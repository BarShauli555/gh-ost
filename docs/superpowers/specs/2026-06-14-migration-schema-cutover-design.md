# Design: Cross-Schema Migration via `ghost_migration_schema`

**Date:** 2026-06-14
**Status:** Approved (pending spec review)

## Goal

Add an opt-in mode to gh-ost in which all migration artifacts (the ghost table,
the changelog table, and the checkpoint table) are created in a separate
database — a fixed schema named `ghost_migration_schema` — instead of alongside
the original table. At cut-over, the swap happens **across schemas**:

- The new (ghost) table moves into the **original** schema, becoming the live table.
- The old original table moves into **`ghost_migration_schema`** (as the `_del` table).

This keeps the original schema free of all gh-ost temporary artifacts during and
after the migration.

## Decisions (locked)

| Decision | Choice |
|----------|--------|
| Schema name | Hardcoded constant `ghost_migration_schema` |
| Rollout | Opt-in via a new boolean flag; default behavior unchanged |
| Helper tables (`_ghc`, `_ghk`) | Live in `ghost_migration_schema` |
| Schema creation | Assume it already exists; error clearly if missing |
| Cut-over mode | Atomic only (gh-ost's default). Two-step is rejected when the flag is on. |

## Non-Goals (YAGNI)

- Configurable / derived schema names (hardcoded constant only).
- Supporting the two-step cut-over path with this feature.
- Auto-creating the migration schema.
- Changing default behavior for existing users.

## Approach

**Add a single `GhostDatabaseName` field to `MigrationContext`, defaulting to
`DatabaseName`.**

When the new flag is off, `GhostDatabaseName == DatabaseName`, so every existing
code path produces byte-for-byte identical SQL — zero behavior change and minimal
test risk. When the flag is on, `GhostDatabaseName = "ghost_migration_schema"`,
and the ~20 code sites that currently hardcode `DatabaseName` for ghost/
changelog/checkpoint/old tables switch to the new accessor. References to the
**original** table continue to use `DatabaseName`.

This was chosen over (B) building all queries at runtime with explicit schema
args (far more invasive, no benefit since schema is fixed at startup) and over
(C) moving only the ghost table while leaving helpers in the original schema
(contradicts the "keep original schema clean" decision).

### Important correctness note

MySQL **does** support cross-database `LOCK TABLES` (locking tables from
different schemas in one statement) and atomic cross-database `RENAME TABLE`
(`RENAME TABLE a.t1 TO b.t2`). The atomic cut-over's safety mechanism therefore
works unchanged across schemas; the sentinel `_del` table simply must be created
in `ghost_migration_schema` (its rename target).

## Components & Changes

### 1. Configuration

- **`go/cmd/gh-ost/main.go`**: new boolean flag `--use-migration-schema`
  (default `false`).
- **New constant** `ghost_migration_schema` (define near other gh-ost name
  constants).
- **`go/base/context.go`**:
  - New field `GhostDatabaseName string` on `MigrationContext`.
  - Initialization: at startup, `GhostDatabaseName = DatabaseName`; if the flag
    is set, `GhostDatabaseName = "ghost_migration_schema"`.
  - Validation: if the flag is on, error if `--cut-over=two-step` is also set.

### 2. Table creation (`go/logic/applier.go`)

- `CreateGhostTable`: `CREATE TABLE ghost_migration_schema._<t>_gho LIKE <orig_db>.<t>`.
- `CreateChangelogTable`, `CreateCheckpointTable`: create in `GhostDatabaseName`.
- `AlterGhost`: `ALTER TABLE ghost_migration_schema._<t>_gho ...`.

### 3. Row copy & DML

- **`go/sql/builder.go`** `BuildRangeInsertPreparedQuery` (and
  `ApplyIterationInsertQuery` in applier): add a separate ghost-schema argument
  so the SELECT source uses the original schema and the INSERT target uses
  `GhostDatabaseName`. (Today a single `databaseName` is used for both.)
- **`prepareQueries`** in `applier.go`: the cached DML insert/update/delete
  builders and the checkpoint insert builder use `GhostDatabaseName`.
- `WriteChangelog`, `ReadLastCheckpoint` → `GhostDatabaseName`.
- **`go/logic/throttler.go`** `collectControlReplicasLag` heartbeat read →
  `GhostDatabaseName`.

### 4. Binlog streaming (`go/logic/migrator.go`, `go/logic/streamer.go`)

- The changelog listener (`initiateStreaming` → `AddListener`) registers with
  `GhostDatabaseName` instead of `DatabaseName`. Binlog events from any schema
  already arrive in the replication stream; only the listener filter key changes.
- The original-table DML listener (`addDMLEventsListener`) stays on `DatabaseName`.

### 5. Validation, existence checks, cleanup (`go/logic/applier.go`)

- `showTableStatus` / `tableExists` / `dropTable`: take a schema parameter.
  Ghost/changelog/checkpoint/old checks pass `GhostDatabaseName`; original-table
  checks pass `DatabaseName`.
- `ValidateOrDropExistingTables`: ghost and old table checks use `GhostDatabaseName`.
- **`go/logic/inspect.go`** `applyColumnTypes` for the ghost table uses
  `GhostDatabaseName`.

### 6. Atomic cut-over (`go/logic/applier.go`)

- `CreateAtomicCutOverSentryTable`: create the sentinel `_<t>_del` in
  `GhostDatabaseName`.
- `AtomicCutOverMagicLock`: `LOCK TABLES <orig_db>.<t> WRITE,
  ghost_migration_schema._<t>_del WRITE` (cross-schema lock); drop / unlock the
  sentinel in `GhostDatabaseName`.
- `AtomicCutoverRename`:
  ```sql
  RENAME TABLE <orig_db>.<t> TO ghost_migration_schema._<t>_del,
               ghost_migration_schema._<t>_gho TO <orig_db>.<t>
  ```

### 7. Cleanup logging (`go/logic/migrator.go`)

- `dropOldTable` and related cleanup log lines reference `GhostDatabaseName`
  for old/checkpoint/changelog tables.

## Data Flow (flag on)

```
CREATE  ghost_migration_schema._t_gho LIKE orig_db.t
CREATE  ghost_migration_schema._t_ghc        (changelog)
CREATE  ghost_migration_schema._t_ghk        (checkpoint)
COPY    INSERT INTO ghost_migration_schema._t_gho SELECT ... FROM orig_db.t
DML     binlog events on orig_db.t  ->  applied to ghost_migration_schema._t_gho
CUTOVER LOCK   orig_db.t WRITE, ghost_migration_schema._t_del WRITE
        RENAME orig_db.t TO ghost_migration_schema._t_del,
               ghost_migration_schema._t_gho TO orig_db.t
RESULT  new table lives in orig_db.t ; old table lives in ghost_migration_schema._t_del
```

## Error Handling

- Flag on + `ghost_migration_schema` missing → fatal error at startup with a
  message telling the operator to create the schema.
- Flag on + `--cut-over=two-step` → fatal error (unsupported combination).
- Insufficient grants on either schema → surfaced via the existing privilege
  checks; cross-schema operations need CREATE/ALTER/DROP/INSERT/SELECT/LOCK on
  both `DatabaseName` and `ghost_migration_schema`.

## Risks

- **Grants**: migration user needs the above privileges on *both* schemas.
- **Replication**: downstream replicas must have `ghost_migration_schema`
  present, since the ghost-schema DDL/DML replicates.
- **Foreign keys**: the original table name is unchanged, so FKs referencing it
  survive. An FK pinned to the *old* table would now point into the migration
  schema; acceptable since the old table is disposable.

## Testing

- **Unit**: with flag off, generated SQL is identical to current output
  (regression guard around `GhostDatabaseName == DatabaseName`). With flag on,
  builders emit the correct cross-schema SQL.
- **Localtests** (gh-ost's integration suite under `localtests/`): add a
  scenario exercising the flag end-to-end — create `ghost_migration_schema`,
  run a migration, assert the new table is in the original schema and the old
  table landed in `ghost_migration_schema`.
- **Negative**: flag + two-step errors out; flag + missing schema errors out.
