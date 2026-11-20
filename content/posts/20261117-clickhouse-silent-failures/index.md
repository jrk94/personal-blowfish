---
title: "ClickHouse Report Errors That Return No Error: Nine Failures and How to Fix Them"
description: "A report on ClickHouse rarely throws an error when it is wrong. It returns duplicates, deleted rows, truncated results or an empty column. Nine failure modes in MES reporting data sets, each with the wrong result, the reason and the fix."
summary: "Writing MES reports on ClickHouse after years of T-SQL: duplicates without FINAL, deleted rows that stay, partitioning and version columns, JSON extraction that returns the wrong value, window functions that return defaults, empty attributes, case sensitivity, truncation at 1000 rows and comments that break data sets."
categories: ["Analytics"]
tags: ["ClickHouse", "Data Platform", "Reporting", "OData", "Grafana"]
date: 2026-11-17
showInRSS: true
draft: false
authors:
  - Roque
---
A SQL Server query that is wrong usually fails loudly: a missing column, a bad join, a conversion error. A ClickHouse query that is wrong usually returns a result, and the result looks fine.

Duplicate rows, deleted records that are still there, a report cut at 1000 rows, a column that exists and is never filled. Each one is a silent failure that has a cause and a fix.

## Before You Start

This is the second post of two. The [architecture post](https://j-roque.com/posts/20261116-clickhouse-architecture/) is the prerequisite: it owns the model, meaning the data sources, the databases, denormalization and JSON, cubes, data sets, OData and the tools. This post covers the failures. Read the architecture post first, because the failures below assume you know where the tables live.

The [earlier post on reports](https://j-roque.com/posts/20260730-reports/) covered the pipeline. This one is for the person who has to write the query, in two parts. Part 1 covers failures that come from how the ClickHouse engine behaves (1 to 5). Part 2 covers failures that come from the platform and its Data Manager (6 to 9). Each one gets the wrong result, the reason and the fix.

Table and column names come from the 11.x-era Critical Manufacturing data model (`CoreDataModel_T_MaterialHistory`, `CoreDataModel_T_Material`, `CoreDataModel_T_ProductionOrderHistory`). They can differ by version, so confirm them with `SHOW CREATE TABLE` on your system. SQL labeled as a sketch was not run against a real instance. Reference material lives in the [Critical Manufacturing documentation](https://help.criticalmanufacturing.com/).

ODS objects are referenced as `schema_ObjectName` instead of `schema.ObjectName`, which is why the tables here read `CoreDataModel_T_Material`.

## Part 1: Engine Behavior

> ClickHouse gave up OLTP semantics to read large volumes of data quickly.

You do not fix a row in ClickHouse: you insert a new version, or run a heavy [mutation](https://clickhouse.com/docs/updating-data/overview). That is where the first silent failure comes from.

The failures also assume the denormalized model: service and operation columns on every history table, and step, product and attributes as JSON snapshots instead of joins. The [architecture post](https://j-roque.com/posts/20261116-clickhouse-architecture/#how-the-data-is-shaped-denormalization-and-json) covers that shape and what it costs.

### Failure 1: Duplicate Rows Without FINAL

**The wrong result:** the same logical row appears twice, or you read a superseded version of it. Totals are inflated and no error is raised.

**Why:** ClickHouse writes data in parts and merges them in the background. Our tables use `ReplacingMergeTree`, which keeps one row per sorting key. The main (state) tables have a version column and keep the highest version. The history tables have none, so on an equal key the last inserted row wins. Either way, the deduplication happens at merge time, and merges are asynchronous. Until one runs, the old and new rows both sit on disk.

With generic IDs, run against a scratch table so it never touches platform data. Create the table, then insert the same key (`materialid`, `_servicehistory_servicehistoryid`, `_operationhistory_operationhistoryseq`) twice, 32 minutes apart:

```sql
-- Scratch table with the same engine and sorting key as the history table
CREATE TABLE scratch_material_history AS CoreDataModel_T_MaterialHistory;

INSERT INTO scratch_material_history
    (materialid, _servicehistory_servicehistoryid, _operationhistory_operationhistoryseq, name, modifiedon)
VALUES (1001, 5001, 84, 'Material1', '2026-01-05 10:00:00');

INSERT INTO scratch_material_history
    (materialid, _servicehistory_servicehistoryid, _operationhistory_operationhistoryseq, name, modifiedon)
VALUES (1001, 5001, 84, 'Material1', '2026-01-05 10:32:00');

SELECT * FROM scratch_material_history;

SELECT * FROM scratch_material_history FINAL;
```

`CREATE TABLE ... AS` on a `Replicated` engine may reuse the original replica path, so on a replicated cluster prefer a minimal explicit definition:

```sql
CREATE TABLE scratch_material_history
(
    materialid                              Int64,
    _servicehistory_servicehistoryid        Int64,
    _operationhistory_operationhistoryseq   Int64,
    name                                    Nullable(String),
    modifiedon                              DateTime
)
ENGINE = ReplacingMergeTree
ORDER BY (materialid, _servicehistory_servicehistoryid, _operationhistory_operationhistoryseq);
```

The plain `SELECT` returns both rows, the 10:00 one and the 10:32 one. The `FINAL` one is expected to return only the 10:32 row, as long as no merge has run in between. The two queries read the same data and return different rows.

**The fix:** `FINAL`. It is a modifier on the table in the `FROM` clause that applies the replacing logic at read time. In a join, it goes after the alias, once per table.

```sql
SELECT
    MaterialId,
    Name
FROM CoreDataModel_T_Material AS m FINAL
WHERE m.MaterialId = 1001
```

`FINAL` is not free. The [documentation](https://clickhouse.com/docs/sql-reference/statements/select/from) is clear that it adds overhead that depends on your data, and filtering on the key columns helps. There is also a documented alternative that avoids it: aggregate to the latest version yourself, ordering by the same version column the engine uses. On the main table that is the materialized version column from Failure 3.

```sql
-- Sketch, not run against a real instance
SELECT
    MaterialId,
    argMax(Name, _ServiceHistoryId_OperationHistorySeq) AS Name
FROM CoreDataModel_T_Material
GROUP BY MaterialId
```

`argMax(x, v)` returns `x` from the row where `v` is highest. Order by the version column, not `ModifiedOn`: the engine decides "latest" by its version column, and a different ordering gives you a different answer than `FINAL` on the same data. Both patterns are described in the [ReplacingMergeTree guide](https://clickhouse.com/docs/guides/replacing-merge-tree). The `argMax` form can be cheaper on large tables and is more work to write, so measure on your data. My own practice is to default to `FINAL` for correctness in data sets, and reach for `argMax ... GROUP BY` when profiling tells you to. That is an opinion, not a rule from the docs.

The worked case below puts `FINAL` on the history tables too. On a history table, `FINAL` is expected to remove replayed duplicates of the same key, which would otherwise count twice in every `countIf`. Verify that on your data rather than take it as a guarantee.

> Forget `FINAL` and your report returns quickly and returns the wrong thing. Treat it as part of the table name.

**Try it yourself.** Create the scratch table, run the two inserts, then the `SELECT` without and with `FINAL`.

To see the merge itself, run `OPTIMIZE TABLE scratch_material_history FINAL` and repeat the plain `SELECT`: the duplicate should be gone. That shows the merge; do not use it in a report. The [documentation](https://clickhouse.com/docs/sql-reference/statements/optimize) advises against relying on `OPTIMIZE ... FINAL`, because it rewrites parts and is expensive on real tables. Never put it in a report pipeline.

### Failure 2: Deleted Records That Are Still There

**The wrong result:** a material you deleted shows up in the report.

**Why:** in the ODS state tables of 12.0, deleted records are kept as rows rather than physically removed. A replacing engine represents a delete as another version of the row.

**The fix:** filter them out explicitly, and do not assume `FINAL` does it for you. The query in the worked case below relies on `FINAL` alone, which is enough for the pre-12.0 data model. If your tables carry `_is_deleted`, filter on it too. The ClickHouse rule: `is_deleted` requires a version column, so the engine is declared as `ReplacingMergeTree(ver, is_deleted)`. Deleted rows are kept by default unless you clean them up. Whether `FINAL` and the `_is_deleted` column take part in delete handling depends on the table being defined that way. Check your table with `SHOW CREATE TABLE`. The [12.0 migration notes](https://developer.criticalmanufacturing.com/12.0/explore/guides/migration/version-specific/1200/data-platform/clickhouse_migration_process/) describe the column.

With `FINAL`, the latest version is chosen first and the `WHERE` runs after it, so the filter is safe there:

```sql
SELECT MaterialId, Name
FROM CoreDataModel_T_Material AS m FINAL
WHERE m._is_deleted = 0
```

With `argMax ... GROUP BY` the order matters. Filtering `_is_deleted` in the `WHERE` removes the delete marker before the aggregation and resurrects the previous version. Aggregate the flag too, then filter on the result.

The sketch below targets the pre-12.0 design: the version column is `_ServiceHistoryId_OperationHistorySeq` (Failure 3), and `_is_deleted` is an assumed column on the table. In 12.0 that materialized column is gone, so order by the table's new merge column instead (see the migration notes).

```sql
-- Sketch, pre-12.0 version column + assumed _is_deleted column. Not run against a real instance
SELECT
    MaterialId,
    argMax(Name, _ServiceHistoryId_OperationHistorySeq)        AS Name,
    argMax(_is_deleted, _ServiceHistoryId_OperationHistorySeq) AS IsDeleted
FROM CoreDataModel_T_Material
GROUP BY MaterialId
HAVING IsDeleted = 0
```

Test it once with a record you know was deleted.

### Failure 3: Versions and Partitions That Do Not Do What You Assume

This one needs the table definitions. These are the pre-12.0 definitions, trimmed to the columns that matter. First the main (state) table:

```sql
-- Main table, pre-12.0 design (trimmed)
CREATE TABLE CoreDataModel_T_Material
(
    ...
    `_attributes` array(nullable(string)),
    `materialid`  int64,
    ...
    `_ServiceHistoryId_OperationHistorySeq` Int128
        MATERIALIZED toInt128(rightPad(toString(_ServiceHistory_ServiceHistoryId), 38, '0'))
                     + _OperationHistory_OperationHistorySeq
        CODEC(ZSTD(1))
)
ENGINE = ReplacingMergeTree(_ServiceHistoryId_OperationHistorySeq)
PARTITION BY toYYYYMM(ModifiedOn)
ORDER BY MaterialId
```

Then the history table, which has no version column:

```sql
-- History table, pre-12.0 design (trimmed)
CREATE TABLE CoreDataModel_T_MaterialHistory
(
    ...
    `_servicehistory_servicehistoryid`     int64,
    `_operationhistory_operationname`      nullable(string),
    `_operationhistory_operationhistoryseq` int64,
    `_attributes` array(nullable(string)),
    `materialid`  int64,
    ...
)
ENGINE = ReplacingMergeTree
PARTITION BY toYYYYMM(modifiedon)
ORDER BY (materialid, _servicehistory_servicehistoryid, _operationhistory_operationhistoryseq)
```

Three things in them matter.

**The sorting key defines "the same row".** `ORDER BY MaterialId` fits the main table, where one material has one current row. It would collapse a history table, where one material legitimately has many rows. The history key adds the event: service history ID and operation history sequence. Replacing logic is only as correct as the key you gave it.

**The partition key must be immutable.** Merges never cross partitions. Both tables here partition by the month of `ModifiedOn`, and `ModifiedOn` changes when a row is updated. Versions of one material that are modified in different months land in different partitions and are never collapsed on disk. `FINAL` still deduplicates across partitions by default, at a cost. Partitioning by creation date avoids the question: a row never changes its `CreatedOn`. That is what 12.0 does.

There is a MergeTree-level setting, `do_not_merge_across_partitions_select_final`, that changes this. It is not on the general settings page, so check that your version has it:

```sql
SELECT * FROM system.merge_tree_settings
WHERE name = 'do_not_merge_across_partitions_select_final'
```

Enabling it makes `FINAL` skip cross-partition merging. That is only safe when the partition key never splits rows that share a sorting key. A mutable partition key plus this setting produces duplicates. Leave it alone unless you can prove your partitioning satisfies that condition on your version.

**The version column must be a type the engine can order.** `ReplacingMergeTree(ver)` is documented to accept `UInt*`, `Date`, `DateTime` and `DateTime64`. The pre-12.0 main table builds it from the service history ID, right-padded with zeros to 38 digits, plus the operation history sequence. That is a number, so it orders numerically. If you ever build such a version as a string, `'9_1'` sorts above `'10_1'`, so keep versions numeric in tables you design.

The history table has no version column at all. When two rows share a key, the most recently inserted one wins, per the [engine docs](https://clickhouse.com/docs/engines/table-engines/mergetree-family/replacingmergetree). That suits a replayed event. It is a tie-breaker, not a design, so a version column that produces ties needs rethinking, and `argMax` on tied versions can return either row.

The version semantics changed in 12.0. The 12.0 migration rebuilds the ODS state tables with a new merge column, a new `CreatedOn` partition key and the `_is_deleted` column, and removes the old materialized column above. See the migration notes linked above for the column's definition, and check which design your version runs before you trust any assumption about which row wins.

Date-time columns also become `DateTime64(7, 'UTC')` in 12.0, so compare them as such.

### Failure 4: JSON Extraction That Returns the Wrong Value

**The wrong result:** a step name that belongs to a different object, or the wrong attribute value.

**Why:** the `simpleJSON*` family is fast because it takes shortcuts. Per the [JSON functions docs](https://clickhouse.com/docs/sql-reference/functions/json-functions), it matches the field at any nesting depth, first match wins, and it assumes compact JSON and constant field names. It does not return empty on nested JSON. It returns whatever it finds first. With attribute arrays full of objects that each have a `Name` key, or with nested `Step` and `Product` objects that both carry a `Name`, "first" is often not what you meant.

**The fix:** default to `JSONExtract*`. The fast family is `simpleJSONExtract*` in ClickHouse; it is sometimes written `JSONSimpleExtract*`, which is not a function. Use `simpleJSON*` only on JSON you know is flat, and only after profiling shows it matters. The native `JSON` type is worth a look for tables you design yourself, since it stores paths as typed subcolumns. It does not help with the replicated MES columns, which arrive as Strings, so `JSONExtract*` stays the tool for those.


| Function                                                  | Use                                                              |
| :--------------------------------------------------------- | :---------------------------------------------------------------- |
| `JSONExtractString`, `JSONExtractInt`, `JSONExtractFloat` | Typed values, honors the path you give                           |
| `JSONExtractArrayRaw`                                     | Array of raw JSON elements, for a String that holds a JSON array |
| `simpleJSONExtract*`                                      | Known flat JSON only, after profiling                            |


`_Attributes` is already an array of JSON strings, so you do not need `JSONExtractArrayRaw` for it. Use `arrayFilter` or `ARRAY JOIN`, then `JSONExtractString(x, 'Name')` per element.

Related to this, an alias that matches a real column name shadows it. Name your extracts `StepName` and `ProductName`, not `Step` and `Product`. An alias that matches a real column name shadows it, and the next expression that references `Step` can silently read the alias instead of the column. The worked-case query below uses `AS Step` and `AS Product`, which work until something unqualified references them.

### Failure 5: Window Functions That Return Defaults

**The wrong result:** the "next value" column is zero or empty on every row.

**Why:** the [window functions page](https://clickhouse.com/docs/sql-reference/window-functions) lists plain `lag(value, offset)` and `lead(value, offset)` as supported. It also lists `lagInFrame` and `leadInFrame`, the variants that respect the window frame. The default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, so it ends at the current row. `leadInFrame` looks forward, finds nothing inside that frame and returns the default value. `lagInFrame` still sees the preceding rows, which is why the bug shows up on `lead` and not on `lag`.

**The fix:** prefer plain `lag` and `lead` if your version supports them. `lagInFrame` and `leadInFrame` are often recommended as alternatives to `LAG` and `LEAD`, and they are, as long as you remember that the frame decides what they can see. Use the `*InFrame` variants only with an explicit full frame, one that matches what `lag` and `lead` would see:

```sql
-- Sketch, not run against a real instance
SELECT
    MaterialId,
    _OperationHistory_OperationHistorySeq AS Seq,
    _OperationHistory_OperationName       AS OperationName,
    lead(_OperationHistory_OperationName, 1) OVER (
        PARTITION BY MaterialId
        ORDER BY _OperationHistory_OperationHistorySeq
    ) AS NextOperation,
    leadInFrame(_OperationHistory_OperationName, 1) OVER (
        PARTITION BY MaterialId
        ORDER BY _OperationHistory_OperationHistorySeq
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) AS NextOperationInFrame
FROM CoreDataModel_T_MaterialHistory FINAL
```

Both columns should agree. If they do not, your frame is wrong. Test on a handful of rows before you build a report on a window function.

## Part 2: Platform and Data Manager Behavior

### Failure 6: Attributes That Are Empty by Design

**The wrong result:** an attribute is blank on most rows, or on all rows of the main table.

**Why:** attributes are only written when they are created or updated, so history rows without a change leave them empty. The main tables have an `_Attributes` column too, but by design it is never filled. Reading attributes from a state table returns nothing, with no error.

**The fix:** read attributes from the history tables. For the latest value, expand the array, filter to the attribute name and take `argMax` by modification time, grouped by the owning object. The worked case below shows a query that works, and a cleaner alternative. That pattern returns the latest value written. If you need to know that an attribute was deleted, you need the latest change event, not the latest non-empty value. The pattern depends on the question.

### Failure 7: Names That Match Nothing

**The wrong result:** a data set returns zero rows and gives no hint why.

**Why:** object and column names are case sensitive. A data set created in lowercase and queried in mixed case returns nothing. The same holds for function names, which are mostly case sensitive with some documented exceptions, so copy the casing from the docs. SQL keywords are the one thing that is not.

**The fix:** use the exact case used at creation, and copy it from `SHOW CREATE TABLE`. The rule also applies to the macros the platform replaces for you, `{db_MES}`, `{db_ODS}`, `{db_CDM}` and `{db_DWH}`: the ClickHouse MES database, the ODS (Operational Data Store tables replicated from the MES SQL Server database), the CDM (canonical and IoT events data) and the DWH (aggregated data). Data Manager substitutes each with the name of the matching database before the query runs, so `{db_ODS}.CoreDataModel_T_Material` resolves in whatever database your `ODS` is deployed as. A macro with the wrong case is not replaced.

Attribute names are data, not identifiers, and they fail the same way. Compare `JSONExtractString(x, 'Name') = 'SuperiorOrderNumber'` against an attribute stored as `superiorOrderNumber` and you get no rows. The worked-case query compares `lower(JSONExtractString(x, 'Name')) = 'superiorordernumber'`, which makes the match case-insensitive. Use that pattern.

### Failure 8: Reports Cut at 1000 Rows

**The wrong result:** a report shows exactly 1000 rows and every total is low.

**Why:** per the [OData access tutorial](https://help.criticalmanufacturing.com/12.0/tutorials/modules/iot-data-platform/odataaccess/), the endpoint returns up to `DATAMANAGER_DEFAULT_MAX_ROWS` rows when `$top` is not set, and the documented default is 1000. Nothing in the result marks it as truncated.

**The fix:** set `$top` and `$skip` explicitly when you page, use `$count` to know the real size, or aggregate in the data set so the result is small. Check the setting for your deployment.

`$count` has its own catch. Per the docs it returns the total of the unfiltered data set, so after a `$filter` it can report a size that is not the size of the rows you are paging through. Do not use it to decide when to stop paging a filtered request.

### Failure 9: Comments That Break the Data Set

**The wrong result:** a data set that validated fine in your editor fails to create, or fails after a refresh, with an error that does not mention the comment.

**Why:** the [data set creation page](https://help.criticalmanufacturing.com/12.0/userguide/data-platform/data/data-set/create_data_set/) documents restrictions on comments and semicolons in the query, because the query is split on semicolons. A comment such as `-- don't forget; this is temporary` contains both a semicolon and an apostrophe, which is exactly the shape to avoid. Read that page for the exact rules in your version.

**The fix:** keep comments free of both. If a query will not create and you cannot see why, delete the comments first.

## A Worked Case

The use case from the [architecture post](https://j-roque.com/posts/20261116-clickhouse-architecture/#building-a-data-set): for each production order, find its superior production order (an attribute), the related materials, and whether each step is in process or completed, based on track in and track out events. The result feeds a Grafana dashboard.

The query reads two history tables, so it needs `FINAL` for replayed events and reads `_Attributes` as the only place attributes are filled (Failure 6). It was written and debugged in DBeaver before it went anywhere near the platform:

![The data set query open in DBeaver, with the result grid below](	
https://image.j-roque.com/posts/20261117-clickhouse-silent-failures/dbeaver-dataset-query.png "The data set query in DBeaver. Debug it here, against ClickHouse, before you paste it into a data set.")

This is the real data set query, trimmed to the parts that matter:

```sql
WITH RawMaterials AS
(
    SELECT DISTINCT
        JSONExtractInt(M.ProductionOrder, 'Id')      AS ProductionOrderId,
        JSONExtractString(M.ProductionOrder, 'Name') AS ProductionOrder,
        JSONExtractString(
            arrayFilter(
                x -> lower(JSONExtractString(x, 'Name')) = 'superiorordernumber',
                MAX(PO._Attributes) OVER (
                    PARTITION BY PO.ProductionOrderId,
                                 JSONExtractString(arrayJoin(PO._Attributes), 'Name')
                    ORDER BY PO.ModifiedOn DESC)
            )[1], 'Value')                           AS SuperiorPO,
        PO.DueDate                                   AS DueDate,
        PO.Quantity                                  AS POQty,
        JSONExtractString(PO.Product, 'Name')        AS Product,
        JSONExtractInt(M.Step, 'Id')                 AS StepId,
        JSONExtractString(M.Step, 'Name')            AS Step,
        M.MaterialId,
        M.Name                                       AS Material,
        M._OperationHistory_OperationName            AS OperationName
    FROM CoreDataModel_T_MaterialHistory M FINAL
    JOIN CoreDataModel_T_ProductionOrderHistory PO FINAL
        ON JSONExtractInt(M.ProductionOrder, 'Id') = PO.ProductionOrderId
    WHERE length(arrayFilter(
              x -> lower(JSONExtractString(x, 'Name')) = 'superiorordernumber',
              PO._Attributes)) > 0
      AND M._OperationHistory_OperationName IN ('TrackIn', 'TrackOut')
)
SELECT DISTINCT
    ProductionOrderId, ProductionOrder, SuperiorPO, DueDate, POQty,
    Product, StepId, Step, MaterialId, Material,
    countDistinct(MaterialId) OVER (PARTITION BY ProductionOrderId)         AS MaterialCountByPO,
    countIf(OperationName = 'TrackIn')  OVER (PARTITION BY ProductionOrderId, StepId) AS TrackInCount,
    countIf(OperationName = 'TrackOut') OVER (PARTITION BY ProductionOrderId, StepId) AS TrackOutCount,
    IF(TrackInCount = TrackOutCount, 'Completed', 'InProcess')              AS Status
FROM RawMaterials RM
ORDER BY ProductionOrder, Material
```

The query uses `JSONExtractInt` on the production order snapshot to build the join key, `lower()` on the attribute name, `countIf` as a window function, and `Status` reusing `TrackInCount` and `TrackOutCount` in the same `SELECT`. ClickHouse allows alias reuse within a query. T-SQL does not.

Two things in the query I would change. The `MAX(PO._Attributes) OVER (...)` with an `arrayJoin` inside the `PARTITION BY` is a window maximum over arrays, and it is more complex than needed for "latest value of one attribute per order". And joining every material event to every production order history row fans the result out, so `SELECT DISTINCT` is doing real work to hide it. The query works, but on new data I would check it against a count first.

The cleaner shape is the `argMax` history CTE from Failure 6, grouped by order, with the counts as a plain aggregation:

```sql
-- Sketch, not run against a real instance. Grain is one row per order and step, not per material.
WITH
    superior_orders AS (
        SELECT
            ProductionOrderId,
            argMax(JSONExtractString(attr, 'Value'), ModifiedOn) AS SuperiorPO
        FROM CoreDataModel_T_ProductionOrderHistory FINAL
        ARRAY JOIN _Attributes AS attr
        WHERE lower(JSONExtractString(attr, 'Name')) = 'superiorordernumber'
        GROUP BY ProductionOrderId
    ),
    step_events AS (
        SELECT
            JSONExtractInt(ProductionOrder, 'Id') AS ProductionOrderId,
            JSONExtractInt(Step, 'Id')            AS StepId,
            JSONExtractString(Step, 'Name')       AS StepName,
            MaterialId,
            _OperationHistory_OperationName       AS OperationName
        FROM CoreDataModel_T_MaterialHistory FINAL
        WHERE _OperationHistory_OperationName IN ('TrackIn', 'TrackOut')
    )
SELECT
    se.ProductionOrderId,
    so.SuperiorPO,
    se.StepName,
    uniqExact(se.MaterialId)                AS Materials,
    countIf(se.OperationName = 'TrackIn')   AS TrackIns,
    countIf(se.OperationName = 'TrackOut')  AS TrackOuts,
    if(TrackIns = TrackOuts, 'Completed', 'InProcess') AS Status
FROM step_events AS se
INNER JOIN superior_orders AS so ON se.ProductionOrderId = so.ProductionOrderId
GROUP BY se.ProductionOrderId, so.SuperiorPO, se.StepId, se.StepName
```

The aliases here are `StepName`, not `Step`, for the reason in Failure 4. The original query uses `Step` and `Product` as aliases, and its references to the real columns are qualified (`M.Step`, `PO.Product`).

The inner join drops orders without the attribute, which is what the use case wants. Because `argMax` takes the latest written value, it will not notice an attribute that was later deleted. Decide whether that matters for your report. And if your tables carry `_is_deleted`, filter on it (Failure 2). The original query does not.

## Publishing It as a Data Set

Creating the data set is covered in the [architecture post](https://j-roque.com/posts/20261116-clickhouse-architecture/#building-a-data-set). Three documented limits apply to data sets. Data set names must avoid `\ / : ? | < > * " .`. Remember Failure 9 before you paste a commented query. And parameterized views cannot reference other parameterized views across databases, so read the [data set creation page](https://help.criticalmanufacturing.com/12.0/userguide/data-platform/data/data-set/create_data_set/) before you stack them.

### Parameters and Filter Pushdown

A data set with several CTEs does its work inside the query. OData options such as `$filter` are applied on top of the data set, and the documentation advises being specific in the query for performance. Whether your filter reaches the inner CTEs or only trims the output is something to check on your data set, not assume.

The worked-case data set makes this a real question. The Grafana dashboard filters on `SuperiorPO`, and in the worked-case query `SuperiorPO` is computed by a window function over a CTE. A predicate on a column derived from a window is a poor candidate for pushing below that window, while a predicate on `ProductionOrderId`, the window's partition key, is a better one. This is a suspected cause to check, not a measured finding: I have not measured it on a real instance. A runnable check:

1. Call the data set over OData without a filter, then again with `$filter` on one production order.
2. Compare what ClickHouse actually read:

```sql
SELECT
    event_time,
    query_duration_ms,
    read_rows,
    read_bytes
FROM system.query_log
WHERE type = 'QueryFinish'
  AND query LIKE '%CoreDataModel_T_MaterialHistory%'
ORDER BY event_time DESC
LIMIT 5
```

If `read_rows` is the same in both runs, the filter trims output only and the inner query still scans everything. `EXPLAIN indexes = 1` on the inner query shows which parts and granules the primary key selects.

When the filter does not reach the inside, parameters are the documented way to push values into the query. A query data set takes them in the SQL:

```sql
SELECT *
FROM {db_ODS}.CoreDataModel_T_Material AS m FINAL
WHERE m.Name = {name: String}
  AND m._is_deleted = 0
```

The caller passes values through OData with `parameters=var1=123,var2='example'`, with arrays in brackets. Parameter support depends on your release, so check the release notes for your version.

## Consuming It Without Getting Burned

The URL pattern and the tools are in the [architecture post](https://j-roque.com/posts/20261116-clickhouse-architecture/#odata-every-data-set-is-an-endpoint). Two points from the worked case's Grafana dashboard tie back to the failures above. The `$filter` on `SuperiorPO` depends on the pushdown behavior in the previous subsection, so check it. And row limits apply here too (Failure 8), so aggregate with `$apply` or set `$top`.

## Final Thoughts

ClickHouse works best on denormalized event tables, and T-SQL habits produce wrong results without an error. In all nine failures the query runs and returns numbers, and the numbers are wrong.

Test every data set against a count you trust, because ClickHouse does not report these errors.