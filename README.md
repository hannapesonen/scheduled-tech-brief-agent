# scheduled-tech-brief-agent
An AI-powered daily technology briefing solution built with Azure AI Foundry, Azure Logic Apps and Azure Blob Storage.

The solution generates a daily technology briefing focused on Microsoft cloud, security, AI, and related enterprise technology topics while maintaining memory of previously covered stories to reduce repetition.

The agent's instructions can be found in a separate document. 

---

## Architecture

```text
Azure Blob Storage
       │
       ▼
Daily Memory Files
(yyyy-MM-dd.json)
       │
       ▼
Azure Logic App
       │
       ├── Reads previous memory files
       ├── Builds history context
       ├── Calls Foundry Agent
       ├── Sends email briefing
       └── Saves new memory export
       │
       ▼
Azure AI Foundry Agent
       │
       ▼
Daily Technology Brief email
```

---

## Components

### Azure AI Foundry

- Model: `gpt-5-mini`
- Agent Instructions:
  - Generate daily technology briefings
  - Prioritize new developments
  - Avoid repeating stories from memory
  - Export briefing memory in structured JSON format

### Azure Blob Storage

Stores briefing memory, so the agent doesn't send same stories every morning.

Example:

```text
briefing-memory/
├── 2026-09-04.json
├── 2026-09-05.json
├── 2026-09-06.json
```

Example file:

```json
{
  "generated": "2026-09-06",
  "stories": [
    {
      "fingerprint": "cisa-kev-additions-2026-09-14",
      "headline": "CISA adds five actively exploited flaws (Artifactory, ScreenConnect, RouterOS) to KEV catalog",
      "category": "Cybersecurity",
      "dateCovered": "2026-09-14",
      "priority": "Critical"
    }
  ]
}
```

### Azure Logic Apps

Responsible for orchestration:

- Reading memory
- Calling the Agent
- Parsing the response
- Sending email
- Persisting memory

---

# Prerequisites

## Azure Resources

- Azure AI Foundry Project
- Azure Storage Account
- Azure Logic App (Consumption tier)
- Outlook.com or Microsoft 365 mail connector

## Permissions

Enable a **System Assigned Managed Identity** on the Logic App.

Assign role (RBAC):

```text
Azure AI Foundry User
```

to the managed identity on the Foundry project.

---

# Agent Configuration

## Create Project

1. Open Azure AI Foundry
2. Create a new Project
3. Select:

```text
gpt-5-mini
```

4. Create Agent

## Agent Memory Export

The agent should append a memory export at the end of each briefing listing the stories it has reported that day.

Example:

```text
=== MEMORY_EXPORT ===

{
  "generated": "2026-09-06",
  "stories": [
    {
      "fingerprint": "azure-updates-roadmap-2026-09-14",
      "headline": "Azure Updates hub refreshed — use Cloud Platform roadmap to track preview→GA milestones",
      "category": "Azure",
      "dateCovered": "2026-09-14",
      "priority": "Medium"
    }
  ]
}
```

---

# Storage Account Setup

Create a container:

```text
briefing-memory
```

---

# Logic App Workflow

## 1. Recurrence Trigger

```text
Recurrence
└── Every 1 day
```

---

## 2. Initialize History Block

```text
Initialize Variable
```

Name:

```text
HistoryBlock
```

Type:

```text
String
```

Value:

```text
Previously covered stories:
```

---

## 3. List Memory Files

```text
List blobs (V2)
```

Storage Account:

```text
<storage account>
```

Container:

```text
briefing-memory
```

---

## 4. Build History Block

### For Each

Input:

```text
body('List_history_blobs_(V2)')?['value']
```

### Inside Loop

#### Get Blob Content

```text
Get blob content (V2)
```

Blob:

```text
item()?['Path']
```

#### Append Memory

```text
Append to string variable
```

Variable:

```text
HistoryBlock
```

Value:

```text

=== MEMORY_ENTRY ===

@{body('Get_blob_content_(V2)')}
```

Result:

```text
Previously covered stories:

=== MEMORY_ENTRY ===
{ day 1 memory }

=== MEMORY_ENTRY ===
{ day 2 memory }

=== MEMORY_ENTRY ===
{ day 3 memory }
```

---

## 5. Build Agent Prompt

```text
Compose
```

Contents:

```text
History:

@{variables('HistoryBlock')}

Prepare morning brief for today.
```

---

## 6. Call Azure AI Foundry Agent

### HTTP Action

Method:

```http
POST
```

URI:

```text
https://<resource>.services.ai.azure.com/api/projects/<project>/openai/v1/responses
```

Headers:

```http
Content-Type: application/json
```

Body:

```json
{
  "input": [
    {
      "role": "user",
      "content": "@{outputs('Compose_prompt_for_agent')}"
    }
  ],
  "stream": false,
  "agent_reference": {
    "name": "<agent-name>",
    "type": "agent_reference"
  }
}
```

### Authentication

```text
Authentication Type:
Managed Identity
```

Identity:

```text
System Assigned
```

Audience:

```text
https://ai.azure.com/
```

> **Important:** The trailing slash is required.

---

## 7. Parse Response

### Parse JSON

Content:

```text
body('HTTP')
```

Schema:

```text
Generated from sample response
```

### Filter Array

From:

```text
body('Parse_JSON')?['output']
```

Condition:

```text
item()?['type'] == 'message'
```

### Extract Briefing Text

```text
Compose
```

Expression:

```text
first(body('Filter_array'))?['content'][0]?['text']
```

Result:

```text
Executive Summary
...
```

without API metadata.

---

## 8. Send Email

```text
Send an email (V2)
```

Subject:

```text
Morning Tech Brief - @{convertFromUtc(utcNow(), 'FLE Standard Time', 'dd-MM-yyyy')}
```

Body:

```text
Good morning,

Here is your morning technology briefing for @{convertFromUtc(utcNow(), 'FLE Standard Time', 'dd-MM-yyyy')}.

@{outputs('Compose_brief_text')}
```

---

## 9. Save Daily Memory Export

Extract:

```text
=== MEMORY_EXPORT ===
```

from the agent response.

Create a new blob:

Filename:

```text
@{formatDateTime(utcNow(), 'yyyy-MM-dd')}.json
```

Example:

```text
2026-09-06.json
```

Blob Content:

```json
{
  "generated": "2026-09-06",
  "stories": [...]
}
```

---

# Design Decisions

## Daily Memory Files

The solution stores one memory file per day rather than maintaining a rolling `history.json`.

Benefits:

- No merge logic required
- Easy troubleshooting
- Full historical audit trail
- Easy retention management
- Simpler Logic App implementation

## History Handling

The Logic App concatenates recent memory files into a history block and provides them as context to the agent.

The agent uses this information to suppress stories that have already been reported unless a material change has occurred.

---

# Lessons learned
- Setting daily recurrence on the Logic App is glitchy, it's easier to set it hourly first and then adjust
- Azure AI Foundry calls can be made direcly from the Logic App
- Managed Identity works with Responses API
- Audience must be https://ai.azure.com/ (with last / included)
- Daily memory files were easier to manage than a rolling history.json file
- Parcing the Responses API output requires extracting the message object from the output array
- Refining the agent prompt takes a while and many iterations. A useful middle step was adding instructions to explicitly classify each news story as New / Already seen, and to have the agent justify it's decision to include or exclude

# Future Enhancements

- Teams delivery
- Adaptive Cards
- 14-day rolling memory window
- Automatic cleanup of old memory files
- Memory deduplication
- Monitoring and alerting

---
