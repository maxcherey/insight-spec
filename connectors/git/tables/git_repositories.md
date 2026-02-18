# Table: `git_repositories`

## Overview

**Purpose**: Store repository metadata from both Bitbucket and GitHub sources.

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
| `project_key` | String | REQUIRED | Organization/workspace name |
| `repo_slug` | String | REQUIRED | Repository name |
| `repo_uuid` | String | NULLABLE | Unique repository identifier from API |
| `name` | String | REQUIRED | Repository display name |
| `full_name` | String | NULLABLE | Full repository path (owner/repo) |
| `description` | String | NULLABLE | Repository description |
| `is_private` | Int64 | NULLABLE | 1 if private, 0 if public |
| `created_on` | DateTime64(3) | NULLABLE | Repository creation date |
| `updated_on` | DateTime64(3) | NULLABLE | Last repository update |
| `size` | Int64 | NULLABLE | Repository size in KB |
| `language` | String | NULLABLE | Primary programming language |
| `has_issues` | Int64 | NULLABLE | 1 if issues enabled |
| `has_wiki` | Int64 | NULLABLE | 1 if wiki enabled |
| `fork_policy` | String | NULLABLE | Fork policy (Bitbucket only) |
| `metadata` | String | REQUIRED | Full API response as JSON |
| `first_seen` | DateTime64(3) | NULLABLE | First time seen in our system |
| `last_updated` | DateTime64(3) | NULLABLE | Last time updated in our system |
| `last_commit_date` | DateTime64(3) | NULLABLE | Date of most recent commit |
| `last_commit_date_first_seen` | DateTime64(3) | NULLABLE | First time last_commit_date was seen |
| `last_commit_date_last_checked` | DateTime64(3) | NULLABLE | Last time last_commit_date was checked |
| `is_empty` | Int64 | NULLABLE | 1 if repository has no commits |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Version for deduplication (millisecond timestamp) |

**Indexes**:
- `idx_repo_lookup`: `(project_key, repo_slug, data_source)`

---

## Data Collection

### Bitbucket Source

**API Endpoint**: `/rest/api/1.0/projects/{project}/repos/{repo}`

**Collection Process**:
1. Fetch all projects: `/rest/api/1.0/projects`
2. For each project, fetch repositories: `/rest/api/1.0/projects/{project}/repos`
3. Parse repository metadata from response
4. Store with `data_source = ""` or `"dev_metrics"`

**API Response Structure**:
```json
{
  "slug": "test-core",
  "id": 368,
  "name": "test-core",
  "hierarchyId": "9c8e45623f231cadb3d2",
  "scmId": "git",
  "state": "AVAILABLE",
  "forkable": true,
  "project": {
    "key": "TEST",
    "id": 24,
    "name": "Test Project",
    "public": false
  },
  "public": false,
  "archived": false,
  "links": {
    "clone": [{"href": "ssh://git@git.acronis.com:7989/test/abl-core.git"}],
    "self": [{"href": "https://git.acronis.work/projects/TEST/repos/test-core/browse"}]
  }
}
```

**Field Mapping**:
- `project_key` ← `project.key` (e.g., "TEST")
- `repo_slug` ← `slug` (e.g., "test-core")
- `repo_uuid` ← `id` as string (e.g., "368") or null
- `name` ← `name`
- `full_name` ← null (not populated for Bitbucket)
- `description` ← `description` (often empty or null)
- `is_private` ← derived from `!public` (null or 1)
- `created_on` ← null (not available in Bitbucket Server API)
- `updated_on` ← null (not available in Bitbucket Server API)
- `size` ← null (not available in Bitbucket Server API)
- `language` ← null (not available in Bitbucket Server API)
- `has_issues` ← null (not available in Bitbucket Server API)
- `has_wiki` ← null (not available in Bitbucket Server API)
- `fork_policy` ← null (available in metadata as `forkable: true`)
- `metadata` ← full JSON response
- `is_empty` ← 0 or 1 based on commit count

**Example Values**:
- `project_key`: "TEST"
- `repo_slug`: "test-core"
- `repo_uuid`: "368", null
- `data_source`: "", "dev_metrics"

---

### GitHub Source

**API Endpoint**: `/repos/{owner}/{repo}`

**Collection Process**:
1. Fetch organization repositories: `/orgs/{org}/repos`
2. Or fetch user repositories: `/users/{user}/repos`
3. For each repository, get details: `/repos/{owner}/{repo}`
4. Store with `data_source = "airnd_github"`

**API Response Structure**:
```json
{
  "id": 1067370324,
  "node_id": "R_kgDOP57HVA",
  "name": "gts-go",
  "full_name": "GlobalTypeSystem/gts-go",
  "private": false,
  "owner": {
    "login": "GlobalTypeSystem",
    "id": 235357302
  },
  "description": "Go-lang implementation of GTS library",
  "created_at": "2025-09-30T19:02:49Z",
  "updated_at": "2026-02-10T20:24:50Z",
  "size": 193,
  "language": "Go",
  "has_issues": true,
  "has_wiki": false
}
```

**Field Mapping**:
- `project_key` ← `owner.login` (e.g., "GlobalTypeSystem")
- `repo_slug` ← `name` (e.g., "gts-go")
- `repo_uuid` ← `id` as string (e.g., "1067370324")
- `name` ← `name`
- `full_name` ← `full_name` (e.g., "GlobalTypeSystem/gts-go")
- `description` ← `description`
- `is_private` ← `private` (0 or 1)
- `created_on` ← `created_at` converted to DateTime64(3)
- `updated_on` ← `updated_at` converted to DateTime64(3)
- `size` ← `size` (in KB)
- `language` ← `language` (e.g., "Go", "Python")
- `has_issues` ← `has_issues` (0 or 1)
- `has_wiki` ← `has_wiki` (0 or 1)
- `fork_policy` ← null (always null for GitHub)
- `metadata` ← full JSON response
- `is_empty` ← null (not calculated for GitHub)

**Example Values**:
- `project_key`: "GlobalTypeSystem"
- `repo_slug`: "gts-go"
- `repo_uuid`: "1067370324"
- `full_name`: "GlobalTypeSystem/gts-go"
- `description`: "Go-lang implementation of GTS library"
- `size`: 193
- `language`: "Go"
- `data_source`: "airnd_github"

---

## Field Semantics

### Core Identifiers

**`project_key`** (String, REQUIRED)
- **Purpose**: Organization or workspace name
- **Bitbucket**: Project key from Bitbucket (e.g., "TEST")
- **GitHub**: Owner login name (e.g., "GlobalTypeSystem")
- **Usage**: Part of composite key for repository lookup
- **Format**: Alphanumeric, may contain hyphens or underscores
- **Case**: Bitbucket can be uppercase or lowercase, GitHub preserves case

**`repo_slug`** (String, REQUIRED)
- **Purpose**: Repository name/identifier
- **Bitbucket**: Repository slug (e.g., "test-core")
- **GitHub**: Repository name (e.g., "gts-go")
- **Usage**: Part of composite key for repository lookup
- **Format**: Alphanumeric with hyphens, underscores, dots
- **Case**: Lowercase or mixed case

**`repo_uuid`** (String, NULLABLE)
- **Purpose**: Unique identifier from source API
- **Bitbucket**: Repository ID as string (e.g., "368"), often null
- **GitHub**: Repository ID as string (e.g., "1067370324"), always populated
- **Usage**: Alternative unique identifier, useful for API calls
- **Note**: Not reliable for Bitbucket (often null)

### Repository Metadata

**`name`** (String, REQUIRED)
- **Purpose**: Display name for the repository
- **Both sources**: Usually same as `repo_slug`
- **Example**: "gts-go", "abl-core"

**`full_name`** (String, NULLABLE)
- **Purpose**: Full repository path including owner
- **GitHub**: Always populated (e.g., "GlobalTypeSystem/gts-go")
- **Bitbucket**: Always null (not constructed)
- **Usage**: Display purposes, GitHub API references
- **Optional**: GitHub only

**`description`** (String, NULLABLE)
- **Purpose**: Repository description text
- **Both sources**: Can be empty string or null
- **Example**: "Go-lang implementation of GTS library"
- **Usage**: Display, search, categorization

**`is_private`** (Int64, NULLABLE)
- **Purpose**: Privacy flag
- **Values**: 1 = private, 0 = public, null = unknown
- **GitHub**: Always 0 or 1
- **Bitbucket**: Often null, sometimes 1
- **Usage**: Access control, filtering

### GitHub-Only Fields

**`created_on`** (DateTime64(3), NULLABLE)
- **Purpose**: Repository creation timestamp
- **Format**: "2025-09-30 19:02:49.000"
- **GitHub**: Populated from `created_at` (ISO 8601)
- **Bitbucket**: Always null (not available in API)
- **Optional**: GitHub only

**`updated_on`** (DateTime64(3), NULLABLE)
- **Purpose**: Last repository update timestamp
- **Format**: "2026-02-10 20:24:50.000"
- **GitHub**: Populated from `updated_at` (ISO 8601)
- **Bitbucket**: Always null (not available in API)
- **Optional**: GitHub only

**`size`** (Int64, NULLABLE)
- **Purpose**: Repository size in kilobytes
- **GitHub**: Populated (e.g., 193)
- **Bitbucket**: Always null (not available in API)
- **Optional**: GitHub only

**`language`** (String, NULLABLE)
- **Purpose**: Primary programming language
- **GitHub**: Populated (e.g., "Go", "Python", "JavaScript")
- **Bitbucket**: Always null (not available in API)
- **Optional**: GitHub only
- **Usage**: Categorization, statistics

**`has_issues`** (Int64, NULLABLE)
- **Purpose**: Whether issue tracking is enabled
- **Values**: 1 = enabled, 0 = disabled
- **GitHub**: Populated (0 or 1)
- **Bitbucket**: Always null (not available in API)
- **Optional**: GitHub only

**`has_wiki`** (Int64, NULLABLE)
- **Purpose**: Whether wiki is enabled
- **Values**: 1 = enabled, 0 = disabled
- **GitHub**: Populated (0 or 1)
- **Bitbucket**: Always null (not available in API)
- **Optional**: GitHub only

### Bitbucket-Only Fields

**`fork_policy`** (String, NULLABLE)
- **Purpose**: Fork policy setting
- **Bitbucket**: Available in metadata as `forkable: true/false`, but not extracted to this field (null)
- **GitHub**: Always null (not applicable)
- **Optional**: Bitbucket only (not currently extracted)

### System Fields

**`metadata`** (String, REQUIRED)
- **Purpose**: Complete API response as JSON string
- **Both sources**: Always populated
- **Usage**: Debugging, accessing platform-specific fields, audit trail
- **Size**: Can be large (several KB)

**`first_seen`** (DateTime64(3), NULLABLE)
- **Purpose**: First time this repository was collected
- **Format**: "2026-02-13 12:00:31.642"
- **Both sources**: Set on first insert
- **Usage**: Tracking, analytics

**`last_updated`** (DateTime64(3), NULLABLE)
- **Purpose**: Last time this repository record was updated
- **Format**: "2026-02-13 12:00:31.642"
- **Both sources**: Updated on each collection
- **Usage**: Incremental collection, change tracking

**`last_commit_date`** (DateTime64(3), NULLABLE)
- **Purpose**: Date of most recent commit in repository
- **Format**: "2024-11-14 14:58:00.000"
- **Bitbucket**: Populated from commit collection
- **GitHub**: Usually null
- **Usage**: Repository activity tracking

**`last_commit_date_first_seen`** (DateTime64(3), NULLABLE)
- **Purpose**: First time the last_commit_date value was observed
- **Format**: "2025-12-31 05:09:40.075"
- **Usage**: Change detection

**`last_commit_date_last_checked`** (DateTime64(3), NULLABLE)
- **Purpose**: Last time last_commit_date was checked/updated
- **Format**: "2026-01-03 15:41:44.001"
- **Usage**: Incremental collection optimization

**`is_empty`** (Int64, NULLABLE)
- **Purpose**: Flag indicating repository has no commits
- **Values**: 1 = empty, 0 = has commits, null = unknown
- **Bitbucket**: Populated (0 or 1)
- **GitHub**: Usually null
- **Usage**: Filtering, validation

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version number
- **Format**: Millisecond timestamp (e.g., 1770984031642)
- **Both sources**: Auto-generated on insert
- **Usage**: ReplacingMergeTree deduplication, keeps latest version

**`data_source`** (String, DEFAULT '')
- **Purpose**: Source discriminator
- **Values**: 
  - GitHub: "airnd_github"
  - Bitbucket: "" (empty string) or "dev_metrics"
- **Note**: Bitbucket does NOT use "bitbucket" as originally designed
- **Usage**: Filtering by source, source-specific logic

---

## Relationships

### Parent Of

**`git_commits`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many commits
- **Description**: All commits belong to a repository

**`git_commit_files`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many commit files
- **Description**: All commit file changes belong to a repository

**`git_pull_requests`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many pull requests
- **Description**: All pull requests belong to a repository

**`git_pull_requests_reviewers`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many reviewers (via PRs)
- **Description**: All PR reviewers are associated with a repository

**`git_pr_comments`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many comments (via PRs)
- **Description**: All PR comments are associated with a repository

**`git_pr_commits`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many PR-commit links
- **Description**: Links between PRs and commits in a repository

**`git_pr_participants`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many participants (via PRs)
- **Description**: All PR participants are associated with a repository

**`git_tickets`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many Jira tickets
- **Description**: Jira tickets extracted from PRs and commits in a repository

**`git_repository_branches`**
- **Join**: `(project_key, repo_slug)` → `(project_key, repo_slug)`
- **Cardinality**: One repository to many branches
- **Description**: Branch tracking for incremental collection

---

## Usage Examples

### Query repositories by source

```sql
-- GitHub repositories only
SELECT project_key, repo_slug, full_name, language, size
FROM git_repositories
WHERE data_source = 'insight_github'
ORDER BY size DESC
LIMIT 10;

-- Bitbucket repositories only
SELECT project_key, repo_slug, name, is_private
FROM git_repositories
WHERE data_source IN ('insight_bitbucket_server')
ORDER BY last_commit_date DESC
LIMIT 10;
```

### Find active repositories

```sql
-- Repositories with recent commits
SELECT 
    project_key,
    repo_slug,
    last_commit_date,
    data_source
FROM git_repositories
WHERE last_commit_date > now() - INTERVAL 30 DAY
ORDER BY last_commit_date DESC;
```

### Get repository with all metadata

```sql
-- Full repository details
SELECT 
    project_key,
    repo_slug,
    full_name,
    description,
    language,
    size,
    is_private,
    created_on,
    updated_on,
    metadata
FROM git_repositories
WHERE project_key = 'TEST'
  AND repo_slug = 'test-core'
  AND data_source = 'insight_bitbucket_server';
```

### Count repositories by language (GitHub only)

```sql
SELECT 
    language,
    COUNT(*) as repo_count,
    SUM(size) as total_size_kb
FROM git_repositories
WHERE data_source = 'insight_github'
  AND language IS NOT NULL
GROUP BY language
ORDER BY repo_count DESC;
```

---

## Notes and Considerations

### Field Availability

Many fields are **source-specific**:
- **GitHub only**: `full_name`, `created_on`, `updated_on`, `size`, `language`, `has_issues`, `has_wiki`
- **Bitbucket only**: `fork_policy` (in metadata but not extracted), `is_empty`, `last_commit_date`

When building unified queries, always check for null values or filter by `data_source`.

### Incremental Collection

The `last_updated` and `last_commit_date` fields enable incremental collection:
1. Query existing repositories from ClickHouse
2. Compare with API response timestamps
3. Skip unchanged repositories
4. Update only modified repositories

### Deduplication

The `_version` field with `ReplacingMergeTree` engine ensures:
- Re-running collection doesn't create duplicates
- Latest version is kept automatically
- Use `FINAL` modifier in queries for guaranteed latest data

### Performance

**Indexes**:
- Primary index on `(project_key, repo_slug, data_source)` enables fast lookups
- Always include `data_source` in WHERE clauses for optimal performance

**Best Practices**:
- Filter by `data_source` early in queries
- Use `project_key` and `repo_slug` for joins
- Avoid full table scans on large datasets
