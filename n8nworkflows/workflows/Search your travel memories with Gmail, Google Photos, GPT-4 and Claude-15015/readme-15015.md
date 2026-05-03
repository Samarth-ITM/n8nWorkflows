Search your travel memories with Gmail, Google Photos, GPT-4 and Claude

https://n8nworkflows.xyz/workflows/search-your-travel-memories-with-gmail--google-photos--gpt-4-and-claude-15015


# Search your travel memories with Gmail, Google Photos, GPT-4 and Claude

# Workflow Analysis: Conversational Trip Memory Search Engine

This document provides a detailed technical analysis of the n8n workflow designed to search personal travel memories across Gmail, Google Calendar, and Google Photos using natural language processing.

---

### 1. Workflow Overview

The **Conversational Trip Memory Search Engine** is a sophisticated retrieval system that allows users to query their travel history using natural language (e.g., *"Where did I stay in Paris?"*). The workflow transforms a vague user request into structured search parameters, fetches data from multiple Google APIs, fuses that data into a cohesive set of results, and uses a high-reasoning AI model to generate a conversational summary.

The logic is divided into four functional blocks:
- **1.1 Query Understanding & Parsing:** Converts a webhook request into a structured JSON search object.
- **1.2 Multi-Source Data Retrieval:** Executes parallel searches across Gmail, Calendar, and Photos.
- **1.3 Data Fusion & Ranking:** Combines fragmented data and filters for relevance.
- **1.4 AI Enhancement & Delivery:** Generates a human-friendly response and logs the interaction.

---

### 2. Block-by-Block Analysis

#### 2.1 Query Understanding & Parsing
**Overview:** This block acts as the "brain" of the intake process, translating human language into machine-readable filters.

- **Nodes Involved:** `Webhook Trigger`, `Extract Query`, `AI Parse Query`, `Structure Parameters`.
- **Node Details:**
    - **Webhook Trigger:** Listens for `POST` requests on the path `trip-search`.
    - **Extract Query (Code):** Simple JS to ensure `query` and `userId` exist; initializes a `conversationId` for tracking.
    - **AI Parse Query (HTTP Request):** Calls OpenAI GPT-4. It uses a system prompt to force the AI to return **ONLY** valid JSON containing `location`, `dateRange` (start/end), `tripType`, and `keywords`.
    - **Structure Parameters (Code):** Cleans the AI's response (removing Markdown code blocks like ` ```json `) and maps it to a standardized object for downstream nodes.
    - **Potential Failures:** JSON parsing errors if the AI returns non-JSON text; OpenAI API timeouts.

#### 2.2 Multi-Source Data Retrieval
**Overview:** This block fetches raw data from various Google services based on the structured parameters.

- **Nodes Involved:** `Search Gmail`, `Search Calendar`, `Search Photos`, `Wait 1 - API Cool Down`.
- **Node Details:**
    - **Search Gmail (HTTP Request):** Queries the Gmail API using a combination of known booking domains (booking.com, airbnb.com) and the extracted location/dates.
    - **Search Calendar (HTTP Request):** Queries the Google Calendar API using `timeMin` and `timeMax` based on the parsed dates.
    - **Search Photos (HTTP Request):** Uses the Google Photos Library API to filter media items within the specific date range.
    - **Wait 1 - API Cool Down (Wait):** A 3-second pause to prevent hitting Google API rate limits (429 errors) when triggering three simultaneous requests.
    - **Potential Failures:** OAuth2 token expiration; Google API quota limits; date format mismatches.

#### 2.3 Data Fusion & Ranking
**Overview:** This block aggregates the results from the three disparate sources and removes noise.

- **Nodes Involved:** `Merge All Sources`, `Fuse Data`, `Wait 2 - Data Processing`.
- **Node Details:**
    - **Merge All Sources (Merge):** Combines the output streams from Gmail, Calendar, and Photos into a single data object.
    - **Fuse Data (Code):** 
        - Maps sources to specific types (`booking`, `event`, `image`).
        - Performs a case-insensitive string search to ensure the `location` appears in the results.
        - Calculates the total count of items found per source.
    - **Wait 2 - Data Processing (Wait):** A 2-second buffer to ensure the merged data is fully processed before being sent to the next AI model.
    - **Potential Failures:** Large data payloads causing memory issues during the `JSON.stringify` operation in the code node.

#### 2.4 AI Enhancement & Response Formatting
**Overview:** The final stage transforms raw data into a polished, conversational answer.

- **Nodes Involved:** `AI Enrich Response`, `Format Response`, `Wait 3 - Final Polish`, `Return Response`, `Log to Drive`.
- **Node Details:**
    - **AI Enrich Response (HTTP Request):** Sends the original query and the fused results to **Claude (Anthropic)**. It instructs the AI to create a friendly, rich response including direct answers, key details, and related memories.
    - **Format Response (Code):** Structures the final JSON payload, including the AI's text, a metadata object, a sliced list of the top 10 results, and dynamic "suggested follow-up" queries.
    - **Wait 3 - Final Polish (Wait):** 1-second pause before final delivery.
    - **Return Response (Respond to Webhook):** Sends the final formatted JSON back to the user.
    - **Log to Drive (Google Drive):** Saves the transaction details to a text file in Google Drive for auditing and analytics.
    - **Potential Failures:** Anthropic API timeouts; failures in the Google Drive folder permissions.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Webhook Trigger | Webhook | Entry Point | - | Extract Query | Conversational Trip Memory Search Engine |
| Extract Query | Code | Request Sanitization | Webhook Trigger | AI Parse Query | Conversational Trip Memory Search Engine |
| AI Parse Query | HTTP Request | NLP Intent Extraction | Extract Query | Structure Parameters | Stage 1: Query Understanding & Parsing |
| Structure Parameters | Code | JSON Formatting | AI Parse Query | Search Gmail, Calendar, Photos | Stage 1: Query Understanding & Parsing |
| Search Gmail | HTTP Request | Email Retrieval | Structure Parameters | Wait 1 | Stage 2: Multi-Source Data Retrieval |
| Search Calendar | HTTP Request | Event Retrieval | Structure Parameters | Wait 1 | Stage 2: Multi-Source Data Retrieval |
| Search Photos | HTTP Request | Image Retrieval | Structure Parameters | Wait 1 | Stage 2: Multi-Source Data Retrieval |
| Wait 1 | Wait | API Rate Limiting | Gmail, Calendar, Photos | Merge All Sources | Stage 2: Multi-Source Data Retrieval |
| Merge All Sources | Merge | Data Aggregation | Wait 1 | Fuse Data | Stage 2: Multi-Source Data Retrieval |
| Fuse Data | Code | Deduplication & Filtering | Merge All Sources | Wait 2 | Stage 3: Data Fusion & Ranking |
| Wait 2 | Wait | Processing Buffer | Fuse Data | AI Enrich Response | Stage 3: Data Fusion & Ranking |
| AI Enrich Response | HTTP Request | Conversational Synthesis | Wait 2 | Format Response | Stage 4: AI Enhancement & Response Formatting |
| Format Response | Code | Final Schema Mapping | AI Enrich Response | Wait 3 | Stage 4: AI Enhancement & Response Formatting |
| Wait 3 | Wait | Final Delivery Buffer | Format Response | Return Response, Log to Drive | Stage 4: AI Enhancement & Response Formatting |
| Return Response | Respond to Webhook | Client Response | Wait 3 | - | Stage 4: AI Enhancement & Response Formatting |
| Log to Drive | Google Drive | Audit Logging | Wait 3 | - | Stage 4: AI Enhancement & Response Formatting |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Input & Parsing
1. Create a **Webhook** node: Set path to `trip-search` and method to `POST`.
2. Add a **Code** node (`Extract Query`): Extract `body.query` and `body.userId`. Generate a `conversationId`.
3. Add an **HTTP Request** node (`AI Parse Query`):
    - URL: `https://api.openai.com/v1/chat/completions`
    - Method: `POST`
    - Auth: OpenAI API Key.
    - Body: JSON requesting GPT-4 to extract `location`, `dateRange`, `tripType`, and `keywords`.
4. Add a **Code** node (`Structure Parameters`): Use `JSON.parse()` to clean the AI output and create a `searchParams` object.

#### Step 2: Parallel Retrieval
1. Create three **HTTP Request** nodes:
    - **Gmail:** `https://gmail.googleapis.com/gmail/v1/users/me/messages` (Query using `q` param).
    - **Calendar:** `https://www.googleapis.com/calendar/v3/calendars/primary/events` (Query using `timeMin`/`timeMax`).
    - **Photos:** `https://photoslibrary.googleapis.com/v1/mediaItems:search` (Body with `dateFilter`).
    - *All three require Google OAuth2 credentials.*
2. Connect all three to a **Wait** node: Set to 3 seconds.

#### Step 3: Fusion & Ranking
1. Add a **Merge** node: Set mode to `Combine`. Connect the Wait node to it.
2. Add a **Code** node (`Fuse Data`):
    - Aggregate arrays from Gmail, Calendar, and Photos.
    - Filter items based on the `searchParams.location` string.
3. Add a **Wait** node: Set to 2 seconds.

#### Step 4: Enrichment & Delivery
1. Add an **HTTP Request** node (`AI Enrich Response`):
    - URL: `https://api.anthropic.com/v1/messages`
    - Method: `POST`
    - Auth: Anthropic API Key.
    - Body: Pass the original query and the fused results to Claude-Sonnet.
2. Add a **Code** node (`Format Response`): Map the AI response and top 10 results into a final JSON object.
3. Add a **Wait** node: Set to 1 second.
4. Add a **Respond to Webhook** node: Set response body to the output of the Format node.
5. Add a **Google Drive** node: Set operation to `Create from Text` to log the final response.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| High-level system purpose: Natural Language Travel History Search | Welcome Guide Sticky Note |
| Strategic use of 3 Wait nodes for API protection and processing stability | Workflow Performance |
| Data sources include Gmail, Calendar, Photos, Maps, and Expense tracking | Data Architecture |
| Future enhancements suggested: Voice search, Multi-language, Automatic summaries | Future Roadmap |