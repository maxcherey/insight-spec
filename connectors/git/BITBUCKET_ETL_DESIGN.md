# Bitbucket Server ETL Design Document

## Overview

Comprehensive design for ETL pipeline collecting data from Bitbucket Server/Data Center into unified `git_*` schema.

**Data Source**: `data_source = "insight_bitbucket_server"`  
**API Version**: Bitbucket Server REST API v1.0  
**Authentication**: HTTP Basic Auth, Bearer Token, or ZTA Token

---

## Architecture

### ETL Data Flow

```
┌──────────────────────┐
│  Bitbucket Server    │
│    REST API v1.0     │
│                      │
│  - /projects         │
│  - /repos            │
│  - /commits          │
│  - /pull-requests    │
│  - /activities       │
└──────────┬───────────┘
           │
           │ HTTP/HTTPS
           │ (Paginated Requests)
           │
           ▼
┌──────────────────────┐
│   ETL Script         │
│                      │
│  1. Fetch Data       │
│     - Pagination     │
│     - Rate Limiting  │
│                      │
│  2. Transform        │
│     - Parse JSON     │
│     - Map Fields     │
│     - Extract Tickets│
│                      │
│  3. Load (Batch)     │
│     - Batch Insert   │
│     - Deduplication  │
└──────────┬───────────┘
           │
           │ ClickHouse
           │ Native Protocol
           │
           ▼
┌──────────────────────┐
│   ClickHouse DB      │
│                      │
│  git_repositories    │
│  git_commits         │
│  git_commit_files    │
│  git_pull_requests   │
│  git_pr_reviewers    │
│  git_pr_comments     │
│  git_pr_commits      │
│  git_pr_participants │
│  git_tickets         │
│  git_collection_runs │
│                      │
│  data_source =       │
│  "insight_bitbucket_ │
│   server"            │
└──────────────────────┘
```

### Design Principles

1. **Incremental Collection**: Track timestamps to avoid re-fetching
2. **Idempotency**: ReplacingMergeTree with `_version` field
3. **Pagination**: Server-style with `start`/`limit` parameters
4. **Rate Limiting**: Exponential backoff retry logic
5. **Fault Tolerance**: Checkpoint mechanism for resumability

---

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: bitbucket-etl
spec:
  schedule: "0 */6 * * *"  # Every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: bitbucket-etl
            image: bitbucket-etl:latest
            resources:
              requests:
                cpu: 2
                memory: 4Gi
            env:
            - name: BITBUCKET_BASE_URL
              valueFrom:
                configMapKeyRef:
                  name: bitbucket-etl-config
                  key: base_url
            - name: BITBUCKET_TOKEN
              valueFrom:
                secretKeyRef:
                  name: bitbucket-etl-secrets
                  key: token
            - name: CLICKHOUSE_HOST
              valueFrom:
                configMapKeyRef:
                  name: bitbucket-etl-config
                  key: clickhouse_host
```

---

## Data Flow

### Overall ETL Flow

```
START
  ↓
Initialize Run (create run_id, run record)
  ↓
Fetch Projects (/rest/api/1.0/projects)
  ↓
For Each Project:
  ↓
  Fetch Repositories (/projects/{p}/repos)
    ↓
    For Each Repository:
      ↓
      Collect Repository Metadata → git_repositories
      ↓
      Collect Commits → git_commits, git_commit_files
      ↓
      Collect Pull Requests → git_pull_requests
        ↓
        Collect PR Activities → git_pr_reviewers, git_pr_comments
        ↓
        Collect PR Commits → git_pr_commits
        ↓
        Parse Participants → git_pr_participants
      ↓
      Extract Jira Tickets → git_tickets
  ↓
Finalize Run (update statistics, status)
  ↓
END
```

---

## Collection Algorithms

### Main ETL Orchestrator

```python
def run_bitbucket_etl(config: ETLConfig) -> CollectionRunResult:
    run_id = f"insight_bitbucket_server-{datetime.now().strftime('%Y%m%d-%H%M%S')}"
    stats = CollectionStats()
    
    # Initialize clients
    bb_client = BitbucketClient(config.bitbucket_url, config.token)
    ch_client = ClickHouseClient(config.clickhouse_config)
    
    # Create run record
    create_collection_run(ch_client, run_id, "running", config)
    
    try:
        # Fetch and process projects
        projects = bb_client.get_projects()
        
        for project in projects:
            repos = bb_client.get_repositories(project.key)
            upload_repositories(ch_client, repos)
            stats.repos_processed += len(repos)
            
            for repo in repos:
                # Collect commits
                commits, files = collect_commits(
                    bb_client, ch_client, 
                    project.key, repo.slug
                )
                upload_commits(ch_client, commits, files)
                stats.commits_collected += len(commits)
                
                # Collect PRs
                pr_data = collect_pull_requests(
                    bb_client, ch_client,
                    project.key, repo.slug
                )
                upload_pr_data(ch_client, pr_data)
                stats.prs_collected += len(pr_data.prs)
        
        status = "completed"
    except Exception as e:
        status = "failed"
        logger.error(f"ETL failed: {e}")
    finally:
        finalize_collection_run(ch_client, run_id, status, stats)
    
    return CollectionRunResult(run_id, status, stats)
```

### Incremental Commit Collection

```python
def collect_commits(bb_client, ch_client, project_key, repo_slug):
    # Get last collected commit date
    last_date = ch_client.query(f"""
        SELECT MAX(date) as last_date
        FROM git_commits
        WHERE project_key = '{project_key}'
          AND repo_slug = '{repo_slug}'
          AND data_source = 'insight_bitbucket_server'
    """)[0]['last_date']
    
    commits = []
    commit_files = []
    
    # Fetch branches
    branches = bb_client.get_branches(project_key, repo_slug)
    default_branch = next(b for b in branches if b.is_default)
    
    # Paginate commits
    start = 0
    while True:
        response = bb_client.get_commits(
            project_key, repo_slug,
            until=default_branch.display_id,
            start=start, limit=100
        )
        
        for commit_data in response['values']:
            commit_date = datetime.fromtimestamp(
                commit_data['authorTimestamp'] / 1000
            )
            
            # Early stopping
            if last_date and commit_date < last_date:
                return commits, commit_files
            
            # Fetch diff for file stats
            diff = bb_client.get_commit_diff(
                project_key, repo_slug, commit_data['id']
            )
            
            # Parse commit
            commit = parse_commit(commit_data, diff, project_key, repo_slug)
            commits.append(commit)
            
            # Parse files
            files = parse_commit_files(diff, project_key, repo_slug, commit_data['id'])
            commit_files.extend(files)
        
        if response['isLastPage']:
            break
        start = response['nextPageStart']
    
    return commits, commit_files
```

### Pull Request Collection

```python
def collect_pull_requests(bb_client, ch_client, project_key, repo_slug):
    # Get last PR update
    last_update = ch_client.query(f"""
        SELECT MAX(updated_on) as last_update
        FROM git_pull_requests
        WHERE project_key = '{project_key}'
          AND repo_slug = '{repo_slug}'
          AND data_source = 'insight_bitbucket_server'
    """)[0]['last_update']
    
    result = PRCollectionResult()
    
    # Fetch PRs (all states, newest first)
    start = 0
    while True:
        response = bb_client.get_pull_requests(
            project_key, repo_slug,
            state="ALL", order="NEWEST",
            start=start, limit=100
        )
        
        for pr_data in response['values']:
            updated_on = datetime.fromtimestamp(pr_data['updatedDate'] / 1000)
            
            # Early stopping
            if last_update and updated_on < last_update:
                return result
            
            # Parse PR
            pr = parse_pull_request(pr_data, project_key, repo_slug)
            result.prs.append(pr)
            
            # Fetch activities (reviews, comments)
            activities = bb_client.get_pr_activities(
                project_key, repo_slug, pr_data['id']
            )
            
            for activity in activities['values']:
                if activity['action'] in ['APPROVED', 'UNAPPROVED']:
                    review = parse_review(activity, project_key, repo_slug, pr_data['id'])
                    result.reviews.append(review)
                elif activity['action'] == 'COMMENTED':
                    comment = parse_comment(activity, project_key, repo_slug, pr_data['id'])
                    result.comments.append(comment)
            
            # Fetch PR commits
            pr_commits = bb_client.get_pr_commits(project_key, repo_slug, pr_data['id'])
            for idx, commit in enumerate(pr_commits['values']):
                result.pr_commits.append(PRCommit(
                    project_key=project_key,
                    repo_slug=repo_slug,
                    pr_id=pr_data['id'],
                    commit_hash=commit['id'],
                    commit_order=idx,
                    data_source="insight_bitbucket_server"
                ))
            
            # Parse participants
            for participant in pr_data.get('participants', []):
                result.participants.append(parse_participant(
                    participant, project_key, repo_slug, pr_data['id']
                ))
            
            # Extract tickets
            tickets = extract_jira_tickets(pr_data['title'], pr_data.get('description', ''))
            for ticket in tickets:
                result.tickets.append(Ticket(
                    external_ticket_id=ticket,
                    project_key=project_key,
                    repo_slug=repo_slug,
                    pr_id=pr_data['id'],
                    data_source="insight_bitbucket_server"
                ))
        
        if response['isLastPage']:
            break
        start = response['nextPageStart']
    
    return result
```

---

## API Specifications

### Base Configuration

```python
BASE_URL = "https://git.company.com"
API_BASE = f"{BASE_URL}/rest/api/1.0"

headers = {
    'Authorization': f'Bearer {token}',
    'Content-Type': 'application/json'
}
```

### Key Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/rest/api/1.0/projects` | GET | List projects |
| `/rest/api/1.0/projects/{p}/repos` | GET | List repositories |
| `/rest/api/1.0/projects/{p}/repos/{r}/commits` | GET | List commits |
| `/rest/api/1.0/projects/{p}/repos/{r}/commits/{hash}/diff` | GET | Get commit diff |
| `/rest/api/1.0/projects/{p}/repos/{r}/pull-requests` | GET | List PRs |
| `/rest/api/1.0/projects/{p}/repos/{r}/pull-requests/{id}/activities` | GET | Get PR activities |
| `/rest/api/1.0/projects/{p}/repos/{r}/pull-requests/{id}/commits` | GET | Get PR commits |
| `/rest/api/1.0/projects/{p}/repos/{r}/branches` | GET | List branches |

### Pagination Pattern

All endpoints use:
- `start`: Page start index (default: 0)
- `limit`: Page size (default: 25, max: 1000)
- Response includes: `isLastPage`, `nextPageStart`

```python
def paginate(endpoint_func, **kwargs):
    start = 0
    limit = 100
    all_results = []
    
    while True:
        response = endpoint_func(start=start, limit=limit, **kwargs)
        all_results.extend(response['values'])
        
        if response['isLastPage']:
            break
        start = response['nextPageStart']
    
    return all_results
```

---

## Data Mapping

### Repository Mapping

```python
def map_repository(api_data):
    return {
        'project_key': api_data['project']['key'],
        'repo_slug': api_data['slug'],
        'repo_uuid': str(api_data.get('id')),
        'name': api_data['name'],
        'full_name': None,  # Not available
        'is_private': 1 if not api_data.get('public') else 0,
        'metadata': json.dumps(api_data),
        'data_source': 'insight_bitbucket_server',
        '_version': int(time.time() * 1000)
    }
```

### Commit Mapping

```python
def map_commit(api_data, diff_data, project_key, repo_slug, branch):
    return {
        'project_key': project_key,
        'repo_slug': repo_slug,
        'commit_hash': api_data['id'],
        'branch': branch,
        'author_name': api_data['author']['name'],
        'author_email': api_data['author']['emailAddress'],
        'committer_name': api_data['committer']['name'],
        'committer_email': api_data['committer']['emailAddress'],
        'message': api_data['message'],
        'date': datetime.fromtimestamp(api_data['authorTimestamp'] / 1000),
        'parents': json.dumps([p['id'] for p in api_data.get('parents', [])]),
        'files_changed': len(diff_data.get('diffs', [])),
        'lines_added': calculate_lines_added(diff_data),
        'lines_removed': calculate_lines_removed(diff_data),
        'is_merge_commit': 1 if len(api_data.get('parents', [])) > 1 else 0,
        'metadata': json.dumps(api_data),
        'data_source': 'insight_bitbucket_server',
        '_version': int(time.time() * 1000)
    }
```

### Pull Request Mapping

```python
def map_pull_request(api_data, project_key, repo_slug):
    created_on = datetime.fromtimestamp(api_data['createdDate'] / 1000)
    closed_on = datetime.fromtimestamp(api_data['closedDate'] / 1000) if api_data.get('closedDate') else None
    
    return {
        'project_key': project_key,
        'repo_slug': repo_slug,
        'pr_id': api_data['id'],
        'pr_number': api_data['id'],
        'title': api_data['title'],
        'description': api_data.get('description', ''),
        'state': api_data['state'],  # OPEN/MERGED/DECLINED
        'author_name': api_data['author']['user']['name'],
        'author_uuid': str(api_data['author']['user']['id']),
        'author_email': api_data['author']['user'].get('emailAddress'),
        'created_on': created_on,
        'updated_on': datetime.fromtimestamp(api_data['updatedDate'] / 1000),
        'closed_on': closed_on,
        'merge_commit_hash': api_data.get('properties', {}).get('mergeCommit', {}).get('id'),
        'source_branch': api_data['fromRef']['displayId'],
        'destination_branch': api_data['toRef']['displayId'],
        'duration_seconds': int((closed_on - created_on).total_seconds()) if closed_on else None,
        'metadata': json.dumps(api_data),
        'data_source': 'insight_bitbucket_server',
        '_version': int(time.time() * 1000)
    }
```

---

## Error Handling

### Retry Strategy

```python
def execute_with_retry(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limit
                wait = 2 ** attempt
                time.sleep(wait)
                continue
            elif e.response.status_code >= 500:  # Server error
                wait = 2 ** attempt
                time.sleep(wait)
                continue
            raise
        except requests.exceptions.ConnectionError:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise
    raise Exception("Max retries exceeded")
```

---

## Performance Optimization

### Batch Uploading

```python
class BatchUploader:
    def __init__(self, ch_client, batch_size=1000):
        self.client = ch_client
        self.batch_size = batch_size
        self.batches = {}
    
    def add(self, table, record):
        if table not in self.batches:
            self.batches[table] = []
        self.batches[table].append(record)
        
        if len(self.batches[table]) >= self.batch_size:
            self.flush(table)
    
    def flush(self, table=None):
        if table:
            if self.batches.get(table):
                self.client.insert(table, self.batches[table])
                self.batches[table] = []
        else:
            for t, records in self.batches.items():
                if records:
                    self.client.insert(t, records)
            self.batches = {}
```

### Parallel Processing

```python
from concurrent.futures import ThreadPoolExecutor

def collect_repos_parallel(client, projects, max_workers=5):
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(client.get_repositories, p): p 
            for p in projects
        }
        
        all_repos = []
        for future in as_completed(futures):
            repos = future.result()
            all_repos.extend(repos)
        
        return all_repos
```

---

## Monitoring

### Metrics to Track

- **Collection Run Metrics**: Duration, repos processed, commits/PRs collected
- **API Metrics**: Requests made, rate limit hits, errors
- **Data Quality**: Missing fields, validation failures
- **Performance**: Throughput (records/second), memory usage

### Logging

```python
import logging

logger = logging.getLogger('bitbucket_etl')
logger.setLevel(logging.INFO)

# Log key events
logger.info(f"Starting ETL run: {run_id}")
logger.info(f"Processed {repo_count} repositories")
logger.error(f"Failed to collect commits: {error}")
logger.info(f"ETL completed: {stats}")
```

---

## Implementation Checklist

- [ ] Set up Bitbucket API client with authentication
- [ ] Implement pagination handler
- [ ] Implement retry logic with exponential backoff
- [ ] Create data collectors for each entity type
- [ ] Implement incremental collection logic
- [ ] Set up ClickHouse connection and batch uploader
- [ ] Implement data mapping functions
- [ ] Add error handling and logging
- [ ] Create Kubernetes CronJob deployment
- [ ] Set up monitoring and alerting
- [ ] Test with sample data
- [ ] Perform full backfill
- [ ] Schedule regular incremental runs
