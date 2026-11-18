---
title: "Reporting on the New MES Data Architecture: A Map from Data Source to Report"
description: "The new MES data architecture moves complexity out of joins and into the engine and the query. A map of the data sources, the ClickHouse databases, denormalized JSON, cubes, data sets, OData and the tools on top."
summary: "You write fewer joins on ClickHouse, but you have to understand how the data is shaped and stored. From SQL Server and Analysis Services to ClickHouse, CDM and cubes over GraphQL, and on to data sets, OData, Stimulsoft, Grafana and Power BI."
categories: ["Analytics"]
tags: ["ClickHouse", "Data Platform", "Reporting", "OData", "Grafana"]
date: 2026-11-16
showInRSS: true
draft: false
authors:
  - Roque
---
The new MES data architecture does not remove complexity. It moves it out of the joins and into the engine and the query.

You write fewer joins. In exchange, you have to understand how the data is shaped and how it is stored, because the engine will happily answer a question you did not mean to ask.

## Overview

The [previous post on reports](https://j-roque.com/posts/20260730-reports/) covered the pipeline: events, Kafka, ClickHouse, data sets. This one starts where the data lands and ends at a finished report, using the Critical Manufacturing MES Data Platform. Table names come from the 11.x-era data model and can differ by version, so confirm them with `SHOW CREATE TABLE` on your system. Reference material lives in the [Critical Manufacturing documentation](https://help.criticalmanufacturing.com/).

This post owns the model. The [next post](https://j-roque.com/posts/20261117-clickhouse-silent-failures/) owns the failures.

## Recap: New Data Architecture

In the SQL Server-based architecture we had three data sources: MS Analysis Services cubes, and the ODS and DWH SQL Server databases. The new architecture has six data sources: ODS, DWH, CDM, Apps, IoT Events, and a data source that uses GraphQL to interface with the cubes.

![Diagram titled CM new data architecture. Top: the three SQL Server data sources AS, DWH and ODS. Bottom: the six new data sources DataManager-Apps, CDM, Cubes (GraphQL), DWH, IoTEvents and ODS, five over OData and the cubes over GraphQL](https://image.j-roque.com/posts/20261116-clickhouse-architecture/data-sources-old-vs-new.png "Old versus new. Three data sources became six, and the cubes moved from Analysis Services to GraphQL.")

- **ODS**: data sets over the Operational Data Store tables, replicated from the MES SQL Server database, per the docs. `MES` is the ClickHouse MES database, which you can also pick as a source when you create a data set.
- **DWH**: data warehouse data sets containing aggregated data.
- **CDM**: the **Canonical Data Model**, anchored in the ISA-95 standard, per the [CDM post on the Critical Manufacturing dev blog](https://devblog.criticalmanufacturing.com/blog/20250716_canonical_data_model/). Per the docs, it holds IoT events data, including system-generated CDM events.
- **Apps**: data sets created and used by MES Apps.
- **IoT Events**: data sets associated with IoT events.
- **Cubes**: a GraphQL API for querying data cubes.

Three of the six are new: CDM, Apps and IoT Events. The cubes stay, but they moved from Analysis Services to GraphQL. ODS Data Silos exist too, and only with an EDP license.

## The Databases

ClickHouse has databases, not schemas inside a database. That one fact explains most of the naming you will see.

### ODS: Same Objects, Different Separator

The `ODS` database looks a lot like the SQL Server one, and the objects are mostly the same. SQL Server puts the schema in front of the object with a dot. ClickHouse has no schema level below the database, so the schema becomes part of the table name: `schema.ObjectName` turns into `schema_ObjectName`.

![DBeaver tree of the ClickHouse ODS database showing tables named with the schema_ObjectName pattern](https://image.j-roque.com/posts/20261116-clickhouse-architecture/ods-tables-naming.png "The ODS database in DBeaver. Same objects as SQL Server, with the schema folded into the table name.")

`CoreDataModel_T_Material` is `CoreDataModel.T_Material` in the old world.

### CDM and DWH

`CDM` and `DWH` have their own naming. `CDM` tables read like entities (`Material_MaterialOperations`, `Production_ProductionOrder`), and `DWH` holds the staging, aggregation and summary tables that feed the cubes.

![DBeaver tree of the CDM and DWH databases with their tables and row counts](https://image.j-roque.com/posts/20261116-clickhouse-architecture/cdm-dwh-tables.png "CDM and DWH tables. CDM is organized by business entity, DWH by aggregation and staging.")

## How the Data Is Shaped: Denormalization and JSON

This is the biggest practical change, and the one that makes "fewer joins" true. In SQL Server, most related information was only reachable through joins. In ClickHouse, the bulk of it sits in columns on the table itself, either as derived service and operation columns or as JSON snapshots.

Here is the raw material history table. The step and product arrive as JSON strings:

![Query result on the material history table with the Step and Product columns holding raw JSON strings](https://image.j-roque.com/posts/20261116-clickhouse-architecture/json-columns-raw.png "Raw JSON columns on the material history table. Everything about the step and product is already there.")

Extract what you need in place:

```sql
SELECT
    MaterialId,
    Name                                 AS Material,
    _ServiceHistory_ServiceName,
    _OperationHistory_OperationName,
    JSONExtractString(Step, 'Name')      AS StepName,
    JSONExtractString(Product, 'Name')   AS ProductName
FROM CoreDataModel_T_MaterialHistory MH FINAL
```

![DBeaver query and result grid with material, service name, operation name, step and product as plain columns](https://image.j-roque.com/posts/20261116-clickhouse-architecture/json-columns-extracted.png "The same table after JSONExtractString. One table, no joins.")

The screenshot shows an earlier version of the query, which aliased the extracts as `Step` and `Product`. The SQL above uses `StepName` and `ProductName` instead, because an alias that matches a real column name shadows it. The [failures post](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#failure-4-json-extraction-that-returns-the-wrong-value) has the warning.

One direct query avoided four joins: `T_ServiceHistory`, `T_OperationHistory`, `T_Step` and `T_Product`. The service and operation data come from the derived columns, the step and product from the JSON extract functions.

### The Counterpoint

Denormalized JSON snapshots trade joins for three costs:

- **Staleness.** A snapshot is a copy taken at event time, so it does not follow a rename.
- **Weaker typing.** JSON has no declared types or constraints. A typo in a field name returns an empty string, not an error.
- **Extraction cost.** Parsing a String column on every query is work the engine repeats.

ClickHouse also gives you [dictionaries](https://clickhouse.com/docs/dictionary) and plain joins when you need current state. The design bet is that most reports get what they need from the snapshot and join only for current or deeper detail. Know which one your question needs.

## Cubes: the Pre-Aggregated Path

[Cubes](https://help.criticalmanufacturing.com/12.0/tutorials/modules/iot-data-platform/datacubes/) are the other way to answer a question. The [developer portal](https://developer.criticalmanufacturing.com/12.0/explore/guides/customizations/dataplatform/components/#cube) describes a cube as giving access to the data of the DWH tables through cubes, with GraphQL and REST support. You reach them through the **Cube Playground**, the cube explorer in the Data Platform. It lists the cubes and lets you build a query graphically.

![Cube explorer list of available cubes in the Data Platform](https://image.j-roque.com/posts/20261116-clickhouse-architecture/cube-list.png "The list of cubes in the Data Platform.")

![Cube Playground on the resource_states cube with a results table of resource_name, semi_e10_state, calendar_day and duration](https://image.j-roque.com/posts/20261116-clickhouse-architecture/cube-playground.png "The Cube Playground. Pick measures and dimensions, and the results table shows the answer.")

Once the query returns what you want, the playground generates the GraphQL for you:

![Generated GraphQL query for the selected cube measures and dimensions](https://image.j-roque.com/posts/20261116-clickhouse-architecture/cube-graphql.png "The generated GraphQL query. Copy it, paste it into your tool.")

The Grafana panel uses a trimmed version of that generated query, through the `DataManager-Cubes` data source. This one is on the resource states cube:

```graphql
query CubeQuery {
  cube {
    resource_states {
      resource_name
      semi_e10_state,
      duration
    }
  }
}
```

![Grafana query editor with the cube GraphQL query](https://image.j-roque.com/posts/20261116-clickhouse-architecture/grafana-cube-query.png "The cube query in Grafana, through the cubes data source.")

![Grafana bar chart of resource_states.duration per resource, with no state breakdown](https://image.j-roque.com/posts/20261116-clickhouse-architecture/grafana-cube-chart.png "The result as a bar chart of resource_states.duration per resource, on sample data.")

Cubes replace Analysis Services for the question "how much, by what", where the aggregation is already modeled. For row-level questions you want a data set.

## SQL Server vs ClickHouse

ClickHouse is not a worse SQL Server. It gave up OLTP semantics to be very good at reading a lot of data. Here is the comparison I would put on a wall:

| | SQL Server (T-SQL) | ClickHouse |
| :-- | :-- | :-- |
| Primary use | OLTP and OLAP, general purpose | Analytics (OLAP) |
| Storage | Row-oriented, columnstore optional | Column-oriented |
| Transactions | Full ACID, multi-statement | Experimental and limited, no general multi-statement transactions. A single `INSERT` into one partition is atomic |
| Indexes | B-tree, columnstore, filtered | Sparse primary index used for data skipping |
| Updates and deletes | Cheap and fully supported | Heavy rewrites through [mutations](https://clickhouse.com/docs/updating-data/overview). Lightweight deletes and updates exist |
| Stored procedures | Yes | No |
| Typical size | GBs to TBs | TBs to PBs. A positioning statement, not a limit |

The row that bites reporting people is updates. You do not fix a row in ClickHouse, you supersede it, and that has consequences for every query you write.

## The Few Functions That Matter

ClickHouse SQL has a long function list. A handful do the work that joins and window subqueries used to do:

| Need | Functions |
| :-- | :-- |
| Latest or earliest value | `argMax(value, key)`, `argMin(value, key)` |
| Conditional aggregation | `countIf(cond)`, `sumIf(value, cond)` |
| Arrays to rows, filtering arrays | `arrayJoin`, `arrayFilter` |
| JSON | `JSONExtractString`, `JSONExtractInt`, `JSONExtractFloat` |
| Time buckets | `toStartOfDay`, `toStartOfHour`, `toStartOfWeek`, `toStartOfMonth` |

`argMax(event_type, event_time)` gives you the last event and `argMin` the first, which replaces most of the `ROW_NUMBER() = 1` subqueries you wrote in T-SQL. `uniq` is an approximate distinct count, cheap and not exact. `uniqExact` is its exact counterpart. Conditionals are lowercase `if` and `multiIf`.

Two traps sit next to these. The `simpleJSONExtract*` family is faster than `JSONExtract*`, but it needs compact JSON and matches a field at any nesting depth, so default to `JSONExtract*` ([Failure 4](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#failure-4-json-extraction-that-returns-the-wrong-value)). And `lag` and `lead` are supported window functions, while the `*InFrame` variants only see what the frame lets them see ([Failure 5](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#failure-5-window-functions-that-return-defaults)).

## The Engine in One Paragraph

Tables use the `MergeTree` family. Data is written in immutable parts, and parts are merged asynchronously in the background. `ReplacingMergeTree` deduplicates at merge time, and the `ORDER BY` is the dedup key: rows that share it are candidates for replacement. Because merges are asynchronous, a plain `SELECT` can read old and new versions of a row, and `FINAL` resolves it at read time. How that fails, and why the partition key matters, is in [Failure 1](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#failure-1-duplicate-rows-without-final) and [Failure 3](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#failure-3-versions-and-partitions-that-do-not-do-what-you-assume).

## Building a Data Set

The use case: for each production order, collect the superior production order (an attribute), the related materials, and whether each step is in process or completed, based on track in and track out events. The result feeds a Grafana dashboard.

Write and debug the query in a SQL client such as DBeaver, against the ClickHouse database. When it returns what you want, move it into the platform. Here is the abridged shape:

```sql
-- Abridged. The full query is in the worked case of the failures post
WITH RawMaterials AS
(
    SELECT DISTINCT
        JSONExtractString(M.ProductionOrder, 'Name') AS ProductionOrder,
        -- latest SuperiorOrderNumber attribute of the order
        ...                                          AS SuperiorPO,
        JSONExtractString(M.Step, 'Name')            AS StepName,
        M.Name                                       AS Material,
        M._OperationHistory_OperationName            AS OperationName
    FROM CoreDataModel_T_MaterialHistory M FINAL
    JOIN CoreDataModel_T_ProductionOrderHistory PO FINAL
        ON JSONExtractInt(M.ProductionOrder, 'Id') = PO.ProductionOrderId
    WHERE M._OperationHistory_OperationName IN ('TrackIn', 'TrackOut')
)
SELECT ...,
    if(TrackInCount = TrackOutCount, 'Completed', 'InProcess') AS Status
FROM RawMaterials
```

The full query, a cleaner rewrite and the parts I would not copy are in the [worked case](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#a-worked-case).

In the MES Data Platform, create a user-defined [data set](https://help.criticalmanufacturing.com/12.0/userguide/data-platform/data/data-set/create_data_set/). Pick a name and a folder, the source (`ODS`, `CDM`, `DWH` or `MES`), the owner type (`App`, `Role` or `User`), paste the query and validate it.

![Data set details page with folder UserDefined, data source ODS, owner type User and state Pending](https://image.j-roque.com/posts/20261116-clickhouse-architecture/dataset-details.png "A new user-defined data set. Folder UserDefined, source ODS, owner type User, state Pending.")

After the refresh, the state moves from Pending to Completed and the fields and their types are inferred. Then you can preview the rows:

![Data set preview grid with production order, superior PO, due date, quantity, product and step columns](https://image.j-roque.com/posts/20261116-clickhouse-architecture/dataset-preview.png "Preview after the refresh. If the preview is wrong, the report will be too.")

## OData: Every Data Set Is an Endpoint

Every data set is exposed over OData V4, under a root folder (`Apps`, `CDM`, `DWH`, `IotEvents`, `ODS` or `UserDefined`):

```text
https://<hostname>/datamanager/odata/<Folder>/<DataSet>
```

Nested folders use a dot separator. One example of each:

| Source | Path |
| :-- | :-- |
| `ODS` | `ODS/Entities.Area` |
| `DWH` | `DWH/MaterialAttributes` |
| `CDM` | `CDM/Material.MaterialOperations` |
| `UserDefined` | `UserDefined/<DataSet>` |

The [OData access tutorial](https://help.criticalmanufacturing.com/12.0/tutorials/modules/iot-data-platform/odataaccess/) covers `$select`, `$filter`, `$orderby`, `$top`, `$skip`, `$count` and `$apply`. Without `$top`, the Data Manager returns at most `DATAMANAGER_DEFAULT_MAX_ROWS` rows, 1000 by default, and nothing marks the result as truncated ([Failure 8](https://j-roque.com/posts/20261117-clickhouse-silent-failures/#failure-8-reports-cut-at-1000-rows)). If OData is new to you, start with the [OData post](https://j-roque.com/posts/20260414-odata/). Every tool below talks to this endpoint, except the cubes.

## Reporting Tools

### Stimulsoft

Create a report, add a data source of the user-defined data manager type, and the data sets in your folder appear. A group header on `SuperiorPO` gives you the production orders and their materials grouped underneath. The [previous post](https://j-roque.com/posts/20260730-reports/) has the background on Stimulsoft in the MES.

![Stimulsoft report designer with a group header on SuperiorPO and production order rows underneath](https://image.j-roque.com/posts/20261116-clickhouse-architecture/stimulsoft-report.png "A Stimulsoft report on the data set. The group header on SuperiorPO does the grouping.")

### Grafana

Grafana is where the data set earns its keep. This dashboard uses the OData data source and works like this:

- **A variable from the data set.** A query variable `SuperiorPO` runs `<DataSet>?$select=SuperiorPO` against the data set, with multi-select and an "All" option.
- **The table filters on it.** `$select=...&$filter=SuperiorPO in (${SuperiorPO:singlequote})`, with `usePost` enabled so the long query goes as a POST.
- **Charts aggregate on the server.** `$apply=filter(SuperiorPO in (...))/groupby((SuperiorPO,Material))` feeds the bar chart, and the same with `Product` feeds the pie.

![Grafana dashboard with a SuperiorPO variable, a status table, a bar chart of materials and a pie chart by product](https://image.j-roque.com/posts/20261116-clickhouse-architecture/grafana-po-dashboard.png "The dashboard on top of the data set: table, bar chart and pie, all driven by the SuperiorPO variable.")

Overrides do the formatting. A value mapping turns `Completed` green and `InProcess` yellow, and the cell background option paints the whole cell. A data link on the `Material` column opens the material page in the MES using the hidden `MaterialId`, so the table is also a navigation tool.

The standard table has limits, such as row layouts it cannot produce. The HTML graphics panel is the escape hatch. You write HTML and JavaScript, transform the data yourself, and render whatever you want. Reach for it when the table runs out, not before.

Grafana also reads the cubes through the `DataManager-Cubes` data source, with the GraphQL query from the Cube Playground. Use it for aggregated, pre-modeled measures such as state durations, and data sets when you need rows.

### Power BI and Excel

Point them at the same OData feed. A table with counts per superior order and a pie chart takes minutes.

![Power BI report built on the OData feed of the data set](https://image.j-roque.com/posts/20261116-clickhouse-architecture/powerbi-odata.png "Power BI on the same OData endpoint. No new integration, just another consumer.")

## Final Thoughts

The architecture has a clean shape, and it asks for something in return. A data set exposed over OData is consumed by Stimulsoft, Grafana and Power BI or Excel alike. Cubes go through GraphQL instead.



**Next:** [Reports on ClickHouse Fail Quietly: Nine Silent Failures](https://j-roque.com/posts/20261117-clickhouse-silent-failures/).

## Further Reading

- [OData access](https://help.criticalmanufacturing.com/12.0/tutorials/modules/iot-data-platform/odataaccess/), Critical Manufacturing tutorial
- [Create a data set](https://help.criticalmanufacturing.com/12.0/userguide/data-platform/data/data-set/create_data_set/), Critical Manufacturing user guide
- [Data cubes](https://help.criticalmanufacturing.com/12.0/tutorials/modules/iot-data-platform/datacubes/), Critical Manufacturing tutorial
- [Data platform components: cube](https://developer.criticalmanufacturing.com/12.0/explore/guides/customizations/dataplatform/components/#cube), Critical Manufacturing developer portal
- [The Canonical Data Model](https://devblog.criticalmanufacturing.com/blog/20250716_canonical_data_model/), Critical Manufacturing dev blog
- [ReplacingMergeTree guide](https://clickhouse.com/docs/guides/replacing-merge-tree), ClickHouse
- [JSON functions](https://clickhouse.com/docs/sql-reference/functions/json-functions), ClickHouse
- [Window functions](https://clickhouse.com/docs/sql-reference/window-functions), ClickHouse
