# Connector Specifications

> Version 1.0 — March 2026

Per-source deep-dive specifications for Constructor Insight connectors. Each file expands on the corresponding source in [`../CONNECTORS_REFERENCE.md`](../CONNECTORS_REFERENCE.md) with full table schemas, identity mapping, Silver/Gold pipeline notes, and open questions.

<!-- toc -->

- [Index](#index)
  - [Version Control](#version-control)
  - [Task Tracking](#task-tracking)
  - [Communication](#communication)
  - [AI Dev Tools](#ai-dev-tools)
  - [AI Tools](#ai-tools)
  - [HR / Directory](#hr-directory)
  - [CRM](#crm)
  - [Quality / Testing](#quality-testing)
- [Unified Streams](#unified-streams)
- [How to Use](#how-to-use)

<!-- /toc -->

---

## Index

### Version Control

| # | Source | Spec | Status |
|---|--------|------|--------|
| — | Git (unified schema) | [`git/README.md`](git/README.md) | ✓ Done |
| 1 | GitHub | [`git/github.md`](git/github.md) | ✓ Done |
| 2 | Bitbucket | [`git/bitbucket.md`](git/bitbucket.md) | ✓ Done |
| 3 | GitLab | [`git/gitlab.md`](git/gitlab.md) | ✓ Done |

### Task Tracking

| # | Source | Spec | Status |
|---|--------|------|--------|
| 4 | YouTrack | [`task-tracking/youtrack.md`](task-tracking/youtrack.md) | ✓ Done |
| 5 | Jira | [`task-tracking/jira.md`](task-tracking/jira.md) | ✓ Done |

### Communication

| # | Source | Spec | Status |
|---|--------|------|--------|
| 6 | Microsoft 365 | [`m365.md`](m365.md) | ✓ Done |
| 7 | Zulip | [`zulip.md`](zulip.md) | ✓ Done |

### AI Dev Tools

| # | Source | Spec | Status |
|---|--------|------|--------|
| 8 | Cursor | [`ai-dev/cursor.md`](ai-dev/cursor.md) | ✓ Done |
| 9 | Windsurf | [`ai-dev/windsurf.md`](ai-dev/windsurf.md) | ✓ Done |
| 15 | GitHub Copilot | [`ai-dev/github-copilot.md`](ai-dev/github-copilot.md) | ✓ Done |

### AI Tools

| # | Source | Spec | Status |
|---|--------|------|--------|
| 13 | Claude API | [`ai/claude-api.md`](ai/claude-api.md) | ✓ Done |
| 14 | Claude Team Plan | [`ai/claude-team.md`](ai/claude-team.md) | ✓ Done |
| 18 | OpenAI API | [`ai/openai-api.md`](ai/openai-api.md) | ✓ Done |
| 19 | ChatGPT Team | [`ai/chatgpt-team.md`](ai/chatgpt-team.md) | ✓ Done |

### HR / Directory

| # | Source | Spec | Status |
|---|--------|------|--------|
| 10 | BambooHR | [`hr-directory/bamboohr.md`](hr-directory/bamboohr.md) | ✓ Done |
| 11 | Workday | [`hr-directory/workday.md`](hr-directory/workday.md) | ✓ Done |
| 12 | LDAP / Active Directory | [`hr-directory/ldap.md`](hr-directory/ldap.md) | ✓ Done |

### CRM

| # | Source | Spec | Status |
|---|--------|------|--------|
| 16 | HubSpot | [`crm/hubspot.md`](crm/hubspot.md) | ✓ Done |
| 17 | Salesforce | [`crm/salesforce.md`](crm/salesforce.md) | ✓ Done |

### Quality / Testing

| # | Source | Spec | Status |
|---|--------|------|--------|
| 20 | Allure TestOps | [`allure.md`](allure.md) | ✓ Done |

---

## Unified Streams

| Stream | Sources | Spec |
|--------|---------|------|
| `class_communication_events` | M365 (Email + Teams) + Zulip | Documented in [`../CONNECTORS_REFERENCE.md`](../CONNECTORS_REFERENCE.md#unified-stream-1-classcommunicationevents) |
| Task Tracker (Silver → Gold) | YouTrack + Jira | Documented in [`../CONNECTORS_REFERENCE.md`](../CONNECTORS_REFERENCE.md#unified-stream-2-task-tracker-silver--gold) |

---

## How to Use

- **Main reference** — [`../CONNECTORS_REFERENCE.md`](../CONNECTORS_REFERENCE.md) is the canonical index of all Bronze table schemas and the Bronze → Silver → Gold pipeline overview.
- **Per-source specs** (this directory) — expand on individual sources with additional detail: complete field lists, API notes, identity mapping, Silver channel mappings, and open questions.
- **Generate a new spec** — `/cypilot-generate Connector spec for {Source Name}`
