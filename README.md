# Agentic-AI-Automation-Systems

## Master Thesis - n8n Workflow Documentation

This document provides detailed documentation of all 12 n8n workflows built for the master thesis project. The system implements an AI-powered customer support chatbot with multi-channel virtual assistant capabilities, RAG (Retrieval-Augmented Generation) knowledge base management, and human escalation handling.

---

## System Architecture Overview

The workflows are organized into several functional layers:

1. **Core AI Customer Support** (Phase 1): Main chatbot + human escalation + PageIndex query webhook
2. **RAG Knowledge Base Management**: Vector store ingestion + human cases consolidation
3. **Virtual Assistant - Calendar & Email** (Phase 2): Calendar management via hosted chat UI + email drafting
4. **Telegram Virtual Assistant** (Phase 3): Command handler, daily briefing, event reminders
5. **PageIndex Document Management**: Telegram-based document updater

**Key Technologies**: n8n workflow automation, Anthropic Claude (AI agent), OpenAI GPT-4o, Pinecone vector database, PageIndex document AI, Google Workspace (Gmail, Calendar, Sheets, Drive), Telegram Bot API.

**Timezone**: Europe/Sarajevo

---

## 1. Main AI Customer Support Chatbot Workflow

- **Status**: Active
- **Created**: 2026-02-03
- **Last Updated**: 2026-04-09
- **Node Count**: 35

### Purpose
This is the central workflow of the entire system. It serves as an autonomous AI-powered email customer support agent that processes incoming customer emails, generates context-aware responses using RAG (Retrieval-Augmented Generation), and handles human escalation when the AI cannot confidently resolve an inquiry.

### Trigger
- **Gmail Trigger (Email Support)**: Monitors a Gmail inbox for new incoming customer emails. This is an event-based trigger that fires whenever a new email arrives.

### Detailed Node-by-Node Flow

1. **Gmail Trigger (Email Support)** (`n8n-nodes-base.gmailTrigger`): Listens for new incoming emails in the support inbox.

2. **Get a thread** (`n8n-nodes-base.gmail`): Fetches the full email thread so the AI has complete conversation context, not just the latest message.

3. **Get Thread History** (`n8n-nodes-base.googleSheets`): Looks up previous interaction logs for this thread from the Google Sheets `Customer_Support_Data` spreadsheet. This provides historical context about past interactions with this customer/thread.

4. **Format Conversation History from Sheets** (`n8n-nodes-base.code`): Formats the retrieved sheet data into a structured conversation history. Handles edge cases like empty history (Google Sheets returns `[{}]` when no results). This ensures the AI agent has full context of prior exchanges.

5. **Format Current Email Data** (`n8n-nodes-base.code`): Processes the current email thread data. Filters out AI-sent messages (from `sunflower442356@gmail.com`) to extract only user messages. Counts messages in the thread and prepares structured input for the AI agent.

6. **Detect Priority & Sentiment** (`n8n-nodes-base.code`): Analyzes the email body for priority keywords (e.g., "refund", "cancel", "lawyer", "legal action", "fraud", "chargeback") and sentiment indicators. This pre-classification helps the AI agent and escalation logic make better decisions.

7. **Check Active Escalation** (`n8n-nodes-base.googleSheets`): Checks if this thread has an existing active escalation in the logs. This prevents duplicate escalations and handles re-escalation scenarios.

8. **Escalation Guard** (`n8n-nodes-base.code`): Evaluates whether the thread has been previously escalated. Checks `isEscalated` flags in the sheet data to determine if this is a new query or a re-escalation.

9. **Route: Re-escalated Thread?** (`n8n-nodes-base.if`): Routes the flow based on whether this is a re-escalated thread:
   - **Yes (re-escalation)**: Goes to `Set Re-escalation Flag` -> `Log Re-escalation to Sheets` -> `Call Human Escalation Handler Workflow`
   - **No (new query)**: Proceeds to `Human Escalation Detection`

10. **Human Escalation Detection** (`@n8n/n8n-nodes-langchain.chainLlm`): An LLM chain (using Anthropic Claude) that analyzes whether the customer needs human assistance. It considers:
    - Previous escalation status
    - The customer's message content and tone
    - Whether the query involves sensitive topics requiring human judgment
    - Uses a **Structured Output Parser** to return a structured decision (escalate: true/false with reasoning)

11. **If** (escalation decision router): Based on the Human Escalation Detection output:
    - **Needs escalation**: Goes to `Log Pending Interaction to Sheets` -> `Call Human Escalation Handler Workflow`
    - **AI can handle**: Proceeds to `AI Agent (Context-Aware)`

12. **AI Agent (Context-Aware)** (`@n8n/n8n-nodes-langchain.agent`): The core AI agent powered by **Anthropic Claude**. Key characteristics:
    - **LLM**: Anthropic Claude (via `lmChatAnthropic`)
    - **Memory**: Window Buffer Memory for conversation context
    - **Tools available** (in priority order):
      1. **Search PageIndex HTTP Request** (PRIMARY): Queries the PageIndex document AI API for answers from uploaded company documents. This is tried first for all queries.
      2. **Vector Store Tool** (FALLBACK via Pinecone): Searches the Pinecone vector database if PageIndex fails or returns incomplete results. Uses OpenAI embeddings for vector search.
    - **Output**: Uses a **Structured Output Parser** to produce structured responses with confidence levels
    - **System prompt** instructs it to be an autonomous customer support agent that resolves inquiries accurately using the provided tools

13. **Unwrap Output** (`n8n-nodes-base.code`): Extracts and normalizes the AI agent's structured output, ensuring all required fields (confidenceLevel, confidenceScore, response, query) are present.

14. **Email Formatter** (`@n8n/n8n-nodes-langchain.chainLlm`): Another LLM chain (Anthropic Claude) that transforms the AI's raw response into a professional, well-formatted email. Takes the original user message, AI response, and confidence data as input. Uses a **Structured Output Parser** to output formatted email sections (greeting, mainContent, closing, footer).

15. **Calculate Metrics & Extract Confidence** (`n8n-nodes-base.code`): Computes response metrics including confidence scores, response time, and assembles the final email body from the formatter's output (greeting + mainContent + closing + footer).

16. **Check if Escalation Needed** (`n8n-nodes-base.if`): Final escalation check based on the confidence score. If confidence is below threshold:
    - **Low confidence**: Routes to `Log Pending Interaction to Sheets` -> `Call Human Escalation Handler Workflow`
    - **Sufficient confidence**: Routes to `Log Interaction to Sheets` -> `Reply to Customer`

17. **Reply to Customer** (`n8n-nodes-base.gmail`): Sends the formatted AI response back to the customer via Gmail.

18. **Mark a message as read** (`n8n-nodes-base.gmail`): Marks the processed email as read in Gmail.

19. **Log Interaction to Sheets** / **Log Pending Interaction to Sheets** (`n8n-nodes-base.googleSheets`): Logs all interactions to the `Customer_Support_Data` Google Spreadsheet for analytics, auditing, and future RAG updates.

20. **Call 'Human Escalation Handler Workflow'** (`n8n-nodes-base.executeWorkflow`): Triggers the separate Human Customer Support Agent Workflows when escalation is needed.

### Data Flow Summary
```
Email arrives -> Get thread + history -> Format context -> Detect priority/sentiment
-> Check active escalation -> Guard against duplicates -> Route re-escalations
-> LLM escalation detection -> Route (escalate or AI handle)
-> AI Agent (Claude + PageIndex + Pinecone) -> Format email -> Calculate metrics
-> Final escalation check -> Reply to customer OR escalate to human
-> Log everything to Google Sheets
```

### Key Design Decisions
- **Dual-tool RAG strategy**: PageIndex (primary) + Pinecone (fallback) ensures high answer quality
- **Multi-stage escalation detection**: Pre-routing detection + post-response confidence check
- **Re-escalation handling**: Prevents duplicate escalations for the same thread
- **Full audit trail**: Every interaction is logged to Google Sheets with timestamps, confidence scores, and response metadata

---

## 2. Human Customer Support Agent Workflows

- **Status**: Active
- **Created**: 2026-02-11
- **Last Updated**: 2026-04-09
- **Node Count**: 25

### Purpose
Handles the human escalation side of the customer support system. When the AI chatbot cannot confidently resolve a customer inquiry, this workflow takes over: it labels the email, notifies a human agent via Telegram, tracks the human agent's response, and monitors SLA compliance with automated alerts.

### Three Functional Sections

#### Section 1: Escalation Handler (triggered by Main Chatbot)
- **When Executed by Another Workflow** (`executeWorkflowTrigger`): Receives escalation data from the Main AI Chatbot Workflow.
- **Add label to message** (`gmail`): Adds a "Human Escalation" label to the email in Gmail so the human agent can easily find it.
- **Send Confirmation of Redirection** (`gmail`): Sends an automated email to the customer acknowledging their query has been escalated to a human agent.
- **Mark a message as read** (`gmail`): Marks the message as read after processing.
- **Alert Human Agent** (`telegram`): Sends a Telegram notification to the human support agent with the customer's query details, thread ID, and priority information.

#### Section 2: Operator Reply Tracker
- **Operator Reply Tracker** (`scheduleTrigger`): Runs periodically to check for human agent replies.
- **Get Messages from Sent + Human Escalation** (`gmail`): Fetches recent sent messages and messages with the "Human Escalation" label.
- **Filter Recent Messages (Last 5 Minutes)** (`code`): Filters to only messages sent within the last 15 minutes (safety window). This detects when the human agent has responded to an escalated ticket.
- **Get Previous Ticket Information** (`googleSheets`): Retrieves the ticket data from Google Sheets.
- **Skip Already Processed** (`code`): Filters out tickets that already have a `timestampEnd` (i.e., already resolved).
- **Update Human Escalated Ticket Information** (`googleSheets`): Updates the ticket record with the human agent's response timestamp and resolution data.

#### Section 3: SLA Breach Monitor
- **Schedule Trigger**: Runs periodically to check for SLA breaches.
- **Read Escalation Logs** (`googleSheets`): Reads all escalation records.
- **Filter Overdue Escalations** (`code`): Identifies tickets where `response === 'Pending'` and the elapsed time exceeds the SLA threshold (1 hour = 3,600,000ms).
- **Has Overdue Escalations?** (`if`): Routes based on whether overdue escalations exist.
- **Lookup Alert Count** (`googleSheets`): Checks how many alerts have already been sent for this ticket.
- **Merge Alert Count** (`code`): Combines the alert count with ticket data.
- **Send Urgent Alert** (`telegram`): Sends an urgent Telegram alert to the human agent about the overdue ticket.
- **Is Second Notice?** (`if`): Checks if this is the second alert for the same ticket.
- **Send Backup Alert** (`telegram`): If second notice, sends an additional alert (potentially to a backup agent or manager).
- **Log SLA Breach Alert** (`googleSheets`): Logs the SLA breach event.

### Key Design Decisions
- **Multi-tier alerting**: First alert -> second notice -> backup alert escalation
- **SLA monitoring**: 1-hour threshold for human response
- **Complete tracking**: Every escalation is tracked from creation to resolution
- **Gmail labels**: Used for organizing escalated emails in the inbox

---

## 3. PageIndex Query Webhook

- **Status**: Active
- **Created**: 2026-02-06
- **Last Updated**: 2026-04-09
- **Node Count**: 12
- **Version Counter**: 46 (heavily iterated)

### Purpose
Exposes a REST API webhook endpoint that allows other workflows (specifically the Main AI Chatbot) to query the PageIndex document AI platform. It serves as a middleware between the chatbot and the PageIndex API, handling request validation, document listing, querying, and response logging.

### Trigger
- **Webhook** (POST `/pageindex-query`): Accepts HTTP POST requests with a JSON body containing a `query` field.

### Detailed Flow

1. **Webhook**: Receives POST requests with `{ "query": "user question" }`.

2. **Code in JavaScript** (Input Validation): Validates the incoming request. Checks that the `query` field exists and is not empty. Returns a 400 error if validation fails. Captures start time for response time measurement.

3. **If** (Error Check): Routes based on validation result:
   - **Error**: Returns error response via `Respond with Error Message`
   - **Valid**: Proceeds to query PageIndex

4. **List All PageIndex Docs** (`httpRequest`): Makes a GET request to `https://api.pageindex.ai/docs` to retrieve all available document IDs. Authenticated with a Bearer token.

5. **Build Doc ID Array** (`code`): Extracts document IDs from the PageIndex response and builds an array of all doc IDs to query against.

6. **PageIndex Query** (`httpRequest`): Makes a POST request to `https://api.pageindex.ai/chat/completions` with:
   - The user's query as a chat message
   - All document IDs to search across
   - Authenticated with Bearer token

7. **Extract Answer** (`code`): Extracts the answer from the PageIndex chat completion response (`response.choices[0].message.content`).

8. **Respond to Webhook** (`respondToWebhook`): Returns the answer as JSON to the calling workflow.

9. **Format Response Metadata** (`code`): Calculates response time, formats metadata for logging.

10. **Log API Calls** (`googleSheets`): Logs every API call to the `PageIndex_API_Logs` sheet in the `Customer_Support_Data` spreadsheet with timestamp, query, response, status, and response time.

### Key Design Decisions
- **Centralized PageIndex access**: All PageIndex queries go through this single webhook, enabling centralized logging and monitoring
- **Full document search**: Queries across ALL documents in PageIndex, not just specific ones
- **Response time tracking**: Measures and logs the total response time for performance monitoring
- **Error handling**: Validates input and handles API errors gracefully

---

## 4. RAG Workflows for Knowledge Base

- **Status**: Inactive
- **Created**: 2026-02-16
- **Last Updated**: 2026-04-09
- **Node Count**: 33

### Purpose
Manages the Pinecone vector database that serves as the fallback knowledge base for the AI chatbot. This workflow handles three scenarios: new file uploads, file updates, and file deletions from a Google Drive folder. It ensures the vector store stays synchronized with the company's document repository.

### Three Trigger Paths

#### Path 1: New File Upload (Google Drive File Created Trigger)
1. **Google Drive File Created** (`googleDriveTrigger`): Fires when a new file is added to the monitored Google Drive folder.
2. **Remove Duplicates** (`removeDuplicates`): Filters out duplicate trigger events.
3. **Download File From Google Drive** (`googleDrive`): Downloads the file binary content.
4. **Code in JavaScript** (`code`): Adds metadata (source filename, upload timestamp) to the file data.
5. **Pinecone Vector Store** (`vectorStorePinecone`): Ingests the document into the `company-files` Pinecone index using:
   - **Default Data Loader**: Loads the binary file content
   - **Recursive Character Text Splitter**: Splits documents into chunks for vectorization
   - **Embeddings OpenAI**: Generates vector embeddings using OpenAI's embedding model
6. **File Creation Logger** (`googleSheets`): Logs the file creation event.

#### Path 2: File Update (Google Drive File Updated Trigger)
1. **Google Drive File Updated** (`googleDriveTrigger`): Fires when an existing file is modified.
2. **HTTP Request** (`httpRequest`): Deletes the old vectors for this file from Pinecone (by metadata filter on fileId).
3. **Edit Fields** (`set`): Prepares the file data for re-download.
4. **Download file** (`googleDrive`): Downloads the updated file.
5. **Code in JavaScript1** (`code`): Adds metadata.
6. **Pinecone Vector Store1** (`vectorStorePinecone`): Re-ingests the updated document with fresh embeddings using the same pipeline (Data Loader -> Text Splitter -> OpenAI Embeddings).
7. **File Update Logger** (`googleSheets`): Logs the update event.

#### Path 3: Scheduled File Sync (Periodic Cleanup)
1. **Schedule Trigger**: Runs periodically to detect deleted/missing files.
2. **Search files and folders** (`googleDrive`): Lists all current files in the monitored Google Drive folder.
3. **Read File Tracker Logs** (`googleSheets`): Reads the file tracker spreadsheet.
4. **Compare & Categorize Files** (`code`): Compares Drive files with tracked files to find:
   - Files to delete (in tracker but not in Drive)
   - Files to mark as missing
   - Files to mark as active (back in Drive)
5. **Has Files to Delete** / **Has Files to Mark Missing** / **Has Files to Mark Active** (`if` nodes): Route based on categorization.
6. **Delete Vectors HTTP Request** (`httpRequest`): Deletes orphaned vectors from Pinecone.
7. **Update row in log tracker** (x3) (`googleSheets`): Updates the file tracker with current status.

### Key Design Decisions
- **Three-path synchronization**: Handles create, update, and delete scenarios for complete vector store management
- **Pinecone as vector DB**: Uses the `company-files` index with OpenAI embeddings
- **Metadata-based vector management**: Each vector includes fileId, fileName, and uploadDate metadata for targeted deletion/updates
- **File tracker spreadsheet**: Maintains a log of all files and their sync status

---

## 5. Human Cases RAG Updater

- **Status**: Active
- **Created**: 2026-04-02
- **Last Updated**: 2026-04-08
- **Node Count**: 12

### Purpose
Runs every 24 hours to consolidate all resolved human-escalated customer support cases into a single PDF document stored in Google Drive. This consolidated PDF is then available for the RAG system to learn from past human-resolved cases, creating a feedback loop where human resolutions improve future AI responses.

### Trigger
- **Schedule Trigger (24h)**: Runs once every 24 hours.

### Detailed Flow

1. **Schedule Trigger (24h)**: Fires every 24 hours.

2. **Read Interaction Logs** (`googleSheets`): Reads the `Interaction_Logs` sheet from the `Customer_Support_Data` spreadsheet. This contains all customer support interactions including those resolved by human agents.

3. **Build All Resolved Cases** (`code`): Processes all interaction log rows and:
   - Groups rows by `threadId`
   - Filters for resolved cases (human-escalated cases that have been completed)
   - Builds a consolidated text document containing all resolved case summaries
   - Returns an array of thread IDs that were included

4. **Search Old Consolidated PDF** (`googleDrive`): Searches Google Drive for the existing `Human_Cases_Consolidated` PDF file.

5. **Download file** (`googleDrive`): Downloads the existing PDF if found.

6. **Old PDF Exists?** (`if`): Checks if a previous consolidated PDF exists:
   - **Yes**: Proceeds to delete old vectors and old PDF before creating new one
   - **No**: Skips directly to creating new PDF

7. **Delete Old Pinecone Vectors** (`httpRequest`): Deletes the old vectors associated with the previous consolidated PDF from Pinecone, so stale data doesn't persist.

8. **Delete Old Consolidated PDF** (`googleDrive`): Removes the old PDF file from Google Drive.

9. **Create Consolidated PDF** (`googleDrive`): Creates a new PDF in Google Drive containing all resolved human cases.

10. **Prepare Thread Updates** (`code`): Prepares update records for each thread ID, including the new file ID and timestamp.

11. **Update kbLastUpdated** (`googleSheets`): Updates the `kbLastUpdated` field in the interaction logs for all processed threads, marking them as synced to the knowledge base.

12. **Notify Employee via Telegram** (`telegram`): Sends a Telegram notification confirming the knowledge base has been updated with the latest resolved cases.

### Key Design Decisions
- **Feedback loop**: Human resolutions are fed back into the RAG system, continuously improving AI performance
- **Full replacement strategy**: Old PDF + vectors are deleted and recreated each cycle (ensures consistency)
- **Tracking via kbLastUpdated**: Prevents reprocessing of already-synced cases
- **Telegram notification**: Keeps the team informed of KB update status

---

## 6. PageIndex Human Cases Updater (Telegram)

- **Status**: Inactive
- **Created**: 2026-04-04
- **Node Count**: 10

### Purpose
Provides a Telegram-based interface for manually updating the Human Cases document in PageIndex. A user sends a Google Drive link to a new PDF via Telegram, and the workflow automatically replaces the old document in PageIndex with the new one.

### Trigger
- **Telegram Trigger**: Listens for incoming Telegram messages.

### Detailed Flow

1. **Telegram Trigger**: Receives a message from the Telegram bot.

2. **Extract Drive Link & Validate** (`code`): Extracts a Google Drive file ID from the message text using regex patterns that match various Google Drive URL formats:
   - `drive.google.com/file/d/FILE_ID/view`
   - `drive.google.com/open?id=FILE_ID`
   - `id=FILE_ID`
   Constructs a direct download URL. Returns an error if no valid link is found.

3. **Valid Link?** (`if`): Routes based on validation:
   - **Invalid**: Sends error reply via Telegram explaining the expected format
   - **Valid**: Proceeds to update PageIndex

4. **Send Error Reply** (`telegram`): Sends an error message back to the user explaining no valid Google Drive link was found.

5. **List PageIndex Docs** (`httpRequest`): GET request to `https://api.pageindex.ai/documents` to list all current documents.

6. **Find Old Human Cases Doc** (`code`): Searches the document list for an existing document whose name contains `human_cases_consolidated` (case-insensitive).

7. **Old Doc Exists?** (`if`): Routes based on whether an old document was found:
   - **Yes**: Delete old doc first, then upload new one
   - **No**: Skip deletion, go straight to upload

8. **Delete Old PageIndex Doc** (`httpRequest`): DELETE request to remove the old document from PageIndex.

9. **Upload New PDF to PageIndex** (`httpRequest`): POST request to `https://api.pageindex.ai/documents` with the Google Drive direct download URL.

10. **Send Confirmation** (`telegram`): Sends a confirmation message to the user that the PageIndex document has been successfully updated.

### Key Design Decisions
- **Telegram as interface**: Quick and convenient for ad-hoc document updates
- **Automatic old doc replacement**: Finds and removes the previous version before uploading
- **Google Drive URL parsing**: Supports multiple Drive URL formats for user convenience

---

## 7. Virtual Assistant (Calendar Management - Hosted Chat)

- **Status**: Inactive
- **Created**: 2026-04-05
- **Node Count**: 11
- **Phase**: Phase 2

### Purpose
A calendar management virtual assistant accessible via a hosted chat UI (n8n's built-in chat widget). Users can check availability, list events, and create calendar events through natural language conversation. Implements a mandatory confirmation flow before creating events (related to thesis hypothesis H2 about trust).

### Trigger
- **Chat Trigger** (hosted chat mode): Provides a web-based chat interface titled "Calendar Virtual Assistant" with the subtitle "Powered by AI - Master Thesis Phase 2".

### Core Components

1. **Chat Trigger** (`chatTrigger`): Hosted chat UI with a welcome message explaining capabilities (check availability, list events, add events).

2. **AI Agent** (`@n8n/n8n-nodes-langchain.agent`):
   - **LLM**: OpenAI GPT-4o
   - **Memory**: Window Buffer Memory (10 messages context)
   - **Tools**:
     - **Google Calendar - List Events**: Lists events from primary calendar
     - **Google Calendar - Create Event**: Creates new events (ONLY after user confirmation)
     - **Google Calendar - Check Availability**: Checks free/busy status
   - **System Prompt** defines four intent handlers:
     - `check_availability`: Uses Check Availability tool
     - `list_events`: Uses List Events tool
     - `add_event`: Requires explicit user confirmation before creating (shows summary, waits for yes/no)
     - `confirm_event`: Processes "yes"/"da"/"sure" confirmations
   - **Key rule**: Mandatory confirmation before event creation — the agent MUST show a summary and wait for explicit user approval

3. **Respond to Webhook** (`respondToWebhook`): Returns the AI agent's response to the chat UI.

4. **Prepare Log Data** (`code`): Extracts intent, action taken, and response metadata from the conversation.

5. **Log to Google Sheets** (`googleSheets`): Logs to the `Virtual Assistant Logs` sheet with fields: timestamp, user_message, intent, action, success, agent_response.

### Key Design Decisions
- **Confirmation-first design**: Events are never created without explicit user approval (thesis H2: trust hypothesis)
- **Intent detection logging**: Logs classified intents for analysis (check_availability, add_event, list_events, confirm_event)
- **Bilingual support**: Accepts confirmations in English and Bosnian (yes/da, no/ne)
- **Google Sheets logging**: All interactions logged for thesis data collection

---

## 8. Email VA - Proactive Monitor

- **Status**: Inactive
- **Created**: 2026-04-05
- **Node Count**: 11
- **Phase**: Phase 2

### Purpose
Automatically monitors incoming emails, classifies them using AI, creates draft replies for emails that need responses, and sends Telegram notifications about all incoming emails. Never sends emails automatically — only creates drafts for human review.

### Trigger
- **Gmail Trigger**: Monitors for new unread emails.

### Detailed Flow

1. **Gmail Trigger**: Fires on new unread emails.

2. **Classify Email (OpenAI)** (`openAi`): Uses GPT-4o to classify the email into:
   - **category**: academic, administrative, personal, newsletter, spam, other
   - **priority**: high, medium, low
   - **requires_reply**: boolean
   - **summary**: 1-2 sentence summary
   - **suggested_reply**: Draft reply text (if requires_reply is true)

3. **Parse Classifier Output** (`code`): Parses the JSON output from the classifier, strips markdown code blocks if present, and combines with original email data (from, subject, threadId, messageId).

4. **Needs Reply?** (`if`): Routes based on `requires_reply`:
   - **Yes**: Creates a draft reply and notifies via Telegram
   - **No**: Sends FYI notification via Telegram only

5. **Create Draft Reply** (`gmail`): Creates a Gmail draft with the AI-suggested reply text (as a reply in the same thread).

6. **Notify: Draft Ready** (`telegram`): Sends Telegram notification with email details (from, subject, category, priority, summary) and confirms a draft reply has been created.

7. **Notify: FYI Only** (`telegram`): Sends Telegram notification for informational emails that don't need a reply.

8. **Prepare Log Data (Draft)** / **Prepare Log Data (FYI)** (`code`): Prepares logging data.

9. **Log to Google Sheets** (`googleSheets`): Logs to `VA_Email_Log` sheet with: timestamp, action_type, from_email, subject, category, draft_created, user_confirmed.

### Key Design Decisions
- **Drafts only, never auto-sends**: Critical safety feature — AI never sends emails without human review
- **AI classification**: Categorizes emails by type and priority for better notification filtering
- **Dual notification paths**: Different Telegram messages for "draft ready" vs "FYI only"

---

## 9. Email VA - On-Demand Drafting

- **Status**: Inactive
- **Created**: 2026-04-05
- **Node Count**: 9
- **Phase**: Phase 2

### Purpose
An interactive chat-based assistant for composing email drafts on demand. Users interact via a hosted chat UI, describe what email they want to write, and the AI drafts it. The AI always shows the draft for confirmation before creating it in Gmail.

### Trigger
- **Chat Trigger** (hosted chat mode): Web chat titled "Email Drafting Assistant".

### Core Components

1. **Chat Trigger**: Hosted chat UI with welcome message explaining capabilities (compose drafts, review content, create professional replies).

2. **AI Agent** (`@n8n/n8n-nodes-langchain.agent`):
   - **LLM**: OpenAI GPT-4o
   - **Memory**: Window Buffer Memory (10 messages context)
   - **Tool**: Gmail - Create Draft (only called after user confirms)
   - **System prompt** enforces:
     - Can ONLY create drafts, never send emails
     - Must extract recipient (To), subject, and body
     - Must ask for missing information
     - MANDATORY: Show summary and ask for confirmation before creating
     - Supports any language
     - If user asks to "send", explains drafts-only limitation

3. **Respond to Webhook**: Returns AI response to chat UI.

4. **Prepare Log Data** (`code`): Detects if a draft was created based on keywords in the agent response ("draft has been created", "draft created", etc.). Extracts subject and recipient from the conversation.

5. **Log to Google Sheets**: Logs to `VA_Email_Log` with: timestamp, action_type (on_demand), from_email, subject, category, draft_created, user_confirmed.

### Key Design Decisions
- **Conversational drafting**: Natural language interaction for email composition
- **Confirmation before action**: User must approve before draft creation (trust hypothesis)
- **Draft detection via NLP**: Log preparation uses pattern matching on agent output to detect successful draft creation

---

## 10. Telegram VA - Command Handler

- **Status**: Inactive
- **Created**: 2026-04-05
- **Node Count**: 19
- **Phase**: Phase 3

### Purpose
The main Telegram bot interface that serves as the unified entry point for all Telegram-based virtual assistant interactions. Routes specific commands (/help, /today) to dedicated handlers and all other messages to an AI agent with calendar + email tools.

### Trigger
- **Telegram Trigger**: Listens for all incoming Telegram messages.

### Routing Logic

1. **Telegram Trigger**: Receives all messages.

2. **Command Router** (`switch`): Routes based on message text (case-insensitive):
   - `/help` -> **Send Help** (static help message)
   - `/today` -> **Get Today's Events** (calendar lookup)
   - **Default** (everything else) -> **AI Agent** (natural language processing)

#### /help Route
3. **Send Help** (`telegram`): Sends a formatted help message listing all available commands: /schedule, /today, /check, /email, and natural language.
4. **Prepare Log (Help)** -> **Log to Google Sheets**

#### /today Route
3. **Get Today's Events** (`googleCalendar`): Fetches all events from today (start of day to end of day) from the primary calendar, ordered by start time.
4. **Format Schedule** (`code`): Formats events into a readable Telegram message with times, titles, and locations. Shows "No events" message if schedule is clear.
5. **Send Schedule** (`telegram`): Sends the formatted schedule.
6. **Prepare Log (Today)** -> **Log to Google Sheets**

#### Default Route (AI Agent)
3. **AI Agent** (`@n8n/n8n-nodes-langchain.agent`):
   - **LLM**: OpenAI GPT-4o
   - **Memory**: Window Buffer Memory (10 messages, keyed by chat ID for per-user sessions)
   - **Tools**:
     - GCal - List Events
     - GCal - Create Event (requires confirmation)
     - GCal - Check Availability
     - Gmail - Create Draft (requires confirmation)
   - **System prompt**: Combined Calendar & Email VA. Handles /schedule, /add, /check, /email, /draft commands and free text. Supports English and Bosnian. Mandatory confirmation before creating events or drafts.
4. **Send AI Response** (`telegram`): Sends the agent's response.
5. **Prepare Log (AI)** -> **Log to Google Sheets**

### Logging
All three routes log to the `VA_Metrics` sheet with: timestamp, trigger_type (command/freeform), command, action_completed, response_time_ms, user_confirmed_before_action.

### Key Design Decisions
- **Command routing + AI fallback**: Common commands get fast direct handlers; everything else goes to AI
- **Per-user memory**: Window Buffer Memory keyed by Telegram chat ID for isolated conversation contexts
- **Unified logging**: All interactions (commands and AI) logged to the same sheet for unified analytics
- **Bilingual confirmations**: Accepts yes/da, no/ne, sure/potvrdi

---

## 11. Telegram VA - Daily Briefing

- **Status**: Inactive
- **Created**: 2026-04-05
- **Node Count**: 7
- **Phase**: Phase 3

### Purpose
Sends a formatted morning briefing of today's calendar events via Telegram every day at 7:00 AM (Europe/Sarajevo timezone). Uses OpenAI GPT-4o to format the briefing into a visually appealing Telegram message.

### Trigger
- **Schedule Trigger**: Runs daily at 07:00.

### Detailed Flow

1. **Schedule Trigger**: Fires at 7:00 AM daily.

2. **Get Today's Events** (`googleCalendar`): Fetches all events for the current day from the primary Google Calendar, with single events expanded and ordered by start time.

3. **Format Morning Briefing** (`openAi`): Uses GPT-4o (temperature 0.3 for consistency) with a system prompt that instructs it to:
   - Create a nicely formatted Telegram Markdown message
   - Start with a greeting and date
   - List each event with time, title, and location
   - If no events, send a friendly "no events" message
   - Use emoji sparingly for visual structure

4. **Send Morning Briefing** (`telegram`): Sends the formatted briefing to the user's Telegram chat.

5. **Prepare Log Data** (`code`): Prepares log entry with trigger_type "morning", command "/briefing", and response time.

6. **Log to Google Sheets** (`googleSheets`): Logs to `VA_Metrics` sheet.

### Key Design Decisions
- **AI-formatted output**: Uses GPT-4o for natural, context-aware formatting rather than rigid templates
- **Low temperature (0.3)**: Ensures consistent, reliable formatting
- **Timezone-aware**: Configured for Europe/Sarajevo

---

## 12. Telegram VA - Event Reminders

- **Status**: Inactive
- **Created**: 2026-04-05
- **Last Updated**: 2026-04-06
- **Node Count**: 9
- **Phase**: Phase 3

### Purpose
Checks every 30 minutes for upcoming Google Calendar events that start in approximately 30 minutes, and sends proactive Telegram reminders. Ensures users don't miss meetings or appointments.

### Trigger
- **Schedule Trigger**: Runs every 30 minutes.

### Detailed Flow

1. **Schedule Trigger**: Fires every 30 minutes.

2. **Get Upcoming Events** (`googleCalendar`): Fetches events from now until 1 hour from now, ordered by start time. Uses Google Calendar OAuth2 credentials.

3. **Filter Events (25-35 min)** (`code`): Calculates minutes until each event starts and filters to only events starting in 25-35 minutes. This 10-minute window (centered on 30 minutes) ensures events are caught exactly once given the 30-minute poll interval.

4. **Has Events?** (`if`): Checks if any events matched the filter:
   - **Yes**: Send reminder
   - **No**: No-op (do nothing)

5. **Send Event Reminder** (`telegram`): Sends a formatted reminder with:
   - Minutes until event
   - Event title
   - Event time (formatted HH:MM)
   - Location (if available)

6. **No Events** (`noOp`): Does nothing when no events match.

7. **Prepare Log Data** (`code`): Logs reminder actions.

8. **Log to Google Sheets** (`googleSheets`): Logs to `VA_Metrics` sheet.

### Key Design Decisions
- **25-35 minute window**: Ensures each event triggers exactly one reminder (given 30-min polling)
- **Proactive reminders**: Users don't need to ask — reminders come automatically
- **Smart filtering**: Only events with a valid start dateTime are processed

---

## Cross-Workflow Integration Map

```
┌─────────────────────────────────────────────────────────────┐
│                    CUSTOMER SUPPORT LAYER                     │
│                                                               │
│  Gmail ──> Main AI Customer Support Chatbot ──> Reply        │
│                    │           │                               │
│                    │     ┌─────┘                               │
│                    ▼     ▼                                     │
│            PageIndex   Pinecone                               │
│            Query       Vector Store                           │
│            Webhook     (RAG Workflows)                        │
│                    │                                           │
│                    ▼                                           │
│         Human Customer Support                                │
│         Agent Workflows                                       │
│            │                                                   │
│            ▼                                                   │
│         Telegram Alerts                                       │
│         SLA Monitoring                                        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    KNOWLEDGE BASE LAYER                       │
│                                                               │
│  Google Drive ──> RAG Workflows ──> Pinecone (company-files) │
│                                                               │
│  Schedule ──> Human Cases RAG Updater ──> Google Drive PDF   │
│                                           ──> Pinecone       │
│                                                               │
│  Telegram ──> PageIndex Human Cases Updater ──> PageIndex    │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    VIRTUAL ASSISTANT LAYER                    │
│                                                               │
│  Chat UI ──> Virtual Assistant (Calendar)                    │
│  Chat UI ──> Email VA - On-Demand Drafting                   │
│  Gmail   ──> Email VA - Proactive Monitor ──> Gmail Drafts   │
│                                             ──> Telegram     │
│                                                               │
│  Telegram ──> Telegram VA - Command Handler                  │
│               ──> /help, /today, AI Agent                    │
│  Schedule ──> Telegram VA - Daily Briefing                   │
│  Schedule ──> Telegram VA - Event Reminders                  │
└─────────────────────────────────────────────────────────────┘
```

---

## Shared Infrastructure

### Google Sheets Spreadsheets
1. **Customer_Support_Data** (`18_bjdUsEWR2zhlPb-0FWAXTVd69sbDKuOYGvOOJRXpA`):
   - `Interaction_Logs`: All chatbot interactions (used by Main Chatbot + Human Cases RAG Updater)
   - `PageIndex_API_Logs`: PageIndex Query Webhook logs
   - Escalation logs (used by Human Support Agent Workflows)

2. **Virtual Assistant Logs** (`1x2IWiByDJhd3fNqcwS8Vymp9sxs-TGLH5Yyt6dYncdM`):
   - `Virtual Assistant Logs`: Calendar VA interactions
   - `VA_Metrics`: Telegram VA metrics (command handler, daily briefing, event reminders)
   - `VA_Email_Log`: Email VA interactions (proactive monitor + on-demand drafting)

### AI Models Used
- **Anthropic Claude** (via `lmChatAnthropic`): Used in the Main AI Customer Support Chatbot for the primary agent, escalation detection, and email formatting
- **OpenAI GPT-4o** (via `lmChatOpenAi` / `openAi`): Used in Virtual Assistant workflows, Telegram VA, Email VA, and Daily Briefing formatter
- **OpenAI Embeddings**: Used for Pinecone vector store embeddings

### External Services
- **PageIndex AI** (`api.pageindex.ai`): Document AI platform for querying uploaded PDFs
- **Pinecone** (`company-files` index): Vector database for RAG fallback
- **Google Workspace**: Gmail, Calendar, Sheets, Drive
- **Telegram Bot API**: Chat interface for VA and notifications

### Credentials Used
- Google Sheets OAuth2 (`dFwvRpFrU3hj7eAL`)
- Google Calendar OAuth2
- Gmail OAuth2
- Telegram Bot API (`Z4DjPtixNJvP438S`)
- OpenAI API (`h8C2L4r39uXcmqya`)
- Anthropic API
- Pinecone API (`ESOHs5j1ToO16UUT`)
- PageIndex API (Bearer token)

---

## Thesis-Relevant Design Patterns

1. **Human-in-the-Loop (HITL)**: The Virtual Assistant and Email VA workflows always require user confirmation before taking actions (creating events, drafting emails). This relates to trust hypothesis H2.

2. **Multi-Stage Escalation**: The customer support system uses a two-stage escalation check (pre-routing LLM detection + post-response confidence threshold).

3. **RAG Feedback Loop**: Human-resolved cases are consolidated back into the knowledge base (Human Cases RAG Updater), creating a continuous improvement cycle.

4. **Dual-RAG Strategy**: PageIndex (primary, fast) + Pinecone (fallback, comprehensive) ensures high answer quality.

5. **Comprehensive Logging**: Every workflow logs interactions to Google Sheets for thesis data collection and analysis.

6. **SLA Monitoring**: Automated escalation tracking with multi-tier alerts (first notice, second notice, backup alert).

7. **Proactive vs. Reactive Patterns**: The system includes both proactive workflows (email monitor, daily briefing, event reminders) and reactive workflows (command handler, on-demand drafting, customer support chatbot).
