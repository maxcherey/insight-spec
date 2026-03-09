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

| Source | Spec | Status |
|--------|------|--------|
| Git (unified schema) | [`git/README.md`](git/README.md) | Draft |
| GitHub | [`git/github.md`](git/github.md) | Draft |
| Bitbucket | [`git/bitbucket.md`](git/bitbucket.md) | Draft |
| GitLab | [`git/gitlab.md`](git/gitlab.md) | Draft |

### Task Tracking

| Source | Spec | Status |
|--------|------|--------|
| YouTrack | [`task-tracking/youtrack.md`](task-tracking/youtrack.md) | Proposed |
| Jira | [`task-tracking/jira.md`](task-tracking/jira.md) | Proposed |

### Communication

| Source | Spec | Status |
|--------|------|--------|
| Microsoft 365 | [`m365.md`](m365.md) | Proposed |
| Zulip | [`zulip.md`](zulip.md) | Proposed |

### AI Dev Tools

| Source | Spec | Status |
|--------|------|--------|
| Cursor | [`ai-dev/cursor.md`](ai-dev/cursor.md) | Proposed |
| Windsurf | [`ai-dev/windsurf.md`](ai-dev/windsurf.md) | Proposed |
| GitHub Copilot | [`ai-dev/github-copilot.md`](ai-dev/github-copilot.md) | Proposed |

### AI Tools

| Source | Spec | Status |
|--------|------|--------|
| Claude API | [`ai/claude-api.md`](ai/claude-api.md) | Proposed |
| Claude Team Plan | [`ai/claude-team.md`](ai/claude-team.md) | Proposed |
| OpenAI API | [`ai/openai-api.md`](ai/openai-api.md) | Proposed |
| ChatGPT Team | [`ai/chatgpt-team.md`](ai/chatgpt-team.md) | Proposed |

### HR / Directory

| Source | Spec | Status |
|--------|------|--------|
| BambooHR | [`hr-directory/bamboohr.md`](hr-directory/bamboohr.md) | Proposed |
| Workday | [`hr-directory/workday.md`](hr-directory/workday.md) | Proposed |
| LDAP / Active Directory | [`hr-directory/ldap.md`](hr-directory/ldap.md) | Proposed |

### CRM

| Source | Spec | Status |
|--------|------|--------|
| HubSpot | [`crm/hubspot.md`](crm/hubspot.md) | Proposed |
| Salesforce | [`crm/salesforce.md`](crm/salesforce.md) | Proposed |

### Quality / Testing

| Source | Spec | Status |
|--------|------|--------|
| Allure TestOps | [`allure.md`](allure.md) | Proposed |

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
