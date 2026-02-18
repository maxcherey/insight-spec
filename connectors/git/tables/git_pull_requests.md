# Table: `git_pull_requests`

## Overview

**Purpose**: Store pull request metadata including state, reviewers, statistics, and lifecycle information.

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
| `pr_id` | Int64 | REQUIRED | PR database ID |
| `pr_number` | Int64 | REQUIRED | PR display number |
| `title` | String | REQUIRED | PR title |
| `description` | String | NULLABLE | PR body/description |
| `state` | String | REQUIRED | PR state (OPEN/MERGED/CLOSED/DECLINED) |
| `author_name` | String | REQUIRED | PR author username |
| `author_uuid` | String | REQUIRED | Author unique identifier |
| `created_on` | DateTime64(3) | REQUIRED | PR creation time |
| `updated_on` | DateTime64(3) | REQUIRED | Last update time |
| `closed_on` | DateTime64(3) | NULLABLE | Close/merge time |
| `merge_commit_hash` | String | NULLABLE | Hash of merge commit |
| `source_branch` | String | REQUIRED | Source/head branch |
| `destination_branch` | String | REQUIRED | Target/base branch |
| `commit_count` | Int64 | NULLABLE | Number of commits in PR |
| `comment_count` | Int64 | NULLABLE | Number of comments |
| `task_count` | Int64 | NULLABLE | Number of tasks (Bitbucket only) |
| `files_changed` | Int64 | NULLABLE | Files modified |
| `lines_added` | Int64 | NULLABLE | Lines added |
| `lines_removed` | Int64 | NULLABLE | Lines removed |
| `duration_seconds` | Int64 | NULLABLE | Time from creation to close (seconds) |
| `jira_tickets` | String | NULLABLE | JSON array of Jira ticket IDs |
| `metadata` | String | REQUIRED | Full API response as JSON |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `diffstat_metadata` | String | DEFAULT '' | JSON array of file changes |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_pr_lookup`: `(project_key, repo_slug, pr_id, data_source)`
- `idx_pr_updated`: `(updated_on)`
- `idx_pr_state`: `(state)`

---

## Data Collection

### Bitbucket Source

**API Endpoints**:
1. PR list: `/rest/api/1.0/projects/{project}/repos/{repo}/pull-requests`
2. PR details: `/rest/api/1.0/projects/{project}/repos/{repo}/pull-requests/{id}`
3. PR changes: `/rest/api/1.0/projects/{project}/repos/{repo}/pull-requests/{id}/changes`

**Collection Process**:
1. Fetch PRs with state filter (OPEN, MERGED, DECLINED, ALL)
2. For each PR, get detailed metadata
3. Optionally fetch file changes for diffstat
4. Store with `data_source = "insight_bitbucket_server"`

**API Response Structure**:
```json
{
  "id": 10492,
  "version": 3,
  "title": "Hotfix/fix automerge conflicts 25 12",
  "description": "",
  "state": "MERGED",
  "open": false,
  "closed": true,
  "createdDate": 1763963867000,
  "updatedDate": 1763964689000,
  "closedDate": 1763964689000,
  "fromRef": {
    "id": "refs/heads/hotfix/fix_automerge_conflicts_25_12",
    "displayId": "hotfix/fix_automerge_conflicts_25_12",
    "latestCommit": "e2e02927dd0c862ee31bb73ab76f60b30be0f2ce",
    "repository": {...}
  },
  "toRef": {
    "id": "refs/heads/release/25.12",
    "displayId": "release/25.12",
    "latestCommit": "c1e4b6cfcf398ed89ffb48eb795d73ce508e7112",
    "repository": {...}
  },
  "author": {
    "user": {
      "name": "jsmith",
      "emailAddress": "john.smith@company.com",
      "id": 152,
      "slug": "jsmith"
    },
    "role": "AUTHOR",
    "approved": false,
    "status": "UNAPPROVED"
  },
  "reviewers": [...],
  "participants": [...],
  "properties": {
    "mergeCommit": {
      "id": "db1147f6b0da1cdec3ec5a18dd9379aacfa0f869"
    }
  }
}
```

**Field Mapping**:
- `pr_id` ← `id` (same as pr_number for Bitbucket)
- `pr_number` ← `id`
- `title` ← `title`
- `description` ← `description`
- `state` ← `state` (OPEN/MERGED/DECLINED)
- `author_name` ← `author.user.name`
- `author_uuid` ← `author.user.id` as string
- `created_on` ← `createdDate` / 1000 (milliseconds → datetime)
- `updated_on` ← `updatedDate` / 1000
- `closed_on` ← `closedDate` / 1000
- `merge_commit_hash` ← `properties.mergeCommit.id`
- `source_branch` ← `fromRef.displayId`
- `destination_branch` ← `toRef.displayId`
- `task_count` ← count from activities
- `duration_seconds` ← `(closedDate - createdDate) / 1000`

**Example Values**:
- `pr_id`: 10492
- `pr_number`: 10492
- `title`: "Hotfix/fix automerge conflicts 25 12"
- `state`: "MERGED"
- `author_name`: "bob"
- `author_uuid`: "152"
- `source_branch`: "hotfix/fix_automerge_conflicts_25_12"
- `destination_branch`: "release/25.12"
- `duration_seconds`: 822 (13.7 minutes)
- `data_source`: "insight_bitbucket_server"

---

### GitHub Source

**API Endpoints**:
1. REST: `/repos/{owner}/{repo}/pulls`
2. REST details: `/repos/{owner}/{repo}/pulls/{number}`
3. GraphQL: `repository.pullRequests` (bulk fetch with nested data)

**Collection Process**:
1. **With authentication (GraphQL)**:
   - Fetch 25-50 PRs per call with reviews, comments, commits
   - Much more efficient than REST
2. **Without authentication (REST)**:
   - Fetch PR list
   - For each PR, fetch details, reviews, comments, commits
   - ~5 API calls per PR

**GraphQL Query**:
```graphql
query {
  repository(owner: $owner, name: $repo) {
    pullRequests(first: 25, orderBy: {field: UPDATED_AT, direction: DESC}) {
      nodes {
        databaseId
        number
        title
        body
        state
        merged
        author { login databaseId }
        createdAt
        updatedAt
        closedAt
        mergeCommit { oid }
        headRefName
        baseRefName
        commits { totalCount }
        additions
        deletions
        changedFiles
        reviews(first: 100) { nodes {...} }
        comments(first: 100) { nodes {...} }
      }
    }
  }
}
```

**REST API Response**:
```json
{
  "id": 3018797339,
  "number": 4,
  "title": "feat: cli",
  "body": "",
  "state": "closed",
  "merged": true,
  "user": {
    "login": "alice",
    "id": 239822647
  },
  "created_at": "2025-11-17T19:45:14Z",
  "updated_at": "2025-11-22T10:07:07Z",
  "closed_at": "2025-11-22T10:07:07Z",
  "merge_commit_sha": "e6c0dad65ca9fb6c7858377e8740cf608d15a0cf",
  "head": {
    "ref": "feat/cli"
  },
  "base": {
    "ref": "main"
  },
  "commits": 2,
  "additions": 1108,
  "deletions": 14,
  "changed_files": 18
}
```

**Field Mapping**:
- `pr_id` ← `id` or `databaseId` (GitHub's internal ID)
- `pr_number` ← `number` (display number, different from id)
- `title` ← `title`
- `description` ← `body`
- `state` ← "MERGED" if `merged=true`, else "OPEN" or "CLOSED"
- `author_name` ← `user.login` or `author.login`
- `author_uuid` ← `user.id` or `author.databaseId` as string
- `created_on` ← `created_at` or `createdAt` (ISO 8601 → datetime)
- `updated_on` ← `updated_at` or `updatedAt`
- `closed_on` ← `closed_at` or `closedAt`
- `merge_commit_hash` ← `merge_commit_sha` or `mergeCommit.oid`
- `source_branch` ← `head.ref` or `headRefName`
- `destination_branch` ← `base.ref` or `baseRefName`
- `commit_count` ← `commits` or `commits.totalCount`
- `comment_count` ← null (not directly available)
- `task_count` ← null (not applicable to GitHub)
- `files_changed` ← `changed_files` or `changedFiles`
- `lines_added` ← `additions`
- `lines_removed` ← `deletions`
- `duration_seconds` ← `(closed_at - created_at).total_seconds()`

**Example Values**:
- `pr_id`: 3018797339
- `pr_number`: 4
- `title`: "feat: cli"
- `state`: "MERGED"
- `author_name`: "alice"
- `author_uuid`: "239822647"
- `source_branch`: "feat/cli"
- `destination_branch`: "main"
- `commit_count`: 2
- `files_changed`: 18
- `lines_added`: 1108
- `lines_removed`: 14
- `duration_seconds`: 397313 (4.6 days)
- `data_source`: "airnd_github"

---

## Field Semantics

### Core Identifiers

**`pr_id`** (Int64, REQUIRED)
- **Purpose**: PR database identifier
- **GitHub**: Internal database ID (e.g., 3018797339)
- **Bitbucket**: Same as pr_number (e.g., 10492)
- **Usage**: Primary identifier, API references

**`pr_number`** (Int64, REQUIRED)
- **Purpose**: PR display number
- **GitHub**: Sequential number per repository (e.g., #4)
- **Bitbucket**: Same as pr_id (e.g., 10492)
- **Usage**: User-facing references, URLs

### PR Content

**`title`** (String, REQUIRED)
- **Purpose**: PR title/summary
- **GitHub**: "feat: cli"
- **Bitbucket**: "Hotfix/fix automerge conflicts 25 12"
- **Usage**: Display, search, Jira ticket extraction

**`description`** (String, NULLABLE)
- **Purpose**: PR body/description text
- **Both sources**: Can be empty string
- **Usage**: Context, Jira ticket extraction, search

### PR State

**`state`** (String, REQUIRED)
- **Purpose**: Current PR state
- **Values**:
  - GitHub: "OPEN", "MERGED", "CLOSED"
  - Bitbucket: "OPEN", "MERGED", "DECLINED"
- **Note**: "DECLINED" is Bitbucket-specific (closed without merging)
- **Usage**: Filtering, statistics, workflow tracking

### Author Information

**`author_name`** (String, REQUIRED)
- **Purpose**: PR author username
- **GitHub**: "alice"
- **Bitbucket**: "bob"
- **Usage**: Attribution, statistics

**`author_uuid`** (String, REQUIRED)
- **Purpose**: Author unique identifier
- **GitHub**: User ID as string (e.g., "239822647")
- **Bitbucket**: User ID as string (e.g., "152")
- **Usage**: User identification, deduplication

### Timestamps

**`created_on`** (DateTime64(3), REQUIRED)
- **Purpose**: PR creation timestamp
- **Format**: "2025-11-17 19:45:14.000"
- **Both sources**: Always populated
- **Usage**: Age calculation, time-series analysis

**`updated_on`** (DateTime64(3), REQUIRED)
- **Purpose**: Last update timestamp
- **Format**: "2025-11-22 10:07:07.000"
- **Both sources**: Always populated
- **Usage**: Incremental collection, activity tracking

**`closed_on`** (DateTime64(3), NULLABLE)
- **Purpose**: Close/merge timestamp
- **Format**: "2025-11-22 10:07:07.000"
- **Null**: If PR is still open
- **Usage**: Duration calculation, completion tracking

**`duration_seconds`** (Int64, NULLABLE)
- **Purpose**: Time from creation to close
- **GitHub**: 397313 seconds (4.6 days)
- **Bitbucket**: 822 seconds (13.7 minutes)
- **Calculation**: `closed_on - created_on`
- **Usage**: Cycle time metrics, performance analysis

### Branch Information

**`source_branch`** (String, REQUIRED)
- **Purpose**: Source/head branch name
- **GitHub**: "feat/cli"
- **Bitbucket**: "hotfix/fix_automerge_conflicts_25_12"
- **Usage**: Branch tracking, merge analysis

**`destination_branch`** (String, REQUIRED)
- **Purpose**: Target/base branch name
- **GitHub**: "main"
- **Bitbucket**: "release/25.12"
- **Usage**: Release tracking, branch strategy analysis

### Merge Information

**`merge_commit_hash`** (String, NULLABLE)
- **Purpose**: SHA of merge commit
- **GitHub**: "e6c0dad65ca9fb6c7858377e8740cf608d15a0cf"
- **Bitbucket**: "db1147f6b0da1cdec3ec5a18dd9379aacfa0f869"
- **Null**: If not merged yet
- **Usage**: Linking PRs to commits, merge tracking

### Statistics

**`commit_count`** (Int64, NULLABLE)
- **Purpose**: Number of commits in PR
- **GitHub**: 2
- **Bitbucket**: 1
- **Usage**: PR size metrics

**`comment_count`** (Int64, NULLABLE)
- **Purpose**: Number of comments
- **GitHub**: null (not directly tracked)
- **Bitbucket**: 0
- **Usage**: Discussion activity metrics

**`task_count`** (Int64, NULLABLE)
- **Purpose**: Number of tasks
- **GitHub**: null (not applicable)
- **Bitbucket**: 0 (Bitbucket-specific feature)
- **Optional**: Bitbucket only
- **Usage**: Task completion tracking

**`files_changed`** (Int64, NULLABLE)
- **Purpose**: Number of files modified
- **GitHub**: 18
- **Bitbucket**: 1
- **Usage**: PR size metrics, complexity analysis

**`lines_added`** (Int64, NULLABLE)
- **Purpose**: Total lines added
- **GitHub**: 1108
- **Bitbucket**: 1
- **Usage**: Code churn metrics, PR size

**`lines_removed`** (Int64, NULLABLE)
- **Purpose**: Total lines removed
- **GitHub**: 14
- **Bitbucket**: 1
- **Usage**: Code churn metrics, refactoring detection

### Additional Fields

**`jira_tickets`** (String, NULLABLE)
- **Purpose**: Extracted Jira ticket IDs
- **Format**: JSON array `["PROJ-123", "PROJ-456"]`
- **Both sources**: Extracted from title and description
- **Usage**: Jira integration, ticket tracking

**`diffstat_metadata`** (String, DEFAULT '')
- **Purpose**: Detailed file change information
- **Format**: JSON array of file changes
- **Both sources**: Often empty in current data
- **Usage**: File-level analysis when populated

### System Fields

**`metadata`** (String, REQUIRED)
- **Purpose**: Complete API response
- **Both sources**: Always populated
- **Usage**: Debugging, platform-specific fields

**`collected_at`** (DateTime64(3), REQUIRED)
- **Purpose**: Collection timestamp
- **Format**: "2026-02-13 12:08:15.565"
- **Usage**: Data freshness tracking

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version
- **Format**: Millisecond timestamp
- **Usage**: ReplacingMergeTree deduplication

**`data_source`** (String, DEFAULT '')
- **Purpose**: Source discriminator
- **Values**: "airnd_github", "dev_metrics", ""
- **Usage**: Filtering by source

---

## Relationships

### Parent

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many PRs to one repository
- **Description**: All PRs belong to a repository

### Children

**`git_pull_requests_reviewers`**
- **Join**: `pr_id` → `pr_id`
- **Cardinality**: One PR to many reviewers
- **Description**: Reviewers assigned to PR

**`git_pr_comments`**
- **Join**: `pr_id` → `pr_id`
- **Cardinality**: One PR to many comments
- **Description**: Comments and discussions on PR

**`git_pr_commits`**
- **Join**: `pr_id` → `pr_id`
- **Cardinality**: One PR to many commits
- **Description**: Commits included in PR

**`git_pr_participants`**
- **Join**: `pr_id` → `pr_id`
- **Cardinality**: One PR to many participants
- **Description**: All participants in PR (Bitbucket concept)

**`git_tickets`**
- **Join**: `pr_id` → `pr_id`
- **Cardinality**: One PR to many Jira tickets
- **Description**: Jira tickets extracted from PR

---

## Usage Examples

### Query open PRs

```sql
SELECT 
    project_key,
    repo_slug,
    pr_number,
    title,
    author_name,
    created_on,
    files_changed,
    lines_added,
    lines_removed
FROM git_pull_requests
WHERE state = 'OPEN'
  AND data_source = 'insight_bitbucket_server'
ORDER BY created_on DESC
LIMIT 50;
```

### PR cycle time analysis

```sql
SELECT 
    project_key,
    repo_slug,
    AVG(duration_seconds) / 3600 as avg_hours,
    MEDIAN(duration_seconds) / 3600 as median_hours,
    COUNT(*) as pr_count
FROM git_pull_requests
WHERE state = 'MERGED'
  AND closed_on >= '2026-01-01'
  AND data_source = 'insight_bitbucket_server'
GROUP BY project_key, repo_slug
ORDER BY avg_hours DESC;
```

### Find large PRs

```sql
SELECT 
    project_key,
    repo_slug,
    pr_number,
    title,
    files_changed,
    lines_added + lines_removed as total_changes
FROM git_pull_requests
WHERE files_changed > 50
   OR (lines_added + lines_removed) > 1000
ORDER BY total_changes DESC
LIMIT 20;
```

### Author statistics

```sql
SELECT 
    author_name,
    COUNT(*) as pr_count,
    SUM(CASE WHEN state = 'MERGED' THEN 1 ELSE 0 END) as merged_count,
    AVG(duration_seconds) / 3600 as avg_hours,
    SUM(lines_added) as total_added,
    SUM(lines_removed) as total_removed
FROM git_pull_requests
WHERE created_on >= '2026-01-01'
  AND data_source = 'insight_github'
GROUP BY author_name
ORDER BY pr_count DESC
LIMIT 20;
```

---

## Notes and Considerations

### PR ID vs PR Number

**Critical difference**:
- **GitHub**: `pr_id` (3018797339) ≠ `pr_number` (4)
  - `pr_id` is internal database ID
  - `pr_number` is sequential per repository
- **Bitbucket**: `pr_id` = `pr_number` (both are 10492)

Always use `pr_number` for display and URLs.

### State Differences

**GitHub states**: OPEN, MERGED, CLOSED
**Bitbucket states**: OPEN, MERGED, DECLINED

"DECLINED" is Bitbucket-specific (PR closed without merging).

### Timestamp Formats

**Bitbucket**: Milliseconds since epoch
**GitHub**: ISO 8601 strings

Both converted to DateTime64(3): "2025-11-22 10:07:07.000"

### Incremental Collection

Use `updated_on` for incremental collection:
1. Query `max(updated_on)` per repository
2. Fetch PRs with `updated_at > last_update`
3. Sort by `updated_at DESC` for early stopping

### Performance

**Indexes**:
- `(project_key, repo_slug, pr_id, data_source)` for lookups
- `(updated_on)` for incremental collection
- `(state)` for filtering by state

**GraphQL Efficiency**:
- GitHub GraphQL: 1 call per 25-50 PRs with nested data
- GitHub REST: ~5 calls per PR
- **5x performance improvement** with GraphQL
