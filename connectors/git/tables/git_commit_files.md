# Table: `git_commit_files`

## Overview

**Purpose**: Store file-level details for each commit, including line changes, file paths, and license/copyright metadata.

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
| `commit_hash` | String | REQUIRED | Parent commit SHA |
| `diff_hash` | String | REQUIRED | Hash of the diff content |
| `file_path` | String | REQUIRED | Full file path |
| `file_extension` | String | NULLABLE | File extension |
| `lines_added` | Int64 | NULLABLE | Lines added in this file |
| `lines_removed` | Int64 | NULLABLE | Lines removed in this file |
| `ai_thirdparty_flag` | UInt8 | DEFAULT 0 | AI-detected third-party code (0 or 1) |
| `scancode_thirdparty_flag` | UInt8 | DEFAULT 0 | Scancode-detected third-party (0 or 1) |
| `scancode_metadata` | String | NULLABLE | License and copyright info as JSON |
| `collected_at` | DateTime64(3) | REQUIRED | Collection timestamp |
| `data_source` | String | REQUIRED | Source discriminator |
| `_version` | UInt64 | REQUIRED | Deduplication version |

**Indexes**:
- `idx_file_lookup`: `(project_key, repo_slug, commit_hash, file_path, data_source)`

---

## Data Collection

### Bitbucket Source

**API Endpoint**: `/rest/api/1.0/projects/{project}/repos/{repo}/commits/{hash}/diff`

**Collection Process**:
1. For each commit, fetch diff details
2. Parse diff response to extract file-level changes
3. Run scancode on file content for license/copyright detection
4. Store with `data_source = "insight_bitbucket_server"`

**API Response Structure**:
```json
{
  "diffs": [
    {
      "source": {
        "toString": "Classes/Helpers/PEZPSPDFKitHelper.h"
      },
      "destination": {
        "toString": "Classes/Helpers/PEZPSPDFKitHelper.h"
      },
      "hunks": [
        {
          "sourceLine": 1,
          "sourceSpan": 10,
          "destinationLine": 1,
          "destinationSpan": 6,
          "segments": [
            {
              "type": "REMOVED",
              "lines": [{"line": 1, "text": "old code"}]
            },
            {
              "type": "ADDED",
              "lines": [{"line": 1, "text": "new code"}]
            }
          ]
        }
      ]
    }
  ]
}
```

**Field Mapping**:
- `commit_hash` ← parent commit SHA
- `diff_hash` ← SHA-256 hash of diff content
- `file_path` ← `source.toString` or `destination.toString`
- `file_extension` ← extracted from file_path (e.g., "h", "java", "py")
- `lines_added` ← count of ADDED segments
- `lines_removed` ← count of REMOVED segments
- `scancode_metadata` ← result from scancode analysis

**Example Values**:
- `commit_hash`: "10ab40f5841265dc925025167bf84eafd2dbbad3"
- `diff_hash`: "da93104036e6be885d151791c968baf9a111a9b364323ffef2e9f44e32b8e7c1"
- `file_path`: "Classes/Helpers/PEZPSPDFKitHelper.h"
- `file_extension`: "h"
- `lines_added`: 1
- `lines_removed`: 5
- `data_source`: "insight_bitbucket_server"

---

### GitHub Source

**Current Status**: Not implemented

**Potential Implementation**:
- API: `/repos/{owner}/{repo}/commits/{sha}` → `files[]` array
- Each file has: `filename`, `additions`, `deletions`, `status`, `patch`
- Could be populated similarly to Bitbucket

---

## Field Semantics

### Core Identifiers

**`commit_hash`** (String, REQUIRED)
- **Purpose**: Parent commit reference
- **Format**: 40-character SHA-1 hash
- **Example**: "10ab40f5841265dc925025167bf84eafd2dbbad3"
- **Usage**: Join key to commits table

**`diff_hash`** (String, REQUIRED)
- **Purpose**: Unique identifier for this specific file change
- **Format**: SHA-256 hash of diff content
- **Example**: "da93104036e6be885d151791c968baf9a111a9b364323ffef2e9f44e32b8e7c1"
- **Usage**: Deduplication, change tracking

**`file_path`** (String, REQUIRED)
- **Purpose**: Full path to the file in repository
- **Examples**: 
  - "Classes/Helpers/PEZPSPDFKitHelper.h"
  - "src/main/java/com/example/File.java"
  - "README.md"
- **Usage**: File identification, filtering by path patterns

**`file_extension`** (String, NULLABLE)
- **Purpose**: File extension for language detection
- **Examples**: "h", "java", "py", "js", "md"
- **Extracted from**: `file_path` (everything after last dot)
- **Usage**: Language statistics, filtering by file type

### Change Statistics

**`lines_added`** (Int64, NULLABLE)
- **Purpose**: Number of lines added in this file
- **Example**: 1, 50, 200
- **Usage**: Code churn metrics, file-level statistics

**`lines_removed`** (Int64, NULLABLE)
- **Purpose**: Number of lines removed in this file
- **Example**: 5, 20, 100
- **Usage**: Code churn metrics, refactoring detection

### Third-Party Detection

**`ai_thirdparty_flag`** (UInt8, DEFAULT 0)
- **Purpose**: AI-based third-party code detection
- **Values**: 0 = not third-party, 1 = third-party
- **Usage**: Identifying external code, license compliance

**`scancode_thirdparty_flag`** (UInt8, DEFAULT 0)
- **Purpose**: Scancode-based third-party detection
- **Values**: 0 = not third-party, 1 = third-party
- **Usage**: License-based third-party identification

### License and Copyright Metadata

**`scancode_metadata`** (String, NULLABLE)
- **Purpose**: Rich license and copyright information
- **Format**: JSON array with detailed scan results
- **Bitbucket**: Populated with comprehensive data
- **GitHub**: Not currently populated

**Example Structure**:
```json
[{
  "path": "Classes/Helpers/PEZPSPDFKitHelper.h",
  "type": "file",
  "package_data": [],
  "is_legal": false,
  "is_manifest": false,
  "is_readme": false,
  "is_top_level": true,
  "is_key_file": false,
  "detected_license_expression": null,
  "detected_license_expression_spdx": null,
  "license_detections": [],
  "license_clues": [],
  "percentage_of_license_text": 0,
  "copyrights": [{
    "copyright": "Copyright (c) 2026 Company",
    "start_line": 6,
    "end_line": 6
  }],
  "holders": [{
    "holder": "Company",
    "start_line": 6,
    "end_line": 6
  }],
  "authors": [{
    "author": "John Doe",
    "start_line": 5,
    "end_line": 5
  }],
  "scan_errors": []
}]
```

**Fields in scancode_metadata**:
- `copyrights`: Copyright statements found in file
- `holders`: Copyright holders
- `authors`: File authors
- `detected_license_expression`: SPDX license expression
- `license_detections`: Detected licenses with confidence scores
- `is_legal`, `is_manifest`, `is_readme`: File type flags
- `percentage_of_license_text`: How much of file is license text

**Usage**:
- License compliance checking
- Copyright attribution
- Author tracking
- Legal file identification

### System Fields

**`collected_at`** (DateTime64(3), REQUIRED)
- **Purpose**: Collection timestamp
- **Format**: "2026-02-13 16:07:23.857"
- **Usage**: Data freshness tracking

**`_version`** (UInt64, REQUIRED)
- **Purpose**: Deduplication version
- **Format**: Millisecond timestamp
- **Usage**: ReplacingMergeTree deduplication

**`data_source`** (String, REQUIRED)
- **Purpose**: Source discriminator
- **Values**: "dev_metrics" (Bitbucket)
- **Usage**: Filtering by source

---

## Relationships

### Parent

**`git_commits`**
- **Join**: `commit_hash` ← `commit_hash`
- **Cardinality**: Many files to one commit
- **Description**: All file changes belong to a commit

**`git_repositories`**
- **Join**: `(project_key, repo_slug)` ← `(project_key, repo_slug)`
- **Cardinality**: Many files to one repository
- **Description**: All file changes belong to a repository

---

## Usage Examples

### Query files changed in a commit

```sql
SELECT 
    file_path,
    file_extension,
    lines_added,
    lines_removed,
    ai_thirdparty_flag,
    scancode_thirdparty_flag
FROM git_commit_files
WHERE commit_hash = '10ab40f5841265dc925025167bf84eafd2dbbad3'
  AND data_source = 'insight_bitbucket_server'
ORDER BY lines_added + lines_removed DESC;
```

### Find files with copyright information

```sql
SELECT 
    project_key,
    repo_slug,
    file_path,
    scancode_metadata
FROM git_commit_files
WHERE scancode_metadata LIKE '%copyright%'
  AND scancode_metadata != ''
LIMIT 10;
```

### Analyze file extension statistics

```sql
SELECT 
    file_extension,
    COUNT(*) as file_count,
    SUM(lines_added) as total_added,
    SUM(lines_removed) as total_removed
FROM git_commit_files
WHERE data_source = 'insight_bitbucket_server'
  AND file_extension IS NOT NULL
GROUP BY file_extension
ORDER BY file_count DESC
LIMIT 20;
```

### Find third-party code

```sql
SELECT 
    project_key,
    repo_slug,
    commit_hash,
    file_path,
    ai_thirdparty_flag,
    scancode_thirdparty_flag
FROM git_commit_files
WHERE (ai_thirdparty_flag = 1 OR scancode_thirdparty_flag = 1)
  AND data_source = 'insight_bitbucket_server'
ORDER BY collected_at DESC
LIMIT 100;
```

### Extract copyright holders

```sql
SELECT 
    project_key,
    repo_slug,
    file_path,
    JSONExtractString(scancode_metadata, 'holders[0].holder') as copyright_holder
FROM git_commit_files
WHERE scancode_metadata LIKE '%holders%'
  AND scancode_metadata != ''
LIMIT 50;
```

---

## Notes and Considerations

### Bitbucket-Specific Table

This table is **primarily for Bitbucket data**. GitHub file statistics are stored in the `git_commits` table's `files_changed`, `lines_added`, `lines_removed`, and `language_breakdown` fields.

### Scancode Integration

The `scancode_metadata` field provides **rich license and copyright information**:
- Automatic license detection
- Copyright statement extraction
- Author identification
- Legal file classification

This is valuable for:
- License compliance audits
- Copyright attribution
- Open source policy enforcement
- Legal risk assessment

### Performance Considerations

**Index Usage**:
- Primary index on `(project_key, repo_slug, commit_hash, file_path, data_source)`
- Always include these fields in WHERE clauses for optimal performance

**Large Metadata**:
- `scancode_metadata` can be large (several KB per file)
- Consider filtering before extracting JSON fields
- Use `LIKE` for simple searches, JSON functions for complex queries

### Future Enhancements

**GitHub Support**:
- Could populate this table from GitHub commits API
- Would enable consistent file-level analysis across both platforms
- Scancode could be run on GitHub files as well

**Additional Analysis**:
- Code complexity metrics per file
- Dependency detection
- Security vulnerability scanning
- Code quality scores
