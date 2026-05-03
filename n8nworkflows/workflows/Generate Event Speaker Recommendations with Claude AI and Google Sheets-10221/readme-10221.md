Generate Event Speaker Recommendations with Claude AI and Google Sheets

https://n8nworkflows.xyz/workflows/generate-event-speaker-recommendations-with-claude-ai-and-google-sheets-10221


# Generate Event Speaker Recommendations with Claude AI and Google Sheets

### 1. Workflow Overview

This n8n workflow, titled **"AI-Driven Speaker & Session Recommendation Engine"**, is designed to generate tailored speaker recommendations for events by integrating event requirements, speaker databases, and audience interests. It is specifically targeted at event organizers or voice agent platforms that seek AI-optimized speaker and session suggestions in real time.

The workflow logically divides into five main blocks:

- **1.1 Input Reception:** Receives event data via a webhook from voice agents or other external systems.
- **1.2 Data Retrieval:** Fetches speaker profiles and audience interest data from Google Sheets.
- **1.3 Data Aggregation & AI Processing:** Combines all fetched data and sends it to an AI agent (Claude AI via Langchain) for analysis and recommendation generation.
- **1.4 Response Formatting:** Parses and formats AI responses into structured data and natural language summaries for voice agents.
- **1.5 Output & Error Handling:** Returns recommendations to the requesting system, saves data to Google Sheets, and manages error scenarios gracefully.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming requests with event details and audience preferences via a webhook to initiate the recommendation process.

- **Nodes Involved:**  
  - Webhook Trigger  
  - Parse Voice Request  
  - Sticky Note - Webhook

- **Node Details:**  

  - **Webhook Trigger**  
    - *Type:* Webhook node  
    - *Role:* Entry point for HTTP POST requests containing event data  
    - *Configuration:* HTTP POST method at path `/speaker-recommendations`  
    - *Input:* External HTTP request payload  
    - *Output:* Raw JSON payload forwarded downstream  
    - *Edge Cases:* Invalid HTTP method or malformed payload could cause failures; no authentication configured, so control exposure is critical.

  - **Parse Voice Request**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Validates and enriches received payload with defaults for missing properties  
    - *Key Logic:* Extracts properties `eventType`, `eventGoals`, `audienceSize`, `preferredTopics`, `sessionCount`, `requestId`, `timestamp`, and `source`  
    - *Input:* Raw webhook JSON  
    - *Output:* Structured JSON with normalized event request data  
    - *Edge Cases:* Missing fields are replaced with defaults (e.g., event type defaults to "Technology Conference"); malformed JSON input could cause parsing errors.

  - **Sticky Note - Webhook**  
    - *Content:* Explains expected webhook payload structure:  
      ```json
      {
        "eventType": "string",
        "eventGoals": "string",
        "audienceSize": number,
        "preferredTopics": []
      }
      ```  
    - *Purpose:* Documentation for users and maintainers.

---

#### 1.2 Data Retrieval

- **Overview:**  
  Fetches speaker profiles and audience interest data from specified Google Sheets to provide context for AI recommendations.

- **Nodes Involved:**  
  - Fetch Speakers Data  
  - Fetch Audience Data  
  - Sticky Note - Fetch

- **Node Details:**  

  - **Fetch Speakers Data**  
    - *Type:* Google Sheets node  
    - *Role:* Retrieves speaker database including profiles, expertise, past sessions, ratings, and availability  
    - *Configuration:* Reads from sheet with `gid=0` in a specified spreadsheet (ID obfuscated as `YOUR_SPREADSHEET_ID`) using a Google Service Account for authentication  
    - *Input:* Triggered after parsing request  
    - *Output:* List of speaker records  
    - *Edge Cases:* Google API authentication errors, empty speaker data, or sheet access issues.

  - **Fetch Audience Data**  
    - *Type:* Google Sheets node  
    - *Role:* Retrieves audience interests and preferences from sheet `gid=1` in the same spreadsheet  
    - *Configuration:* Similar to Fetch Speakers Data node  
    - *Input:* Triggered concurrently with speaker data fetch  
    - *Output:* Audience interest records  
    - *Edge Cases:* Same as Fetch Speakers Data.

  - **Sticky Note - Fetch**  
    - *Content:* Clarifies the retrieval of speaker profiles, expertise, past sessions, ratings, and availability.

---

#### 1.3 Data Aggregation & AI Processing

- **Overview:**  
  Combines event request, speakers, and audience data into a single structured object, then uses Claude AI (via Langchain AI Agent) to generate speaker recommendations optimized for event goals.

- **Nodes Involved:**  
  - Aggregate All Data  
  - AI Agent  
  - Anthropic Chat Model  
  - Sticky Note - Process  
  - Sticky Note - AI Analysis

- **Node Details:**  

  - **Aggregate All Data**  
    - *Type:* Code node  
    - *Role:* Merges data from previous nodes into one structured JSON object for AI consumption  
    - *Key Logic:* Normalizes speakers and audience data as arrays, combines with event details and metadata (requestId, timestamp, source)  
    - *Input:* Outputs of Parse Voice Request, Fetch Speakers Data, Fetch Audience Data  
    - *Output:* Combined JSON with keys `request`, `speakers`, `audienceInterests`, `eventDetails`, and `metadata`  
    - *Edge Cases:* Missing or malformed data from any input could affect AI input quality.

  - **AI Agent**  
    - *Type:* Langchain Agent node  
    - *Role:* Orchestrates AI processing workflow, passing input to the language model and receiving the response  
    - *Configuration:* Connected to Anthropic Chat Model as language model  
    - *Input:* Aggregated data JSON  
    - *Output:* Raw AI recommendation response  
    - *Version:* Requires Langchain integration v2.2 or higher  
    - *Edge Cases:* API errors, rate limits, or malformed AI responses.

  - **Anthropic Chat Model**  
    - *Type:* Langchain Anthropic LM Chat node  
    - *Role:* AI model providing natural language generation and analysis (Claude Sonnet 4)  
    - *Configuration:* Model set to `claude-sonnet-4-20250514` with credentials linked to Anthropic API account  
    - *Input:* Text prompt based on aggregated data  
    - *Output:* AI-generated text response with recommendations  
    - *Edge Cases:* Authentication failures, timeout, or model unavailability.

  - **Sticky Note - Process**  
    - *Content:* Describes combining voice request data with speaker and audience profiles for matching.

  - **Sticky Note - AI Analysis**  
    - *Content:* Notes AI’s role in optimizing recommendations based on fit, diversity, and engagement potential.

---

#### 1.4 Response Formatting

- **Overview:**  
  Parses AI-generated raw response, extracts structured recommendation details, and formats a natural language summary suitable for voice agents.

- **Nodes Involved:**  
  - Format for Voice Response  
  - Sticky Note - Format

- **Node Details:**  

  - **Format for Voice Response**  
    - *Type:* Code node  
    - *Role:* Parses AI JSON output, handles parsing errors, and constructs a voice-friendly response message summarizing recommendations  
    - *Key Logic:*  
      - Tries to extract JSON object from AI text using regex  
      - Builds output including status, requestId, timestamp, total recommendations, recommendations array, alternative speakers, diversity score, topic coverage, and summary  
      - Creates a natural language string highlighting the top speaker and inviting the user to hear more  
    - *Input:* AI Agent raw response  
    - *Output:* Structured JSON including `voiceResponse` string  
    - *Edge Cases:* JSON parsing failures, malformed AI output, no recommendations returned.

  - **Sticky Note - Format**  
    - *Content:* Explains formatting output for voice agents and system storage, including match scores and reasoning.

---

#### 1.5 Output & Error Handling

- **Overview:**  
  Sends final or error responses to the webhook caller, saves recommendation data to Google Sheets, and manages error detection.

- **Nodes Involved:**  
  - Send Voice Agent Response  
  - Save to Google Sheets  
  - Check for Errors  
  - Format Error Response  
  - Send Error Response  
  - Sticky Note - Save  
  - Sticky Note - Response

- **Node Details:**  

  - **Send Voice Agent Response**  
    - *Type:* Respond to Webhook node  
    - *Role:* Returns 200 OK with JSON-formatted recommendations to the requesting voice agent or external system  
    - *Input:* Formatted recommendation JSON  
    - *Output:* HTTP response  
    - *Edge Cases:* Network errors or delayed responses.

  - **Save to Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Appends recommendation data to `gid=2` sheet for tracking and analytics  
    - *Configuration:* Uses same Google Sheets document and service account credentials  
    - *Input:* Formatted recommendation JSON  
    - *Edge Cases:* API errors, write permission issues.

  - **Check for Errors**  
    - *Type:* If node  
    - *Role:* Evaluates if the formatted recommendations contain errors or zero recommendations  
    - *Conditions:* Checks non-empty error field or zero-length recommendations array  
    - *Input:* Formatted recommendations  
    - *Output:* Routes to error handling or normal flow accordingly

  - **Format Error Response**  
    - *Type:* Code node  
    - *Role:* Constructs a standardized error response JSON with a polite voice message for failure scenarios  
    - *Output:* Error JSON with status "error", requestId, timestamp, error message, and voiceResponse apologizing for failure

  - **Send Error Response**  
    - *Type:* Respond to Webhook node  
    - *Role:* Sends HTTP 500 error response with JSON error details to the caller

  - **Sticky Note - Save**  
    - *Content:* Notes saving recommendation history and analytics for tracking/reporting.

  - **Sticky Note - Response**  
    - *Content:* Describes returning formatted recommendations with natural language summary to voice agents.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                               | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                   |
|-------------------------|---------------------------------|-----------------------------------------------|---------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| Webhook Trigger         | Webhook                         | Receives incoming event requests               | —                               | Parse Voice Request                | ## Webhook Trigger Receives requests from voice agents or external systems with event requirements and audience preferences. Expected payload: { "eventType": "string", "eventGoals": "string", "audienceSize": number, "preferredTopics": [] } |
| Parse Voice Request     | Code                            | Extracts and normalizes webhook payload         | Webhook Trigger                 | Fetch Speakers Data, Fetch Audience Data, Aggregate All Data |                                                                                               |
| Fetch Speakers Data     | Google Sheets                   | Retrieves speaker data                           | Parse Voice Request             | Aggregate All Data                | ## Fetch Speaker Database Retrieve speaker profiles, expertise areas, past sessions, ratings, and availability from Google Sheets. |
| Fetch Audience Data     | Google Sheets                   | Retrieves audience interest data                 | Parse Voice Request             | Aggregate All Data                |                                                                                               |
| Aggregate All Data      | Code                            | Combines all input data for AI processing       | Fetch Speakers Data, Fetch Audience Data, Parse Voice Request | AI Agent                        | ## Calculate & Analyze Combine voice request data with speaker profiles and audience insights for comprehensive matching. |
| AI Agent                | Langchain Agent                 | Orchestrates AI recommendation generation       | Aggregate All Data              | Format for Voice Response         | ## AI Optimization Engine Claude analyzes speaker-audience fit, diversity, and engagement potential to recommend optimal session lineup. |
| Anthropic Chat Model    | Langchain LM Chat (Anthropic)  | Executes Claude AI model                          | AI Agent (ai_languageModel input) | AI Agent (ai_languageModel output) |                                                                                               |
| Format for Voice Response| Code                            | Parses AI response and formats voice/system output | AI Agent                       | Send Voice Agent Response, Save to Google Sheets, Check for Errors | ## Format Recommendations Structure AI output for both voice agent responses and system storage with match scores and reasoning. |
| Send Voice Agent Response| Respond to Webhook              | Returns successful JSON response to caller      | Format for Voice Response       | —                                 | ## Voice Agent Response Return formatted recommendations to the voice agent with natural language summary and structured data. |
| Save to Google Sheets   | Google Sheets                   | Saves recommendation history for tracking       | Format for Voice Response       | —                                 | ## Update Tracking Sheet Save recommendation history and analytics to Google Sheets for tracking and reporting. |
| Check for Errors        | If                             | Checks for errors or empty recommendations       | Format for Voice Response       | Format Error Response (on error), — (on success) |                                                                                               |
| Format Error Response   | Code                            | Formats error response JSON for the caller       | Check for Errors                | Send Error Response               |                                                                                               |
| Send Error Response     | Respond to Webhook              | Returns error JSON response with HTTP 500        | Format Error Response           | —                                 |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `speaker-recommendations`  
   - No authentication configured (consider security implications)  

2. **Add Code node "Parse Voice Request"**  
   - Place after webhook trigger  
   - JavaScript code to extract and normalize payload with defaults:  
     - `eventType` (default: "Technology Conference")  
     - `eventGoals` (default: "Engaging and innovative sessions")  
     - `audienceSize` (default: 500)  
     - `preferredTopics` (default: ["AI", "Cloud Computing", "Innovation"])  
     - `sessionCount` (default: 5)  
     - `requestId` (default: timestamp string)  
     - `timestamp` (current ISO datetime)  
     - `source` (default: "voice-agent")  

3. **Add Google Sheets node "Fetch Speakers Data"**  
   - Operation: Read (default)  
   - Spreadsheet ID: Your Google Sheets document ID  
   - Sheet Name: Use sheet with `gid=0`  
   - Authentication: Google Service Account credentials configured  
   - Connect from "Parse Voice Request"  

4. **Add Google Sheets node "Fetch Audience Data"**  
   - Same configuration as above, but Sheet Name: `gid=1`  
   - Connect from "Parse Voice Request"  

5. **Add Code node "Aggregate All Data"**  
   - Combine outputs of Parse Voice Request, Fetch Speakers Data, and Fetch Audience Data into one JSON object with keys: `request`, `speakers`, `audienceInterests`, `eventDetails`, `metadata`  
   - Connect all three previous nodes outputs into this node (fan-in)  

6. **Add Langchain Anthropic Chat Model node**  
   - Model: `claude-sonnet-4-20250514` (or latest available Claude model)  
   - Set Anthropic API credentials  
   - No special options required  

7. **Add Langchain AI Agent node**  
   - Connect input from "Aggregate All Data"  
   - Connect language model input to Anthropic Chat Model node  
   - Output connects to "Format for Voice Response"  

8. **Add Code node "Format for Voice Response"**  
   - Parses AI text response, extracts JSON recommendations, handles errors  
   - Constructs a voice-friendly summary string and structured response  
   - Connect from AI Agent node  

9. **Add Respond to Webhook node "Send Voice Agent Response"**  
   - HTTP Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Response Body: `={{ JSON.stringify($json, null, 2) }}`  
   - Connect from "Format for Voice Response"  

10. **Add Google Sheets node "Save to Google Sheets"**  
    - Operation: Append  
    - Spreadsheet ID: same as before  
    - Sheet Name: `gid=2` (recommendations tracking)  
    - Map incoming data for append  
    - Connect from "Format for Voice Response"  

11. **Add If node "Check for Errors"**  
    - Condition:  
      - Check if `$json.error` is non-empty OR  
      - Number of recommendations equals zero  
    - Connect from "Format for Voice Response"  

12. **Add Code node "Format Error Response"**  
    - Generates error JSON with status "error" and voice-friendly apology  
    - Connect from "Check for Errors" (error output)  

13. **Add Respond to Webhook node "Send Error Response"**  
    - HTTP Response Code: 500  
    - Response Headers: Content-Type: application/json  
    - Response Body: `={{ JSON.stringify($json, null, 2) }}`  
    - Connect from "Format Error Response"  

14. **Connect "Check for Errors" success output to normal flow (no further nodes needed)**

15. **Add Sticky Notes at appropriate positions** to document each block as described in Section 2.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow expects a secure environment or additional authentication on the webhook node to avoid unauthorized use. | Security best practices for webhooks: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/                  |
| Claude AI model (`claude-sonnet-4-20250514`) used via Anthropic API requires an API key and proper quota management. | Anthropic API docs: https://console.anthropic.com/docs                                                        |
| Google Sheets access uses Service Account credentials; ensure the service account has read/write permissions on the spreadsheet. | Google Sheets API docs: https://developers.google.com/sheets/api                                             |
| Voice-friendly response generation uses a simple heuristic summarizing top recommendations and inviting further interaction. | Can be extended with richer natural language generation if needed.                                        |
| The workflow assumes a JSON response format from Claude AI; malformed or unexpected responses trigger error handling. | Consider adding schema validation for AI responses to improve robustness.                                |
| Project inspired by event management use cases requiring dynamic, AI-driven speaker selection.       | No external blog links provided.                                                                          |

---

**Disclaimer:**  
The provided text is exclusively extracted from an automated n8n workflow. It fully complies with content policies and contains no illegal, offensive, or protected materials. All processed data is legal and public.