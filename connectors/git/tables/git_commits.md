# Table: `git_commits`

## Overview

**Purpose**: Store commit history with file statistics from both Bitbucket and GitHub sources.

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
| `commit_hash` | String | REQUIRED | Git SHA-1 hash (40 characters) |
| `branch` | String | NULLABLE | Branch name where commit was found |
| `author_name` | String | REQUIRED | Commit author name |
| `author_email` | String | REQUIRED | Author email address |
| `committer_name` | String | REQUIRED | Committer name |
| `committer_email` | String | REQUIRED | Committer email address |
| `message` | String | REQUIRED | Commit message |
| `date` | DateTime64(3) | REQUIRED | Commit timestamp |
| `parents` | String | REQUIRED | JSON array of parent commit hashes |
| `files_changed` | Int64 | NULLABLE | Number of files modified |
| `lines_added` | Int64 | NULLABLE | Lines added |
| `lines_removed` | Int64 | NULLABLE | Lines removed |
| `language_breakdown` | String | NULLABLE | JSON object: {language: line_count} |
| `is_merge_commit` | Int64 | NULLABLE | 1 if merge commit (multiple parents) |
| `metadata` | String | REQUIRED | Full API response as JSON |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `ai_percentage` | Float32 | DEFAULT 0 | AI-generated code percentage (0.0-1.0) |
| `hash_sum` | String | NULLABLE | Deduplication hash (comma-separated) |
| `scancode_metadata` | String | NULLABLE | License/copyright scanning metadata |
| `ai_thirdparty_repos` | String | NULLABLE | Third-party repository detection |
| `ai_thirdparty_flag` | UInt8 | DEFAULT 0 | AI-detected third-party flag (0 or 1) |
| `scancode_thirdparty_flag` | UInt8 | DEFAULT 0 | Scancode-detected third-party (0 or 1) |
| `diffstat_metadata` | String | DEFAULT '' | JSON array of file changes with path details |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_commit_lookup`: `(project_key, repo_slug, commit_hash, data_source)`
- `idx_commit_date`: `(date)`

---

## Data Collection

### Bitbucket Source

**API Endpoints**:
1. Commit list: `/rest/api/1.0/projects/{project}/repos/{repo}/commits`
2. Commit details: `/rest/api/1.0/projects/{project}/repos/{repo}/commits/{hash}`
3. Commit diff: `/rest/api/1.0/projects/{project}/repos/{repo}/commits/{hash}/diff`

**Collection Process**:
1. Fetch commits from repository (paginated with `limit=100`)
2. For each commit, get detailed metadata
3. Optionally fetch diff for file-level statistics
4. Parse and store with `data_source = "insight_bitbucket_server"`

**API Response Structure**:
```json
{
  "id": "e7c304462a0e0123e2997fe9541a321cfe6595a5",
  "displayId": "e7c304462a0",
  "author": {
    "name": "John.Smith",
    "emailAddress": "john.smith@company.com",
    "displayName": "John Smith"
  },
  "authorTimestamp": 1763632375000,
  "committer": {
    "name": "John.Smith",
    "emailAddress": "John.Smith@company.com"
  },
  "committerTimestamp": 1763731118000,
  "message": "PLTFRM-84253 fix unstable test for alert manager",
  "parents": [{
    "id": "1056d1151d0615f63a6ab07bce5c4cd3ca0f0330",
    "displayId": "1056d1151d0",
    "author": {...},
    "authorTimestamp": 1763655688000
  }],
  "properties": {
    "jira-key": ["PLTFRM-84253"]
  }
}
```

**Field Mapping**:
- `commit_hash` ← `id` (full SHA-1)
- `author_name` ← `author.name` (dot-separated format: "John.Smith")
- `author_email` ← `author.emailAddress`
- `committer_name` ← `committer.name`
- `committer_email` ← `committer.emailAddress`
- `message` ← `message`
- `date` ← `authorTimestamp` / 1000 (convert milliseconds to datetime)
- `parents` ← JSON array with full parent objects (includes displayId, author, timestamps)
- `files_changed` ← count from diff API
- `lines_added` ← sum from diff API hunks
- `lines_removed` ← sum from diff API hunks
- `language_breakdown` ← null (not calculated for Bitbucket)
- `is_merge_commit` ← 1 if `parents.length > 1`, else 0
- `metadata` ← full JSON response
- `hash_sum` ← comma-separated hashes for deduplication

**Example Values**:
- `commit_hash`: "e7c304462a0e0123e2997fe9541a321cfe6595a5"
- `branch`: "release/25.12"
- `author_name`: "John.Smith"
- `date`: "2025-11-20 09:52:55.000"
- `files_changed`: 3
- `lines_added`: 42
- `lines_removed`: 36
- `data_source`: "insight_bitbucket_server"

---

### GitHub Source

**API Endpoints**:
1. REST: `/repos/{owner}/{repo}/commits`
2. REST details: `/repos/{owner}/{repo}/commits/{sha}`
3. GraphQL: `repository.ref.target.history` (bulk fetch - 100x faster)

**Collection Process**:
1. **With authentication (GraphQL)**:
   - Fetch 100 commits per API call with metadata
   - Includes additions, deletions, changedFiles
   - Much more efficient than REST
2. **Without authentication (REST)**:
   - Fetch commit list (no file details)
   - For each commit, fetch details (includes files)
   - 1 API call per commit for full details

**GraphQL Query**:
```graphql
query {
  repository(owner: $owner, name: $repo) {
    ref(qualifiedName: $branch) {
      target {
        ... on Commit {
          history(first: 100, since: $since) {
            nodes {
              oid
              message
              committedDate
              additions
              deletions
              changedFiles
              author { name email }
              committer { name email }
              parents(first: 10) { nodes { oid } }
            }
          }
        }
      }
    }
  }
}
```

**REST API Response**:
```json
{
  "sha": "1167e2a04964da044b891a766f2b57d7fb012d3f",
  "commit": {
    "author": {
      "name": "alice",
      "email": "alice@example.com",
      "date": "2026-02-10T20:24:45Z"
    },
    "committer": {
      "name": "GitHub",
      "email": "noreply@github.com",
      "date": "2026-02-10T20:24:45Z"
    },
    "message": "Add NOTICE file with copyright and license information"
  },
  "parents": [
    {"sha": "79262ab145e119a2c4be9c65d9e6d2af327893a1"}
  ],
  "files": [
    {
      "filename": "NOTICE",
      "additions": 15,
      "deletions": 0,
      "changes": 15,
      "status": "added"
    }
  ],
  "stats": {
    "additions": 15,
    "deletions": 0,
    "total": 15
  }
}
```

**Field Mapping**:
- `commit_hash` ← `sha` or `oid` (GraphQL)
- `author_name` ← `commit.author.name` or `author.name`
- `author_email` ← `commit.author.email` or `author.email`
- `committer_name` ← `commit.committer.name` (often "GitHub" for web commits)
- `committer_email` ← `commit.committer.email` (often "noreply@github.com")
- `message` ← `commit.message` or `message`
- `date` ← `commit.author.date` or `committedDate` (ISO 8601 → DateTime64)
- `parents` ← JSON array of SHA strings: `["79262ab..."]`
- `files_changed` ← `changedFiles` or `files.length`
- `lines_added` ← `additions` or `stats.additions`
- `lines_removed` ← `deletions` or `stats.deletions`
- `language_breakdown` ← calculated from file extensions: `{"Other": 15}`
- `is_merge_commit` ← 1 if `parents.length > 1`, else 0
- `metadata` ← full JSON response
- `hash_sum` ← null (not used for GitHub)

**Example Values**:
- `commit_hash`: "1167e2a04964da044b891a766f2b57d7fb012d3f"
- `branch`: "main"
- `author_name`: "alice"
- `committer_name`: "GitHub"
- `committer_email`: "noreply@github.com"
- `date`: "2026-02-10 20:24:45.000"
- `files_changed`: 15
- `lines_added`: 15
- `lines_removed`: 0
- `language_breakdown`: `{"Other": 15}`
- `data_source`: "insight_github"

---

## Field Semantics

### Core Identifiers

**`commit_hash`** (String, REQUIRED)
- **Purpose**: Unique Git SHA-1 hash
- **Format**: 40-character hexadecimal string
- **Both sources**: Standard Git hash
- **Example**: "1167e2a04964da044b891a766f2b57d7fb012d3f"
- **Usage**: Primary identifier for commits, join key

**`branch`** (String, NULLABLE)
- **Purpose**: Branch name where commit was found
- **GitHub**: "main", "develop", "feature/xyz"
- **Bitbucket**: "release/25.12", "master"
- **Note**: Same commit can appear on multiple branches
- **Usage**: Filtering, branch-specific analysis

### Author Information

**`author_name`** (String, REQUIRED)
- **Purpose**: Commit author's name
- **GitHub**: Username or full name (e.g., "alice")
- **Bitbucket**: Dot-separated format (e.g., "John.Smith")
- **Usage**: Developer attribution, statistics

**`author_email`** (String, REQUIRED)
- **Purpose**: Commit author's email
- **GitHub**: "alice@example.com"
- **Bitbucket**: Corporate email (e.g., "John.Smith@company.com")
- **Usage**: Developer identification, deduplication

**`committer_name`** (String, REQUIRED)
- **Purpose**: Person who committed the change
- **GitHub**: Often "GitHub" for web-based commits
- **Bitbucket**: Usually same as author
- **Note**: Different from author for cherry-picks, merges

**`committer_email`** (String, REQUIRED)
- **Purpose**: Committer's email
- **GitHub**: Often "noreply@github.com" for web commits
- **Bitbucket**: Corporate email
- **Usage**: Tracking commit source (web vs local)

### Commit Content

**`message`** (String, REQUIRED)
- **Purpose**: Commit message text
- **GitHub**: "Add NOTICE file with copyright..."
- **Bitbucket**: "PLTFRM-84253 fix unstable test..."
- **Usage**: Search, Jira ticket extraction, understanding changes
- **Note**: May contain multiple lines

**`date`** (DateTime64(3), REQUIRED)
- **Purpose**: Commit timestamp
- **Format**: "2026-02-10 20:24:45.000"
- **GitHub**: Converted from ISO 8601 string
- **Bitbucket**: Converted from milliseconds since epoch
- **Usage**: Time-series analysis, filtering, ordering

**`parents`** (String, REQUIRED)
- **Purpose**: Parent commit references
- **GitHub**: JSON array of SHA strings: `["79262ab145e119a2c4be9c65d9e6d2af327893a1"]`
- **Bitbucket**: JSON array of full objects with metadata
- **Format difference**: Bitbucket includes displayId, author, timestamps
- **Usage**: Commit graph traversal, merge detection

### Statistics

**`files_changed`** (Int64, NULLABLE)
- **Purpose**: Number of files modified in commit
- **GitHub**: 15
- **Bitbucket**: 3
- **Usage**: Commit size metrics, filtering large commits

**`lines_added`** (Int64, NULLABLE)
- **Purpose**: Total lines added
- **GitHub**: 15
- **Bitbucket**: 42
- **Usage**: Code churn metrics, developer productivity

**`lines_removed`** (Int64, NULLABLE)
- **Purpose**: Total lines removed
- **GitHub**: 0
- **Bitbucket**: 36
- **Usage**: Code churn metrics, refactoring detection

**`language_breakdown`** (String, NULLABLE)
- **Purpose**: Lines changed per programming language
- **Format**: JSON object `{"Java": 150, "Python": 80, "XML": 15}`
- **GitHub**: Calculated from file extensions: `{"Other": 15}`
- **Bitbucket**: null (not calculated)
- **Optional**: GitHub only
- **Usage**: Language-specific metrics

**`is_merge_commit`** (Int64, NULLABLE)
- **Purpose**: Flag for merge commits
- **Values**: 1 = merge (multiple parents), 0 = regular commit
- **Both sources**: Calculated from parents array length
- **Usage**: Filtering merge commits, commit type analysis

### Advanced Fields

**`ai_percentage`** (Float32, DEFAULT 0)
- **Purpose**: Percentage of AI-generated code
- **Range**: 0.0 to 1.0
- **Default**: 0
- **Usage**: AI code detection, quality metrics

**`hash_sum`** (String, NULLABLE)
- **Purpose**: Deduplication hash
- **GitHub**: null (not used)
- **Bitbucket**: Comma-separated hashes (e.g., "221d85917479ba90...,5d832ba77bb30...")
- **Optional**: Bitbucket only
- **Usage**: Detecting duplicate commits

**`scancode_metadata`** (String, NULLABLE)
- **Purpose**: License and copyright scanning results
- **Format**: JSON array with license detections
- **Both sources**: null or populated
- **Usage**: License compliance, copyright tracking

**`ai_thirdparty_repos`** (String, NULLABLE)
- **Purpose**: Third-party repository detection
- **Both sources**: null in current data
- **Usage**: Dependency tracking

**`ai_thirdparty_flag`** (UInt8, DEFAULT 0)
- **Purpose**: AI-detected third-party code flag
- **Values**: 0 or 1
- **Usage**: Third-party code identification

**`scancode_thirdparty_flag`** (UInt8, DEFAULT 0)
- **Purpose**: Scancode-detected third-party flag
- **Values**: 0 or 1
- **Usage**: License-based third-party detection

### System Fields

**`metadata`** (String, REQUIRED)
- **Purpose**: Complete API response as JSON
- **Both sources**: Always populated
- **Usage**: Debugging, platform-specific fields, audit trail

**`collected_at`** (DateTime64(3), REQUIRED)
- **Purpose**: When commit was collected
- **Format**: "2026-02-13 12:00:32.923"
- **Usage**: Collection tracking, data freshness

**`data_source`** (String, DEFAULT '')
- **Purpose**: Source discriminator
- **Values**: "airnd_github", "dev_metrics", or ""
- **Usage**: Filtering by source

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version
- **Format**: Millisecond timestamp (e.g., 1770984032928)
- **Usage**: ReplacingMergeTree deduplication

---

## Relationships

### Parent

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many commits to one repository
- **Description**: All commits belong to a repository

### Children

**`git_commit_files`**
- **Join**: `commit_hash` → `commit_hash`
- **Cardinality**: One commit to many files
- **Description**: File-level details for each commit

**`git_pr_commits`**
- **Join**: `commit_hash` → `commit_hash`
- **Cardinality**: One commit to many PR associations
- **Description**: Links commits to pull requests

**`git_tickets`**
- **Join**: `commit_hash` → `commit_hash`
- **Cardinality**: One commit to many tickets
- **Description**: Jira tickets extracted from commit messages

---

## Collection Logic

### Incremental Collection

**Strategy**:
1. Query `max(date)` from ClickHouse per repository
2. Fetch commits with `since` parameter (GitHub) or filter client-side (Bitbucket)
3. Stop pagination when reaching already-collected commits
4. Update `last_commit_date` in repositories table

**GitHub Optimization**:
- Use GraphQL for bulk fetching (100 commits per call)
- Include file statistics in single query
- 100x faster than REST API

**Bitbucket Optimization**:
- Use `limit=100` for maximum page size
- Early stopping when `since` date reached
- Separate diff API calls only when needed

### Branch Collection

**GitHub**:
- Supports collecting from all branches or default only
- Tracks `last_commit_hash` per branch
- Avoids duplicates using `seen_commits` set
- Always re-collects default branch commits

**Bitbucket**:
- Typically collects from default branch only
- Can be extended for multi-branch support

---

## Usage Examples

### Query recent commits

```sql
SELECT 
    project_key,
    repo_slug,
    commit_hash,
    author_name,
    message,
    date,
    files_changed,
    lines_added,
    lines_removed
FROM git_commits
WHERE date > now() - INTERVAL 7 DAY
  AND data_source = 'insight_github'
ORDER BY date DESC
LIMIT 100;
```

### Find merge commits

```sql
SELECT 
    commit_hash,
    author_name,
    message,
    date,
    parents
FROM git_commits
WHERE is_merge_commit = 1
  AND project_key = 'GlobalTypeSystem'
  AND repo_slug = 'gts-go'
ORDER BY date DESC;
```

### Developer statistics

```sql
SELECT 
    author_name,
    COUNT(*) as commit_count,
    SUM(files_changed) as total_files,
    SUM(lines_added) as total_added,
    SUM(lines_removed) as total_removed
FROM git_commits
WHERE date >= '2026-01-01'
  AND data_source = 'insight_github'
GROUP BY author_name
ORDER BY commit_count DESC
LIMIT 20;
```

### Language breakdown (GitHub only)

```sql
SELECT 
    project_key,
    repo_slug,
    language_breakdown
FROM git_commits
WHERE language_breakdown IS NOT NULL
  AND language_breakdown != ''
  AND data_source = 'insight_github'
LIMIT 10;
```

---

## Notes and Considerations

### Timestamp Formats

**Critical difference**:
- **Bitbucket**: Milliseconds since epoch (1763632375000)
- **GitHub**: ISO 8601 string ("2026-02-10T20:24:45Z")

Both are converted to DateTime64(3) format: "2026-02-10 20:24:45.000"

### Parents Field Structure

**GitHub**: Simple array of SHA strings
```json
["79262ab145e119a2c4be9c65d9e6d2af327893a1"]
```

**Bitbucket**: Full objects with metadata
```json
[{
  "id": "1056d1151d0615f63a6ab07bce5c4cd3ca0f0330",
  "displayId": "1056d1151d0",
  "author": {...},
  "authorTimestamp": 1763655688000
}]
```

### Performance Considerations

**GraphQL vs REST**:
- GitHub GraphQL: 100 commits per call with full metadata
- GitHub REST: 1 call per commit for file details
- **100x performance improvement** with GraphQL

**Indexes**:
- Use `(project_key, repo_slug, commit_hash, data_source)` for lookups
- Use `(date)` index for time-range queries

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Same commit can be re-collected without duplicates
- Latest version is kept automatically
- Use `FINAL` modifier for guaranteed latest data
