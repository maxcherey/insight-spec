# Table: `git_collection_runs`

## Overview

**Purpose**: Track data collection executions for monitoring, debugging, and performance analysis.

**Data Sources Examples**: 
- Bitbucket: `data_source = "insight_bitbucket_server"`
- GitHub: `data_source = "insight_github"`
- GitLab: `data_source = "insight_gitlab"`
- CustomGit: `data_source = "custom_etl"`

---

## Schema Definition

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | Int64 | PRIMARY KEY | Auto-generated unique identifier |
| `run_id` | String | REQUIRED | Unique run identifier |
| `started_at` | DateTime64(3) | REQUIRED | Run start time |
| `completed_at` | DateTime64(3) | NULLABLE | Run completion time |
| `status` | String | REQUIRED | Run status (running/completed/failed) |
| `repos_processed` | Int64 | NULLABLE | Number of repositories processed |
| `commits_collected` | Int64 | NULLABLE | Number of commits collected |
| `prs_collected` | Int64 | NULLABLE | Number of PRs collected |
| `api_calls` | Int64 | NULLABLE | Number of API calls made |
| `errors` | Int64 | NULLABLE | Number of errors encountered |
| `settings` | String | REQUIRED | JSON of collection settings |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

---

## Data Collection

### All Analyzers

**Collection Process**:
1. **At start of collection run**:
   - Generate unique `run_id`
   - Set `started_at` timestamp
   - Set `status = "running"`
   - Store initial settings
2. **During collection**:
   - Track statistics (repos, commits, PRs, API calls, errors)
   - Update periodically (optional)
3. **At completion**:
   - Set `completed_at` timestamp
   - Set `status = "completed"` or `"failed"`
   - Store final statistics

**Run ID Format**:
- GitHub: `"insight_github-2026-02-13-120000"` or `"insight_github-{timestamp}"`
- Bitbucket: `"insight_bitbucket_server-2026-02-13-080000"` or `"insight_bitbucket_server-{timestamp}"`

**Settings JSON Example**:
```json
{
  "since": "2026-01-01",
  "until": "2026-02-01",
  "repositories": ["repo1", "repo2"],
  "collect_commits": true,
  "collect_prs": true,
  "collect_reviews": true,
  "collect_comments": true,
  "branches": "all",
  "force_refetch": false,
  "batch_size": 100
}
```

**Example Values**:
- `run_id`: "github-2026-02-13-120000"
- `started_at`: "2026-02-13 12:00:00.000"
- `completed_at`: "2026-02-13 12:15:30.000"
- `status`: "completed"
- `repos_processed`: 24
- `commits_collected`: 1036
- `prs_collected`: 432
- `api_calls`: 2500
- `errors`: 0
- `data_source`: "insight_github"

---

## Field Semantics

### Run Identification

**`run_id`** (String, REQUIRED)
- **Purpose**: Unique identifier for collection run
- **Format**: "{source}-{date}-{time}" or "{source}-{timestamp}"
- **GitHub**: "github-2026-02-13-120000"
- **Bitbucket**: "bitbucket-2026-02-13-080000"
- **Usage**: Tracking specific runs, debugging

### Timestamps

**`started_at`** (DateTime64(3), REQUIRED)
- **Purpose**: When collection run started
- **Format**: "2026-02-13 12:00:00.000"
- **Both sources**: Always populated
- **Usage**: Run duration calculation, timeline analysis

**`completed_at`** (DateTime64(3), NULLABLE)
- **Purpose**: When collection run completed
- **Format**: "2026-02-13 12:15:30.000"
- **Null**: If run is still in progress or failed before completion
- **Usage**: Run duration calculation, completion tracking

**Duration Calculation**:
```sql
SELECT 
    run_id,
    started_at,
    completed_at,
    EXTRACT(EPOCH FROM (completed_at - started_at)) / 60 as duration_minutes
FROM git_collection_runs
WHERE completed_at IS NOT NULL;
```

### Status

**`status`** (String, REQUIRED)
- **Purpose**: Current run status
- **Values**:
  - **"running"**: Collection in progress
  - **"completed"**: Successfully completed
  - **"failed"**: Failed with errors
- **Usage**: Monitoring, filtering, alerting

### Statistics

**`repos_processed`** (Int64, NULLABLE)
- **Purpose**: Number of repositories processed
- **GitHub**: 24
- **Bitbucket**: 150
- **Usage**: Progress tracking, capacity planning

**`commits_collected`** (Int64, NULLABLE)
- **Purpose**: Number of commits collected
- **GitHub**: 1036
- **Bitbucket**: 5000
- **Usage**: Data volume tracking, performance analysis

**`prs_collected`** (Int64, NULLABLE)
- **Purpose**: Number of pull requests collected
- **GitHub**: 432
- **Bitbucket**: 1200
- **Usage**: Data volume tracking, performance analysis

**`api_calls`** (Int64, NULLABLE)
- **Purpose**: Total API calls made during run
- **GitHub**: 2500
- **Bitbucket**: 8000
- **Usage**: Rate limit tracking, efficiency analysis

**`errors`** (Int64, NULLABLE)
- **Purpose**: Number of errors encountered
- **Values**: 0, 5, 10, etc.
- **Usage**: Error rate tracking, quality monitoring

### Configuration

**`settings`** (String, REQUIRED)
- **Purpose**: Collection configuration as JSON
- **Format**: JSON object with collection parameters
- **Always populated**: Contains run configuration
- **Usage**: Reproducing runs, debugging, auditing

**Settings Fields**:
- `since`: Start date for incremental collection
- `until`: End date (optional)
- `repositories`: List of repositories to process
- `collect_commits`: Whether to collect commits
- `collect_prs`: Whether to collect PRs
- `collect_reviews`: Whether to collect reviews
- `collect_comments`: Whether to collect comments
- `branches`: "all" or "default"
- `force_refetch`: Whether to force full refetch
- `batch_size`: Batch size for uploads

### System Fields

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version
- **Format**: Millisecond timestamp
- **Usage**: ReplacingMergeTree deduplication

**`data_source`** (String, DEFAULT '')
- **Purpose**: Source discriminator
- **Values**: "insight_github", "custom_etl", ""
- **Usage**: Filtering by source

---

## Usage Examples

### Query recent collection runs

```sql
SELECT 
    run_id,
    started_at,
    completed_at,
    status,
    repos_processed,
    commits_collected,
    prs_collected,
    api_calls,
    errors
FROM git_collection_runs
WHERE data_source = 'insight_github'
ORDER BY started_at DESC
LIMIT 10;
```

### Calculate run duration

```sql
SELECT 
    run_id,
    started_at,
    completed_at,
    EXTRACT(EPOCH FROM (completed_at - started_at)) / 60 as duration_minutes,
    status,
    repos_processed,
    commits_collected,
    prs_collected
FROM git_collection_runs
WHERE completed_at IS NOT NULL
  AND data_source = 'insight_github'
ORDER BY started_at DESC
LIMIT 20;
```

### Find failed runs

```sql
SELECT 
    run_id,
    started_at,
    completed_at,
    errors,
    settings
FROM git_collection_runs
WHERE status = 'failed'
  AND data_source = 'insight_github'
ORDER BY started_at DESC;
```

### Collection efficiency analysis

```sql
SELECT 
    run_id,
    repos_processed,
    commits_collected,
    prs_collected,
    api_calls,
    ROUND(commits_collected::FLOAT / NULLIF(api_calls, 0), 2) as commits_per_call,
    ROUND(prs_collected::FLOAT / NULLIF(api_calls, 0), 2) as prs_per_call,
    EXTRACT(EPOCH FROM (completed_at - started_at)) / 60 as duration_minutes
FROM bitbucket_collection_runs
WHERE status = 'completed'
  AND completed_at IS NOT NULL
  AND data_source = 'insight_github'
ORDER BY started_at DESC
LIMIT 20;
```

### Average collection statistics

```sql
SELECT 
    data_source,
    COUNT(*) as total_runs,
    AVG(repos_processed) as avg_repos,
    AVG(commits_collected) as avg_commits,
    AVG(prs_collected) as avg_prs,
    AVG(api_calls) as avg_api_calls,
    AVG(EXTRACT(EPOCH FROM (completed_at - started_at)) / 60) as avg_duration_minutes,
    SUM(CASE WHEN status = 'failed' THEN 1 ELSE 0 END) as failed_count
FROM bitbucket_collection_runs
WHERE started_at >= '2026-01-01'
  AND completed_at IS NOT NULL
GROUP BY data_source;
```

### Find long-running collections

```sql
SELECT 
    run_id,
    started_at,
    completed_at,
    EXTRACT(EPOCH FROM (completed_at - started_at)) / 60 as duration_minutes,
    repos_processed,
    commits_collected,
    prs_collected
FROM bitbucket_collection_runs
WHERE completed_at IS NOT NULL
  AND EXTRACT(EPOCH FROM (completed_at - started_at)) / 60 > 30
ORDER BY duration_minutes DESC
LIMIT 20;
```

### Extract settings for a run

```sql
SELECT 
    run_id,
    started_at,
    status,
    settings
FROM bitbucket_collection_runs
WHERE run_id = 'github-2026-02-13-120000';

-- Parse specific settings
SELECT 
    run_id,
    JSONExtractString(settings, 'since') as since_date,
    JSONExtractString(settings, 'branches') as branches,
    JSONExtractBool(settings, 'force_refetch') as force_refetch
FROM bitbucket_collection_runs
WHERE data_source = 'insight_github'
ORDER BY started_at DESC
LIMIT 10;
```

---

## Notes and Considerations

### Monitoring Use Cases

**Real-time Monitoring**:
- Track running collections
- Monitor progress
- Detect stuck runs

**Historical Analysis**:
- Collection performance trends
- API efficiency improvements
- Error rate tracking

**Capacity Planning**:
- Estimate collection times
- Plan API rate limits
- Resource allocation

### Status Transitions

**Normal Flow**:
1. Start: `status = "running"`
2. Complete: `status = "completed"`

**Error Flow**:
1. Start: `status = "running"`
2. Fail: `status = "failed"`

**Stuck Runs**:
- `status = "running"` but `started_at` is old
- May indicate crashed process
- Should be monitored and cleaned up

### API Efficiency

**GitHub GraphQL**:
- Higher commits/PRs per API call
- More efficient than REST
- Reflected in `api_calls` metric

**Bitbucket REST**:
- Lower efficiency (REST only)
- More API calls needed
- Higher `api_calls` for same data volume

**Efficiency Metrics**:
```sql
-- Commits per API call
commits_collected / api_calls

-- PRs per API call
prs_collected / api_calls

-- Items per minute
(commits_collected + prs_collected) / duration_minutes
```

### Error Tracking

**`errors` field**:
- Count of errors during collection
- Doesn't include error details
- For details, check application logs

**Error Types**:
- API rate limit errors
- Network timeouts
- Authentication failures
- Data parsing errors

**Error Rate**:
```sql
SELECT 
    AVG(errors::FLOAT / NULLIF(api_calls, 0)) * 100 as error_rate_percent
FROM bitbucket_collection_runs
WHERE status = 'completed';
```

### Settings Reproducibility

**Settings JSON** enables:
- Reproducing collection runs
- Understanding what was collected
- Debugging issues
- Auditing collection parameters

**Important Settings**:
- `since`: Affects incremental vs full collection
- `force_refetch`: Affects deduplication
- `branches`: Affects commit volume
- `repositories`: Affects scope

### Performance Optimization

**Track Metrics**:
- Duration per repository
- API calls per item
- Error rates
- Throughput (items/minute)

**Optimize Based On**:
- Use GraphQL when possible (GitHub)
- Increase batch sizes
- Parallelize collection
- Implement better caching

### Cleanup

**Old Runs**:
- Consider archiving or deleting old runs
- Keep recent runs for monitoring
- Retain failed runs for debugging

**Stuck Runs**:
- Identify runs with `status = "running"` and old `started_at`
- Update to `status = "failed"`
- Investigate cause

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Re-inserting same run doesn't create duplicates
- Latest version is kept
- Use `FINAL` modifier for guaranteed latest data
