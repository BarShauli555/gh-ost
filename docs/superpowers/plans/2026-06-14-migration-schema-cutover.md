# Cross-Schema Migration via `ghost_migration_schema` Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an opt-in `--use-migration-schema` mode where gh-ost creates the ghost/changelog/checkpoint tables in a separate `ghost_migration_schema` database and the atomic cut-over swaps tables across schemas — the new table lands in the original schema, the old table lands in `ghost_migration_schema`.

**Architecture:** Introduce one `MigrationContext.GhostDatabaseName` field plus a `GetGhostDatabaseName()` accessor that defaults to `DatabaseName`. When the flag is off, the accessor returns `DatabaseName`, so all SQL is byte-for-byte identical to today. When on, it returns the hardcoded `ghost_migration_schema`. Every code site that references the ghost/changelog/checkpoint/old tables switches from `DatabaseName` to `GetGhostDatabaseName()`; original-table references stay on `DatabaseName`. The row-copy and atomic cut-over SQL become cross-schema.

**Tech Stack:** Go, MySQL, gh-ost's existing `go/sql` query builders, `go test`, gh-ost `localtests` integration suite.

**Spec:** `docs/superpowers/specs/2026-06-14-migration-schema-cutover-design.md`

---

## Notes for the implementer

- Run unit tests with: `cd /Users/barsh/git_projects/gh-ost && go test ./go/...` (or narrow to a package, e.g. `go test ./go/base/ -run TestGhostDatabaseName -v`).
- `go/sql/builder_test.go` tests are pure string-comparison tests requiring no database — fast.
- The `localtests` suite (Task 7) requires a live MySQL and is run via `./localtests/test.sh`; it is optional to run locally but the test files must be added.
- **Do NOT** change references to the *original* table — only ghost/changelog/checkpoint/old artifacts move schemas. The clearest tell: in `CreateGhostTable` the `LIKE <db>.<original>` side stays `DatabaseName`; the `create table <db>.<ghost>` side becomes `GetGhostDatabaseName()`.

---

## Task 1: Context field, accessor, constant, flag, and guards

**Files:**
- Modify: `go/base/context.go` (add constant near other constants; add `GhostDatabaseName` field ~line 83; add `GetGhostDatabaseName()` accessor near `GetGhostTableName` ~line 382)
- Modify: `go/cmd/gh-ost/main.go` (add flag ~line 117; wire + guard after the cut-over switch ~line 350)
- Test: `go/base/context_test.go`

- [ ] **Step 1: Write the failing test**

Add to `go/base/context_test.go`:

```go
func TestGetGhostDatabaseName(t *testing.T) {
	t.Run("defaults to DatabaseName when unset", func(t *testing.T) {
		context := NewMigrationContext()
		context.DatabaseName = "mydb"
		require.Equal(t, "mydb", context.GetGhostDatabaseName())
	})
	t.Run("returns GhostDatabaseName when set", func(t *testing.T) {
		context := NewMigrationContext()
		context.DatabaseName = "mydb"
		context.GhostDatabaseName = "ghost_migration_schema"
		require.Equal(t, "ghost_migration_schema", context.GetGhostDatabaseName())
	})
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd /Users/barsh/git_projects/gh-ost && go test ./go/base/ -run TestGetGhostDatabaseName -v`
Expected: FAIL to compile — `context.GetGhostDatabaseName undefined` and `context.GhostDatabaseName undefined`.

- [ ] **Step 3: Add the constant in `go/base/context.go`**

Find the existing `const (` block that defines `CutOverAtomic CutOver = iota` (around line 40). Immediately after that const block's closing `)`, add a new constant:

```go
// MigrationSchemaName is the fixed schema into which ghost/changelog/checkpoint
// tables are created when --use-migration-schema is enabled.
const MigrationSchemaName = "ghost_migration_schema"
```

- [ ] **Step 4: Add the field in `go/base/context.go`**

In the `MigrationContext` struct, change the `DatabaseName` line (line 83) block to add the new field:

```go
	DatabaseName          string
	GhostDatabaseName     string
	OriginalTableName     string
```

- [ ] **Step 5: Add the accessor in `go/base/context.go`**

Immediately after `GetGhostTableName()` (ends ~line 382), add:

```go
// GetGhostDatabaseName returns the schema in which the ghost, changelog,
// checkpoint and "old" tables live. When --use-migration-schema is not set this
// equals DatabaseName, preserving the original single-schema behavior.
func (mctx *MigrationContext) GetGhostDatabaseName() string {
	if mctx.GhostDatabaseName != "" {
		return mctx.GhostDatabaseName
	}
	return mctx.DatabaseName
}
```

- [ ] **Step 6: Run the test to verify it passes**

Run: `cd /Users/barsh/git_projects/gh-ost && go test ./go/base/ -run TestGetGhostDatabaseName -v`
Expected: PASS (both subtests).

- [ ] **Step 7: Add the flag in `go/cmd/gh-ost/main.go`**

Immediately after the `cut-over` flag (line 117), add a local var declaration:

```go
	cutOver := flag.String("cut-over", "atomic", "choose cut-over type (default|atomic, two-step)")
	useMigrationSchema := flag.Bool("use-migration-schema", false, fmt.Sprintf("create ghost/changelog/checkpoint tables in the '%s' schema and swap tables across schemas at cut-over (atomic cut-over only)", base.MigrationSchemaName))
```

- [ ] **Step 8: Wire the flag and add guards in `go/cmd/gh-ost/main.go`**

Immediately after the `switch *cutOver { ... }` block (closes line 350), add:

```go
	if *useMigrationSchema {
		if migrationContext.CutOverType != base.CutOverAtomic {
			migrationContext.Log.Fatalf("--use-migration-schema requires the atomic cut-over; remove --cut-over=two-step")
		}
		if migrationContext.Revert {
			migrationContext.Log.Fatalf("--use-migration-schema is not supported together with revert")
		}
		migrationContext.GhostDatabaseName = base.MigrationSchemaName
	}
```

- [ ] **Step 9: Verify it compiles**

Run: `cd /Users/barsh/git_projects/gh-ost && go build ./...`
Expected: builds with no errors.

- [ ] **Step 10: Commit**

```bash
git add go/base/context.go go/base/context_test.go go/cmd/gh-ost/main.go
git commit -m "Add GhostDatabaseName context field, accessor, and --use-migration-schema flag

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Route applier ghost/changelog/checkpoint table SQL to the ghost schema

These are mechanical replacements of `apl.migrationContext.DatabaseName` → `apl.migrationContext.GetGhostDatabaseName()` for every site referencing a ghost/changelog/checkpoint/old table. No behavior change when the flag is off (accessor returns `DatabaseName`).

**Files:**
- Modify: `go/logic/applier.go` (functions: `prepareQueries`, `showTableStatus`, `CreateGhostTable`, `AlterGhost`, `AlterGhostAutoIncrement`, `CreateChangelogTable`, `CreateCheckpointTable`, `dropTable`, `WriteChangelog`, `ReadLastCheckpoint`)

- [ ] **Step 1: `prepareQueries` — DML + checkpoint builders use the ghost schema**

In `prepareQueries` (lines 299–337), replace `apl.migrationContext.DatabaseName` with `apl.migrationContext.GetGhostDatabaseName()` in all four builder constructor calls (delete builder line 301, insert builder line 309, update builder line 318, checkpoint builder line 329). For example the delete builder becomes:

```go
	if apl.dmlDeleteQueryBuilder, err = sql.NewDMLDeleteQueryBuilder(
		apl.migrationContext.GetGhostDatabaseName(),
		apl.migrationContext.GetGhostTableName(),
		apl.migrationContext.OriginalTableColumns,
		&apl.migrationContext.UniqueKey.Columns,
	); err != nil {
		return err
	}
```

Apply the identical `DatabaseName` → `GetGhostDatabaseName()` swap to the insert, update, and checkpoint builder calls in the same function.

- [ ] **Step 2: `showTableStatus` uses the ghost schema**

`showTableStatus` (line 391) is only ever called for ghost/old/changelog/checkpoint tables (verified callers: `tableExists` for ghost+old, `DropAtomicCutOverSentryTableIfExists`). Change line 392:

```go
func (apl *Applier) showTableStatus(tableName string) (rowMap sqlutils.RowMap) {
	query := fmt.Sprintf(`show /* gh-ost */ table status from %s like '%s'`, sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()), tableName)
```

- [ ] **Step 3: `CreateGhostTable` — ghost in ghost schema, LIKE original in original schema**

In `CreateGhostTable` (line 489), change ONLY the target (ghost) schema, NOT the `LIKE` source:

```go
	query := fmt.Sprintf(`create /* gh-ost */ table %s.%s like %s.%s`,
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetGhostTableName()),
		sql.EscapeName(apl.migrationContext.DatabaseName),
		sql.EscapeName(apl.migrationContext.OriginalTableName),
	)
	apl.migrationContext.Log.Infof("Creating ghost table %s.%s",
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetGhostTableName()),
	)
```

- [ ] **Step 4: `AlterGhost` uses the ghost schema**

In `AlterGhost` (line 530) replace both `DatabaseName` references (the query at 531 and the log at 536) with `GetGhostDatabaseName()`:

```go
	query := fmt.Sprintf(`alter /* gh-ost */ table %s.%s %s`,
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetGhostTableName()),
		apl.migrationContext.AlterStatementOptions,
	)
	apl.migrationContext.Log.Infof("Altering ghost table %s.%s",
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetGhostTableName()),
	)
```

- [ ] **Step 5: `AlterGhostAutoIncrement` uses the ghost schema**

In `AlterGhostAutoIncrement` (line 571) replace both `DatabaseName` references (query line 572 and log line 577) with `GetGhostDatabaseName()`.

- [ ] **Step 6: `CreateChangelogTable` uses the ghost schema**

In `CreateChangelogTable` (line 593) replace the `DatabaseName` in the query (line 601) and the log (line 606) with `GetGhostDatabaseName()`.

- [ ] **Step 7: `CreateCheckpointTable` uses the ghost schema**

In `CreateCheckpointTable` (line 647) replace `DatabaseName` (line 648) with `GetGhostDatabaseName()`.

- [ ] **Step 8: `dropTable` uses the ghost schema**

`dropTable` (line 660) is only called for changelog/checkpoint/old/ghost tables (verified callers). Replace both `DatabaseName` references (query line 662 and log line 666) with `GetGhostDatabaseName()`.

- [ ] **Step 9: `WriteChangelog` uses the ghost schema**

In `WriteChangelog` (line 776) replace `DatabaseName` (line 796) with `GetGhostDatabaseName()`.

- [ ] **Step 10: `ReadLastCheckpoint` uses the ghost schema**

In `ReadLastCheckpoint` (line 830) replace `DatabaseName` (line 831) with `GetGhostDatabaseName()`.

- [ ] **Step 11: Verify it compiles and existing tests pass**

Run: `cd /Users/barsh/git_projects/gh-ost && go build ./... && go test ./go/logic/ ./go/base/`
Expected: builds; tests PASS (flag-off behavior unchanged since accessor returns `DatabaseName`).

- [ ] **Step 12: Commit**

```bash
git add go/logic/applier.go
git commit -m "Route ghost/changelog/checkpoint applier SQL through GetGhostDatabaseName

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Cross-schema row-copy INSERT

The row-copy SELECTs from the original table (original schema) and INSERTs into the ghost table (ghost schema). Today `BuildRangeInsertQuery`/`BuildRangeInsertPreparedQuery` take a single `databaseName` used for both. Add a `ghostDatabaseName` parameter.

**Files:**
- Modify: `go/sql/builder.go` (`BuildRangeInsertQuery` line 263, `buildRangeInsertQueryTwoColumn` line 344, `BuildRangeInsertPreparedQuery` line 422)
- Modify: `go/logic/applier.go` (`ApplyIterationInsertQuery` line 1078)
- Test: `go/sql/builder_test.go`

- [ ] **Step 1: Write a failing cross-schema test**

Add to `go/sql/builder_test.go`:

```go
func TestBuildRangeInsertQueryCrossSchema(t *testing.T) {
	originalDatabaseName := "mydb"
	ghostDatabaseName := "ghost_migration_schema"
	originalTableName := "tbl"
	ghostTableName := "ghost"
	sharedColumns := []string{"id", "name"}
	uniqueKey := "PRIMARY"
	uniqueKeyColumns := NewColumnList([]string{"id"})
	rangeStartValues := []string{"@v1s"}
	rangeEndValues := []string{"@v1e"}
	rangeStartArgs := []interface{}{3}
	rangeEndArgs := []interface{}{103}

	query, _, err := BuildRangeInsertQuery(originalDatabaseName, ghostDatabaseName, originalTableName, ghostTableName, sharedColumns, sharedColumns, uniqueKey, uniqueKeyColumns, rangeStartValues, rangeEndValues, rangeStartArgs, rangeEndArgs, true, true, true)
	require.NoError(t, err)
	// INSERT target must be the ghost schema; SELECT source must be the original schema.
	require.Contains(t, query, "`ghost_migration_schema`.`ghost`")
	require.Contains(t, query, "`mydb`.`tbl`")
	// The ghost table must NOT be referenced under the original schema.
	require.NotContains(t, query, "`mydb`.`ghost`")
}
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `cd /Users/barsh/git_projects/gh-ost && go test ./go/sql/ -run TestBuildRangeInsertQueryCrossSchema -v`
Expected: FAIL to compile — `BuildRangeInsertQuery` called with too many arguments.

- [ ] **Step 3: Add `ghostDatabaseName` to `BuildRangeInsertQuery`**

Change the signature (line 263) and the escaping/use of the ghost table's schema. New signature and the relevant edits:

```go
func BuildRangeInsertQuery(databaseName, ghostDatabaseName, originalTableName, ghostTableName string, sharedColumns []string, mappedSharedColumns []string, uniqueKey string, uniqueKeyColumns *ColumnList, rangeStartValues, rangeEndValues []string, rangeStartArgs, rangeEndArgs []interface{}, includeRangeStartValues bool, transactionalTable bool, noWait bool) (result string, explodedArgs []interface{}, err error) {
	if len(sharedColumns) == 0 {
		return "", explodedArgs, fmt.Errorf("got 0 shared columns in BuildRangeInsertQuery")
	}
	databaseName = EscapeName(databaseName)
	ghostDatabaseName = EscapeName(ghostDatabaseName)
	originalTableName = EscapeName(originalTableName)
	ghostTableName = EscapeName(ghostTableName)
```

Update the two-column delegation call (was line 298) to pass `ghostDatabaseName`:

```go
	if uniqueKeyColumns.Len() == 2 {
		return buildRangeInsertQueryTwoColumn(
			databaseName, ghostDatabaseName, originalTableName, ghostTableName,
			sharedColumnsListing, mappedSharedColumnsListing,
			uniqueKey, uniqueKeyColumns,
			rangeStartValues, rangeEndValues,
			rangeStartArgs, rangeEndArgs,
			minRangeComparisonSign, transactionalClause,
		)
	}
```

Update the final `fmt.Sprintf` (was lines 317–333) so the INSERT target uses `ghostDatabaseName` while the SELECT source keeps `databaseName`:

```go
	result = fmt.Sprintf(`
		insert /* gh-ost %s.%s */ ignore
		into
			%s.%s
			(%s)
		(
			select %s
			from
				%s.%s
			force index (%s)
			where
				(%s and %s)
				%s
		)`,
		databaseName, originalTableName, ghostDatabaseName, ghostTableName, mappedSharedColumnsListing,
		sharedColumnsListing, databaseName, originalTableName, uniqueKey,
		rangeStartComparison, rangeEndComparison, transactionalClause)
	return result, explodedArgs, nil
}
```

- [ ] **Step 4: Add `ghostDatabaseName` to `buildRangeInsertQueryTwoColumn`**

Change its signature (line 344) to accept `ghostDatabaseName` right after `databaseName`:

```go
func buildRangeInsertQueryTwoColumn(
	databaseName, ghostDatabaseName, originalTableName, ghostTableName string,
	sharedColumnsListing, mappedSharedColumnsListing string,
	uniqueKey string,
	uniqueKeyColumns *ColumnList,
	rangeStartValues, rangeEndValues []string,
	rangeStartArgs, rangeEndArgs []interface{},
	minRangeComparisonSign ValueComparisonSign,
	transactionalClause string,
```

Then in this function's body, locate the `fmt.Sprintf` that builds the `insert ... into %s.%s` statement. The INSERT `into` target schema (the `%s` paired with `ghostTableName`) must use `ghostDatabaseName`; every reference paired with `originalTableName` in the `from`/sub-select stays `databaseName`. (Read the function body and change only the INSERT-target schema argument.)

- [ ] **Step 5: Add `ghostDatabaseName` to `BuildRangeInsertPreparedQuery`**

Change its signature (line 422) and its delegation to `BuildRangeInsertQuery`:

```go
func BuildRangeInsertPreparedQuery(databaseName, ghostDatabaseName, originalTableName, ghostTableName string, sharedColumns []string, mappedSharedColumns []string, uniqueKey string, uniqueKeyColumns *ColumnList, rangeStartArgs, rangeEndArgs []interface{}, includeRangeStartValues bool, transactionalTable bool, noWait bool) (result string, explodedArgs []interface{}, err error) {
```

Inside the function body it calls `BuildRangeInsertQuery(databaseName, originalTableName, ghostTableName, ...)`. Update that call to pass `ghostDatabaseName` as the second argument:

```go
	return BuildRangeInsertQuery(databaseName, ghostDatabaseName, originalTableName, ghostTableName, sharedColumns, mappedSharedColumns, uniqueKey, uniqueKeyColumns, rangeStartValues, rangeEndValues, rangeStartArgs, rangeEndArgs, includeRangeStartValues, transactionalTable, noWait)
```

(The `rangeStartValues`/`rangeEndValues` names are the locals built inside `BuildRangeInsertPreparedQuery` — keep whatever they are currently named; only insert `ghostDatabaseName` as the new second arg.)

- [ ] **Step 6: Update the caller `ApplyIterationInsertQuery`**

In `go/logic/applier.go` (line 1078) pass the ghost schema as the new second arg:

```go
	query, explodedArgs, err := sql.BuildRangeInsertPreparedQuery(
		apl.migrationContext.DatabaseName,
		apl.migrationContext.GetGhostDatabaseName(),
		apl.migrationContext.OriginalTableName,
		apl.migrationContext.GetGhostTableName(),
		apl.migrationContext.SharedColumns.Names(),
		apl.migrationContext.MappedSharedColumns.Names(),
		apl.migrationContext.UniqueKey.Name,
		&apl.migrationContext.UniqueKey.Columns,
		apl.migrationContext.MigrationIterationRangeMinValues.AbstractValues(),
		apl.migrationContext.MigrationIterationRangeMaxValues.AbstractValues(),
		apl.migrationContext.GetIteration() == 0,
		apl.migrationContext.IsTransactionalTable(),
		// TODO: Don't hardcode this
		strings.HasPrefix(apl.migrationContext.ApplierMySQLVersion, "8."),
	)
```

- [ ] **Step 7: Fix existing builder tests (compile breakage)**

In `go/sql/builder_test.go`, every existing call to `BuildRangeInsertQuery(databaseName, originalTableName, ghostTableName, ...)` (lines 174, 203, 225, 252, 274, 296, 325, 354, and any others) must insert the ghost schema as the second arg. Since the existing tests are same-schema, pass `databaseName` as the new `ghostDatabaseName`:

```go
		query, explodedArgs, err := BuildRangeInsertQuery(databaseName, databaseName, originalTableName, ghostTableName, sharedColumns, sharedColumns, uniqueKey, uniqueKeyColumns, rangeStartValues, rangeEndValues, rangeStartArgs, rangeEndArgs, true, true, true)
```

Apply the same `databaseName, databaseName` doubling to every existing `BuildRangeInsertQuery(...)` and `BuildRangeInsertPreparedQuery(...)` call in the test file. Use a search to find them all:

Run: `cd /Users/barsh/git_projects/gh-ost && grep -n "BuildRangeInsert" go/sql/builder_test.go`
Then edit each call site so the original schema is passed twice (source and ghost are identical in these legacy tests).

- [ ] **Step 8: Run the sql package tests**

Run: `cd /Users/barsh/git_projects/gh-ost && go test ./go/sql/ -v -run TestBuildRangeInsert`
Expected: PASS, including the new `TestBuildRangeInsertQueryCrossSchema`.

- [ ] **Step 9: Verify whole build + tests**

Run: `cd /Users/barsh/git_projects/gh-ost && go build ./... && go test ./go/...`
Expected: builds; all tests PASS.

- [ ] **Step 10: Commit**

```bash
git add go/sql/builder.go go/sql/builder_test.go go/logic/applier.go
git commit -m "Support cross-schema row-copy INSERT in range query builders

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Changelog binlog listener, throttler heartbeat, and inspector column types

The changelog table now lives in the ghost schema, so the binlog listener filter, the throttler's heartbeat read, and the inspector's ghost-table column inspection must target it.

**Files:**
- Modify: `go/logic/migrator.go` (`initiateStreaming` line 1448; cleanup drop-hint logs lines 1936/1939)
- Modify: `go/logic/throttler.go` (`collectControlReplicasLag` line 185)
- Modify: `go/logic/inspect.go` (`InspectOriginalAndGhostTables` ghost call, line 190)

- [ ] **Step 1: Changelog listener filters on the ghost schema**

In `go/logic/migrator.go` `initiateStreaming` (line 1448), change the changelog `AddListener` database arg from `DatabaseName` to `GetGhostDatabaseName()`:

```go
	mgtr.eventsStreamer.AddListener(
		false,
		mgtr.migrationContext.GetGhostDatabaseName(),
		mgtr.migrationContext.GetChangelogTableName(),
		func(dmlEntry *binlog.BinlogEntry) error {
			return mgtr.onChangelogEvent(dmlEntry)
		},
	)
```

Leave `addDMLEventsListener` (line 1483) unchanged — it listens on the *original* table in `DatabaseName`.

- [ ] **Step 1b: Cleanup drop-hint logs name the ghost schema**

In `go/logic/migrator.go`, the operator-facing "drop table" hints (lines 1936 and 1939) print where the old/checkpoint tables live. Change the schema arg from `DatabaseName` to `GetGhostDatabaseName()`:

```go
		mgtr.migrationContext.Log.Infof("-- drop table %s.%s", sql.EscapeName(mgtr.migrationContext.GetGhostDatabaseName()), sql.EscapeName(mgtr.migrationContext.GetOldTableName()))
		if mgtr.migrationContext.Checkpoint {
			mgtr.migrationContext.Log.Infof("Am not dropping checkpoint table without `--ok-to-drop-table`. To drop the checkpoint table, issue:")
			mgtr.migrationContext.Log.Infof("-- drop table %s.%s", sql.EscapeName(mgtr.migrationContext.GetGhostDatabaseName()), sql.EscapeName(mgtr.migrationContext.GetCheckpointTableName()))
		}
```

- [ ] **Step 2: Throttler heartbeat reads from the ghost schema**

In `go/logic/throttler.go` `collectControlReplicasLag` (line 185), change line 188:

```go
	replicationLagQuery := fmt.Sprintf(`
		select value from %s.%s where hint = 'heartbeat' and id <= 255
		`,
		sql.EscapeName(thlr.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(thlr.migrationContext.GetChangelogTableName()),
	)
```

- [ ] **Step 3: Inspector reads ghost-table column types from the ghost schema**

In `go/logic/inspect.go` (line 190), change ONLY the ghost-table `applyColumnTypes` call's schema arg. Line 189 (original table) stays `DatabaseName`:

```go
	isp.applyColumnTypes(isp.migrationContext.DatabaseName, isp.migrationContext.OriginalTableName, isp.migrationContext.OriginalTableColumns, isp.migrationContext.SharedColumns, &isp.migrationContext.UniqueKey.Columns)
	isp.applyColumnTypes(isp.migrationContext.GetGhostDatabaseName(), isp.migrationContext.GetGhostTableName(), isp.migrationContext.GhostTableColumns, isp.migrationContext.MappedSharedColumns)
```

- [ ] **Step 4: Verify build + tests**

Run: `cd /Users/barsh/git_projects/gh-ost && go build ./... && go test ./go/...`
Expected: builds; all tests PASS.

- [ ] **Step 5: Commit**

```bash
git add go/logic/migrator.go go/logic/throttler.go go/logic/inspect.go
git commit -m "Point changelog listener, heartbeat read, and ghost column inspection at ghost schema

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Cross-schema atomic cut-over

The atomic cut-over creates the sentinel `_del` table in the ghost schema, locks original (original schema) + sentinel (ghost schema) together, and renames across schemas. MySQL supports cross-database `LOCK TABLES` and atomic cross-database `RENAME TABLE`.

**Files:**
- Modify: `go/logic/applier.go` (`CreateAtomicCutOverSentryTable` line 1408, `AtomicCutOverMagicLock` lines 1507/1538/1563, `AtomicCutoverRename` line 1602)

- [ ] **Step 1: Sentinel table created in the ghost schema**

In `CreateAtomicCutOverSentryTable` (line 1408) replace both `DatabaseName` references (the CREATE at line 1418 and the log at line 1424) with `GetGhostDatabaseName()`:

```go
	query := fmt.Sprintf(`
		create /* gh-ost */ table %s.%s (
			id int auto_increment primary key
		) engine=%s comment='%s'`,
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(tableName),
		apl.migrationContext.TableEngine,
		atomicCutOverMagicHint,
	)
	apl.migrationContext.Log.Infof("Creating magic cut-over table %s.%s",
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(tableName),
	)
```

(`DropAtomicCutOverSentryTableIfExists` uses `showTableStatus`/`dropTable`, both already routed to the ghost schema in Task 2 — no change needed here.)

- [ ] **Step 2: Cross-schema LOCK TABLES**

In `AtomicCutOverMagicLock`, the `lock ... tables` statement (lines 1507–1512) must lock the original table in the original schema and the sentinel in the ghost schema:

```go
	query = fmt.Sprintf(`lock /* gh-ost */ tables %s.%s write, %s.%s write`,
		sql.EscapeName(apl.migrationContext.DatabaseName),
		sql.EscapeName(apl.migrationContext.OriginalTableName),
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetOldTableName()),
	)
	apl.migrationContext.Log.Infof("Locking %s.%s, %s.%s",
		sql.EscapeName(apl.migrationContext.DatabaseName),
		sql.EscapeName(apl.migrationContext.OriginalTableName),
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetOldTableName()),
	)
```

- [ ] **Step 3: Drop the sentinel from the ghost schema**

In `AtomicCutOverMagicLock`, the `drop ... table if exists` statement (lines 1538–1541) and the "Releasing lock" log (lines 1563–1568) reference the sentinel/old table. Change the sentinel's schema to `GetGhostDatabaseName()` (keep the original table's schema as `DatabaseName`):

```go
	query = fmt.Sprintf(`drop /* gh-ost */ table if exists %s.%s`,
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetOldTableName()),
	)
```

and the release-lock log:

```go
	apl.migrationContext.Log.Infof("Releasing lock from %s.%s, %s.%s",
		sql.EscapeName(apl.migrationContext.DatabaseName),
		sql.EscapeName(apl.migrationContext.OriginalTableName),
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetOldTableName()),
	)
```

- [ ] **Step 4: Cross-schema atomic RENAME**

In `AtomicCutoverRename` (line 1602), the rename moves original→ghost-schema._del and ghost-schema._gho→original:

```go
	query = fmt.Sprintf(`rename /* gh-ost */ table %s.%s to %s.%s, %s.%s to %s.%s`,
		sql.EscapeName(apl.migrationContext.DatabaseName),
		sql.EscapeName(apl.migrationContext.OriginalTableName),
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetOldTableName()),
		sql.EscapeName(apl.migrationContext.GetGhostDatabaseName()),
		sql.EscapeName(apl.migrationContext.GetGhostTableName()),
		sql.EscapeName(apl.migrationContext.DatabaseName),
		sql.EscapeName(apl.migrationContext.OriginalTableName),
	)
```

This yields: `RENAME TABLE orig_db.t TO ghost_migration_schema._t_del, ghost_migration_schema._t_gho TO orig_db.t` — the new table lands in the original schema, the old table lands in the migration schema.

- [ ] **Step 5: Verify build + tests**

Run: `cd /Users/barsh/git_projects/gh-ost && go build ./... && go test ./go/...`
Expected: builds; all tests PASS.

- [ ] **Step 6: Commit**

```bash
git add go/logic/applier.go
git commit -m "Make atomic cut-over swap tables across schemas

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Validate the migration schema exists

When the flag is on, fail fast with a clear error if `ghost_migration_schema` does not exist (per spec: assume-exists, error if missing). Add the check in `ValidateOrDropExistingTables`, which runs on the applier before any ghost tables are created.

**Files:**
- Modify: `go/logic/applier.go` (`ValidateOrDropExistingTables` line 408; add a small helper)

- [ ] **Step 1: Add a schema-existence helper**

In `go/logic/applier.go`, add a helper just above `ValidateOrDropExistingTables` (above line 406):

```go
// schemaExists checks whether a given schema (database) exists on the applier.
func (apl *Applier) schemaExists(schemaName string) (bool, error) {
	query := `select /* gh-ost */ count(*) from information_schema.schemata where schema_name = ?`
	var count int
	if err := apl.db.QueryRow(query, schemaName).Scan(&count); err != nil {
		return false, err
	}
	return count > 0, nil
}
```

- [ ] **Step 2: Call it at the top of `ValidateOrDropExistingTables`**

Insert at the very start of `ValidateOrDropExistingTables` (immediately after line 408 `func ... {`), before the `InitiallyDropGhostTable` block:

```go
	if apl.migrationContext.GhostDatabaseName != "" {
		exists, err := apl.schemaExists(apl.migrationContext.GhostDatabaseName)
		if err != nil {
			return err
		}
		if !exists {
			return fmt.Errorf("migration schema %s does not exist; create it before running gh-ost with --use-migration-schema", sql.EscapeName(apl.migrationContext.GhostDatabaseName))
		}
	}
```

(We check `GhostDatabaseName != ""` rather than the accessor, so the validation only runs when the flag explicitly set a distinct schema.)

- [ ] **Step 3: Verify build + tests**

Run: `cd /Users/barsh/git_projects/gh-ost && go build ./... && go test ./go/...`
Expected: builds; all tests PASS.

- [ ] **Step 4: Commit**

```bash
git add go/logic/applier.go
git commit -m "Validate migration schema exists when --use-migration-schema is set

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: End-to-end localtests scenario

gh-ost's `localtests` suite runs a real migration against MySQL and validates results. Add a scenario that runs with `--use-migration-schema` and asserts the swap moved tables across schemas.

**Files:**
- Read first: `localtests/README.md` and an existing test dir (e.g. `localtests/tz/`) to learn the file conventions (`create.sql`, `extra_args`, `expect_table_structure`, `ghost_columns`, etc.).
- Create: `localtests/use-migration-schema/create.sql`
- Create: `localtests/use-migration-schema/extra_args`

- [ ] **Step 1: Inspect the localtests conventions**

Run: `cd /Users/barsh/git_projects/gh-ost && cat localtests/README.md && ls localtests/tz/ && cat localtests/tz/create.sql && cat localtests/tz/extra_args`
Expected: you learn that each test dir has a `create.sql` (DDL + seed data + an event that drives DML during migration) and an `extra_args` file with extra gh-ost flags. Note whether the harness supports a per-test setup hook for creating the schema (look in `localtests/test.sh` for how `create.sql` is sourced and whether a separate schema can be created there).

- [ ] **Step 2: Create the migration schema and seed data in `create.sql`**

Create `localtests/use-migration-schema/create.sql`. The harness runs `create.sql` against the test database; create the migration schema at the top so it exists before gh-ost runs:

```sql
create database if not exists ghost_migration_schema;

drop table if exists gh_ost_test;
create table gh_ost_test (
  id int auto_increment,
  i int not null,
  color varchar(32),
  primary key(id)
) auto_increment=1;

drop event if exists gh_ost_test;
delimiter ;;
create event gh_ost_test
  on schedule every 1 second
  starts current_timestamp
  ends current_timestamp + interval 60 second
  on completion not preserve
  enable
  do
begin
  insert into gh_ost_test values (null, 11, 'red');
  insert into gh_ost_test values (null, 13, 'green');
  update gh_ost_test set color='blue' where i=11 order by id desc limit 1;
  delete from gh_ost_test where i=13 order by id desc limit 1;
end ;;
```

- [ ] **Step 3: Add the flag in `extra_args`**

Create `localtests/use-migration-schema/extra_args` with:

```
--use-migration-schema
```

- [ ] **Step 4: Run the single localtest (requires a live MySQL configured per localtests/README.md)**

Run: `cd /Users/barsh/git_projects/gh-ost && ./localtests/test.sh use-migration-schema`
Expected: the test passes — gh-ost migrates `gh_ost_test`, the migrated table ends up in the original test schema, and the `_gh_ost_test_del` table ends up in `ghost_migration_schema`. If your environment has no MySQL, note that this step is deferred to CI / a machine with the localtests DB; the test files are still committed.

- [ ] **Step 5: Commit**

```bash
git add localtests/use-migration-schema/
git commit -m "Add localtests scenario for --use-migration-schema cross-schema cut-over

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Task 8: Documentation

**Files:**
- Read first: `doc/` directory to find the cut-over / command flags docs.
- Modify: the appropriate flags doc (likely `doc/command-line-flags.md`).

- [ ] **Step 1: Find the flags doc**

Run: `cd /Users/barsh/git_projects/gh-ost && ls doc/ && grep -l "cut-over" doc/*.md`
Expected: identifies the command-line flags doc.

- [ ] **Step 2: Document the new flag**

Add an entry for `--use-migration-schema` to the flags doc, in the same style as neighboring entries. Suggested text:

```markdown
### use-migration-schema

When provided, gh-ost creates the ghost, changelog and checkpoint tables in a
dedicated `ghost_migration_schema` database instead of alongside the original
table. At cut-over the tables are swapped across schemas: the migrated table is
renamed into the original schema and the original table is moved into
`ghost_migration_schema` as the `_del` table.

Requirements and limitations:

- The `ghost_migration_schema` database must already exist; gh-ost does not create it.
- The migration user needs CREATE/ALTER/DROP/INSERT/SELECT/LOCK privileges on both the original schema and `ghost_migration_schema`.
- Only the atomic cut-over is supported; combining with `--cut-over=two-step` is rejected.
- Not supported together with revert operations.
- Downstream replicas must also have the `ghost_migration_schema` database present.
```

- [ ] **Step 3: Commit**

```bash
git add doc/
git commit -m "Document --use-migration-schema flag

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Final verification

- [ ] Run the full unit suite: `cd /Users/barsh/git_projects/gh-ost && go build ./... && go test ./go/...` — all PASS.
- [ ] Confirm flag-off behavior is unchanged: with `--use-migration-schema` absent, `GetGhostDatabaseName()` returns `DatabaseName`, so generated SQL is identical to pre-change. (The pre-existing `go/sql` and `go/logic` tests passing is the regression guard.)
- [ ] Run `gofmt -l go/` and ensure no files are listed (formatting clean).
