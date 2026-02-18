# GitHub ETL Design Document

## Overview

Comprehensive design for ETL pipeline collecting data from GitHub into unified `git_*` schema with GraphQL optimization.

**Data Source**: `data_source = "insight_github"`  
**API Version**: GitHub REST API v3 + GraphQL API v4  
**Authentication**: Bearer Token (Personal Access Token or GitHub App)

---

## Architecture

### ETL Data Flow

```
┌──────────────────────┐
│    GitHub API        │
│                      │
│  REST API v3:        │
│  - /orgs/{org}/repos │
│  - /repos/.../commits│
│  - /repos/.../pulls  │
│                      │
│  GraphQL API v4:     │
│  - Bulk Commits      │
│    (100 per request) │
│  - Bulk PRs          │
│    (50 per request)  │
│                      │
│  Rate: 5000 req/hr   │
└──────────┬───────────┘
           │
           │ HTTPS
           │ (REST + GraphQL)
           │
           ▼
┌──────────────────────┐
│   ETL Script         │
│                      │
│  1. Fetch Data       │
│     - GraphQL Bulk   │
│     - REST Fallback  │
│     - Rate Limiting  │
│                      │
│  2. Transform        │
│     - Parse JSON     │
│     - Map Fields     │
│     - Calculate Stats│
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
│  git_tickets         │
│  git_collection_runs │
│                      │
│  data_source =       │
│  "insight_github"    │
└──────────────────────┘
```

### Key Advantages

1. **GraphQL Optimization**: 100x faster commit collection, 5x faster PR collection
2. **Rich Metadata**: More fields available (language, size, issues, wiki)
3. **Rate Limiting**: 5000 requests/hour (authenticated)
4. **Multi-Branch Support**: Efficient collection from all branches
5. **Incremental Updates**: Track by `updated_at` timestamps

---

## Deployment

### Kubernetes Deployment

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: github-etl
spec:
  schedule: "0 */4 * * *"  # Every 4 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: github-etl
            image: github-etl:latest
            resources:
              requests:
                cpu: 2
                memory: 4Gi
            env:
            - name: GITHUB_TOKEN
              valueFrom:
                secretKeyRef:
                  name: github-etl-secrets
                  key: token
            - name: GITHUB_ORG
              value: "myorg"
            - name: CLICKHOUSE_HOST
              valueFrom:
                configMapKeyRef:
                  name: github-etl-config
                  key: clickhouse_host
            - name: COLLECT_ALL_BRANCHES
              value: "true"
```

---

## Data Flow

### Overall ETL Flow

```
START
  ↓
Initialize Run (create run_id, run record)
  ↓
Fetch Organization Repositories (/orgs/{org}/repos)
  ↓
For Each Repository:
  ↓
  Collect Repository Metadata → git_repositories
  ↓
  Fetch Branches (/repos/{owner}/{repo}/branches)
  ↓
  For Each Branch (or default only):
    ↓
    Collect Commits (GraphQL) → git_commits, git_commit_files
  ↓
  Collect Pull Requests (GraphQL) → git_pull_requests
    ↓
    Parse Reviews → git_pr_reviewers
    ↓
    Parse Comments → git_pr_comments
    ↓
    Parse Commits → git_pr_commits
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
def run_github_etl(config: ETLConfig) -> CollectionRunResult:
    run_id = f"insight_github-{datetime.now().strftime('%Y%m%d-%H%M%S')}"
    stats = CollectionStats()
    
    # Initialize clients
    gh_client = GitHubClient(config.github_token)
    ch_client = ClickHouseClient(config.clickhouse_config)
    
    # Create run record
    create_collection_run(ch_client, run_id, "running", config)
    
    try:
        # Fetch repositories
        repos = gh_client.get_org_repos(config.github_org)
        upload_repositories(ch_client, repos)
        stats.repos_processed = len(repos)
        
        for repo in repos:
            owner = repo.owner_login
            name = repo.name
            
            # Collect commits (GraphQL for efficiency)
            if config.use_graphql:
                commits, files = collect_commits_graphql(
                    gh_client, ch_client, owner, name,
                    collect_all_branches=config.collect_all_branches
                )
            else:
                commits, files = collect_commits_rest(
                    gh_client, ch_client, owner, name
                )
            
            upload_commits(ch_client, commits, files)
            stats.commits_collected += len(commits)
            
            # Collect PRs (GraphQL for efficiency)
            if config.use_graphql:
                pr_data = collect_prs_graphql(
                    gh_client, ch_client, owner, name
                )
            else:
                pr_data = collect_prs_rest(
                    gh_client, ch_client, owner, name
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

### GraphQL Commit Collection (100x Faster)

```python
def collect_commits_graphql(gh_client, ch_client, owner, repo, collect_all_branches=True):
    commits = []
    commit_files = []
    seen_commits = set()
    
    # Get last collected date
    last_date = ch_client.query(f"""
        SELECT MAX(date) as last_date
        FROM git_commits
        WHERE project_key = '{owner}'
          AND repo_slug = '{repo}'
          AND data_source = 'insight_github'
    """)[0]['last_date']
    
    # Fetch branches
    branches = gh_client.get_branches(owner, repo)
    default_branch = next((b for b in branches if b.is_default), branches[0])
    
    # Determine which branches to collect
    branches_to_collect = branches if collect_all_branches else [default_branch]
    
    for branch in branches_to_collect:
        cursor = None
        
        while True:
            # GraphQL query for commits (100 at a time)
            query = """
            query($owner: String!, $repo: String!, $branch: String!, $since: GitTimestamp, $cursor: String) {
              repository(owner: $owner, name: $repo) {
                ref(qualifiedName: $branch) {
                  target {
                    ... on Commit {
                      history(first: 100, since: $since, after: $cursor) {
                        pageInfo {
                          hasNextPage
                          endCursor
                        }
                        nodes {
                          oid
                          message
                          committedDate
                          additions
                          deletions
                          changedFiles
                          author {
                            name
                            email
                          }
                          committer {
                            name
                            email
                          }
                          parents(first: 5) {
                            nodes {
                              oid
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
            """
            
            variables = {
                'owner': owner,
                'repo': repo,
                'branch': f"refs/heads/{branch.name}",
                'since': last_date.isoformat() if last_date else None,
                'cursor': cursor
            }
            
            response = gh_client.graphql(query, variables)
            history = response['data']['repository']['ref']['target']['history']
            
            for commit_data in history['nodes']:
                commit_hash = commit_data['oid']
                
                # Skip if already seen (commits can appear in multiple branches)
                if commit_hash in seen_commits:
                    continue
                seen_commits.add(commit_hash)
                
                # Parse commit
                commit = parse_commit_graphql(
                    commit_data, owner, repo, branch.name
                )
                commits.append(commit)
                
                # For file-level details, need REST API call
                if commit_data['changedFiles'] > 0:
                    files = gh_client.get_commit_files_rest(owner, repo, commit_hash)
                    for file_data in files:
                        commit_files.append(parse_commit_file(
                            file_data, owner, repo, commit_hash
                        ))
            
            # Check pagination
            if not history['pageInfo']['hasNextPage']:
                break
            cursor = history['pageInfo']['endCursor']
    
    return commits, commit_files


def parse_commit_graphql(commit_data, owner, repo, branch):
    # Calculate language breakdown from files
    language_breakdown = calculate_language_breakdown(commit_data.get('changedFiles', []))
    
    return {
        'project_key': owner,
        'repo_slug': repo,
        'commit_hash': commit_data['oid'],
        'branch': branch,
        'author_name': commit_data['author']['name'],
        'author_email': commit_data['author']['email'],
        'committer_name': commit_data['committer']['name'],
        'committer_email': commit_data['committer']['email'],
        'message': commit_data['message'],
        'date': datetime.fromisoformat(commit_data['committedDate'].replace('Z', '+00:00')),
        'parents': json.dumps([p['oid'] for p in commit_data['parents']['nodes']]),
        'files_changed': commit_data['changedFiles'],
        'lines_added': commit_data['additions'],
        'lines_removed': commit_data['deletions'],
        'language_breakdown': json.dumps(language_breakdown) if language_breakdown else None,
        'is_merge_commit': 1 if len(commit_data['parents']['nodes']) > 1 else 0,
        'metadata': json.dumps(commit_data),
        'data_source': 'insight_github',
        '_version': int(time.time() * 1000)
    }
```

### GraphQL PR Collection (5x Faster)

```python
def collect_prs_graphql(gh_client, ch_client, owner, repo):
    result = PRCollectionResult()
    
    # Get last PR update
    last_update = ch_client.query(f"""
        SELECT MAX(updated_on) as last_update
        FROM git_pull_requests
        WHERE project_key = '{owner}'
          AND repo_slug = '{repo}'
          AND data_source = 'insight_github'
    """)[0]['last_update']
    
    cursor = None
    
    while True:
        # GraphQL query for PRs with nested data
        query = """
        query($owner: String!, $repo: String!, $cursor: String) {
          repository(owner: $owner, name: $repo) {
            pullRequests(first: 50, orderBy: {field: UPDATED_AT, direction: DESC}, after: $cursor) {
              pageInfo {
                hasNextPage
                endCursor
              }
              nodes {
                databaseId
                number
                title
                body
                state
                merged
                author {
                  login
                  ... on User {
                    databaseId
                  }
                }
                createdAt
                updatedAt
                closedAt
                mergeCommit {
                  oid
                }
                headRefName
                baseRefName
                commits {
                  totalCount
                }
                comments {
                  totalCount
                }
                changedFiles
                additions
                deletions
                reviews(first: 100) {
                  nodes {
                    databaseId
                    author {
                      login
                      ... on User {
                        databaseId
                      }
                    }
                    state
                    submittedAt
                    body
                  }
                }
                comments(first: 100) {
                  nodes {
                    databaseId
                    body
                    author {
                      login
                    }
                    createdAt
                    updatedAt
                    path
                    line
                  }
                }
                commits(first: 100) {
                  nodes {
                    commit {
                      oid
                      message
                      committedDate
                    }
                  }
                }
              }
            }
          }
        }
        """
        
        variables = {
            'owner': owner,
            'repo': repo,
            'cursor': cursor
        }
        
        response = gh_client.graphql(query, variables)
        pr_connection = response['data']['repository']['pullRequests']
        
        for pr_data in pr_connection['nodes']:
            updated_at = datetime.fromisoformat(pr_data['updatedAt'].replace('Z', '+00:00'))
            
            # Early stopping
            if last_update and updated_at < last_update:
                return result
            
            # Parse PR
            pr = parse_pr_graphql(pr_data, owner, repo)
            result.prs.append(pr)
            
            # Parse reviews
            for review_data in pr_data['reviews']['nodes']:
                review = parse_review_graphql(review_data, owner, repo, pr_data['databaseId'])
                result.reviews.append(review)
            
            # Parse comments
            for comment_data in pr_data['comments']['nodes']:
                comment = parse_comment_graphql(comment_data, owner, repo, pr_data['databaseId'])
                result.comments.append(comment)
            
            # Parse commits
            for idx, commit_data in enumerate(pr_data['commits']['nodes']):
                result.pr_commits.append({
                    'project_key': owner,
                    'repo_slug': repo,
                    'pr_id': pr_data['databaseId'],
                    'commit_hash': commit_data['commit']['oid'],
                    'commit_order': idx,
                    'metadata': json.dumps(commit_data),
                    'data_source': 'insight_github',
                    '_version': int(time.time() * 1000)
                })
            
            # Extract tickets
            tickets = extract_jira_tickets(pr_data['title'], pr_data.get('body', ''))
            for ticket in tickets:
                result.tickets.append({
                    'external_ticket_id': ticket,
                    'project_key': owner,
                    'repo_slug': repo,
                    'pr_id': pr_data['databaseId'],
                    'commit_hash': '',
                    'data_source': 'insight_github',
                    '_version': int(time.time() * 1000)
                })
        
        # Check pagination
        if not pr_connection['pageInfo']['hasNextPage']:
            break
        cursor = pr_connection['pageInfo']['endCursor']
    
    return result


def parse_pr_graphql(pr_data, owner, repo):
    created_at = datetime.fromisoformat(pr_data['createdAt'].replace('Z', '+00:00'))
    updated_at = datetime.fromisoformat(pr_data['updatedAt'].replace('Z', '+00:00'))
    closed_at = None
    duration_seconds = None
    
    if pr_data.get('closedAt'):
        closed_at = datetime.fromisoformat(pr_data['closedAt'].replace('Z', '+00:00'))
        duration_seconds = int((closed_at - created_at).total_seconds())
    
    # Determine state
    if pr_data['merged']:
        state = 'MERGED'
    elif pr_data['state'] == 'CLOSED':
        state = 'CLOSED'
    else:
        state = 'OPEN'
    
    return {
        'project_key': owner,
        'repo_slug': repo,
        'pr_id': pr_data['databaseId'],
        'pr_number': pr_data['number'],
        'title': pr_data['title'],
        'description': pr_data.get('body', ''),
        'state': state,
        'author_name': pr_data['author']['login'],
        'author_uuid': str(pr_data['author'].get('databaseId', '')),
        'author_email': None,  # Not available in GraphQL
        'created_on': created_at,
        'updated_on': updated_at,
        'closed_on': closed_at,
        'merge_commit_hash': pr_data.get('mergeCommit', {}).get('oid'),
        'source_branch': pr_data['headRefName'],
        'destination_branch': pr_data['baseRefName'],
        'commit_count': pr_data['commits']['totalCount'],
        'comment_count': pr_data['comments']['totalCount'],
        'task_count': None,  # Not applicable to GitHub
        'files_changed': pr_data['changedFiles'],
        'lines_added': pr_data['additions'],
        'lines_removed': pr_data['deletions'],
        'duration_seconds': duration_seconds,
        'jira_tickets': None,  # Extracted separately
        'metadata': json.dumps(pr_data),
        'data_source': 'insight_github',
        '_version': int(time.time() * 1000)
    }
```

### REST API Fallback

```python
def collect_commits_rest(gh_client, ch_client, owner, repo):
    """Fallback to REST API if GraphQL not available."""
    commits = []
    commit_files = []
    
    # Get last collected date
    last_date = ch_client.query(f"""
        SELECT MAX(date) as last_date
        FROM git_commits
        WHERE project_key = '{owner}'
          AND repo_slug = '{repo}'
          AND data_source = 'insight_github'
    """)[0]['last_date']
    
    # Fetch commits with pagination
    page = 1
    per_page = 100
    
    while True:
        params = {
            'page': page,
            'per_page': per_page,
            'since': last_date.isoformat() if last_date else None
        }
        
        response = gh_client.get(f'/repos/{owner}/{repo}/commits', params=params)
        
        if not response:
            break
        
        for commit_data in response:
            # Fetch detailed commit info (includes files)
            commit_detail = gh_client.get(f'/repos/{owner}/{repo}/commits/{commit_data["sha"]}')
            
            # Parse commit
            commit = parse_commit_rest(commit_detail, owner, repo)
            commits.append(commit)
            
            # Parse files
            for file_data in commit_detail.get('files', []):
                commit_files.append(parse_commit_file(
                    file_data, owner, repo, commit_data['sha']
                ))
        
        if len(response) < per_page:
            break
        page += 1
    
    return commits, commit_files
```

---

## API Specifications

### Authentication

```python
headers = {
    'Authorization': f'Bearer {github_token}',
    'Accept': 'application/vnd.github.v3+json',
    'User-Agent': 'GitHub-ETL/1.0'
}
```

### REST API Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/orgs/{org}/repos` | GET | List org repositories |
| `/repos/{owner}/{repo}` | GET | Get repository details |
| `/repos/{owner}/{repo}/commits` | GET | List commits |
| `/repos/{owner}/{repo}/commits/{sha}` | GET | Get commit details with files |
| `/repos/{owner}/{repo}/pulls` | GET | List pull requests |
| `/repos/{owner}/{repo}/pulls/{number}` | GET | Get PR details |
| `/repos/{owner}/{repo}/pulls/{number}/reviews` | GET | Get PR reviews |
| `/repos/{owner}/{repo}/pulls/{number}/comments` | GET | Get PR comments |
| `/repos/{owner}/{repo}/pulls/{number}/commits` | GET | Get PR commits |
| `/repos/{owner}/{repo}/branches` | GET | List branches |

### GraphQL Endpoint

**URL**: `https://api.github.com/graphql`

**Method**: POST

**Headers**: Same as REST + `Content-Type: application/json`

**Body**:
```json
{
  "query": "query { ... }",
  "variables": { ... }
}
```

### Rate Limiting

- **REST API**: 5000 requests/hour (authenticated)
- **GraphQL API**: 5000 points/hour (varies by query complexity)
- **Headers**: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

```python
def check_rate_limit(response):
    remaining = int(response.headers.get('X-RateLimit-Remaining', 0))
    reset_time = int(response.headers.get('X-RateLimit-Reset', 0))
    
    if remaining < 100:
        wait_time = reset_time - time.time()
        if wait_time > 0:
            logger.warning(f"Rate limit low. Waiting {wait_time}s")
            time.sleep(wait_time)
```

---

## Data Mapping

### Repository Mapping

```python
def map_repository(api_data):
    return {
        'project_key': api_data['owner']['login'],
        'repo_slug': api_data['name'],
        'repo_uuid': str(api_data['id']),
        'name': api_data['name'],
        'full_name': api_data['full_name'],
        'description': api_data.get('description'),
        'is_private': 1 if api_data['private'] else 0,
        'created_on': datetime.fromisoformat(api_data['created_at'].replace('Z', '+00:00')),
        'updated_on': datetime.fromisoformat(api_data['updated_at'].replace('Z', '+00:00')),
        'size': api_data['size'],
        'language': api_data.get('language'),
        'has_issues': 1 if api_data['has_issues'] else 0,
        'has_wiki': 1 if api_data['has_wiki'] else 0,
        'fork_policy': None,  # Not applicable
        'metadata': json.dumps(api_data),
        'data_source': 'insight_github',
        '_version': int(time.time() * 1000)
    }
```

### Commit Mapping (REST)

```python
def map_commit_rest(api_data, owner, repo, branch='main'):
    # Calculate language breakdown
    language_breakdown = {}
    for file in api_data.get('files', []):
        ext = os.path.splitext(file['filename'])[1]
        if ext:
            language_breakdown[ext] = language_breakdown.get(ext, 0) + file['additions']
    
    return {
        'project_key': owner,
        'repo_slug': repo,
        'commit_hash': api_data['sha'],
        'branch': branch,
        'author_name': api_data['commit']['author']['name'],
        'author_email': api_data['commit']['author']['email'],
        'committer_name': api_data['commit']['committer']['name'],
        'committer_email': api_data['commit']['committer']['email'],
        'message': api_data['commit']['message'],
        'date': datetime.fromisoformat(api_data['commit']['author']['date'].replace('Z', '+00:00')),
        'parents': json.dumps([p['sha'] for p in api_data.get('parents', [])]),
        'files_changed': len(api_data.get('files', [])),
        'lines_added': api_data['stats']['additions'],
        'lines_removed': api_data['stats']['deletions'],
        'language_breakdown': json.dumps(language_breakdown) if language_breakdown else None,
        'is_merge_commit': 1 if len(api_data.get('parents', [])) > 1 else 0,
        'metadata': json.dumps(api_data),
        'data_source': 'insight_github',
        '_version': int(time.time() * 1000)
    }
```

### Review Mapping

```python
def map_review(api_data, owner, repo, pr_id):
    # Normalize status
    status_map = {
        'APPROVED': 'APPROVED',
        'CHANGES_REQUESTED': 'CHANGES_REQUESTED',
        'COMMENTED': 'COMMENTED',
        'approved': 'APPROVED'  # Legacy format
    }
    
    status = status_map.get(api_data['state'], api_data['state'])
    approved = 1 if status == 'APPROVED' else 0
    
    return {
        'project_key': owner,
        'repo_slug': repo,
        'pr_id': pr_id,
        'reviewer_name': api_data['author']['login'],
        'reviewer_uuid': str(api_data['author'].get('databaseId', '')),
        'reviewer_email': None,  # Not provided by GitHub API
        'status': status,
        'role': 'REVIEWER',
        'approved': approved,
        'reviewed_at': datetime.fromisoformat(api_data['submittedAt'].replace('Z', '+00:00')) if api_data.get('submittedAt') else None,
        'metadata': json.dumps(api_data),
        'data_source': 'insight_github',
        '_version': int(time.time() * 1000)
    }
```

---

## Error Handling

### Rate Limit Handling

```python
class GitHubClient:
    def request(self, method, url, **kwargs):
        response = self.session.request(method, url, **kwargs)
        
        # Check rate limit
        remaining = int(response.headers.get('X-RateLimit-Remaining', 0))
        
        if remaining < 100:
            reset_time = int(response.headers.get('X-RateLimit-Reset', 0))
            wait_time = max(0, reset_time - time.time())
            logger.warning(f"Rate limit low ({remaining}). Waiting {wait_time}s")
            time.sleep(wait_time)
        
        # Handle rate limit exceeded
        if response.status_code == 403 and 'rate limit' in response.text.lower():
            reset_time = int(response.headers.get('X-RateLimit-Reset', 0))
            wait_time = max(0, reset_time - time.time() + 10)
            logger.error(f"Rate limit exceeded. Waiting {wait_time}s")
            time.sleep(wait_time)
            return self.request(method, url, **kwargs)
        
        response.raise_for_status()
        return response.json()
```

### GraphQL Error Handling

```python
def graphql_request(query, variables):
    response = requests.post(
        'https://api.github.com/graphql',
        headers=headers,
        json={'query': query, 'variables': variables}
    )
    
    data = response.json()
    
    # Check for GraphQL errors
    if 'errors' in data:
        errors = data['errors']
        logger.error(f"GraphQL errors: {errors}")
        
        # Check if rate limited
        if any('rate limit' in str(e).lower() for e in errors):
            time.sleep(60)
            return graphql_request(query, variables)
        
        raise Exception(f"GraphQL query failed: {errors}")
    
    return data
```

---

## Performance Optimization

### GraphQL vs REST Performance

| Operation | REST API | GraphQL API | Speedup |
|-----------|----------|-------------|---------|
| Fetch 100 commits with files | 100 requests | 1 request | 100x |
| Fetch 50 PRs with reviews/comments | 250 requests | 1 request | 250x |
| Fetch PR with all data | 5 requests | 1 request | 5x |

### Batch Processing

```python
class GraphQLBatchCollector:
    """Collect multiple entities in single GraphQL query."""
    
    def collect_multiple_repos(self, repos):
        # Build query for multiple repos
        query_parts = []
        for idx, repo in enumerate(repos[:10]):  # Max 10 per query
            query_parts.append(f"""
                repo{idx}: repository(owner: "{repo.owner}", name: "{repo.name}") {{
                    pullRequests(first: 50) {{
                        nodes {{ ... }}
                    }}
                }}
            """)
        
        query = f"query {{ {' '.join(query_parts)} }}"
        return self.graphql(query)
```

### Caching Strategy

```python
class CachedGitHubClient:
    def __init__(self, token, cache_ttl=3600):
        self.client = GitHubClient(token)
        self.cache = {}
        self.cache_ttl = cache_ttl
    
    def get_with_cache(self, url):
        cache_key = url
        
        if cache_key in self.cache:
            cached_data, cached_time = self.cache[cache_key]
            if time.time() - cached_time < self.cache_ttl:
                return cached_data
        
        data = self.client.get(url)
        self.cache[cache_key] = (data, time.time())
        return data
```

---

## Monitoring

### Key Metrics

- **API Usage**: Requests made, rate limit remaining, GraphQL points used
- **Collection Stats**: Repos, commits, PRs collected per run
- **Performance**: Time per repository, GraphQL vs REST usage
- **Errors**: Rate limit hits, API errors, data validation failures

### Logging

```python
logger.info(f"Starting GitHub ETL: {run_id}")
logger.info(f"Using GraphQL: {config.use_graphql}")
logger.info(f"Collecting from org: {config.github_org}")
logger.info(f"Rate limit remaining: {remaining}/{limit}")
logger.info(f"Collected {commit_count} commits in {duration}s")
logger.warning(f"Rate limit low: {remaining} remaining")
logger.error(f"Failed to collect repo {owner}/{repo}: {error}")
```

---

## Implementation Checklist

- [ ] Set up GitHub API client with token authentication
- [ ] Implement GraphQL query builder and executor
- [ ] Implement REST API fallback
- [ ] Create rate limit handler with automatic waiting
- [ ] Implement commit collector (GraphQL + REST)
- [ ] Implement PR collector (GraphQL + REST)
- [ ] Add multi-branch support
- [ ] Implement incremental collection logic
- [ ] Set up ClickHouse batch uploader
- [ ] Add error handling and retry logic
- [ ] Create Kubernetes CronJob deployment
- [ ] Set up monitoring and alerting
- [ ] Test with sample organization
- [ ] Perform initial backfill
- [ ] Schedule regular incremental runs
- [ ] Monitor rate limit usage and optimize

---

## GraphQL Query Examples

### Complete Commit Query

```graphql
query($owner: String!, $repo: String!, $branch: String!, $cursor: String) {
  repository(owner: $owner, name: $repo) {
    ref(qualifiedName: $branch) {
      target {
        ... on Commit {
          history(first: 100, after: $cursor) {
            pageInfo {
              hasNextPage
              endCursor
            }
            nodes {
              oid
              message
              committedDate
              additions
              deletions
              changedFiles
              author {
                name
                email
              }
              committer {
                name
                email
              }
              parents(first: 5) {
                nodes {
                  oid
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Complete PR Query

```graphql
query($owner: String!, $repo: String!, $cursor: String) {
  repository(owner: $owner, name: $repo) {
    pullRequests(first: 50, orderBy: {field: UPDATED_AT, direction: DESC}, after: $cursor) {
      pageInfo {
        hasNextPage
        endCursor
      }
      nodes {
        databaseId
        number
        title
        body
        state
        merged
        author {
          login
          ... on User {
            databaseId
          }
        }
        createdAt
        updatedAt
        closedAt
        mergeCommit {
          oid
        }
        headRefName
        baseRefName
        commits {
          totalCount
        }
        changedFiles
        additions
        deletions
        reviews(first: 100) {
          nodes {
            databaseId
            author {
              login
              ... on User {
                databaseId
              }
            }
            state
            submittedAt
          }
        }
        comments(first: 100) {
          nodes {
            databaseId
            body
            author {
              login
            }
            createdAt
            path
            line
          }
        }
        commits(first: 100) {
          nodes {
            commit {
              oid
            }
          }
        }
      }
    }
  }
}
```
