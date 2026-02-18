# Table: `git_pull_requests_reviewers`

## Overview

**Purpose**: Store PR review submissions and approvals, tracking who reviewed each pull request and their review status.

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
| `reviewer_name` | String | REQUIRED | Reviewer username |
| `reviewer_uuid` | String | REQUIRED | Reviewer unique identifier |
| `reviewer_email` | String | NULLABLE | Reviewer email address |
| `status` | String | REQUIRED | Review status |
| `role` | String | REQUIRED | Reviewer role (always "REVIEWER") |
| `approved` | Int64 | REQUIRED | 1 if approved, 0 if not |
| `reviewed_at` | DateTime64(3) | NULLABLE | Review submission timestamp |
| `metadata` | String | REQUIRED | Review metadata as JSON |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `data_source` | String | DEFAULT '' | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_reviewer_lookup`: `(project_key, repo_slug, pr_id, reviewer_uuid, data_source)`

---

## Data Collection

### Bitbucket Source

**API Endpoint**: `/rest/api/1.0/projects/{project}/repos/{repo}/pull-requests/{id}/activities`

**Collection Process**:
1. Fetch PR activities (includes reviews, comments, approvals)
2. Filter activities by type: `APPROVED`, `UNAPPROVED`, `REVIEWED`
3. Extract reviewer information from each activity
4. Store with `data_source = "dev_metrics"`

**API Response Structure**:
```json
{
  "values": [
    {
      "id": 67960978,
      "createdDate": 1746519437307,
      "user": {
        "name": "reviewer1",
        "emailAddress": "reviewer1@company.com",
        "id": 660,
        "slug": "reviewer1",
        "displayName": "Reviewer One"
      },
      "action": "APPROVED",
      "commentAction": "ADDED"
    }
  ]
}
```

**Also from PR details**:
```json
{
  "reviewers": [
    {
      "user": {
        "name": "reviewer2",
        "emailAddress": "reviewer2@company.com",
        "id": 2727,
        "slug": "reviewer2"
      },
      "lastReviewedCommit": "60a8ef554009cbfe80d33f0ea915e1a8c2ffd8e0",
      "role": "REVIEWER",
      "approved": true,
      "status": "APPROVED"
    }
  ]
}
```

**Field Mapping**:
- `reviewer_name` ← `user.name` or `user.slug`
- `reviewer_uuid` ← `user.id` as string
- `reviewer_email` ← `user.emailAddress`
- `status` ← `status` or `action` (APPROVED/UNAPPROVED)
- `role` ← `role` (always "REVIEWER" for this table)
- `approved` ← 1 if status is APPROVED, else 0
- `reviewed_at` ← `createdDate` / 1000 (milliseconds → datetime), often null

**Example Values**:
- `reviewer_name`: "reviewer1", "reviewer2"
- `reviewer_uuid`: "660", "2727"
- `reviewer_email`: "reviewer1@company.com"
- `status`: "UNAPPROVED", "APPROVED"
- `approved`: 0 or 1
- `reviewed_at`: null or timestamp
- `data_source`: "dev_metrics"

---

### GitHub Source

**API Endpoints**:
1. REST: `/repos/{owner}/{repo}/pulls/{number}/reviews`
2. GraphQL: `pullRequest.reviews.nodes[]`

**Collection Process**:
1. **With authentication (GraphQL)**:
   - Fetch reviews as part of PR query
   - Includes nested review data
2. **Without authentication (REST)**:
   - Fetch reviews separately for each PR
   - Parse review submissions

**GraphQL Query**:
```graphql
pullRequest {
  reviews(first: 100) {
    nodes {
      id
      author { login databaseId }
      state
      submittedAt
      body
    }
  }
}
```

**REST API Response**:
```json
{
  "id": 2603317285,
  "user": {
    "login": "bob",
    "id": 192502778
  },
  "state": "APPROVED",
  "submitted_at": "2025-11-22T10:07:03Z",
  "body": "LGTM"
}
```

**Field Mapping**:
- `reviewer_name` ← `user.login` or `author.login`
- `reviewer_uuid` ← `user.id` or `author.databaseId` as string
- `reviewer_email` ← "" (empty, not provided by GitHub API)
- `status` ← `state` (APPROVED/CHANGES_REQUESTED/COMMENTED/approved)
- `role` ← "REVIEWER" (always)
- `approved` ← 1 if state is APPROVED or "approved", else 0
- `reviewed_at` ← `submitted_at` or `submittedAt` (ISO 8601 → datetime)

**Example Values**:
- `reviewer_name`: "bob", "alice"
- `reviewer_uuid`: "192502778", "239822647"
- `reviewer_email`: "" (empty)
- `status`: "APPROVED", "CHANGES_REQUESTED", "COMMENTED", "approved"
- `approved`: 1 or 0
- `reviewed_at`: "2025-11-22 10:07:03.000"
- `data_source`: "airnd_github"

---

## Field Semantics

### Core Identifiers

**`pr_id`** (Int64, REQUIRED)
- **Purpose**: Parent pull request identifier
- **Both sources**: Links to git_pull_requests table
- **Usage**: Join key, filtering by PR

**`reviewer_uuid`** (String, REQUIRED)
- **Purpose**: Unique reviewer identifier
- **GitHub**: User ID as string (e.g., "192502778")
- **Bitbucket**: User ID as string (e.g., "660")
- **Usage**: User identification, deduplication

### Reviewer Information

**`reviewer_name`** (String, REQUIRED)
- **Purpose**: Reviewer username
- **GitHub**: GitHub username (e.g., "bob")
- **Bitbucket**: Bitbucket username/slug (e.g., "jane.smith")
- **Usage**: Display, attribution

**`reviewer_email`** (String, NULLABLE)
- **Purpose**: Reviewer email address
- **GitHub**: Empty string (not provided by API)
- **Bitbucket**: Corporate email (e.g., "Jane.Smith@company.com")
- **Usage**: Contact information, user matching

### Review Status

**`status`** (String, REQUIRED)
- **Purpose**: Review state
- **GitHub values**: 
  - "APPROVED" - Reviewer approved the PR
  - "CHANGES_REQUESTED" - Reviewer requested changes
  - "COMMENTED" - Reviewer commented without approval
  - "approved" - Alternative format for approved
- **Bitbucket values**:
  - "APPROVED" - Reviewer approved
  - "UNAPPROVED" - Reviewer has not approved (default state)
- **Note**: Bitbucket has simpler review workflow (only APPROVED/UNAPPROVED)
- **Usage**: Filtering, approval tracking

**`approved`** (Int64, REQUIRED)
- **Purpose**: Boolean approval flag
- **Values**: 1 = approved, 0 = not approved
- **Derived from**: status field
- **Usage**: Quick filtering for approved reviews, statistics

**`role`** (String, REQUIRED)
- **Purpose**: Reviewer role
- **Both sources**: Always "REVIEWER" for this table
- **Note**: Bitbucket has PARTICIPANT role, but those go in separate table
- **Usage**: Role-based filtering (though always same value)

### Timestamps

**`reviewed_at`** (DateTime64(3), NULLABLE)
- **Purpose**: When review was submitted
- **Format**: "2025-11-22 10:07:03.000"
- **GitHub**: Populated from `submitted_at`
- **Bitbucket**: Often null (not always available)
- **Usage**: Timeline analysis, review speed metrics

### System Fields

**`metadata`** (String, REQUIRED)
- **Purpose**: Complete review metadata as JSON
- **Both sources**: Always populated
- **Usage**: Debugging, accessing additional fields

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
- **Cardinality**: Many reviewers to one PR
- **Description**: All reviewers belong to a PR

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many reviewers to one repository
- **Description**: All reviewers are associated with a repository

---

## Usage Examples

### Query PR reviewers

```sql
SELECT 
    pr_id,
    reviewer_name,
    reviewer_email,
    status,
    approved,
    reviewed_at
FROM git_pull_requests_reviewers
WHERE project_key = 'GlobalTypeSystem'
  AND repo_slug = 'gts-go'
  AND pr_id = 3018797339
  AND data_source = 'airnd_github'
ORDER BY reviewed_at DESC;
```

### Find PRs with approvals

```sql
SELECT 
    pr.project_key,
    pr.repo_slug,
    pr.pr_number,
    pr.title,
    COUNT(r.id) as total_reviewers,
    SUM(r.approved) as approvals
FROM git_pull_requests pr
LEFT JOIN git_pull_requests_reviewers r 
    ON pr.pr_id = r.pr_id 
    AND pr.data_source = r.data_source
WHERE pr.state = 'OPEN'
  AND pr.data_source = 'insight_github'
GROUP BY pr.project_key, pr.repo_slug, pr.pr_number, pr.title
HAVING approvals >= 2
ORDER BY pr.created_on DESC;
```

### Reviewer statistics

```sql
SELECT 
    reviewer_name,
    COUNT(*) as review_count,
    SUM(approved) as approval_count,
    AVG(approved) as approval_rate,
    COUNT(DISTINCT pr_id) as unique_prs
FROM git_pull_requests_reviewers
WHERE data_source = 'insight_github'
  AND reviewed_at >= '2026-01-01'
GROUP BY reviewer_name
ORDER BY review_count DESC
LIMIT 20;
```

### Review turnaround time

```sql
SELECT 
    r.reviewer_name,
    AVG(EXTRACT(EPOCH FROM (r.reviewed_at - pr.created_on))) / 3600 as avg_hours_to_review,
    COUNT(*) as review_count
FROM git_pull_requests_reviewers r
JOIN git_pull_requests pr 
    ON r.pr_id = pr.pr_id 
    AND r.data_source = pr.data_source
WHERE r.reviewed_at IS NOT NULL
  AND r.data_source = 'insight_github'
  AND pr.created_on >= '2026-01-01'
GROUP BY r.reviewer_name
HAVING review_count >= 5
ORDER BY avg_hours_to_review ASC
LIMIT 20;
```

### Find change requests

```sql
SELECT 
    pr.project_key,
    pr.repo_slug,
    pr.pr_number,
    pr.title,
    r.reviewer_name,
    r.status,
    r.reviewed_at
FROM git_pull_requests_reviewers r
JOIN git_pull_requests pr 
    ON r.pr_id = pr.pr_id 
    AND r.data_source = pr.data_source
WHERE r.status = 'CHANGES_REQUESTED'
  AND r.data_source = 'insight_github'
ORDER BY r.reviewed_at DESC
LIMIT 50;
```

---

## Notes and Considerations

### Review Status Differences

**GitHub** has granular review states:
- **APPROVED**: Reviewer explicitly approved
- **CHANGES_REQUESTED**: Reviewer requested changes
- **COMMENTED**: Reviewer commented without explicit approval/rejection
- **approved**: Alternative lowercase format

**Bitbucket** has simpler workflow:
- **APPROVED**: Reviewer approved
- **UNAPPROVED**: Default state (not yet approved)

Bitbucket doesn't have "changes requested" concept - reviewers either approve or don't.

### Email Availability

**Critical difference**:
- **GitHub**: `reviewer_email` is always empty (not provided by API)
- **Bitbucket**: `reviewer_email` is populated with corporate email

For user matching across sources, use `reviewer_uuid` or `reviewer_name`.

### Reviewed At Timestamp

**GitHub**: Always populated when review is submitted
**Bitbucket**: Often null (not always tracked in activities)

When analyzing review speed, filter for `reviewed_at IS NOT NULL`.

### Multiple Reviews

A reviewer can submit **multiple reviews** for the same PR:
- Initial review
- Follow-up after changes
- Re-approval after updates

Each review creates a separate row. To get latest review per reviewer:
```sql
SELECT *
FROM git_pull_requests_reviewers
WHERE (pr_id, reviewer_uuid, reviewed_at) IN (
    SELECT pr_id, reviewer_uuid, MAX(reviewed_at)
    FROM git_pull_requests_reviewers
    WHERE reviewed_at IS NOT NULL
    GROUP BY pr_id, reviewer_uuid
)
```

### Approval Requirements

Different repositories may have different approval requirements:
- Some require 1 approval
- Some require 2+ approvals
- Some require specific reviewers

This table tracks all reviewers and their status, but doesn't enforce approval policies.

### Performance

**Index Usage**:
- Primary index on `(project_key, repo_slug, pr_id, reviewer_uuid, data_source)`
- Always include these fields in WHERE clauses for optimal performance

**Join Optimization**:
- When joining with PRs, always include `data_source` in join condition
- Use `pr_id` for efficient lookups

### Deduplication

The `_version` field with `ReplacingMergeTree` ensures:
- Re-collecting reviews doesn't create duplicates
- Latest review state is kept
- Use `FINAL` modifier for guaranteed latest data
