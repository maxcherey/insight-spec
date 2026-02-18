# Table: `git_pr_commits`

## Overview

**Purpose**: Link commits to pull requests, tracking which commits are included in each PR.

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
| `pr_id` | Int64 | REQUIRED | Parent PR ID |
| `commit_hash` | String | REQUIRED | Commit SHA |
| `commit_order` | Int64 | REQUIRED | Order of commit in PR (0-indexed) |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_pr_commit_lookup`: `(project_key, repo_slug, pr_id, commit_hash, data_source)`

---

## Data Collection

### Bitbucket Source

**API Endpoint**: `/rest/api/1.0/projects/{project}/repos/{repo}/pull-requests/{id}/commits`

**Collection Process**:
1. For each PR, fetch associated commits
2. Extract commit hashes
3. Assign order based on API response sequence
4. Store with `data_source = "insight_bitbucket_server"`

**API Response Structure**:
```json
{
  "values": [
    {
      "id": "a749767deeac85d702fe97bcfab57ea1797e7511",
      "displayId": "a749767deea",
      "author": {
        "name": "alice",
        "emailAddress": "alice@company.com"
      },
      "authorTimestamp": 1763963129000,
      "committer": {
        "name": "alice",
        "emailAddress": "alice@company.com"
      },
      "committerTimestamp": 1763963129000,
      "message": "Pull request #10486: Bumped components versions...",
      "parents": [...]
    }
  ]
}
```

**Field Mapping**:
- `commit_hash` ← `id`
- `commit_order` ← index in response array (0-indexed)

**Example Values**:
- `pr_id`: 10489
- `commit_hash`: "a749767deeac85d702fe97bcfab57ea1797e7511"
- `commit_order`: 1
- `data_source`: "insight_bitbucket_server"

---

### GitHub Source

**API Endpoints**:
1. REST: `/repos/{owner}/{repo}/pulls/{number}/commits`
2. GraphQL: `pullRequest.commits.nodes[].commit.oid`

**Collection Process**:
1. **With authentication (GraphQL)**:
   - Fetch commits as part of PR query
   - Extract commit OIDs
2. **Without authentication (REST)**:
   - Fetch commits separately for each PR

**GraphQL Query**:
```graphql
pullRequest {
  commits(first: 100) {
    nodes {
      commit {
        oid
        message
        committedDate
        author { name email }
      }
    }
  }
}
```

**REST API Response**:
```json
{
  "sha": "044295536b349f74e154b0770582c548d710f856",
  "commit": {
    "author": {
      "name": "bob",
      "email": "bob@example.com",
      "date": "2025-11-17T19:45:10Z"
    },
    "message": "Initial commit for CLI feature"
  }
}
```

**Field Mapping**:
- `commit_hash` ← `sha` or `commit.oid`
- `commit_order` ← index in response array (0-indexed)

**Example Values**:
- `pr_id`: 3018797339
- `commit_hash`: "044295536b349f74e154b0770582c548d710f856"
- `commit_order`: 0
- `data_source`: "insight_github"

---

## Field Semantics

### Core Identifiers

**`pr_id`** (Int64, REQUIRED)
- **Purpose**: Parent pull request identifier
- **Both sources**: Links to git_pull_requests table
- **Usage**: Join key, filtering by PR

**`commit_hash`** (String, REQUIRED)
- **Purpose**: Git SHA-1 hash
- **Format**: 40-character hexadecimal string
- **Both sources**: Standard Git hash
- **Example**: "044295536b349f74e154b0770582c548d710f856"
- **Usage**: Links to git_commits table

### Commit Order

**`commit_order`** (Int64, REQUIRED)
- **Purpose**: Order of commit in PR
- **Format**: 0-indexed integer
- **GitHub**: 0, 1, 2, ...
- **Bitbucket**: 0, 1, 2, ...
- **Usage**: Ordering commits chronologically within PR


### System Fields

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

**`git_pull_requests`**
- **Join**: `pr_id` ← `pr_id`
- **Cardinality**: Many commits to one PR
- **Description**: All commits belong to a PR

**`git_commits`**
- **Join**: `commit_hash` ← `commit_hash`
- **Cardinality**: Many PR associations to one commit
- **Description**: Same commit can be in multiple PRs

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many PR-commits to one repository
- **Description**: All PR-commit links are in a repository

---

## Usage Examples

### Query commits in a PR

```sql
SELECT 
    pc.commit_hash,
    pc.commit_order,
    c.author_name,
    c.message,
    c.date,
    c.files_changed,
    c.lines_added,
    c.lines_removed
FROM git_pr_commits pc
LEFT JOIN git_commits c 
    ON pc.commit_hash = c.commit_hash 
    AND pc.data_source = c.data_source
WHERE pc.pr_id = 3018797339
  AND pc.data_source = 'insight_github'
ORDER BY pc.commit_order ASC;
```

### Find PRs containing a specific commit

```sql
SELECT 
    pr.project_key,
    pr.repo_slug,
    pr.pr_number,
    pr.title,
    pr.state,
    pc.commit_order
FROM git_pr_commits pc
JOIN git_pull_requests pr 
    ON pc.pr_id = pr.pr_id 
    AND pc.data_source = pr.data_source
WHERE pc.commit_hash = '044295536b349f74e154b0770582c548d710f856'
  AND pc.data_source = 'insight_bitbucket_server'
ORDER BY pr.created_on DESC;
```

### PR size by commit count

```sql
SELECT 
    pr.project_key,
    pr.repo_slug,
    pr.pr_number,
    pr.title,
    COUNT(pc.commit_hash) as commit_count,
    pr.files_changed,
    pr.lines_added,
    pr.lines_removed
FROM git_pull_requests pr
LEFT JOIN git_pr_commits pc 
    ON pr.pr_id = pc.pr_id 
    AND pr.data_source = pc.data_source
WHERE pr.data_source = 'insight_bitbucket_server'
  AND pr.created_on >= '2026-01-01'
GROUP BY pr.project_key, pr.repo_slug, pr.pr_number, pr.title, 
         pr.files_changed, pr.lines_added, pr.lines_removed
ORDER BY commit_count DESC
LIMIT 50;
```

### Find commits that appear in multiple PRs

```sql
SELECT 
    commit_hash,
    COUNT(DISTINCT pr_id) as pr_count,
    GROUP_CONCAT(DISTINCT pr_id) as pr_ids
FROM git_pr_commits
WHERE data_source = 'insight_bitbucket_server'
GROUP BY commit_hash
HAVING pr_count > 1
ORDER BY pr_count DESC
LIMIT 100;
```

---

## Notes and Considerations

### Many-to-Many Relationship

This table implements a **many-to-many relationship**:
- One PR can have many commits
- One commit can be in many PRs (cherry-picks, backports, rebases)

### Commit Order

The `commit_order` field preserves the **chronological order** of commits within a PR:
- 0 = first commit
- 1 = second commit
- etc.

This is useful for:
- Displaying commits in correct order
- Analyzing commit sequence
- Understanding PR evolution

### Join Optimization

When joining with commits table:
```sql
-- Always include data_source in join
LEFT JOIN git_commits c 
    ON pc.commit_hash = c.commit_hash 
    AND pc.data_source = c.data_source
```

This ensures correct matching across sources.

### Cherry-Picks and Backports

The same commit can appear in multiple PRs:
- Cherry-picked to different branches
- Backported to release branches
- Rebased and force-pushed

Query for such commits:
```sql
SELECT commit_hash, COUNT(DISTINCT pr_id) as pr_count
FROM git_pr_commits
GROUP BY commit_hash
HAVING pr_count > 1
```

### Performance

**Index Usage**:
- Primary index on `(project_key, repo_slug, pr_id, commit_hash, data_source)`
- Efficient for PR-based queries
- Include all fields in WHERE for optimal performance

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Re-collecting PR commits doesn't create duplicates
- Latest version is kept
- Use `FINAL` modifier for guaranteed latest data
