# Table: `git_tickets`

## Overview

**Purpose**: Store ticket references extracted from PR titles, descriptions, and commit messages.

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
| `external_ticket_id` | String | REQUIRED | External ticket ID (e.g., Jira ticket ID PROJ-123) |
| `project_key` | String | REQUIRED | Repository owner |
| `repo_slug` | String | REQUIRED | Repository name |
| `pr_id` | Int64 | NULLABLE | Associated PR ID (0 if from commit) |
| `commit_hash` | String | NULLABLE | Associated commit hash (empty if from PR) |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_jira_lookup`: `(external_ticket_id, project_key, repo_slug, data_source)`
- `idx_jira_pr`: `(pr_id)`
- `idx_jira_commit`: `(commit_hash)`

---

## Data Collection

### Extraction Logic

**Regex Pattern**: `\b([A-Z][A-Z0-9]+-\d+)\b`

**Sources**:
1. **PR title**: Extract from pull request title
2. **PR description**: Extract from pull request body/description
3. **Commit message**: Extract from commit message text

**Both Analyzers** use the same extraction logic.

### Bitbucket Source

**Collection Process**:
1. For each PR, extract Jira tickets from:
   - `title` field
   - `description` field
2. For each commit, extract Jira tickets from:
   - `message` field
3. Also available in Bitbucket metadata: `properties.jira-key` array
4. Store with `data_source = "insight_bitbucket_server"`

**Bitbucket Metadata Example**:
```json
{
  "properties": {
    "jira-key": ["PLTFRM-84253", "PLTFRM-84254"]
  }
}
```

**Field Mapping**:
- `external_ticket_id` ← extracted ticket ID (e.g., "PLTFRM-84253")
- `pr_id` ← PR ID if from PR, 0 if from commit
- `commit_hash` ← commit SHA if from commit, "" if from PR

**Example Values**:
- `external_ticket_id`: "PLTFRM-84253"
- `project_key`: "abc"
- `repo_slug`: "k8s-platform-chart"
- `pr_id`: 10492 (or 0 if from commit)
- `commit_hash`: "" (or SHA if from commit)
- `data_source`: "insight_bitbucket_server"

---

### GitHub Source

**Collection Process**:
1. For each PR, extract tickets from:
   - `title` field
   - `body` field
2. For each commit, extract Jira tickets from:
   - `message` field
3. Store with `data_source = "insight_github"`

**Field Mapping**:
- `external_ticket_id` ← extracted ticket ID (e.g., "PLTFRM-84867")
- `pr_id` ← PR ID if from PR, 0 if from commit
- `commit_hash` ← commit SHA if from commit, "" if from PR

**Example Values**:
- `external_ticket_id`: "PLTFRM-84867"
- `project_key`: "acronis"
- `repo_slug`: "go-authkit"
- `pr_id`: 3137177218 (or 0 if from commit)
- `commit_hash`: "" (or SHA if from commit)
- `data_source`: "insight_github"

---

## Field Semantics

### Core Identifiers

**`external_ticket_id`** (String, REQUIRED)
- **Purpose**: External ticket identifier
- **Format**: PROJECT-NUMBER (e.g., "PLTFRM-84253", "ABC-123")
- **Pattern**: One or more uppercase letters/digits, hyphen, one or more digits
- **Both sources**: Extracted using same regex
- **Usage**: Linking code changes to external issues

**`project_key`** (String, REQUIRED)
- **Purpose**: Repository owner
- **Both sources**: Required for context
- **Usage**: Filtering by repository

**`repo_slug`** (String, REQUIRED)
- **Purpose**: Repository name
- **Both sources**: Required for context
- **Usage**: Filtering by repository

### Source References

**`pr_id`** (Int64, NULLABLE)
- **Purpose**: Associated pull request ID
- **Values**: 
  - PR ID if ticket extracted from PR
  - 0 if ticket extracted from commit
- **Usage**: Linking tickets to PRs

**`commit_hash`** (String, NULLABLE)
- **Purpose**: Associated commit SHA
- **Values**:
  - Commit hash if ticket extracted from commit
  - Empty string "" if ticket extracted from PR
- **Usage**: Linking tickets to commits

**Note**: A ticket is either from a PR (pr_id > 0, commit_hash = "") or from a commit (pr_id = 0, commit_hash != "").

### System Fields

**`collected_at`** (DateTime64(3), REQUIRED)
- **Purpose**: Collection timestamp
- **Format**: "2026-02-13 12:09:14.220"
- **Usage**: Data freshness tracking

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version
- **Format**: Millisecond timestamp
- **Usage**: ReplacingMergeTree deduplication

**`data_source`** (String, DEFAULT '')
- **Purpose**: Source discriminator
- **Values**: "insight_bitbucket_server", "insight_github", "insight_gitlab", "custom_etl"
- **Usage**: Filtering by source

---

## Relationships

### Parent

**`git_pull_requests`**
- **Join**: `pr_id` ← `pr_id` (when pr_id > 0)
- **Cardinality**: Many tickets to one PR
- **Description**: Tickets extracted from PR

**`git_commits`**
- **Join**: `commit_hash` ← `commit_hash` (when commit_hash != "")
- **Cardinality**: Many tickets to one commit
- **Description**: Tickets extracted from commit

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many tickets to one repository
- **Description**: All tickets are associated with a repository

---

## Usage Examples

### Find all Jira tickets in a repository

```sql
SELECT DISTINCT
    jira_ticket,
    COUNT(DISTINCT pr_id) as pr_count,
    COUNT(DISTINCT commit_hash) as commit_count
FROM git_tickets
WHERE project_key = 'acronis'
  AND repo_slug = 'go-authkit'
  AND data_source = 'airnd_github'
GROUP BY jira_ticket
ORDER BY pr_count + commit_count DESC;
```

### Find PRs for a specific Jira ticket

```sql
SELECT 
    pr.project_key,
    pr.repo_slug,
    pr.pr_number,
    pr.title,
    pr.state,
    pr.created_on
FROM git_tickets jt
JOIN git_pull_requests pr 
    ON jt.pr_id = pr.pr_id 
    AND jt.data_source = pr.data_source
WHERE jt.jira_ticket = 'PLTFRM-84867'
  AND jt.pr_id > 0
  AND jt.data_source = 'airnd_github'
ORDER BY pr.created_on DESC;
```

### Find commits for a specific Jira ticket

```sql
SELECT 
    c.commit_hash,
    c.author_name,
    c.message,
    c.date,
    c.files_changed,
    c.lines_added,
    c.lines_removed
FROM git_tickets jt
JOIN git_commits c 
    ON jt.commit_hash = c.commit_hash 
    AND jt.data_source = c.data_source
WHERE jt.external_ticket_id = 'PLTFRM-84253'
  AND jt.commit_hash != ''
  AND jt.data_source = 'insight_bitbucket_server'
ORDER BY c.date DESC;
```

### Most referenced Jira tickets

```sql
SELECT 
    jira_ticket,
    COUNT(*) as reference_count,
    COUNT(CASE WHEN pr_id > 0 THEN 1 END) as in_prs,
    COUNT(CASE WHEN commit_hash != '' THEN 1 END) as in_commits,
    MIN(collected_at) as first_seen,
    MAX(collected_at) as last_seen
FROM git_tickets
WHERE data_source = 'insight_bitbucket_server'
  AND collected_at >= '2026-01-01'
GROUP BY jira_ticket
ORDER BY reference_count DESC
LIMIT 50;
```

### Jira tickets by project prefix

```sql
SELECT 
    SUBSTRING(jira_ticket, 1, POSITION('-' IN jira_ticket) - 1) as jira_project,
    COUNT(DISTINCT jira_ticket) as unique_tickets,
    COUNT(*) as total_references
FROM git_tickets
WHERE data_source = 'insight_bitbucket_server'
GROUP BY jira_project
ORDER BY unique_tickets DESC;
```

### Find PRs and commits for a ticket

```sql
-- Combined view of PRs and commits for a ticket
SELECT 
    'PR' as source_type,
    pr.pr_number as identifier,
    pr.title as description,
    pr.created_on as timestamp
FROM git_tickets jt
JOIN git_pull_requests pr 
    ON jt.pr_id = pr.pr_id 
    AND jt.data_source = pr.data_source
WHERE jt.external_ticket_id = 'PLTFRM-84253'
  AND jt.pr_id > 0

UNION ALL

SELECT 
    'Commit' as source_type,
    c.commit_hash as identifier,
    c.message as description,
    c.date as timestamp
FROM git_tickets jt
JOIN git_commits c 
    ON jt.commit_hash = c.commit_hash 
    AND jt.data_source = c.data_source
WHERE jt.external_ticket_id = 'PLTFRM-84253'
  AND jt.commit_hash != ''

ORDER BY timestamp DESC;
```

---

## Notes and Considerations

### Extraction Method

**Regex Pattern**: `\b([A-Z][A-Z0-9]+-\d+)\b`

**Matches**:
- "PLTFRM-84253" ✓
- "ABC-123" ✓
- "PROJECT-1" ✓
- "A1B2-999" ✓

**Doesn't Match**:
- "abc-123" (lowercase)
- "PROJ_123" (underscore instead of hyphen)
- "123-PROJ" (number first)

**Multiple Tickets**:
- Can extract multiple tickets from same text
- Each ticket creates separate row
- Example: "PROJ-123 and PROJ-124" → 2 rows

### Source Disambiguation

**From PR** (pr_id > 0, commit_hash = ""):
- Extracted from PR title or description
- Links to pull request

**From Commit** (pr_id = 0, commit_hash != ""):
- Extracted from commit message
- Links to commit

**Never both**:
- A row is either from PR or from commit, never both

### Bitbucket Built-in Integration

Bitbucket has **built-in Jira integration**:
- `properties.jira-key` array in metadata
- Automatically detected by Bitbucket
- More reliable than regex extraction

**Recommendation**: Use both sources:
1. Extract from `properties.jira-key` (Bitbucket only)
2. Fall back to regex extraction

### Duplicate Tickets

**Same ticket can appear multiple times**:
- In multiple PRs
- In multiple commits
- In both PR and commits

**Deduplication**:
- Use `DISTINCT jira_ticket` for unique tickets
- Use `COUNT(*)` for total references
- Group by ticket for aggregation

### Cross-Platform Tracking

**Same Jira ticket** can appear in:
- Bitbucket PRs and commits
- GitHub PRs and commits
- Multiple repositories

Query across sources:
```sql
SELECT 
    jira_ticket,
    data_source,
    COUNT(*) as references
FROM git_tickets
WHERE jira_ticket = 'PLTFRM-84253'
GROUP BY jira_ticket, data_source;
```

### Performance

**Indexes**:
- `(jira_ticket, project_key, repo_slug, data_source)` for ticket lookups
- `(pr_id)` for PR-based queries
- `(commit_hash)` for commit-based queries

**Optimization**:
- Always filter by `data_source` when possible
- Use indexes for efficient lookups
- Consider partitioning by date for large datasets

### Use Cases

**Jira ticket tracking**:
- Link code changes to Jira issues
- Track ticket progress through PRs and commits
- Generate release notes from tickets
- Measure ticket completion time

**Traceability**:
- Find all code changes for a ticket
- Understand ticket implementation
- Audit ticket-to-code mapping

**Reporting**:
- Tickets per repository
- Tickets per developer
- Ticket reference frequency
- Cross-repository ticket tracking

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Re-collecting tickets doesn't create duplicates
- Latest extraction is kept
- Use `FINAL` modifier for guaranteed latest data
