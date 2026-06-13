# IT Support Knowledge Base Automation

An n8n workflow system that automatically analyzes resolved Jira IT support tickets, detects recurring issues, and generates knowledge base articles in Confluence — without any human intervention.

---

## Overview

When support teams resolve the same type of issue repeatedly, that knowledge rarely gets captured. This project automates the full pipeline: from scanning Jira for recently resolved tickets, to publishing a structured knowledge base (KB) article on Confluence when a pattern is detected.

The system runs on a weekly schedule and is composed of 4 linked sub-workflows plus a shared logger.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    0 - Main Orchestrator                     │
│  (Weekly trigger → chains sub-workflows → email notification)│
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────▼──────────────┐
              │  1 - Fetch Jira Tickets      │
              │  Retrieves resolved tickets  │
              │  + comments from Jira        │
              └──────────────┬──────────────┘
                             │
              ┌──────────────▼──────────────┐
              │  2 - Check KB &             │
              │  Find Similar Tickets        │
              │  Searches Confluence for     │
              │  existing KB articles, then  │
              │  clusters unmatched tickets  │
              └──────────────┬──────────────┘
                             │ (if 3+ similar tickets found)
              ┌──────────────▼──────────────┐
              │  3 - Write KB Article        │
              │  to Confluence               │
              │  Generates & publishes a     │
              │  structured page via LLM     │
              └─────────────────────────────┘
```

A shared **Logger** sub-workflow records every execution to Google Sheets and sends Discord alerts on errors.

---

## Workflows

### `0 - Main Orchestrator`
The entry point. Triggered weekly (or manually for testing). Calls each sub-workflow in sequence, routes the output, and sends an email notification when a new KB article is created.

<img width="2157" height="556" alt="image" src="https://github.com/user-attachments/assets/64ec7f05-ce05-4050-bc65-cb669ed23b6e" />


### `1 - Fetch Jira Tickets`
Queries Jira for tickets that are:
- In a resolved/done/closed status
- Updated in the last 60 days
- Not yet linked to a knowledge base article (`customfield_10108` is empty or set to "NON")

For each ticket, it fetches all comments and assembles a structured JSON object (ticket fields + comments) passed to the next workflow.

<img width="2126" height="538" alt="image" src="https://github.com/user-attachments/assets/50cf91cd-bf2b-4cd8-97bc-7a633f33289d" />


### `2 - Check KB & Find Similar Tickets`
The core intelligence layer. For each ticket:
1. **Extracts search keywords** from the ticket summary and description (with French stopword filtering)
2. **Searches Confluence** (space `BDC`) for existing KB pages matching those keywords via CQL
3. **Calls an LLM** (GPT-4.1-mini) to determine if an existing article already covers the issue
4. If a match is found → updates the Jira custom field to mark the ticket as covered
5. If no match → aggregates all unmatched tickets and calls a second LLM to detect if at least 3 tickets share the same root cause
6. If a cluster of 3+ similar tickets is found → returns `action: "creer_fiche"` to the orchestrator

<img width="1513" height="430" alt="image" src="https://github.com/user-attachments/assets/856bff82-e9a8-4b51-9981-5e1b90f45b30" />
<img width="1091" height="433" alt="image" src="https://github.com/user-attachments/assets/0376b5d3-c59a-4b8f-905d-b9188f3a2f18" />


### `3 - Write KB Article to Confluence`
Receives the identified ticket cluster and:
1. Fetches a Confluence page template
2. Sends tickets + template to an LLM (GPT-4o-mini) with a prompt to act as a Knowledge Manager and write a structured article in HTML
3. Parses the JSON response (title + HTML content)
4. Creates the page in Confluence via REST API (space `BDC`)

<img width="1979" height="519" alt="image" src="https://github.com/user-attachments/assets/0f64f9d5-0bd5-4f81-9425-175861830b74" />


### `Logger - Execution Tracker`
A reusable utility sub-workflow called by all other workflows. Logs each execution (workflow name, execution ID, status, error message) to a Google Sheet, and posts a Discord webhook message when `status = error`.

<img width="1708" height="383" alt="image" src="https://github.com/user-attachments/assets/96eebd17-4904-4a14-8c52-ca9475aeea6a" />


---

## Tech Stack

| Component | Tool |
|---|---|
| Workflow automation | [n8n](https://n8n.io) |
| Ticket management | Jira (Cloud) |
| Knowledge base | Confluence (Cloud) |
| LLM | OpenAI GPT-4o-mini / GPT-4.1-mini |
| Logging | Google Sheets + Discord webhook |
| Notifications | Gmail |

---

## Setup

### Prerequisites
- n8n instance (self-hosted or cloud)
- Jira Cloud account with API credentials
- Confluence Cloud account with API credentials
- OpenAI API key
- Google Sheets OAuth2 credentials
- Gmail OAuth2 credentials
- Discord webhook URL (for error alerts)

### Installation

1. Import all 5 JSON workflow files into your n8n instance

2. Configure credentials for each node (Jira, Confluence, OpenAI, Google Sheets, Gmail).

3. In the Logger workflow, replace the Google Sheet ID with your own sheet. The sheet must have an `logs` tab with these columns: `timestamp`, `workflow_name`, `execution_id`, `status`, `error_message`.

4. Update the following values to match your environment:

| Location | Field | Default value |
|---|---|---|
| Workflow 1 & 2 | Jira JQL filter | `updatedDate >= -60d` |
| Workflow 2 | Confluence space key | `BDC` |
| Workflow 3 | Confluence base URL | `https://mainiaproject.atlassian.net/` |
| Workflow 3 | Template ID | `1212417` |
| Workflow 0 | Notification email |  |

5. Activate the workflows, starting with the sub-workflows before the orchestrator.

---

## How It Works — End to End

```
Every Monday (weekly trigger)
  │
  ├─► Fetch all resolved Jira tickets from the last 60 days (not yet KB-tagged)
  │
  ├─► For each ticket:
  │     ├─ Extract keywords from summary + description
  │     ├─ Search Confluence for matching KB articles
  │     ├─ LLM decides: does an existing article cover this ticket?
  │     │     ├─ YES → tag the Jira ticket as "KB found" (customfield_10108)
  │     │     └─ NO  → add to "no KB" pool
  │
  ├─► Aggregate all tickets in the "no KB" pool
  │     └─ LLM decides: are there 3+ tickets with the same root cause?
  │           ├─ NO  → stop, nothing to create
  │           └─ YES → identify the cluster and the common issue
  │
  ├─► LLM writes a structured KB article based on the real ticket data
  │     └─ Published to Confluence using a standard template
  │
  └─► Send email notification to Knowledge Managers
        └─ Log execution result to Google Sheets
```

---

## Project Context

This is a proof-of-concept built in 2 days as the capstone project of an agentic AI training course. It uses test data and is intended to demonstrate how an orchestrator + specialized sub-agents pattern can be applied to a real-world IT support use case — not for production use.
