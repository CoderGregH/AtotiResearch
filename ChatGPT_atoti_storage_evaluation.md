# Data Storage Options for Atoti Cube -- Evaluation

## Initial Scenario / Question

-   We have active pivot with an in memory cube from Atoti
-   We load in snapshots of data that we load into that cube
-   We build measures and dimension in cube but otherwise avoid logic in
    the cube. We do this in preprocessing

We currently put the data for these cube in object storage (HCP which is
like S3 but on prem) which is working quite well

### Other options suggested

-   MSSQL server
-   Minio with Iceberg

### Request

Evaluate the tech solution and provide: - Pros and cons of each - Other
strong contenders - Ranking of recommendations

------------------------------------------------------------------------

## Summary Recommendation

Best default: Keep object storage snapshot pattern.\
Upgrade path: Object storage + Iceberg if stronger governance or
multi-tool access is needed.\
Weakest fit: MSSQL as primary snapshot store.

------------------------------------------------------------------------

## Ranking

1.  Object storage snapshots (HCP / S3-style)
2.  Object storage + Iceberg
3.  MSSQL Server
4.  PostgreSQL (alternative relational option)
5.  Delta Lake / Hudi (if Spark-heavy ecosystem)

------------------------------------------------------------------------

## 1. Object Storage Snapshots (Current Approach)

### Pros

-   Simple, robust, and reproducible
-   Aligns well with immutable snapshot pattern
-   Cheap and scalable
-   Easy rollback
-   Keeps logic in preprocessing
-   Works well with Parquet

### Cons

-   Weak governance and metadata
-   Requires conventions (naming, manifests)
-   Limited concurrency handling
-   Harder for multi-tool access

### Best When

-   Atoti is main consumer
-   Snapshots are batch-based
-   Simplicity is preferred

------------------------------------------------------------------------

## 2. Object Storage + Iceberg

### Pros

-   Strong table semantics
-   Built-in snapshot history
-   Schema evolution support
-   Multi-engine compatibility
-   Future-proof architecture

### Cons

-   More complexity
-   Operational overhead
-   May be overkill for Atoti-only usage

### Best When

-   Multiple tools need access
-   Schema evolves frequently
-   Governance matters

------------------------------------------------------------------------

## 3. MSSQL Server

### Pros

-   Strong governance and security
-   Familiar enterprise tooling
-   Good SQL access

### Cons

-   Poor fit for snapshot model
-   Higher cost and overhead
-   Encourages logic in database
-   Less aligned with Atoti pattern

### Best When

-   SQL-first organisation
-   DBA ownership required

------------------------------------------------------------------------

## Other Considerations

### Lightweight Metadata Layer

Keep object storage but add: - Manifest tracking - Snapshot catalog -
Audit logging

### Iceberg on Existing Storage

Focus on table format rather than switching storage vendor.

------------------------------------------------------------------------

## Final Recommendation

1.  Stay with object storage and improve discipline
2.  Adopt Iceberg if needed for governance
3.  Use MSSQL only if organisationally required

------------------------------------------------------------------------

Generated on: 2026-03-24
