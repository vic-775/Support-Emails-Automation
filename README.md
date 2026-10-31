# AI Email Support Agent with a Human Safety Net

> **Customers wait hours for answers your FAQ already has. Bots that answer everything send wrong answers about refunds. This project does neither.**
> It replies instantly when your knowledge base can answer, and when it can't, it pulls in a human on Slack, drafts the reply *with* them, and only sends what they approve.

Built with **n8n**, **Gmail**, **Pinecone Assistant (RAG)**, **Slack**, **Airtable** and **Postgres**. Each tool can be swapped for an alternative (see the tech stack).

**GitHub "About" line (copy-paste):**
`AI email support automation in n8n: answers from your FAQ via RAG, escalates the rest to a human on Slack with one-click approval.`

**Suggested topics:** `n8n` `ai-agent` `rag` `pinecone` `customer-support` `human-in-the-loop` `slack` `gmail` `automation` `llm`

---

## The problem

Small support teams hit the same wall:

- Most emails ask things the FAQ already answers, but someone still has to type the reply.
- A fully automatic bot is risky. One wrong answer about a refund, a double charge or a legal complaint costs more than a slow reply.
- When an AI is unsure, the usual fallback is "forward it to a person", which loses context and the customer waits anyway.

## The solution

Three connected n8n workflows that split the work by risk:

| Situation | What happens |
|---|---|
| The FAQ clearly answers it | The AI agent replies in the same Gmail thread automatically, after a safety check |
| The FAQ doesn't cover it, or the topic is risky (billing, refunds, fraud, legal, angry customer) | The case goes to a human on Slack. An escalation agent explains the gap, asks one question at a time and drafts the reply |
| The human reads the draft | **Approve and send** with one button, or reply in the thread to request changes. The agent redrafts until it's right |
| Nobody responds in 7 days | The thread is told the case timed out and no reply is sent to the customer |

The customer never gets an unreviewed answer on a sensitive topic, and the human never starts from a blank page.

---

## Architecture

![Architecture overview](docs/email%20architecture.drawio.png)

### Workflow 1: Email Support Agent (main)
1. **Receive and classify.** A Gmail trigger polls every minute. The newest message is separated from quoted history, and an LLM decides whether it is a support request (newsletters and spam are ignored).
2. **Retrieve knowledge.** Pinecone Assistant returns FAQ snippets. Snippets scoring below 0.4 are dropped and the best three are kept.
3. **Decide.**
   - **Enough knowledge:** the answering agent writes a reply from the snippets only. A safety node blocks sending if the agent asked to escalate, the email mentions refunds, charges, fraud or legal wording, the draft promises an outcome only a human can decide, or the reply is empty.
   - **Not enough, or unsafe:** the escalation agent takes over.
4. **Human in the loop.** A human reviewer gets the case on Slack. The agent replies with one of three actions: **ASK** (needs one missing fact), **DRAFT** (full customer email plus an Approve button) or **SEND**. The workflow pauses on a Wait node until she responds.
5. **Send and close.** On approval, the exact approved draft is sent through Gmail, the Slack message is updated, the Airtable case is closed and the send is written to the agent's memory.

### Workflow 2: Slack Interaction Handler
Receives Slack events and button clicks, answers Slack within 3 seconds, looks up the open case in Airtable and posts the click or reply to the saved resume URL of the paused workflow. If the case is already closed, it tells the user so.

### Workflow 3: Knowledge Base Ingestion
Run manually whenever the FAQ changes. It downloads the FAQ from Google Drive as plain text, replaces the old file in Pinecone Assistant and uploads the new one, with no duplicates.

---

## Features

- **Safe by design.** Risky topics always go to a human, whatever the model says.
- **Answers only from your FAQ.** No invented policies, prices or deadlines.
- **One-click approval in Slack.** Approving sends the exact text shown. Used buttons are replaced with a status line so they can't be clicked twice.
- **Conversational drafting.** The human talks to the agent in a Slack thread. Rejected drafts are marked and a new draft follows.
- **Per-case memory.** Each case has its own Postgres chat history, so the agent remembers what was already asked and answered.
- **Multi-turn safe.** A case ID links the email, the Slack thread and the Airtable record.
- **Timeout handling.** Stale cases don't hang forever.

## Tech stack

| Layer | Used here | Other options that would work |
|---|---|---|
| Orchestration | n8n | Make, Zapier (with changes) |
| Email | Gmail (trigger and reply) | Outlook / Microsoft 365, IMAP and SMTP |
| Knowledge base and RAG | Pinecone Assistant, FAQ stored in Google Drive | Any vector store (Qdrant, Supabase, Weaviate), FAQ from Notion or a website |
| LLMs | Groq (classifier), OpenRouter (agents) | OpenAI, Anthropic, Gemini, local models. Models are configurable |
| Human review | Slack (Block Kit buttons, Events API, Interactivity) | Microsoft Teams, WhatsApp, Telegram, Discord |
| Case tracking | Airtable | Google Sheets, Notion, Postgres, any database |
| Agent memory | Postgres (n8n chat memory) | Redis, Supabase, n8n window buffer memory |

> The pattern matters more than the tools. Any chat app that can show a message with an approve action and send replies back to a webhook can replace Slack, and any table that stores a case ID, status and resume URL can replace Airtable.

---

## Repository structure

```
.
├── workflows/
│   ├── 01-email-support-agent.json
│   ├── 02-slack-interaction-handler.json
│   └── 03-knowledge-base-ingestion.json
├── docs/
│   ├── email_support_automation_architecture.drawio
│   ├── email_support_workflow_detail.drawio
│   └── architecture.png
└── README.md
```

---

## Setup

### 1. Prerequisites
- n8n instance (self-hosted or cloud) that is reachable from the internet
- Accounts: Gmail, Slack, Airtable, Pinecone, Google Drive, Postgres, Groq, OpenRouter

### 2. Import the workflows
Import all three JSON files in `workflows/`, then create and attach your own credentials in each node. **Do not reuse credential IDs from the exports.**

### 3. Airtable (or Google Sheets, Notion, any database)
Create a table with these fields (names must match exactly). Google Sheets, Notion or a database table work just as well, you only need to swap the Airtable nodes:

| Field | Type |
|---|---|
| `subject_id` | Text (the case ID) |
| `Email Subject` | Text |
| `thread_ts` | Text |
| `Channel` | Text |
| `User` | Text |
| `resume_url` | Text |
| `Status` | Single select: `open`, `closed` |
| `Escaltion-Status` | Single select: `Escalated`, `Not Escalated` |

> The field name `Escaltion-Status` contains a typo in the current build. Rename it everywhere, or keep it as is, but keep the table and the nodes consistent.

### 4. Postgres
n8n creates the `n8n_chat_histories` table automatically on first use. Use the same Postgres credential for all memory and history nodes.

### 5. Pinecone knowledge base
1. Put your FAQ in a Google Doc.
2. Create a Pinecone Assistant and set it in the Pinecone nodes.
3. Run **Workflow 3** manually to load the FAQ.

### 6. Slack app (or Teams, WhatsApp, Telegram)
Using another chat app? Replace the Slack nodes and point that app's webhook at Workflow 2. The steps below are for Slack.

1. Create a Slack app and add a bot with the `chat:write` scope.
2. Under **Event Subscriptions**, set the Request URL to the **production URL of the Webhook node** in Workflow 2 and subscribe to message events for the place where the bot posts (for example `message.im` for direct messages).
3. Under **Interactivity**, set the same URL.
4. Install the app and attach the bot token to the Slack nodes. In `Post Email Subject`, choose who receives escalations.

### 7. Activate
Activate Workflows 1 and 2. Workflow 3 stays manual.

### 8. Test
Send yourself an email in each of these cases and watch the result:

| Test email | Expected result |
|---|---|
| A question your FAQ answers | Automatic reply in the same thread |
| "What items do you have in stock?" (FAQ gap) | Slack escalation and a draft |
| "I was charged twice" | Always escalated |
| Newsletter or spam | Ignored |
| Reply in the Slack thread asking for changes | Old draft marked rejected, new draft appears |
| Click **Approve and send** | Email sent, button replaced, case closed |
| Click the old button again | "Case already closed" message |
