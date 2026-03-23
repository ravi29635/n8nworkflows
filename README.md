Jira & Confluence Keyword Intelligence Workflow

> An automated n8n workflow that searches Jira and Confluence for configured keywords, uses **Claude AI** to generate intelligent summaries, and delivers a weekly intelligence report via Confluence page, Jira ticket, and email.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Workflow Architecture](#workflow-architecture)
- [Node-by-Node Explanation](#node-by-node-explanation)
- [Data Flow Diagram](#data-flow-diagram)
- [Environment Variables](#environment-variables)
- [Required Credentials](#required-credentials)
- [Keywords Configuration](#keywords-configuration)
- [Outputs](#outputs)
- [Setup Instructions](#setup-instructions)
- [Scheduling](#scheduling)
- [Error Handling](#error-handling)

---

## Overview

This workflow runs every weekday at **8:00 AM** (or on-demand via webhook) and:

1. Searches Jira and Confluence for a predefined list of strategic keywords
2. Normalises and aggregates the results
3. Sends the data to **Claude AI (claude-opus-4-5)** for intelligent summarisation
4. Builds a full HTML intelligence report
5. Publishes the report to **Confluence**, creates a **Jira ticket**, and sends it by **email**

---

## Workflow Architecture

```
[Schedule / Webhook Trigger]
        │
        ▼
[🔑 Prepare Keywords]
        │
   ┌────┴────┐
   ▼         ▼
[Jira]  [Confluence]   ← parallel searches per keyword
   │         │
   ▼         ▼
[Process] [Process]    ← normalise results
   │         │
   └────┬────┘
        ▼
[🔀 Merge Jira + Confluence]
        │
        ▼
[📊 Aggregate All Data]
        │
        ▼
[💬 Prepare Claude AI Prompts]
        │
        ▼
[🤖 Claude AI – Generate Summary]
        │
        ▼
[📝 Extract AI Summary]
        │
        ▼
[📄 Build Final Report]
        │
   ┌────┼────┐
   ▼    ▼    ▼
[Confluence] [Jira Ticket] [Email]
        │
        ▼
[✅ Webhook Response]
```

---

## Node-by-Node Explanation

### 1. ⏰ Schedule Trigger (Weekdays 8AM)
- **Type:** `n8n-nodes-base.scheduleTrigger`
- **Cron:** `0 8 * * 1-5` — fires Monday–Friday at 08:00
- Starts the automated daily run without any manual intervention

### 2. 🔗 Manual Webhook Trigger
- **Type:** `n8n-nodes-base.webhook`
- **Path:** `POST /dl-manual-run`
- Allows on-demand triggering from external systems or manual testing
- Both this and the schedule trigger feed into the same downstream pipeline

### 3. 🔑 Prepare Keywords
- **Type:** Code node (JavaScript)
- Defines the list of strategic keywords to search:
  - `project-A`
  - `Project-B`
  - `Security`
  - `Security Architecture`
- Calculates a `sinceDate` (7 days ago) to filter only recent results
- Emits one item per keyword so downstream nodes run **in parallel** for each

### 4. 🔍 Jira Search by Keyword
- **Type:** HTTP Request → Jira REST API v3
- **Endpoint:** `GET {JIRA_BASE_URL}/rest/api/3/search`
- **Auth:** Basic Auth (email + API token)
- Uses **JQL query:** `text ~ "{keyword}" AND updated >= "{sinceDate}" ORDER BY updated DESC`
- Fetches up to **50 issues** with fields: summary, description, status, assignee, reporter, priority, type, labels, components, created, updated, comments, attachments

### 5. 🔍 Confluence Search by Keyword
- **Type:** HTTP Request → Confluence REST API
- **Endpoint:** `GET {CONFLUENCE_BASE_URL}/rest/api/content/search`
- **Auth:** Basic Auth (email + API token)
- Uses **CQL query:** `text ~ "{keyword}" AND lastModified >= "{sinceDate}" ORDER BY lastModified DESC`
- Fetches up to **50 pages** with body content, space, version, ancestors, and labels

### 6. ⚙️ Process Jira Results
- **Type:** Code node (JavaScript)
- Normalises raw Jira API response into a clean, consistent structure
- Extracts: `id`, `key`, `type`, `summary`, `description` (truncated to 500 chars), `status`, `priority`, `assignee`, `reporter`, `labels`, `components`, `created`, `updated`, `commentCount`, `url`
- Returns `{ keyword, source: 'jira', items[], count }` — empty array if no results

### 7. ⚙️ Process Confluence Results
- **Type:** Code node (JavaScript)
- Strips HTML tags from page body content using regex cleanup
- Normalises into: `id`, `type`, `title`, `space`, `spaceKey`, `excerpt` (600 chars), `labels`, `ancestors` (breadcrumb path), `lastModified`, `modifiedBy`, `url`
- Returns `{ keyword, source: 'confluence', items[], count }`

### 8. 🔀 Merge Jira + Confluence
- **Type:** `n8n-nodes-base.merge` (mergeByPosition)
- Combines the Jira and Confluence results for the same keyword into a single data stream
- Input 0 = Jira results | Input 1 = Confluence results

### 9. 📊 Aggregate All Data
- **Type:** Code node (JavaScript)
- Groups all per-keyword results into a unified object keyed by keyword
- Computes totals: `totalJiraIssues`, `totalConfluencePages`
- Produces a `keywordBreakdown` map used by Claude for analysis

### 10. 💬 Prepare Claude AI Prompts
- **Type:** Code node (JavaScript)
- Builds a rich, structured prompt per keyword for Claude, including:
  - Role context: *"You are a technical program manager at DeLaval..."*
  - Instructions to produce: executive summary, key themes, active work items, next actions, meeting agenda
  - The top 10 Jira issues with status, priority, assignee, description
  - The top 10 Confluence pages with space, path, last modifier, excerpt
- If no results exist for any keyword, produces a graceful empty-state item

### 11. 🤖 Claude AI – Generate Summary
- **Type:** HTTP Request → Anthropic API
- **Endpoint:** `POST https://api.anthropic.com/v1/messages`
- **Model:** `claude-opus-4-5`
- **Max tokens:** 2000
- **Auth:** API Key in `x-api-key` header
- Sends the prompt built in the previous node and receives structured AI analysis

### 12. 📝 Extract AI Summary
- **Type:** Code node (JavaScript)
- Extracts `content[0].text` from Claude's API response
- Pairs the AI-generated text back with keyword metadata for use in the final report

### 13. 📄 Build Final Report
- **Type:** Code node (JavaScript)
- Collects all per-keyword AI summaries and assembles two formats:
  - **Markdown meeting notes** (`meetingNotes`) — suitable for Jira ticket body
  - **Full HTML report** (`htmlReport`) — styled with DeLaval branding (navy/blue gradient header, stat cards, collapsible Jira/Confluence tables)
- Report includes stats: keywords tracked, total Jira issues, total Confluence pages
- Title format: `DeLaval Intelligence Report — {weekday}, {date}`

### 14. 📤 Publish to Confluence
- **Type:** HTTP Request → Confluence REST API
- **Method:** `POST {CONFLUENCE_BASE_URL}/rest/api/content`
- Creates a new Confluence page under a configured parent page
- Publishes the full HTML report to the designated reporting space

### 15. 🎫 Create Jira Report Ticket
- **Type:** HTTP Request → Jira REST API v3
- **Method:** `POST {JIRA_BASE_URL}/rest/api/3/issue`
- Creates a Jira task with:
  - Summary: `Weekly Intelligence Report — {date}`
  - Description: First 2000 chars of meeting notes
  - Labels: `intelligence-report`, `automated`

### 16. 📧 Email Intelligence Report
- **Type:** `n8n-nodes-base.emailSend`
- Sends the full HTML report via SMTP
- From/To addresses configured via environment variables
- Subject = report title

### 17. ✅ Webhook Response
- **Type:** `n8n-nodes-base.respondToWebhook`
- Returns a JSON success response when triggered via webhook:
  ```json
  { "success": true, "message": "Report generated successfully", "title": "...", "keywords": [...] }
  ```

### 18. 🚨 Error Handler
- **Type:** Code node (JavaScript)
- Logs failures with timestamp, error message, and failing node name
- Connected as fallback for critical failures in the pipeline

---

## Environment Variables

Configure these in your n8n instance under **Settings → Variables**:

| Variable | Description | Example |
|---|---|---|
| `JIRA_BASE_URL` | Your Jira instance base URL | `https://yourcompany.atlassian.net` |
| `CONFLUENCE_BASE_URL` | Your Confluence instance base URL | `https://yourcompany.atlassian.net/wiki` |
| `JIRA_PROJECT_KEY` | Project key for created report tickets | `INTEL` |
| `CONFLUENCE_REPORT_SPACE_KEY` | Space key where reports are published | `REPORTS` |
| `CONFLUENCE_PARENT_PAGE_ID` | Page ID to nest reports under | `123456789` |
| `NOTIFICATION_FROM_EMAIL` | Sender email address | `noreply@company.com` |
| `NOTIFICATION_TO_EMAILS` | Recipient email(s), comma-separated | `team@company.com` |

---

## Required Credentials

Set these up in n8n under **Credentials**:

| Credential ID | Type | Used By | Notes |
|---|---|---|---|
| `jira-basic-auth` | HTTP Basic Auth | Jira Search, Create Ticket | Email + Jira API token |
| `confluence-basic-auth` | HTTP Basic Auth | Confluence Search, Publish | Email + Confluence API token |
| `anthropic-api-key` | HTTP Header Auth | Claude AI node | Header name: `x-api-key` |
| `smtp-credentials` | SMTP | Email Send node | Your mail server config |

> **Getting Atlassian API tokens:** Go to [id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens) and create a token. Use your Atlassian email as username and the token as password for Basic Auth.

---

## Keywords Configuration

Keywords are defined in the **🔑 Prepare Keywords** node. To modify them, edit the `keywords` array in the node's JavaScript code:

```javascript
const keywords = [
  "Project-A",
  "Project-B",
  "Security",
  "Security Architecture",
];
```

The workflow searches both Jira and Confluence for each keyword independently.

---

## Outputs

Each run produces three outputs:

| Output | Description |
|---|---|
| **Confluence Page** | Styled HTML report published under the configured parent page |
| **Jira Ticket** | A `Task` issue tagged `intelligence-report` and `automated` |
| **Email** | Full HTML report sent to configured recipients |

---

## Setup Instructions

1. **Import the workflow** into your n8n instance:
   - Go to **Workflows → Import from File**
   - Select `jira_confluence_workflow.json`

2. **Create credentials** for Jira, Confluence, Anthropic, and SMTP as listed above

3. **Set environment variables** for all the variables listed in the table above

4. **Activate the workflow** — it will run automatically at 8AM on weekdays

5. **Test manually** by calling the webhook:
   ```bash
   curl -X POST https://your-n8n-instance.com/webhook/delaval-manual-run
   ```

---

## Scheduling

The workflow runs on cron `0 8 * * 1-5` — every Monday through Friday at 08:00 in the timezone configured in your n8n instance. To change the time, edit the **⏰ Schedule Trigger** node's cron expression.

---

## Error Handling

The **🚨 Error Handler** node captures failures and logs:
- Error message
- Timestamp
- Failing node name

To receive error notifications, connect the error handler to the email node or configure n8n's built-in error workflow setting (`settings.errorWorkflow`).

---

*Generated workflow tags: `n8n`, `Intelligence`, `Jira`, `Confluence`*

