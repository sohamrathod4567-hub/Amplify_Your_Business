# Amplify_Your_Business  - n8n Sales Automation System

An end-to-end AI-powered sales pipeline built in n8n that automatically finds prospects, stores them in a CRM, drafts outreach emails, books demos, and records deals — all triggered from a single ICP document drop.

---

## Overview

This system is composed of 7 interconnected n8n workflows that together automate the full sales development lifecycle for **AccountAI**, an accounting software firm.

```
ICP PDF Drop (Google Drive)
        ↓
Business Development Manager  ← Orchestrator Agent
        ↓              ↓              ↓
  Prospecting     RevOps Sub-     SDR Sub-Agent
  Sub-Agent       Agent           (Email Drafts)
        ↓              ↓
  Firecrawl      Pipedrive CRM
  Web Search

Separate Webhook-Triggered Flows:
  ├── Deal Recording Sub-Agent  →  Pipedrive Deal
  └── Demo Booking Sub-Agent    →  Google Calendar
```

---

## Workflows

### 1. Business Development Manager
**File:** `Business_Development_Manager.json`  
**Trigger:** New PDF file dropped in a Google Drive folder (`ICP File Drop`)

The orchestrator agent. It reads an Ideal Customer Profile (ICP) PDF, then sequentially:
1. Calls the **Prospecting Sub-Agent** to find matching leads online
2. Calls the **RevOps Sub-Agent** once per lead to save them in Pipedrive
3. Calls the **SDR Sub-Agent** to draft outreach emails for all stored leads
4. Sends a Telegram notification reporting success or failure

**Model:** Google Gemini  
**Integrations:** Google Drive, Pipedrive, Gmail, Telegram

---

### 2. Prospecting Sub-Agent
**File:** `Prospecting_sub-agent.json`  
**Trigger:** Called by the Business Development Manager (or via MCP)

Takes a natural language query (e.g. "Find accountants in the US to sell accounting software") and uses **Firecrawl** to search the web for matching prospects. Returns up to 2 structured leads including:
- First name, last name
- Company & role
- Predicted email address
- Rationale for qualification

**Model:** Google Gemma 3 (via OpenRouter)

---

### 3. Prospecting MCP
**File:** `Prospecting_Mcp.json`  
**Trigger:** MCP Server Trigger (exposes the Prospecting Sub-Agent as an MCP tool)

Wraps the Prospecting Sub-Agent as an MCP-accessible tool so it can be called from external agents or Claude Desktop. Useful for integrating prospecting into broader agentic workflows.

---

### 4. RevOps Sub-Agent
**File:** `RevOps_Sub-Agent.json`  
**Trigger:** Called by the Business Development Manager

Receives a natural language description of a sales lead and:
1. Parses it into structured fields (name, company, role, email) using an AI agent
2. Creates an **Organization** record in Pipedrive
3. Creates a **Person** record linked to that organization
4. Creates a **Lead** record linking person and organization

**Model:** Arcee AI Trinity Large (via OpenRouter)  
**Integration:** Pipedrive

---

### 5. SDR Sub-Agent
**File:** `SDR_Sub-Agent.json`  
**Trigger:** Called by the Business Development Manager

Acts as an AI Sales Development Representative. It:
1. Retrieves all person records from Pipedrive
2. Drafts a personalized HTML outreach email from Soham (CTO, AccountAI) for one lead
3. Saves it as a **Gmail draft** ready for review and sending

The email highlights AccountAI's software, prompts interest via a call to **+919879979098**, and is professionally formatted in HTML.

**Model:** Arcee AI Trinity Large (via OpenRouter)  
**Integrations:** Pipedrive, Gmail

---

### 6. Deal Recording Sub-Agent
**File:** `Deal_Recording_Sub-Agent.json`  
**Trigger:** Webhook (`POST /webhook/540a05b8-...`)

Receives a prospect's name via webhook, looks them up in Pipedrive, and if found, creates a **Deal** record associated with that person. Sends a Telegram notification on success or failure.

**Model:** Google Gemma 4 (via Google Gemini)  
**Integration:** Pipedrive, Telegram

**Sample Request:**
```json
POST /webhook/540a05b8-65ce-4704-b4b9-8490708195d3
{
  "name": "John Smith"
}
```

---

### 7. Demo Booking Sub-Agent
**File:** `Demo_Booking-Subagent.json`  
**Trigger:** Webhook (`POST /webhook/8b22e673-...`)

Books a product demo on Google Calendar. It:
1. Checks calendar availability for the requested time slot
2. Creates a calendar event if the slot is free
3. Sends a Telegram notification and responds to the webhook with a summary

**Model:** Google Gemini  
**Integration:** Google Calendar, Telegram

**Sample Request:**
```json
POST /webhook/8b22e673-285a-4570-b2d3-ae4de6d93843
{
  "name": "Jane Doe",
  "timeslot": "2026-05-01T10:00:00"
}
```

---

### 8. AI (Web Search Utility)
**File:** `AI.json`  
**Trigger:** Chat message

A standalone utility workflow. Takes a user message, generates an optimised search query using an AI agent, runs a **Firecrawl MapSearch**, and forwards the results to a webhook endpoint.

**Model:** Arcee AI Trinity Large Preview (via OpenRouter)

---

## Integrations & Credentials Required

| Service | Used For |
|---|---|
| **OpenRouter** | LLM access (Arcee AI, Google Gemma) |
| **Google Gemini (PaLM API)** | LLM access for orchestrator & demo booking |
| **Firecrawl** | Web search & scraping for prospecting |
| **Pipedrive** | CRM — leads, persons, organizations, deals |
| **Gmail (OAuth2)** | Drafting outreach emails |
| **Google Drive (OAuth2)** | Watching for ICP file drops |
| **Google Calendar (OAuth2)** | Checking availability & booking demos |
| **Telegram** | Success/failure notifications |

---

## How to Use

### Automated Pipeline (Recommended)
1. Upload an ICP PDF to the watched Google Drive folder (`ICP File Drop`)
2. The **Business Development Manager** workflow fires automatically
3. Prospects are found, stored in Pipedrive, and outreach emails are drafted in Gmail
4. Review and send the Gmail drafts manually

### Book a Demo
Send a POST request to the Demo Booking webhook with the prospect's name and preferred time slot. A calendar event will be created and you'll receive a Telegram confirmation.

### Record a Deal
Send a POST request to the Deal Recording webhook with the prospect's name. The workflow will find them in Pipedrive and create a deal record.

---

## Architecture Notes

- All sub-agents use **Structured Output Parsers** to ensure clean, typed data flows between workflow steps
- The orchestrator uses `maxIterations: 30` to handle multi-step tool use reliably
- Telegram notifications are used throughout as lightweight observability
- The Prospecting MCP workflow exposes the prospecting capability to external agents via the MCP protocol

---

## Author

Built by **Soham** 
