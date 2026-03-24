# Atoti ActivePivot Storage Layer Evaluation

---

## Initial Brief

We have an active pivot with an in-memory cube from Atoti. We load in snapshots of data into that cube and build measures and dimensions in the cube but otherwise avoid logic in the cube — we do this in preprocessing.

We currently put the data for these cubes in object storage (HCP, which is like S3 but on-prem) which is working quite well.

### Options Under Consideration

People within the firm have suggested the following alternatives:

- **MSSQL Server** — available in the firm today
- **MinIO with Iceberg** — available in the firm today

The request was to evaluate each solution with pros and cons, surface any strong contenders not yet considered, and rank each recommendation.

---

## Clarifying Questions

Before evaluating, the following questions were asked to ensure a fair and targeted assessment:

### Q1: How large are the snapshots and how frequently are they loaded?

**Answer:** Snapshots are mainly daily cuts of millions (3 to 5 million) of rows which can be broken down by region and legal entity. There are a few intraday loads (under a million rows in small batches throughout the day).

### Q2: What matters most for this solution? (Ranked)

| Priority | Rank |
|---|---|
| Query performance / low latency reads | 1 |
| Cost & operational simplicity | 2 |
| Time-travel / auditability of snapshots | 3 |
| Schema evolution & future flexibility | 4 |

### Q3: Who primarily consumes the data beyond Atoti?

**Answer:** Atoti + some ad-hoc SQL/BI queries, with the biggest slant towards Atoti ActivePivot cubes.

---

## The Critical Insight

Since Atoti ActivePivot loads everything into an **in-memory cube**, the storage layer is fundamentally a **staging and persistence layer**, not a hot query engine. Atoti bulk-reads snapshots, builds the cube, and then serves queries from memory. This means the storage layer needs to be great at **bulk reads and organised writes** — not low-latency point queries. That distinction reshapes the entire comparison.

---

## Option 1: HCP Object Storage (Current Solution)

**How it works today:** Snapshot files (presumably Parquet or CSV, partitioned by region/legal entity) are written to HCP. Atoti reads them in bulk at load time.

### Pros

- Already working and proven in the environment — there's real value in not fixing what isn't broken.
- Object storage is purpose-built for this pattern: write-once bulk files, read in full at load time. It's the natural fit for snapshot staging.
- Operationally minimal — no database engine to patch, tune, or failover. Files are just files.
- Partitioning by region/legal entity is trivial at the filesystem level (prefix/folder structure), which aligns perfectly with the data shape.
- Scales effortlessly for these data volumes (3–5M rows daily is modest for object storage).

### Cons

- No built-in time-travel or schema versioning — snapshot history is managed through folder naming conventions or a lightweight metadata catalogue.
- Ad-hoc SQL/BI queries require an additional tool (e.g. DuckDB, Trino, or a BI connector that reads Parquet). You can't just point a SQL client at it.
- No ACID transactions — if a write fails mid-upload, cleanup logic is needed.

---

## Option 2: MSSQL Server

**How it works:** Snapshots land in relational tables. Atoti connects via JDBC, runs a query, and loads results into the cube.

### Pros

- Already available in the firm, so procurement and support are sorted.
- Ad-hoc SQL queries are trivial — anyone with SSMS or a BI tool can explore the data without Atoti.
- Strong tooling ecosystem: backups, monitoring, security, audit trails are all mature and well-understood.
- Transactional writes mean partial loads don't leave corrupted state.

### Cons

- **Paying for capabilities not needed.** MSSQL's strength is indexed point queries, joins, and transactional consistency — but Atoti doesn't need any of that. It just needs "give me all the rows for this snapshot." A full RDBMS would be running as a glorified file server.
- Operational overhead is real: index management, storage growth, backups, patching, and licensing. SQL Server licensing in particular is expensive relative to the value it adds here.
- Bulk loading 3–5M rows daily into SQL Server requires tuning (bulk insert, minimal logging, index maintenance). Solvable but unnecessary work.
- The JDBC read path adds latency compared to Atoti reading files directly — serialising millions of rows through a query engine and network protocol for no analytical benefit.
- Schema changes require DDL migrations, which is more friction than just writing a new file with updated columns.

---

## Option 3: MinIO + Apache Iceberg

**How it works:** MinIO provides the S3-compatible object storage (similar to HCP). Iceberg adds a table format layer on top — giving a metadata catalogue, schema evolution, time-travel, and partition management over the same Parquet files.

### Pros

- Directly addresses the third priority (time-travel/auditability). Iceberg's snapshot isolation lets you query any historical version of a table trivially — "show me the data as it looked on March 15th" becomes a one-liner.
- Schema evolution is built in — add/rename/drop columns without rewriting historical data.
- Partitioning by region and legal entity is a first-class Iceberg feature with hidden partitioning (consumers don't need to know the partition scheme).
- Opens the door for Spark, Trino, DuckDB, or other engines to query the same data for ad-hoc BI use — without duplicating anything.
- MinIO is already available internally, so the object storage component is proven.

### Cons

- **Added complexity for a problem that may not exist today.** Iceberg requires a catalogue (Hive Metastore, AWS Glue equivalent, REST catalogue, or Nessie). That's another component to deploy and maintain.
- The current HCP setup is already working. Migrating to MinIO + Iceberg is a meaningful project — new write pipelines, catalogue setup, and validating that Atoti's read path works cleanly with Iceberg tables.
- Iceberg's strengths (time-travel, schema evolution) are ranked 3rd and 4th on the priority list. This would be optimising for secondary concerns.
- For these data volumes (3–5M rows daily), Iceberg's overhead may not be justified. Iceberg shines at scale — billions of rows, petabyte-range datasets, many concurrent readers. This use case is well within the range where simple partitioned Parquet files work perfectly.
- Compaction and maintenance jobs (Iceberg accumulates small files from intraday loads) add operational burden.

---

## Option 4: DuckDB as a Sidecar Query Layer over Existing HCP (Additional Contender)

**How it works:** Keep HCP as-is. For the ad-hoc SQL/BI need, deploy DuckDB (embedded or as a lightweight service) that reads Parquet directly from object storage. Atoti continues to load from HCP unchanged.

### Pros

- Zero disruption to the working Atoti pipeline.
- Gives the SQL queryability that HCP alone lacks, satisfying the "some ad-hoc SQL/BI" need.
- No new infrastructure — DuckDB is an embedded engine, runs in-process or as a tiny service.
- Extremely fast for analytical queries over Parquet at these data volumes.

### Cons

- Not a full-featured shared query engine — if BI needs grow significantly, it would be outgrown.
- No built-in time-travel (though it can be pointed at different snapshot folders).
- Less well-known internally, so there may be governance or support concerns.

---

## Final Rankings

| Rank | Solution | Rationale |
|---|---|---|
| **1** | **HCP (current) + DuckDB for ad-hoc SQL** | Top two priorities are query performance and operational simplicity. HCP already delivers both. Atoti's bulk-read pattern is ideally served by object storage, and it's already proven. The only gap is ad-hoc SQL, which DuckDB closes cheaply. Don't introduce complexity to solve a problem that doesn't exist. |
| **2** | **MinIO + Iceberg** | The right answer if time-travel/auditability becomes a hard requirement (regulatory or audit-driven). But the operational cost is real and should be justified by a concrete need, not architectural fashion. |
| **3** | **MSSQL Server** | Weakest fit for this specific pattern. Adds cost, complexity, and latency to serve a workload that doesn't benefit from what MSSQL is good at. Only wins if firm-wide policy mandates an RDBMS for all data, or if the ad-hoc BI demand is much larger than described. |

---

## Bottom Line

The current HCP setup is well-suited to the workload. The strongest improvement would be adding DuckDB as a lightweight SQL layer for ad-hoc queries, preserving everything that already works while closing the only meaningful gap. Moving to MSSQL or Iceberg should be driven by concrete new requirements, not architectural aspiration.
