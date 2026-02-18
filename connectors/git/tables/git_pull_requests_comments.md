# Table: `git_pr_comments`

## Overview

**Purpose**: Store PR discussion and review comments, including both general comments and inline code review comments.

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
| `comment_id` | Int64 | REQUIRED | Comment unique ID from API |
| `content` | String | REQUIRED | Comment text/body (supports Markdown) |
| `author_name` | String | REQUIRED | Comment author username |
| `author_uuid` | String | NULLABLE | Author unique identifier |
| `author_email` | String | NULLABLE | Author email address |
| `created_at` | DateTime64(3) | REQUIRED | Comment creation time |
| `updated_at` | DateTime64(3) | REQUIRED | Last update time |
| `state` | String | NULLABLE | Thread state (Bitbucket: OPEN/RESOLVED) |
| `severity` | String | NULLABLE | Comment severity (Bitbucket: NORMAL/BLOCKER) |
| `thread_resolved` | Int64 | NULLABLE | 1 if thread resolved (Bitbucket only) |
| `file_path` | String | NULLABLE | File path for inline comments |
| `line_number` | Int64 | NULLABLE | Line number for inline comments |
| `metadata` | String | REQUIRED | Comment metadata as JSON |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_comment_lookup`: `(project_key, repo_slug, pr_id, comment_id, data_source)`

---

## Data Collection

### Bitbucket Source

**API Endpoint**: `/rest/api/1.0/projects/{project}/repos/{repo}/pull-requests/{id}/activities`

**Collection Process**:
1. Fetch PR activities
2. Filter activities by type: `COMMENTED`
3. Extract comment information from each activity
4. Store with `data_source = "insight_bitbucket_server"`

**API Response Structure**:
```json
{
  "id": 67960978,
  "createdDate": 1746519437307,
  "user": {
    "name": "commenter1",
    "emailAddress": "commenter1@example.com",
    "id": 5794,
    "slug": "commenter1",
    "displayName": "Commenter One"
  },
  "action": "COMMENTED",
  "commentAction": "ADDED",
  "comment": {
    "id": 2850386,
    "version": 0,
    "text": "✅ **Security scan passed**\\n\\nScan log: [link](https://...)",
    "author": {...},
    "createdDate": 1746519437307,
    "updatedDate": 1746519437307,
    "comments": [],
    "threadResolved": false,
    "severity": "NORMAL",
    "state": "OPEN",
    "anchor": {
      "path": "src/main/java/File.java",
      "line": 42,
      "lineType": "CONTEXT"
    }
  }
}
```

**Field Mapping**:
- `comment_id` ← `comment.id`
- `content` ← `comment.text`
- `author_name` ← `user.name` or `user.slug`
- `author_uuid` ← `user.id` as string
- `author_email` ← `user.emailAddress`
- `created_at` ← `comment.createdDate` / 1000
- `updated_at` ← `comment.updatedDate` / 1000
- `state` ← `comment.state` (OPEN/RESOLVED)
- `severity` ← `comment.severity` (NORMAL/BLOCKER)
- `thread_resolved` ← 1 if `comment.threadResolved` is true, else 0
- `file_path` ← `comment.anchor.path` (if inline comment)
- `line_number` ← `comment.anchor.line` (if inline comment)

**Example Values**:
- `comment_id`: 2850386
- `content`: "✅ **Security scan passed**\\n\\nScan log for security team: [link](...)"
- `author_name`: "johndoe"
- `author_uuid`: "5794"
- `author_email`: "johndoe@company.com"
- `created_at`: "2025-05-06 08:10:37.307"
- `state`: "OPEN", "RESOLVED"
- `severity`: "NORMAL", "BLOCKER"
- `thread_resolved`: 0 or 1
- `file_path`: null (for general comments) or path (for inline)
- `data_source`: "dev_metrics"

---

### GitHub Source

**API Endpoints**:
1. REST review comments: `/repos/{owner}/{repo}/pulls/{number}/comments`
2. REST issue comments: `/repos/{owner}/{repo}/issues/{number}/comments`
3. GraphQL: `pullRequest.comments.nodes[]` + `pullRequest.reviewThreads.nodes[].comments.nodes[]`

**Collection Process**:
1. **With authentication (GraphQL)**:
   - Fetch comments as part of PR query
   - Includes both issue comments and review comments
2. **Without authentication (REST)**:
   - Fetch review comments separately
   - Fetch issue comments separately

**GraphQL Query**:
```graphql
pullRequest {
  comments(first: 100) {
    nodes {
      id
      databaseId
      author { login databaseId }
      body
      createdAt
      updatedAt
    }
  }
  reviewThreads(first: 100) {
    nodes {
      comments(first: 50) {
        nodes {
          id
          databaseId
          author { login databaseId }
          body
          createdAt
          updatedAt
          path
          line
        }
      }
    }
  }
}
```

**REST API Response (Review Comment)**:
```json
{
  "id": 2603317285,
  "user": {
    "login": "john",
    "id": 192502778
  },
  "body": "> The test field indicates...\\n\\nSuggestion for improvement",
  "created_at": "2025-12-09T16:03:18Z",
  "updated_at": "2025-12-09T16:03:18Z",
  "path": "gts-macros/Cargo.toml",
  "line": 15,
  "position": 3
}
```

**Field Mapping**:
- `comment_id` ← `id` or `databaseId`
- `content` ← `body`
- `author_name` ← `user.login` or `author.login`
- `author_uuid` ← `user.id` or `author.databaseId` as string (often empty)
- `author_email` ← "" (empty, not provided)
- `created_at` ← `created_at` or `createdAt` (ISO 8601 → datetime)
- `updated_at` ← `updated_at` or `updatedAt`
- `state` ← null (not applicable to GitHub)
- `severity` ← null (not applicable to GitHub)
- `thread_resolved` ← null (not applicable to GitHub)
- `file_path` ← `path` (for review comments)
- `line_number` ← `line` (for review comments)

**Example Values**:
- `comment_id`: 2603317285
- `content`: "> The test field indicates...\\n\\nSuggestion for improvement"
- `author_name`: "reviewer1"
- `author_uuid`: "" (empty)
- `author_email`: "" (empty)
- `created_at`: "2025-12-09 16:03:18.000"
- `state`: null
- `severity`: null
- `thread_resolved`: null
- `file_path`: "gts-macros/Cargo.toml"
- `line_number`: 15
- `data_source`: "airnd_github"

---

## Field Semantics

### Core Identifiers

**`pr_id`** (Int64, REQUIRED)
- **Purpose**: Parent pull request identifier
- **Both sources**: Links to git_pull_requests table
- **Usage**: Join key, filtering by PR

**`comment_id`** (Int64, REQUIRED)
- **Purpose**: Unique comment identifier from API
- **GitHub**: Comment ID (e.g., 2603317285)
- **Bitbucket**: Comment ID (e.g., 2850386)
- **Usage**: Deduplication, comment tracking

### Comment Content

**`content`** (String, REQUIRED)
- **Purpose**: Comment text/body
- **Format**: Markdown-formatted text
- **GitHub**: "> The test field indicates...\\n\\nSuggestion"
- **Bitbucket**: "✅ **Security scan passed**\\n\\nScan log: [link](...)"
- **Note**: Supports Markdown formatting, line breaks, links
- **Usage**: Display, search, analysis

### Author Information

**`author_name`** (String, REQUIRED)
- **Purpose**: Comment author username
- **GitHub**: GitHub username (e.g., "johndoe")
- **Bitbucket**: Bitbucket username (e.g., "johndoe")
- **Usage**: Attribution, filtering

**`author_uuid`** (String, NULLABLE)
- **Purpose**: Author unique identifier
- **GitHub**: Empty string (not provided)
- **Bitbucket**: User ID as string (e.g., "5794")
- **Usage**: User identification

**`author_email`** (String, NULLABLE)
- **Purpose**: Author email address
- **GitHub**: Empty string (not provided)
- **Bitbucket**: Corporate email (e.g., "johndoe@company.com")
- **Usage**: Contact information

### Timestamps

**`created_at`** (DateTime64(3), REQUIRED)
- **Purpose**: Comment creation timestamp
- **Format**: "2025-12-09 16:03:18.000", "2025-05-06 08:10:37.307"
- **Both sources**: Always populated
- **Usage**: Timeline analysis, ordering

**`updated_at`** (DateTime64(3), REQUIRED)
- **Purpose**: Last update timestamp
- **Format**: "2025-12-09 16:03:18.000"
- **Both sources**: Always populated
- **Note**: Same as created_at if never edited
- **Usage**: Edit tracking

### Bitbucket-Specific Fields

**`state`** (String, NULLABLE)
- **Purpose**: Thread state
- **Values**: "OPEN", "RESOLVED"
- **GitHub**: null (not applicable)
- **Bitbucket**: OPEN or RESOLVED
- **Optional**: Bitbucket only
- **Usage**: Thread resolution tracking

**`severity`** (String, NULLABLE)
- **Purpose**: Comment severity level
- **Values**: "NORMAL", "BLOCKER"
- **GitHub**: null (not applicable)
- **Bitbucket**: NORMAL or BLOCKER
- **Optional**: Bitbucket only
- **Usage**: Priority tracking, filtering critical comments

**`thread_resolved`** (Int64, NULLABLE)
- **Purpose**: Thread resolution flag
- **Values**: 1 = resolved, 0 = not resolved, null = not applicable
- **GitHub**: null (not applicable)
- **Bitbucket**: 0 or 1
- **Optional**: Bitbucket only
- **Usage**: Quick filtering for unresolved threads

### Inline Comment Fields

**`file_path`** (String, NULLABLE)
- **Purpose**: File path for inline code review comments
- **GitHub**: "gts-macros/Cargo.toml"
- **Bitbucket**: "src/main/java/File.java" or null
- **Null**: For general PR comments (not inline)
- **Usage**: Filtering inline comments, file-specific analysis

**`line_number`** (Int64, NULLABLE)
- **Purpose**: Line number for inline comments
- **GitHub**: 15
- **Bitbucket**: 42 or null
- **Null**: For general PR comments
- **Usage**: Code location tracking

### System Fields

**`metadata`** (String, REQUIRED)
- **Purpose**: Complete comment metadata as JSON
- **Both sources**: Always populated
- **Usage**: Debugging, accessing additional fields

**`collected_at`** (DateTime64(3), REQUIRED)
- **Purpose**: Collection timestamp
- **Format**: "2026-02-13 12:08:14.015"
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
- **Cardinality**: Many comments to one PR
- **Description**: All comments belong to a PR

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many comments to one repository
- **Description**: All comments are associated with a repository

---

## Usage Examples

### Query PR comments

```sql
SELECT 
    comment_id,
    author_name,
    content,
    created_at,
    file_path,
    line_number
FROM git_pr_comments
WHERE project_key = 'GlobalTypeSystem'
  AND repo_slug = 'gts-go'
  AND pr_id = 3018797339
  AND data_source = 'insight_github'
ORDER BY created_at ASC;
```

### Find inline code review comments

```sql
SELECT 
    pr_id,
    file_path,
    line_number,
    author_name,
    content,
    created_at
FROM git_pr_comments
WHERE file_path IS NOT NULL
  AND line_number IS NOT NULL
  AND data_source = 'insight_bitbucket_server'
ORDER BY created_at DESC
LIMIT 100;
```

### Find unresolved threads (Bitbucket)

```sql
SELECT 
    pr.pr_number,
    pr.title,
    c.author_name,
    c.content,
    c.severity,
    c.created_at
FROM git_pr_comments c
JOIN git_pull_requests pr 
    ON c.pr_id = pr.pr_id 
    AND c.data_source = pr.data_source
WHERE c.state = 'OPEN'
  AND c.thread_resolved = 0
  AND c.data_source = 'insight_bitbucket_server'
  AND pr.state = 'OPEN'
ORDER BY c.severity DESC, c.created_at ASC;
```

### Comment activity statistics

```sql
SELECT 
    author_name,
    COUNT(*) as comment_count,
    COUNT(DISTINCT pr_id) as prs_commented,
    COUNT(CASE WHEN file_path IS NOT NULL THEN 1 END) as inline_comments,
    COUNT(CASE WHEN file_path IS NULL THEN 1 END) as general_comments
FROM git_pr_comments
WHERE data_source = 'insight_bitbucket_server'
  AND created_at >= '2026-01-01'
GROUP BY author_name
ORDER BY comment_count DESC
LIMIT 20;
```

### Find blocker comments (Bitbucket)

```sql
SELECT 
    pr.project_key,
    pr.repo_slug,
    pr.pr_number,
    pr.title,
    c.author_name,
    c.content,
    c.created_at
FROM git_pr_comments c
JOIN git_pull_requests pr 
    ON c.pr_id = pr.pr_id 
    AND c.data_source = pr.data_source
WHERE c.severity = 'BLOCKER'
  AND c.state = 'OPEN'
  AND c.data_source = 'insight_bitbucket_server'
ORDER BY c.created_at DESC
LIMIT 50;
```

### Search comments by content

```sql
SELECT 
    pr_id,
    author_name,
    content,
    created_at,
    file_path
FROM git_pr_comments
WHERE content LIKE '%security%'
  AND data_source = 'insight_bitbucket_server'
ORDER BY created_at DESC
LIMIT 100;
```

---

## Notes and Considerations

### Comment Types

**GitHub** has two types of comments:
1. **Issue comments**: General PR discussion (no file_path)
2. **Review comments**: Inline code review (has file_path and line_number)

**Bitbucket** combines both in activities:
- General comments have no anchor
- Inline comments have anchor with path and line

### Bitbucket-Specific Features

**Thread Management**:
- `state`: OPEN or RESOLVED
- `thread_resolved`: Boolean flag
- `severity`: NORMAL or BLOCKER

These fields enable:
- Tracking unresolved discussions
- Prioritizing blocker comments
- Ensuring all feedback is addressed

**GitHub doesn't have**:
- Thread resolution tracking
- Comment severity levels
- Explicit thread state

### Markdown Support

Both sources support **Markdown formatting**:
- Bold, italic, code blocks
- Links, images
- Quotes, lists
- Line breaks (`\n`)

When displaying, render as Markdown for proper formatting.

### Author Information Gaps

**GitHub** doesn't provide:
- `author_uuid` (empty string)
- `author_email` (empty string)

Use `author_name` for GitHub user identification.

**Bitbucket** provides complete author information.

### Nested Comments

**Bitbucket** supports nested comment threads:
- Parent comment
- Replies to comment
- Nested replies

The `metadata` field contains `comments` array with nested structure.

**GitHub** has flat comment structure with quote-based threading.

### Performance

**Index Usage**:
- Primary index on `(project_key, repo_slug, pr_id, comment_id, data_source)`
- Always include these fields in WHERE clauses

**Full-Text Search**:
- Use `LIKE` for simple content searches
- Consider external search engine for advanced queries
- `content` field can be large (several KB)

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Re-collecting comments doesn't create duplicates
- Latest comment version is kept (for edits)
- Use `FINAL` modifier for guaranteed latest data
