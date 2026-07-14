# RAG Pipeline - Job Failure Monitor

A Retrieval-Augmented Generation (RAG) system for monitoring and analyzing Databricks job failures. This pipeline automatically ingests job run logs, vectorizes failure records, and enables semantic search and contextual Q&A over your job failure history.

## Overview

This project implements a complete RAG pipeline running on Databricks, with **zero external dependencies** — all models and infrastructure run on Databricks Foundation Model API and Delta tables.

### Problem Solved

When jobs fail in production, teams need quick answers:
- What caused this error?
- Have we seen this before?
- What's the recommended fix?

This pipeline lets you ask natural language questions about job failures and get contextual answers based on your organization's failure history.

### Architecture

```
001: Schema Setup
    ↓
    Creates: rag_jobs schema, jobs_run_log table, error_registry table
    
002: Data Ingestion (Scheduled)
    ↓
    Fetches Databricks job runs → extracts errors → stores in jobs_run_log
    
003: Embedding & Vectorization (Scheduled)
    ↓
    Builds rich documents per failure → vectorizes with databricks-gte-large-en
    → stores in job_embeddings Delta table
    
004: Query & Retrieval (Interactive)
    ↓
    Embeds user query → finds top-k similar failures → generates answer with LLM
```

## Notebooks

### 001 - Schema & Seed Setup
**One-time setup notebook**

- Creates `rag_jobs` schema
- Creates `jobs_run_log` table (stores all job run history)
- Creates `error_registry` table (known error types with remediation steps)
- Seeds error registry with common Databricks job failures
- Validates setup with sanity checks

**Run this first.** Safe to re-run.

### 002 - Data Ingestion
**Scheduled notebook** (e.g., every 30 minutes)

- Polls Databricks Jobs API for recent runs
- Filters to terminal states (SUCCESS, FAILED, INTERNAL_ERROR, SKIPPED)
- For failed runs: fetches full traceback and classifies error type
- Merges into `jobs_run_log` (insert-only, no duplicates)

**Prerequisite:** Run notebook 001 first.

### 003 - Embedding & Vectorization
**Scheduled notebook** (e.g., hourly, or after notebook 002)

- Reads unembedded failed runs from `jobs_run_log`
- Builds rich documents (error type + root cause + actions + traceback)
- Vectorizes using `databricks-gte-large-en` (1024-dimensional embeddings)
- Stores embeddings in `job_embeddings` Delta table
- Marks rows as `embedded_at` upon completion

**Prerequisites:** Notebooks 001 and 002.

### 004 - Query & Retrieval
**Interactive notebook**

- Load embeddings from Delta table (from notebook 003)
- Embed user query using same model (`databricks-gte-large-en`)
- Retrieve top-k most relevant failures (cosine similarity)
- Generate contextual answer using Databricks Foundation Model API LLM

**Prerequisites:** Notebooks 001, 002, and 003.

## Tables

### `rag_jobs.jobs_run_log`
Databricks job run history with failure details.

| Column | Type | Purpose |
|--------|------|---------|
| run_id | LONG | Databricks job run_id (part of primary key) |
| job_id | STRING | Databricks job_id (part of primary key) |
| job_name | STRING | Human-readable job name |
| run_date | DATE | Run trigger date (UTC) |
| status | STRING | SUCCESS or FAILED |
| cluster_id | STRING | Cluster that executed the run |
| triggered_by | STRING | Trigger source (scheduler, manual, etc.) |
| traceback | STRING | Full stack trace for failed runs |
| error_message | STRING | Extracted error message |
| error_type | STRING | Classified error type (e.g., NullPointerException) |
| duration_secs | INT | Run duration in seconds |
| embedded_at | TIMESTAMP | Set by notebook 003; NULL if pending embedding |

### `rag_jobs.error_registry`
Known error types with root causes and remediation steps.

| Column | Type | Purpose |
|--------|------|---------|
| error_type | STRING | Classified error type (matches jobs_run_log) |
| root_cause | STRING | Explanation of why this error occurs |
| actions | STRING | Numbered remediation steps |
| email | STRING | Owner / on-call contact |
| severity | STRING | HIGH / MEDIUM / LOW |
| last_updated | TIMESTAMP | When this entry was last modified |

### `rag_jobs.job_embeddings`
Vectorized job failures for RAG retrieval.

| Column | Type | Purpose |
|--------|------|---------|
| doc_id | STRING | String representation of run_id (primary key) |
| run_id | LONG | Databricks job run_id |
| job_id | STRING | Job identifier |
| job_name | STRING | Human-readable job name |
| run_date | STRING | Run date |
| error_type | STRING | Error classification |
| severity | STRING | Error severity level |
| email | STRING | Owner contact |
| actions | STRING | Recommended remediation steps |
| cluster_id | STRING | Cluster that ran the job |
| content | STRING | Full document text that was vectorized |
| embedding | ARRAY<FLOAT> | 1024-dimensional vector from databricks-gte-large-en |
| created_at | TIMESTAMP | When this record was embedded |

## Prerequisites

### System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| **Databricks Runtime** | DBR 13.3 LTS | DBR 14.3+ LTS |
| **Python** | 3.9 | 3.10+ |
| **Cluster Size** | 2 workers (2 cores each) | 4+ workers (4+ cores each) |
| **Workspace Region** | Any | Same as job cluster region |

### Databricks Workspace Setup

1. **Workspace Configuration**
   - Admin access to workspace for initial setup
   - Storage: Minimum 10GB DBFS/Delta Lake capacity
   - Catalog: `hive_metastore` (default) - no Unity Catalog required

2. **Compute & Permissions**
   - Create all-purpose cluster or job cluster (DBR 14.3+)
   - Service principal or user with permissions to:
     - Create schemas in `hive_metastore`
     - Create and modify Delta tables
     - Query Databricks Jobs REST API
     - Use Foundation Model API

3. **Network & API Access**
   - Outbound HTTPS access to Databricks Jobs API endpoints
   - Databricks workspace URL accessible from cluster
   - Personal access token (for service principal or user auth)

### Foundation Model API Access

- Verify Foundation Model API is enabled in workspace
- Ensure your workspace region supports:
  - `databricks-gte-large-en` (embedding model)
  - `databricks-meta-llama-3-1-70b-instruct` (LLM)
  - Other optional models

**Check access:**
```python
from databricks.sdk import WorkspaceClient
w = WorkspaceClient()
# Available models will be listed in Notebooks 003 & 004 widget dropdowns
```

### IAM & Security

**Required Permissions:**
```
- catalogs:USE (on hive_metastore)
- schemas:CREATE, USE, MODIFY (on rag_jobs)
- tables:CREATE, SELECT, INSERT, MODIFY (on rag_jobs tables)
- jobs:READ (to query Jobs API)
- tokens:GENERATE (for notebook access token)
```

**Recommended Setup:**
- Use a **service principal** for scheduled notebooks (notebooks 002 & 003)
- Use **personal tokens** for interactive notebooks (notebook 004)
- Rotate tokens every 90 days
- Store tokens in Databricks Secrets (not hardcoded)

## Getting Started

### Initial Setup (One-Time)

1. **Clone/Import Notebooks**
   ```
   Import all 4 notebooks into your Databricks workspace
   RAG_pipeline_01_schema_&_seed_setup.ipynb
   RAG_pipeline_02_data_ingestion.ipynb
   RAG_pipeline_03_embedding_&_vectorization.ipynb
   RAG_pipeline_04_query_&_retrieval.ipynb
   ```

2. **Run Setup Notebook**
   ```
   Open: RAG_pipeline_01_schema_&_seed_setup.ipynb
   Execute all cells
   Verify: Check rag_jobs schema and tables are created
   ```

3. **Run Ingestion (Manual First Run)**
   ```
   Open: RAG_pipeline_02_data_ingestion.ipynb
   Set widget: lookback_hours = 24 (for initial load)
   Execute all cells
   Verify: Check jobs_run_log is populated
   ```

4. **Run Embedding**
   ```
   Open: RAG_pipeline_03_embedding_&_vectorization.ipynb
   Execute "Create job_embeddings table" section first
   Then execute embedding pipeline
   Verify: Check job_embeddings table has vectors
   ```

5. **Test Query Interface**
   ```
   Open: RAG_pipeline_04_query_&_retrieval.ipynb
   Execute all cells
   Test a sample query
   Verify: Get contextual answers about failures
   ```

### Production Deployment

**Schedule Notebooks as Databricks Jobs:**

1. **Notebook 002 - Data Ingestion Job**
   ```
   Frequency: Every 30 minutes
   Cluster: Shared single-node cluster (cost-effective)
   Timeout: 15 minutes
   Max concurrent runs: 1
   Notifications: Send on failure
   Widget: lookback_hours = 1
   ```

2. **Notebook 003 - Embedding Job**
   ```
   Frequency: Every 60 minutes (or hourly after notebook 002)
   Cluster: Shared multi-node cluster (if large dataset)
   Timeout: 30 minutes
   Max concurrent runs: 1
   Notifications: Send on failure
   Depends on: Notebook 002 (optional orchestration)
   ```

3. **Notebook 004 - Query Interface**
   ```
   Deploy as: Interactive endpoint or all-purpose cluster
   Access: Team dashboard or Databricks notebooks
   No scheduling needed (on-demand)
   ```

## Monitoring & Observability

### Key Metrics to Track

1. **Ingestion Health (Notebook 002)**
   - `SELECT COUNT(*) FROM rag_jobs.jobs_run_log` — total runs ingested
   - `SELECT COUNT(*) FROM rag_jobs.jobs_run_log WHERE status = 'FAILED'` — failure count
   - Latest run date: `SELECT MAX(run_date) FROM rag_jobs.jobs_run_log`

2. **Embedding Coverage (Notebook 003)**
   - `SELECT COUNT(*) FROM rag_jobs.jobs_run_log WHERE embedded_at IS NULL` — pending embeddings
   - `SELECT COUNT(*) FROM rag_jobs.job_embeddings` — total embedded records
   - Latest embedding: `SELECT MAX(created_at) FROM rag_jobs.job_embeddings`

3. **Query Performance (Notebook 004)**
   - Average retrieval time (measured in notebook)
   - Top-k precision (does notebook return relevant failures?)
   - User feedback: Are answers helpful?

### Logging & Alerting

**In Notebooks:**
- Add logging via `dbutils.notebook.run()` return values
- Log Foundation Model API errors with full stack traces
- Track start/end times and durations

**Databricks Job Alerts:**
- Enable email notifications on job failure
- Set up Slack/Teams alerts via Jobs API webhooks
- Monitor cluster provisioning failures

### Data Quality Checks

Run daily in notebook 001:
```python
# Sanity checks
assert spark.sql("SELECT COUNT(*) FROM rag_jobs.jobs_run_log").collect()[0][0] > 0
assert spark.sql("SELECT COUNT(*) FROM rag_jobs.error_registry").collect()[0][0] > 0
assert spark.sql("SELECT COUNT(*) FROM rag_jobs.job_embeddings WHERE embedding IS NOT NULL").collect()[0][0] > 0
```

## Configuration & Best Practices

### Environment Variables

Use Databricks Secrets for sensitive data:

```python
import os
from databricks.sdk import WorkspaceClient

w = WorkspaceClient()

# Get secrets (don't hardcode tokens)
DATABRICKS_TOKEN = dbutils.secrets.get(scope="rag-jobs", key="databricks_token")
WORKSPACE_URL = dbutils.secrets.get(scope="rag-jobs", key="workspace_url")
```

### Performance Tuning

| Setting | Default | Tuning |
|---------|---------|--------|
| **Batch Size (Embedding)** | 32 | Increase to 64 if API allows; decrease if rate-limited |
| **Top-K (Query)** | 5 | Increase to 10+ for broader context; decrease for precision |
| **Lookback Hours (Ingestion)** | 1 | Increase for initial load; keep small for steady-state |
| **Delta Table Optimization** | None | Run `VACUUM` and `OPTIMIZE` weekly |

**Optimize Delta Tables (Weekly):**
```sql
-- Consolidate small files
OPTIMIZE rag_jobs.jobs_run_log;
OPTIMIZE rag_jobs.job_embeddings;

-- Remove old versions (retention = 7 days default)
VACUUM rag_jobs.jobs_run_log RETAIN 7 DAYS;
VACUUM rag_jobs.job_embeddings RETAIN 7 DAYS;
```

### Error Handling

**Common Issues & Recovery:**

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Foundation Model API rate limit** | "429 Too Many Requests" | Reduce batch size in notebook 003; add exponential backoff |
| **Jobs API timeout** | "Connection timeout" | Increase `lookback_hours` to reduce pages; add retry logic |
| **Out of memory** | Spark executor failures | Reduce batch size; increase cluster memory; add partitioning |
| **Duplicate embeddings** | Vectors for same run_id appear multiple times | Re-run notebook 001 VACUUM; use run_id as unique key |

### Backup & Recovery

**Backup Strategy:**
```python
# Weekly snapshot (run monthly)
spark.sql("""
  CREATE TABLE rag_jobs.jobs_run_log_backup_YYYY_MM_DD AS
  SELECT * FROM rag_jobs.jobs_run_log
""")
```

**Recovery:**
```python
# Restore from backup
spark.sql("""
  TRUNCATE TABLE rag_jobs.jobs_run_log;
  INSERT INTO rag_jobs.jobs_run_log
  SELECT * FROM rag_jobs.jobs_run_log_backup_YYYY_MM_DD
""")
```

## Maintenance

### Regular Tasks

| Task | Frequency | Owner | Time |
|------|-----------|-------|------|
| Monitor job failures | Daily | On-call | 5 min |
| Review embedding quality | Weekly | Data team | 15 min |
| Optimize Delta tables | Weekly | Data team | 10 min |
| Update error registry | As needed | L3 support | 5 min |
| Rotate service principal tokens | 90 days | Platform team | 10 min |
| Archive old runs (>1 year) | Quarterly | Data team | 20 min |

### Scaling Considerations

- **Low volume (<100 failures/day):** Single small cluster for all notebooks
- **Medium volume (100-1k failures/day):** Separate ingestion and embedding clusters
- **High volume (>1k failures/day):** Consider partitioning by job_id; use auto-scaling clusters

### Cost Optimization

1. Use **single-node clusters** for lightweight ingestion/embedding
2. **Auto-terminate** clusters after 10 minutes of inactivity
3. **Spot instances** for non-critical batches (saving ~70%)
4. **Job scheduling:** Run during off-peak hours if possible
5. Monitor DBU spend via Databricks billing dashboard

## Security

### Data Protection

- ✅ Delta table ACLs: Restrict access to `rag_jobs` schema to authorized users
- ✅ Secrets management: Use Databricks Secrets, not hardcoded tokens
- ✅ API tokens: Use service principals; rotate every 90 days
- ✅ Audit logging: Enable Databricks audit logs to track data access

### Compliance

- ✅ Data retention: Implement retention policies per org requirements
- ✅ PII handling: Ensure error messages don't contain sensitive data
- ✅ Access control: Use workspace-level ACLs for notebook access
- ✅ Encryption: Delta tables auto-encrypted at rest by Databricks

## Models Used

- **Embedding Model:** `databricks-gte-large-en` (1024-dim vectors)
- **LLM Options:** 
  - `databricks-meta-llama-3-1-70b-instruct` (recommended)
  - `databricks-meta-llama-3-1-8b-instruct`
  - `databricks-dbrx-instruct`
  - `databricks-mixtral-8x7b-instruct`

All via Databricks Foundation Model API — **no external API keys required**.

## No External Dependencies

✅ No external vector database (embeddings stored in Delta)  
✅ No external LLM API (Databricks Foundation Model API only)  
✅ No pip installs needed (pre-installed in DBR 14.3+)  
✅ No Unity Catalog required (hive_metastore only)

## Example Queries (Notebook 004)

```
"Why is my customer_etl_daily job failing?"
"What does OutOfMemoryError mean?"
"Show me all NullPointerException failures from the last week"
"How do I fix a schema mismatch error?"
```

The pipeline retrieves similar past failures and generates contextual answers.

## Troubleshooting

### No embeddings generated
- Check: Has notebook 002 run and ingested failures?
- Check: Are there unembedded failures in `rag_jobs.jobs_run_log` with `embedded_at IS NULL`?
- Check: Notebook 003 logs for Foundation Model API errors

### Query returns no results
- Ensure notebook 003 has completed at least once
- Try a broader query (e.g., "Tell me about job failures")
- Check `job_embeddings` table is populated: `SELECT COUNT(*) FROM rag_jobs.job_embeddings`

### Slow ingestion
- Increase `lookback_hours` widget in notebook 002 to reduce API calls
- Reduce `BATCH_SIZE` in notebook 003 if hitting Foundation Model API rate limits

## Contributing

To add new error types to the registry, edit the `error_data` list in notebook 001's "Seed error_registry" cell.

## Testing & Validation

### Unit Testing Approach

**Notebook 001 (Setup)** - Validate table creation:
```python
# After running notebook 001, run these checks
tables = spark.sql("SHOW TABLES IN rag_jobs").collect()
assert len(tables) == 3, f"Expected 3 tables, got {len(tables)}"

# Check schema
jobs_log_schema = spark.table("rag_jobs.jobs_run_log").schema
assert "run_id" in [f.name for f in jobs_log_schema.fields]
```

**Notebook 002 (Ingestion)** - Validate data quality:
```python
# Check for duplicates
duplicates = spark.sql("""
  SELECT run_id, job_id, COUNT(*) as cnt
  FROM rag_jobs.jobs_run_log
  GROUP BY run_id, job_id
  HAVING cnt > 1
""")
assert duplicates.count() == 0, "Duplicate runs found"

# Check completeness
missing_error_type = spark.sql("""
  SELECT COUNT(*) FROM rag_jobs.jobs_run_log
  WHERE status = 'FAILED' AND error_type IS NULL
""")
```

**Notebook 003 (Embedding)** - Validate vectors:
```python
# Check embedding dimensions
from pyspark.sql.functions import size
embedding_dims = spark.sql("""
  SELECT size(embedding) as dim FROM rag_jobs.job_embeddings LIMIT 1
""").collect()[0][0]
assert embedding_dims == 1024, f"Expected 1024-dim, got {embedding_dims}"

# Check for null embeddings
null_embeddings = spark.sql("""
  SELECT COUNT(*) FROM rag_jobs.job_embeddings WHERE embedding IS NULL
""")
assert null_embeddings.collect()[0][0] == 0, "Found null embeddings"
```

**Notebook 004 (Query)** - Validate retrieval:
```python
# Test with sample query
answer = ask("What is a NullPointerException?")
assert len(answer) > 0, "Query returned empty answer"
assert "NullPointerException" in answer or "null" in answer.lower()
```

## Versioning & Release Notes

### Version History

**v1.0.0** (Current)
- ✅ Full RAG pipeline for job failure monitoring
- ✅ 4-notebook setup with ingestion, embedding, and query
- ✅ Foundation Model API integration
- ✅ Delta table storage (no external dependencies)

**Planned Features**
- v1.1.0: Multi-language support for queries
- v1.2.0: Custom error type classifier
- v1.3.0: Web UI dashboard for failure analytics
- v2.0.0: Integration with Slack/Teams for auto-notifications

## Support & Documentation

### Documentation

- **Notebooks:** Inline comments and markdown cells explain each section
- **README:** This file covers architecture, setup, and operations
- **Error Registry:** Seeds common Databricks errors with solutions

### Getting Help

| Issue | Resource |
|-------|----------|
| **Databricks API docs** | https://docs.databricks.com/api/workspace/jobs/list |
| **Foundation Model API** | https://docs.databricks.com/en/machine-learning/foundation-models |
| **Delta Lake docs** | https://docs.databricks.com/en/delta |
| **Job failures** | Check Databricks job run history or notebook logs |

### Reporting Issues

1. **Check logs** — Review notebook execution logs for errors
2. **Validate setup** — Re-run notebook 001 sanity checks
3. **Test components** — Run each notebook independently
4. **Isolate issue** — Identify if problem is in ingestion, embedding, or query
5. **Document findings** — Include error messages, versions, and reproduction steps

### Contributing Improvements

To improve this project:

1. **Add error types:** Update error_data in notebook 001
2. **Improve queries:** Test queries in notebook 004 and document patterns
3. **Optimize performance:** Benchmark changes and report impact
4. **Bug fixes:** Create a fork, fix, and submit for review

## License

[Specify your license - e.g., MIT, Apache 2.0, etc.]

## Support Contact

- **Issues/Bugs:** [GitHub Issues link or support email]
- **Questions:** [Slack channel or discussion forum]
- **Feature Requests:** [GitHub Discussions or issue template]

---

**Last Updated:** July 2024  
**Maintained By:** Data & Analytics Team  
**Status:** Production Ready ✅

