# Agentic AI Automation Systems

Master's thesis project implementing two production-style **agentic AI automation systems** built on [n8n](https://n8n.io):

1. **Customer Support Chatbot** — an autonomous, email-based AI support agent with a RAG knowledge base, PageIndex document QA, and human-in-the-loop escalation.
2. **Personal Virtual Assistant** — a Telegram- and email-driven personal assistant that manages Google Calendar and Gmail with a plan-then-approve safety gate.

Each system ships with its workflows, source datasets, knowledge base, runtime logs, and a full evaluation suite (test cases + automated test-runner workflows).

---

## Tech Stack

- **Orchestration:** n8n (workflow automation)
- **AI models:** Anthropic Claude, OpenAI GPT-4o, Google Vertex / Gemini
- **Retrieval:** Pinecone (vector store) + PageIndex (document QA)
- **Google Workspace:** Gmail, Calendar, Sheets, Drive
- **Messaging:** Telegram Bot API
- **Timezone:** Europe/Sarajevo

---

## Repository Structure

```
.
├── customer_support_chatbot/
│   ├── datasets/             # Source data + analysis notebook
│   ├── knowledge_base/       # Company KB documents (RAG source)
│   ├── logging_files/        # Runtime log exports
│   ├── workflows/            # Production n8n workflows
│   ├── test_case_workflows/  # n8n workflows that run the test suite
│   └── test_cases/           # category1 … category7 (test inputs/outputs)
│
└── personal_virtual_assistant/
    ├── datasets/             # Analysis notebook
    ├── logging_files/        # Runtime log exports
    ├── workflows/            # Production n8n workflows
    ├── test_case_workflows/  # n8n workflows that run the test suite
    └── test_cases/           # category1 … category5 (test inputs/outputs)
```

> **Workflow documentation lives inside the files.** Every `.json` workflow contains an n8n **sticky note** describing that workflow's purpose and how it works. Open a workflow in n8n (or read the `content` of its sticky-note node) to see its documentation.

> **Test data formats.** Files in `test_cases/` are exported as **`.csv`**. The `test_case_workflows/` read from and write to the live **Google Sheets (`.xlsx`) spreadsheets** that drive each evaluation — the CSVs are the static snapshots of those sheets.

---

## 1. Customer Support Chatbot

An autonomous AI agent that reads customer emails, answers them from the company knowledge base, and escalates to a human when needed.

### `datasets/`
Source data and exploratory analysis for the support domain.
- `customer_support_tickets.csv` — raw support ticket dataset.
- `MT_Customer_Support_Tickets.ipynb` — Jupyter notebook for dataset analysis/preparation.

### `knowledge_base/`
The documents the chatbot retrieves answers from (RAG source of truth). Synced into both Pinecone and PageIndex.
- `01_FAQ_General` … `09_Company_Overview.docx` — FAQ, technical, billing/refunds, product catalog, escalation, response templates, SLA metrics, policies, company overview.
- `Human_Cases.docx` — consolidated answers distilled from past human-handled escalations, regenerated automatically by the *Human Cases RAG Updater* workflow.

### `logging_files/`
CSV exports of the runtime Google Sheets the system writes to during operation.
- `Interaction_Logs` — every customer interaction (classification, escalation, confidence, response time).
- `PageIndex_API_Logs` — PageIndex query calls and results.
- `PageIndex_File_Tracker` / `Rag_File_Tracker` — which KB files are currently embedded in PageIndex / Pinecone.
- `SLA_Breach_Logs` — escalations that breached the human-reply SLA.

### `workflows/`
The production n8n workflows. Each file's sticky note holds its full description.
- **Main AI Customer Support Chatbot Workflow** — central inbound pipeline: reads support email, classifies/routes, drafts a KB-grounded reply, and replies or escalates.
- **PageIndex Query Webhook** — HTTP endpoint exposing PageIndex document-QA; the chatbot's primary KB retrieval tool.
- **HA - Sending Immediate Escalation Alert to a Human Agent** — hands a case off to a human and pings the on-call agent on Telegram.
- **HA - Scrapping and Logging Human Sent Replies** — syncs `Interaction_Logs` with the replies human agents actually send.
- **HA - SLA Breach Escalation Monitor** — alerts when an escalated ticket waits past the 1-hour SLA.
- **Human Cases RAG Updater** — rebuilds the `Human_Cases` KB doc from resolved escalations and re-embeds it.
- **RAG Workflow for Knowledge Base - Addition / Update / Delete** — keep the Pinecone vector store in sync with KB file changes.
- **PageIndex Workflow for Knowledge Base - Addition / Update** — mirror the same KB changes into PageIndex.

### `test_case_workflows/`
n8n workflows that drive the evaluation suite against the live chatbot.
- **Test Email Sender** — sends test-case emails into the chatbot's inbox, one at a time.
- **Test Response Collector** — matches each test case to its interaction log and grades classification, escalation, response time, and confidence.

### `test_cases/`
Test inputs and expected outputs, organized into seven categories (`category1` … `category7`), exported as `.csv`. Larger categories are split into rounds/parts (e.g. Round1/Round2, A–D) of inputs and corresponding results.

---

## 2. Personal Virtual Assistant

A Telegram- and email-driven assistant that manages the owner's Google Calendar and Gmail, using a plan-then-approve flow so nothing is sent or booked without explicit user approval.

### `datasets/`
- `MT_Personal_Assistant.ipynb` — Jupyter notebook for dataset analysis/preparation.

### `logging_files/`
CSV exports of the runtime Google Sheets the assistant writes to.
- `VA_Email_Log` — emails handled/drafted by the assistant.
- `VA_Metrics` — per-run metrics (briefings, reminders, actions).

### `workflows/`
The production n8n workflows. Each file's sticky note holds its full description.
- **Telegram VA - Command Handler** — core command-and-control bot: handles slash commands and free-form English/Bosnian messages, with a Planner → human approval → executing agent flow over Calendar and Gmail.
- **Email VA - Proactive Monitor** — watches the inbox, classifies each email, drafts replies, checks the calendar for proposed meetings, and asks for Telegram approval before acting.
- **Telegram VA - Daily Briefing** — sends a daily morning calendar briefing over Telegram (read-only, no AI).
- **Telegram VA - Event Reminders** — pushes reminders for calendar events starting soon.

### `test_case_workflows/`
n8n workflows that run the assistant's evaluation suite. Most clone a common "Test Runner" pattern: snapshot the calendar/drafts before, drive the VA in `[TEST_MODE]`, snapshot after, diff to grade the result, then clean up.
- **Cat2 - Test Runner** — calendar scheduling.
- **Cat3 - Test Runner** + **Cat3 - EXC Pre-Populator** — conflict handling (the pre-populator seeds existing events before the conflict tests run).
- **Cat4 CAP / RSC / SDA / SRM / TRI - Test Runner** — multistep tasks (cancel-apologize-propose, reschedule-notify, schedule+email, schedule+reminder, triple-step).
- **Cat5 - Test Runner** — ambiguous requests (grades whether the agent asks a clarifying question).
- **Test Auto-Responder (Email VA Approval Simulator)** — replaces the manual Telegram approval click for batch testing the Email VA.

### `test_cases/`
Test inputs and expected outputs, organized into five categories (`category1` … `category5`), exported as `.csv`. Some categories are split into rounds/parts of inputs and outputs.

---

## Notes

- The `.json` workflows are n8n exports — import them into an n8n instance to inspect or run them. Credentials (Google, Pinecone, Telegram, model APIs) must be configured in your own n8n environment.
- This repository is part of a master's thesis and is intended for documentation and reproducibility rather than turnkey deployment.

---

### README best-practice sources

- [Make a README](https://www.makeareadme.com/)
- [How to Structure Your README File — freeCodeCamp](https://www.freecodecamp.org/news/how-to-structure-your-readme-file/)
- [README Best Practices — Tilburg Science Hub](https://www.tilburgsciencehub.com/topics/collaborate-share/share-your-work/content-creation/readme-best-practices/)
- [Writing READMEs for Research Data — Cornell Data Services](https://data.research.cornell.edu/data-management/sharing/readme/)
