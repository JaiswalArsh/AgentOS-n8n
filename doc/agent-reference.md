# AgentOS — Agent & Tool Reference

> Complete developer reference for all agents and external tool integrations in the AgentOS system.

---

## Table of Contents

- [Parent Agent — Ultimate Personal Assistant](#parent-agent--ultimate-personal-assistant)
- [Email Agent](#email-agent)
- [Calendar Agent](#calendar-agent)
- [Contact Agent](#contact-agent)
- [Content Creator Agent](#content-creator-agent)
- [Tool Reference](#tool-reference)

---

## Parent Agent — Ultimate Personal Assistant

### Purpose

The Ultimate Personal Assistant is the root orchestrator of the AgentOS system. It receives all user input, maintains conversational context, and delegates execution to the appropriate child agent or direct tool. It does not execute domain actions (email composition, calendar writes, etc.) directly — it routes only.

### n8n Node Type

`@n8n/n8n-nodes-langchain.agent` — typeVersion 1.7

### LLM

OpenRouter (`@n8n/n8n-nodes-langchain.lmChatOpenRouter`) — model is configurable via the OpenRouter credential and node settings.

### Memory

`@n8n/n8n-nodes-langchain.memoryBufferWindow` — session key: `$('Telegram Trigger').item.json.message.chat.id`

### Inputs

| Field | Source | Type | Description |
|-------|--------|------|-------------|
| `text` | `$json.text` | string | The user's message — either typed text or Whisper-transcribed voice |

The `text` field is populated by one of two upstream nodes:
- **Set 'Text' node** — extracts `message.text` from the Telegram payload for typed messages
- **Transcribe Audio node** — returns the transcribed string from OpenAI Whisper for voice messages

### Outputs

| Field | Type | Description |
|-------|------|-------------|
| `output` | string | Final natural language response sent to the user via Telegram |

### Available Tools

| Tool Name | Type | n8n Node | Purpose |
|-----------|------|----------|---------|
| `emailAgent` | toolWorkflow | Sub-workflow | All Gmail operations |
| `calendarAgent` | toolWorkflow | Sub-workflow | All Google Calendar operations |
| `contactAgent` | toolWorkflow | Sub-workflow | All Airtable contact operations |
| `contentCreator` | toolWorkflow | Sub-workflow | Blog and content generation |
| `Tavily` | toolHttpRequest | HTTP Tool | Direct web search via Tavily API |
| `Calculator` | toolCalculator | Built-in | Arithmetic and math operations |
| `Think` | toolThink | Built-in | Internal reasoning scratchpad for verification |

### Routing Logic

The parent agent uses LLM-based intent classification. The system prompt enumerates each tool with a description. The LLM selects tools based on semantic matching of the user query against tool descriptions. No hard-coded switch/case logic exists in the workflow.

**Pre-resolution rules enforced in the system prompt:**

The following actions require the parent to call `contactAgent` first to resolve a person's name to their email address before calling the target agent:
- Sending emails
- Drafting emails
- Creating calendar events with attendees

### Decision Process

```
1. Receive user query (text or transcribed voice)
2. Load conversation history from memory (window buffer, keyed to chat.id)
3. Classify intent → select tool(s)
4. If action requires contact lookup → call contactAgent first
5. Call primary tool(s) with enriched query
6. Call Think tool to verify result
7. Compose and return natural language response
```

### Example Requests and Expected Tool Calls

| User Input | Tool Sequence |
|-----------|--------------|
| "Send an email to Marcus about the deadline" | `contactAgent` → `emailAgent` |
| "What's on my calendar Friday?" | `calendarAgent` |
| "Write a blog about serverless computing" | `contentCreator` |
| "Add Jane Doe to my contacts, her email is jane@co.com" | `contactAgent` |
| "Search for news about the EU AI Act" | `Tavily` |
| "What is 23% of 15,000?" | `Calculator` |
| "Schedule lunch with Sarah tomorrow at noon" | `contactAgent` → `calendarAgent` |
| "Get my last 5 emails from hr@company.com" | `emailAgent` |

---

## Email Agent

### Purpose

Manages the full lifecycle of Gmail interactions. Accepts a natural language query from the parent agent and translates it into one or more Gmail API operations. All outgoing emails are formatted as HTML and signed off as "Nate."

### n8n Node Type

`@n8n/n8n-nodes-langchain.agent` — typeVersion 1.6

### LLM

OpenAI GPT-4o (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)

### Trigger

`n8n-nodes-base.executeWorkflowTrigger` — `inputSource: passthrough`

### Inputs

| Field | Source | Type | Description |
|-------|--------|------|-------------|
| `query` | `$json.query` | string | Natural language instruction from parent agent |

The query may include pre-resolved contact information, e.g.:  
`"Send an email to Marcus Williams at marcus@email.com asking about the project timeline."`

### Outputs

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | Confirmation message on success, or `"Unable to perform task. Please try again."` on error |

### Supported Operations

| Operation | Gmail Tool | Required Pre-steps | AI Parameters |
|-----------|-----------|-------------------|---------------|
| Send email | `gmailTool` (default operation) | None (email address should be in query) | `emailAddress`, `subject`, `emailBody` |
| Create draft | `gmailTool` (operation: draft) | None | `emailAddress`, `subject`, `emailBody` |
| Get emails | `gmailTool` (operation: getAll) | None | `sender`, `limit` |
| Reply to email | `gmailTool` (operation: reply) | Must call `Get Emails` first | `ID` (messageId), `emailBody` |
| Apply label | `gmailTool` (operation: addLabels) | Must call `Get Emails` + `Get Labels` first | `ID`, `labelID` |
| Mark as unread | `gmailTool` (operation: markAsUnread) | Must call `Get Emails` first | `messageID` |
| List labels | `gmailTool` (resource: label) | None | None |

### Error Handling

- Agent node: `onError: continueErrorOutput`
- Error output routes to `Try Again` set node
- Returns: `"Unable to perform task. Please try again."`

### Example Queries (as received from parent agent)

```
"Send a professional email to john@example.com with subject 'Project Update' asking for the Q3 report status."

"Create a draft email to hr@company.com with subject 'Sick Day' notifying them I won't be in today."

"Get the last 3 emails from invoices@vendor.com"

"Reply to the email with ID 18abc123 confirming the meeting for Thursday at 2 PM."

"Label the email with ID 18abc124 with the 'Finance' label."

"Mark the email with ID 18abc125 as unread."
```

---

## Calendar Agent

### Purpose

Manages Google Calendar events including creation, retrieval, update, and deletion. Handles both solo events and events with external attendees. Automatically defaults to a one-hour duration when not specified.

### n8n Node Type

`@n8n/n8n-nodes-langchain.agent` — typeVersion 1.6

### LLM

OpenAI GPT-4o (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)

### Trigger

`n8n-nodes-base.executeWorkflowTrigger` — `inputSource: passthrough`

### Inputs

| Field | Source | Type | Description |
|-------|--------|------|-------------|
| `query` | `$json.query` | string | Natural language instruction from parent agent |

### Outputs

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | Confirmation message or retrieved event data on success, error string on failure |

### Supported Operations

| Operation | Tool Node | Required Pre-steps | AI Parameters |
|-----------|----------|-------------------|---------------|
| Create solo event | `Create Event` | None | `eventTitle`, `eventStart`, `eventEnd` |
| Create event with attendee | `Create Event with Attendee` | Attendee email pre-resolved by parent | `eventTitle`, `eventStart`, `eventEnd`, `eventAttendeeEmail` |
| Get events | `Get Events` | None | `dayBefore`, `dayAfter` (date window) |
| Delete event | `Delete Event` | Must call `Get Events` first to get event ID | `eventID` |
| Update event time | `Update Event` | Must call `Get Events` first to get event ID | `eventID`, `startTime`, `endTime` |

### Date/Time Handling

- Current date/time is injected via `{{ $now }}` in the system prompt
- The agent infers missing end times by adding 1 hour to the start time
- All datetime values are passed as ISO 8601 strings to the Google Calendar API
- The `Get Events` tool uses a `timeMin`/`timeMax` window; the agent is instructed to use the day before and after the requested date to avoid timezone edge cases

### Error Handling

- Agent node: `onError: continueErrorOutput`
- Error output routes to `Try Again` set node
- Returns: `"Unable to perform task. Please try again."`

### Example Queries (as received from parent agent)

```
"Create a calendar event titled 'Team Standup' on 2025-09-15 from 10:00 AM to 10:30 AM."

"Create a calendar event titled 'Client Call' on 2025-09-16 at 2:00 PM with attendee sarah@client.com."

"Get all calendar events for 2025-09-15."

"Delete the event titled 'Old Meeting' scheduled for 2025-09-17. Get events first to find the ID."

"Update the 'Strategy Session' event on 2025-09-18 to start at 3:00 PM and end at 4:00 PM."
```

---

## Contact Agent

### Purpose

Maintains a personal contact directory stored in Airtable. Supports searching for contacts by name, adding new contacts, and updating existing contact records. Implements upsert logic to prevent duplicate entries.

### n8n Node Type

`@n8n/n8n-nodes-langchain.agent` — typeVersion 1.7

### LLM

OpenAI GPT-4o (`@n8n/n8n-nodes-langchain.lmChatOpenAi`)

### Trigger

`n8n-nodes-base.executeWorkflowTrigger` — `inputSource: passthrough`

### Inputs

| Field | Source | Type | Description |
|-------|--------|------|-------------|
| `query` | `$json.query` | string | Natural language instruction from parent agent |

### Outputs

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | Retrieved contact data, confirmation of add/update, or error string |

### Airtable Schema

| Field Name | Airtable Type | Description |
|-----------|--------------|-------------|
| `name` | Single line text | Full name — used as the upsert matching key |
| `email` | Email | Contact's email address |
| `phoneNumber` | Phone number | Contact's phone number |

### Supported Operations

| Operation | Airtable Tool | n8n Operation | Match Key |
|-----------|--------------|---------------|-----------|
| Look up contact | `Get Contacts` | `search` | Any field |
| Add new contact | `Add or Update Contact` | `upsert` | `name` |
| Update contact | `Add or Update Contact` | `upsert` | `name` |

The `Add or Update Contact` tool uses Airtable's upsert operation matching on the `name` field. If a record with the same name exists, it is updated. If not, a new record is created.

### AI Field Mapping

The upsert tool uses `$fromAI()` expressions for field values:

| Airtable Field | Expression | AI Hint |
|---------------|------------|---------|
| `name` | `$fromAI("name")` | Contact's full name |
| `email` | `$fromAI("emailAddress")` | Contact's email address |
| `phoneNumber` | `$fromAI("phoneNumber")` | Contact's phone number |

### Error Handling

- Agent node: `onError: continueErrorOutput`
- Error output routes to `Try Again` set node
- Returns: `"An error occurred. Please try again."`

### Example Queries (as received from parent agent)

```
"Find the email address for Marcus Williams."

"Add a new contact: name 'Sarah Chen', email 'sarah@techcorp.com', phone '555-0192'."

"Update the phone number for John Smith to 555-4321."

"What is the email address for my accountant?"

"Get all contact information for David Park."
```

---

## Content Creator Agent

### Purpose

Generates SEO-optimized, well-researched blog posts in HTML format. Combines Tavily web search for current information gathering with Anthropic Claude's long-form writing capabilities. All output is formatted as semantic HTML with preserved source citations.

### n8n Node Type

`@n8n/n8n-nodes-langchain.agent` — typeVersion 1.7

### LLM

Anthropic Claude (`@n8n/n8n-nodes-langchain.lmChatAnthropic`) — typeVersion 1.2

### Trigger

`n8n-nodes-base.executeWorkflowTrigger` — `inputSource: passthrough`

### Inputs

| Field | Source | Type | Description |
|-------|--------|------|-------------|
| `query` | `$json.query` | string | Blog topic or content creation instruction from parent agent |

### Outputs

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | Complete HTML-formatted blog post with citations, or error string on failure |

### Output Format

Blog posts are returned as HTML strings using:

- `<h1>` — Blog post title
- `<h2>` — Section headings
- `<p>` — Body paragraphs
- `<ul><li>` — Bulleted lists
- `<a href="URL">` — Clickable citation links from Tavily results

### Supported Operations

| Operation | Description |
|-----------|-------------|
| Blog post generation | Full-length blog post with research, structure, and HTML formatting |
| Topic research | Tavily search is always called before writing to ensure current information |
| Citation injection | All Tavily source URLs are embedded as `<a>` tags in the output |

### Tavily Integration

The Content Creator Agent includes its own Tavily tool node (separate from the parent agent's Tavily tool). Configuration:

| Parameter | Value |
|-----------|-------|
| `search_depth` | `"basic"` |
| `topic` | `"news"` |
| `include_answer` | `true` |
| `include_raw_content` | `true` |
| `max_results` | `3` |

The AI parameter `searchTerm` is populated from the user's blog request topic.

### Error Handling

- Agent node: `onError: continueErrorOutput`
- Error output routes to `Try Again` set node
- Returns: `"Error occurred. Please try again."`

### Example Queries (as received from parent agent)

```
"Write a blog post about the latest developments in multimodal AI models."

"Create a 1000-word article on the benefits of workflow automation for small businesses."

"Write a technical blog post comparing PostgreSQL and MongoDB for production applications."

"Generate a blog post about the impact of AI on the software engineering profession in 2025."

"Write an SEO-optimized blog post about the best productivity tools for remote developers."
```

---

## Tool Reference

### Gmail (n8n-nodes-base.gmailTool)

**Authentication:** Gmail OAuth2 (`gmailOAuth2` credential)  
**API Base:** `https://gmail.googleapis.com`  
**n8n typeVersion:** 2.1

| Operation | n8n Resource/Operation | Key Parameters |
|-----------|----------------------|---------------|
| Send email | default / default | `sendTo`, `subject`, `message` (HTML) |
| Create draft | `draft` / default | `subject`, `message`, `options.sendTo` |
| Get emails | default / `getAll` | `limit`, `filters.sender` |
| Reply to email | default / `reply` | `messageId`, `message` |
| Add labels | default / `addLabels` | `messageId`, `labelIds` |
| Mark as unread | default / `markAsUnread` | `messageId` |
| Get labels | `label` / default | None (returns all labels) |

**Usage Notes:**
- All emails are sent as HTML (`emailType: "html"`). The agent system prompt instructs the LLM to format email bodies in HTML.
- Attribution footer is disabled (`appendAttribution: false`) to avoid the "Sent via n8n" footer.
- When replying or labeling, the email thread must first be fetched via `Get Emails` to obtain the `messageId`.
- `Get Emails` returns `simple: false` to include full headers and metadata needed for follow-up operations.

---

### Google Calendar (n8n-nodes-base.googleCalendarTool)

**Authentication:** Google Calendar OAuth2 (`googleCalendarOAuth2Api` credential)  
**API Base:** `https://www.googleapis.com/calendar/v3`  
**n8n typeVersion:** 1.3

| Operation | n8n Operation | Key Parameters |
|-----------|--------------|---------------|
| Create event | default | `calendar`, `start`, `end`, `additionalFields.summary` |
| Create event with attendee | default | `calendar`, `start`, `end`, `additionalFields.summary`, `additionalFields.attendees[]` |
| Get events | `getAll` | `calendar`, `timeMin`, `timeMax` |
| Delete event | `delete` | `calendar`, `eventId` |
| Update event | `update` | `calendar`, `eventId`, `updateFields.start`, `updateFields.end` |

**Usage Notes:**
- All datetime values must be ISO 8601 format (e.g., `2025-09-15T14:00:00+05:30`).
- The `Get Events` tool uses `dayBefore` and `dayAfter` parameters to bracket the requested date, ensuring events near midnight are not missed due to timezone offsets.
- The `Update Event` tool in the current implementation only updates `start` and `end` times. Title/description updates would require extending the `updateFields` configuration.
- Calendar ID is set to the account email address, which references the primary calendar.

---

### Airtable (n8n-nodes-base.airtableTool)

**Authentication:** Airtable API (Personal Access Token)  
**API Base:** `https://api.airtable.com/v0`  
**n8n typeVersion:** 2.1

| Operation | n8n Operation | Key Parameters |
|-----------|--------------|---------------|
| Search contacts | `search` | `base`, `table` |
| Upsert contact | `upsert` | `base`, `table`, `columns`, `matchingColumns: ["name"]` |

**Airtable Base/Table IDs used:**
- Base: `appK0rbtvf9e7vt6w` (Contacts database)
- Table: `tbl08JGCfUK1RhXsG` (Contacts table)

**Usage Notes:**
- The upsert operation matches on the `name` field. If a record with that name exists, it is updated. Otherwise, a new record is created.
- `$fromAI()` expressions allow the LLM to populate field values dynamically based on the query.
- `attemptToConvertTypes: false` and `convertFieldsToString: false` are set to prevent data type coercion errors.
- Update your base and table IDs in both Airtable tool nodes after importing the Contact Agent workflow.

---

### Tavily (toolHttpRequest — HTTP POST)

**Authentication:** API key in request body  
**Endpoint:** `https://api.tavily.com/search`  
**n8n Node Type:** `@n8n/n8n-nodes-langchain.toolHttpRequest` — typeVersion 1.1

**Request body schema:**

```json
{
  "api_key": "YOUR_TAVILY_API_KEY",
  "query": "{searchTerm}",
  "search_depth": "basic",
  "include_answer": true,
  "topic": "news",
  "include_raw_content": true,
  "max_results": 3
}
```

**Usage Notes:**
- `{searchTerm}` is a placeholder definition of type `string` resolved by the LLM at runtime.
- `include_raw_content: true` returns the full page content of each result for richer answers.
- `include_answer: true` returns Tavily's own AI-generated summary of the search results.
- The same Tavily API key is used in both the parent agent's direct Tavily tool and the Content Creator Agent's Tavily tool. Replace both occurrences when setting up.
- Tavily's free tier has a request limit. Monitor usage at [app.tavily.com](https://app.tavily.com).

---

### Telegram Bot API

**Authentication:** Bot token (via n8n Telegram credential)  
**Trigger:** Webhook — n8n registers the webhook automatically on workflow activation  
**n8n Node Types used:**
- `n8n-nodes-base.telegramTrigger` (typeVersion 1.1) — receives messages
- `n8n-nodes-base.telegram` (typeVersion 1.2) — downloads files and sends messages

| Operation | Node | Description |
|-----------|------|-------------|
| Receive message | Telegram Trigger | Fires on all `message` updates |
| Download voice file | Telegram (resource: file) | Fetches voice binary using `file_id` |
| Send text message | Telegram (default) | Sends reply to `chat.id` |

**Usage Notes:**
- The trigger is configured to listen for `message` update type only. Callback queries, inline queries, and other update types are not handled.
- Voice messages arrive with `message.voice.file_id` which must be fetched before transcription.
- Response messages reference the original `chat.id` via `$('Telegram Trigger').item.json.message.chat.id`.
- `appendAttribution: false` prevents the "Sent via n8n" footer on outgoing messages.

---

### OpenRouter

**Authentication:** API key (Bearer token)  
**API Base:** `https://openrouter.ai/api/v1`  
**n8n Node Type:** `@n8n/n8n-nodes-langchain.lmChatOpenRouter` — typeVersion 1

**Usage Notes:**
- OpenRouter acts as a unified gateway, allowing the parent agent's LLM to be swapped without changing workflow structure — only the credential/model selection needs updating.
- Recommended models: `anthropic/claude-3.5-sonnet`, `openai/gpt-4o`, `google/gemini-1.5-pro`
- OpenRouter charges per token at the upstream provider's rate plus a small routing fee. Monitor costs at [openrouter.ai/activity](https://openrouter.ai/activity).

---

### OpenAI

**Authentication:** API key (Bearer token)  
**API Base:** `https://api.openai.com/v1`  
**n8n Node Types used:**
- `@n8n/n8n-nodes-langchain.lmChatOpenAi` (typeVersion 1 / 1.2) — chat completions for Email, Calendar, Contact agents
- `@n8n/n8n-nodes-langchain.openAi` (typeVersion 1.6) — audio transcription via Whisper

| Usage | Model | Operation |
|-------|-------|-----------|
| Email Agent LLM | `gpt-4o` | Chat completions |
| Calendar Agent LLM | `gpt-4o` | Chat completions |
| Contact Agent LLM | `gpt-4o` | Chat completions |
| Voice Transcription | `whisper-1` | `audio/transcribe` |

**Usage Notes:**
- All three sub-agents use GPT-4o for its strong instruction-following capabilities with structured tool use.
- The Whisper transcription node accepts binary audio data piped from the Telegram download node.
- A single OpenAI API key covers both GPT-4o and Whisper usage.
- Set spending limits and usage alerts at [platform.openai.com/account/limits](https://platform.openai.com/account/limits).

---

### Anthropic (Claude)

**Authentication:** API key (`x-api-key` header)  
**API Base:** `https://api.anthropic.com`  
**n8n Node Type:** `@n8n/n8n-nodes-langchain.lmChatAnthropic` — typeVersion 1.2

**Usage:**
- Used exclusively by the Content Creator Agent for long-form blog post generation.
- Claude is selected for this agent due to its superior performance on extended prose, nuanced tone, and adherence to HTML formatting instructions.

**Usage Notes:**
- The specific Claude model version is selected in the node's `options` parameter. Default is the latest Claude Sonnet model available in n8n's Anthropic integration.
- Monitor token usage at [console.anthropic.com](https://console.anthropic.com).
- Anthropic's API has separate rate limits from OpenAI; factor this in for high-volume content generation.
