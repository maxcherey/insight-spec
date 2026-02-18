# Table: `git_repository_branches`

## Overview

**Purpose**: Track branch state for incremental collection, storing last collected commit per branch to avoid re-fetching.

**Data Sources**: 
- Bitbucket: `data_source = "insight_bitbucket_server"`
- GitHub: `data_source = "insight_github"`
- GitLab: `data_source = "insight_gitlab"`
- CustomGit: `data_source = "custom_etl"`

---

## Schema Definition

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | Int64 | PRIMARY KEY | Auto-generated unique identifier |
| `project_key` | String | REQUIRED | Repository owner |
| `repo_slug` | String | REQUIRED | Repository name |
| `branch_name` | String | REQUIRED | Branch name |
| `is_default` | Int64 | REQUIRED | 1 if default branch, 0 otherwise |
| `last_commit_hash` | String | NULLABLE | Last commit collected from this branch |
| `last_commit_date` | DateTime64(3) | NULLABLE | Date of last commit |
| `last_checked_at` | String | NULLABLE | Last time this branch was checked |
| `metadata` | String | NULLABLE | Branch metadata as JSON |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_branch_lookup`: `(project_key, repo_slug, branch_name, data_source)`

---

## Data Collection

### Bitbucket Source

**API Endpoint**: `/rest/api/1.0/projects/{project}/repos/{repo}/branches`

**Collection Process**:
1. Fetch all branches for repository
2. Identify default branch
3. Track last collected commit per branch
4. Store with `data_source = "insight_bitbucket_server"`

**API Response Structure**:
```json
{
  "values": [
    {
      "id": "refs/heads/master",
      "displayId": "master",
      "type": "BRANCH",
      "latestCommit": "a749767deeac85d702fe97bcfab57ea1797e7511",
      "latestChangeset": "a749767deeac85d702fe97bcfab57ea1797e7511",
      "isDefault": true
    }
  ]
}
```

**Field Mapping**:
- `branch_name` ← `displayId` (e.g., "master", "develop")
- `is_default` ← 1 if `isDefault` is true, else 0
- `last_commit_hash` ← `latestCommit` or tracked during collection
- `last_checked_at` ← current timestamp in milliseconds

**Example Values**:
- `project_key`: "TEST"
- `repo_slug`: "test-repo"
- `branch_name`: "master"
- `is_default`: 1
- `last_commit_hash`: null (not populated in current data)
- `last_commit_date`: null
- `last_checked_at`: "1767454894233" (milliseconds)
- `data_source`: "insight_bitbucket_server"

---

### GitHub Source

**API Endpoint**: `/repos/{owner}/{repo}/branches`

**Collection Process**:
1. Fetch all branches (if multi-branch collection enabled)
2. Identify default branch from repository metadata
3. Track last collected commit per branch
4. Store with `data_source = "insight_github"`

**API Response Structure**:
```json
{
  "name": "main",
  "commit": {
    "sha": "e6c0dad65ca9fb6c7858377e8740cf608d15a0cf",
    "url": "https://api.github.com/repos/owner/repo/commits/e6c0dad..."
  },
  "protected": false
}
```

**Field Mapping**:
- `branch_name` ← `name` (e.g., "main", "develop")
- `is_default` ← 1 if matches repository default branch, else 0
- `last_commit_hash` ← tracked during collection
- `last_commit_date` ← tracked during collection
- `last_checked_at` ← current timestamp

**Example Values**:
- `project_key`: "GlobalTypeSystem"
- `repo_slug`: "gts-go"
- `branch_name`: "main"
- `is_default`: 1
- `last_commit_hash`: "e6c0dad65ca9fb6c7858377e8740cf608d15a0cf"
- `last_commit_date`: "2026-02-10 20:24:45.000"
- `last_checked_at`: "2026-02-13T12:00:00Z"
- `data_source`: "insight_github"

---

## Field Semantics

### Core Identifiers

**`project_key`** (String, REQUIRED)
- **Purpose**: Repository owner
- **Bitbucket**: "FIVENINE"
- **GitHub**: "GlobalTypeSystem"
- **Usage**: Part of composite key

**`repo_slug`** (String, REQUIRED)
- **Purpose**: Repository name
- **Bitbucket**: "test-repo"
- **GitHub**: "gts-go"
- **Usage**: Part of composite key

**`branch_name`** (String, REQUIRED)
- **Purpose**: Branch name
- **Bitbucket**: "master", "develop", "release/1.0"
- **GitHub**: "main", "develop", "feature/xyz"
- **Usage**: Part of composite key, filtering

### Branch Information

**`is_default`** (Int64, REQUIRED)
- **Purpose**: Flag for default branch
- **Values**: 1 = default branch, 0 = other branch
- **Bitbucket**: 1 for "master" typically
- **GitHub**: 1 for "main" or "master" typically
- **Usage**: Identifying primary branch, prioritization

### Tracking Fields

**`last_commit_hash`** (String, NULLABLE)
- **Purpose**: Last commit collected from this branch
- **Format**: 40-character SHA-1 hash
- **Bitbucket**: null (not populated in current data)
- **GitHub**: "e6c0dad65ca9fb6c7858377e8740cf608d15a0cf"
- **Usage**: Incremental collection starting point

**`last_commit_date`** (DateTime64(3), NULLABLE)
- **Purpose**: Date of last collected commit
- **Format**: "2026-02-10 20:24:45.000"
- **Bitbucket**: null (not populated)
- **GitHub**: Populated when tracking
- **Usage**: Time-based incremental collection

**`last_checked_at`** (String, NULLABLE)
- **Purpose**: Last time branch was checked
- **Format**: Varies - milliseconds or ISO string
- **Bitbucket**: "1767454894233" (milliseconds)
- **GitHub**: "2026-02-13T12:00:00Z" (ISO 8601)
- **Usage**: Tracking check frequency

### Metadata

**`metadata`** (String, NULLABLE)
- **Purpose**: Branch metadata as JSON
- **Bitbucket**: null (not populated)
- **GitHub**: May contain branch protection rules, etc.
- **Usage**: Additional branch information

### System Fields

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version
- **Format**: Millisecond timestamp (e.g., 1767636246909)
- **Usage**: ReplacingMergeTree deduplication

**`data_source`** (String, DEFAULT '')
- **Purpose**: Source discriminator
- **Values**: "insight_github", "dev_metrics", ""
- **Usage**: Filtering by source

---

## Relationships

### Parent

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many branches to one repository
- **Description**: All branches belong to a repository

---

## Usage Examples

### Query branches for a repository

```sql
SELECT 
    branch_name,
    is_default,
    last_commit_hash,
    last_commit_date,
    last_checked_at
FROM git_repository_branches
WHERE project_key = 'GlobalTypeSystem'
  AND repo_slug = 'gts-go'
  AND data_source = 'insight_github'
ORDER BY is_default DESC, branch_name ASC;
```

### Find default branches

```sql
SELECT 
    project_key,
    repo_slug,
    branch_name,
    last_commit_date
FROM git_repository_branches
WHERE is_default = 1
  AND data_source = 'insight_github'
ORDER BY last_commit_date DESC NULLS LAST;
```

### Find branches needing update

```sql
-- Branches not checked recently
SELECT 
    project_key,
    repo_slug,
    branch_name,
    last_checked_at,
    last_commit_date
FROM git_repository_branches
WHERE data_source = 'insight_github'
  AND (
    last_checked_at IS NULL 
    OR last_checked_at < '2026-02-01'
  )
ORDER BY last_checked_at ASC NULLS FIRST
LIMIT 100;
```

### Branch statistics per repository

```sql
SELECT 
    project_key,
    repo_slug,
    COUNT(*) as total_branches,
    SUM(is_default) as default_branches,
    COUNT(CASE WHEN last_commit_hash IS NOT NULL THEN 1 END) as tracked_branches
FROM git_repository_branches
WHERE data_source = 'insight_github'
GROUP BY project_key, repo_slug
ORDER BY total_branches DESC;
```

### Find stale branches

```sql
SELECT 
    project_key,
    repo_slug,
    branch_name,
    last_commit_date,
    EXTRACT(DAY FROM (NOW() - last_commit_date)) as days_since_commit
FROM git_repository_branches
WHERE last_commit_date IS NOT NULL
  AND last_commit_date < NOW() - INTERVAL 90 DAY
  AND data_source = 'insight_github'
ORDER BY last_commit_date ASC
LIMIT 100;
```

---

## Usage in Incremental Collection

### Algorithm

**For each branch**:
1. Query `last_commit_hash` from this table
2. Fetch commits from API with `since = last_commit_hash`
3. Process new commits
4. Update `last_commit_hash` and `last_commit_date`
5. Update `last_checked_at`

**Benefits**:
- Avoid re-fetching old commits
- Reduce API calls
- Faster collection
- Lower resource usage

### GitHub Multi-Branch Collection

**When enabled**:
1. Fetch all branches from API
2. For each branch:
   - Check `last_commit_hash` in table
   - Fetch new commits since last hash
   - Update tracking fields
3. Avoid duplicate commits across branches using `seen_commits` set

**When disabled** (default branch only):
1. Only track default branch
2. Simpler, faster
3. May miss commits on feature branches

### Bitbucket Collection

**Current Implementation**:
- Typically collects from default branch only
- Branch tracking table populated but not fully utilized
- Can be extended for multi-branch support

---

## Notes and Considerations

### Data Completeness

**Current State**:
- **Bitbucket**: Table populated but tracking fields often null
- **GitHub**: Fully populated when multi-branch collection enabled

**Tracking Fields**:
- `last_commit_hash`: Critical for incremental collection
- `last_commit_date`: Useful for time-based filtering
- `last_checked_at`: Useful for monitoring

### Timestamp Format Inconsistency

**`last_checked_at` format varies**:
- **Bitbucket**: Milliseconds since epoch ("1767454894233")
- **GitHub**: ISO 8601 string ("2026-02-13T12:00:00Z")

**Recommendation**: Normalize to consistent format (DateTime64(3)).

### Default Branch Detection

**Bitbucket**:
- `isDefault` flag in API response
- Usually "master"

**GitHub**:
- `default_branch` field in repository metadata
- Usually "main" or "master"

**Important**: Always check `is_default` flag, don't assume branch name.

### Branch Lifecycle

**New Branches**:
- Created when first seen in API
- `last_commit_hash` is null initially
- First collection fetches all commits

**Updated Branches**:
- `last_commit_hash` updated after each collection
- Enables incremental collection

**Deleted Branches**:
- Not automatically removed from table
- May need cleanup process

### Performance

**Index Usage**:
- Primary index on `(project_key, repo_slug, branch_name, data_source)`
- Always include these fields in WHERE clauses

**Optimization**:
- Query this table before fetching commits
- Use `last_commit_hash` for incremental collection
- Reduces API calls significantly

### Multi-Branch vs Single-Branch

**Multi-Branch Collection**:
- **Pros**: Complete commit history, all branches tracked
- **Cons**: More API calls, more storage, duplicate commits
- **Use case**: Comprehensive analysis

**Single-Branch Collection** (default only):
- **Pros**: Faster, fewer API calls, simpler
- **Cons**: May miss feature branch commits
- **Use case**: Main branch tracking

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Re-collecting branches doesn't create duplicates
- Latest branch state is kept
- Use `FINAL` modifier for guaranteed latest data

### Future Enhancements

**Better Tracking**:
- Populate all tracking fields consistently
- Normalize timestamp formats
- Track branch protection rules
- Track branch creation/deletion dates

**Cleanup**:
- Detect and remove deleted branches
- Archive stale branches
- Optimize storage
