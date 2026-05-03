Real-time PulsePoint Emergency Alerts to iMessage with AI Summaries

https://n8nworkflows.xyz/workflows/real-time-pulsepoint-emergency-alerts-to-imessage-with-ai-summaries-5533


# Real-time PulsePoint Emergency Alerts to iMessage with AI Summaries

### 1. Workflow Overview

This workflow automates the retrieval of real-time emergency incident alerts from the PulsePoint API, processes and summarizes these alerts using AI, and then sends concise, user-friendly notifications via iMessage through the Blooio.com SMS API. It is designed for users who want to receive timely, readable updates on local emergency incidents directly on their phone.

Logical blocks of the workflow:

- **1.1 Input Reception:** Triggering the workflow either manually or on a schedule to fetch new incident data.
- **1.2 Incident Retrieval & Processing:** Fetching encrypted PulsePoint incident data, decrypting, filtering for new incidents, and structuring the data.
- **1.3 Incident Aggregation & Filtering:** Aggregating all new incidents and determining if there are any to process.
- **1.4 AI Summarization:** Using an AI agent (LangChain with OpenAI) to generate a concise, emoji-rich summary of the incidents.
- **1.5 Message Dispatch:** Sending the AI-generated summary as an SMS/iMessage through the Blooio.com HTTP API.
- **1.6 Documentation and Guidance:** Sticky notes providing configuration guidance, user instructions, and quick visual overview.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block provides two ways to start the workflow: manual execution and periodic scheduling every 60 minutes.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing or ad-hoc runs.  
  - Configuration: No parameters; triggers workflow immediately on user click.  
  - Inputs: None  
  - Outputs: Connected to “Get alerts” node.  
  - Edge Cases: N/A

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers the workflow every 60 minutes to fetch new incidents regularly.  
  - Configuration: Interval set to every 60 minutes (1 hour).  
  - Inputs: None  
  - Outputs: Connected to “Get alerts” node.  
  - Edge Cases: Potential issues if workflow runs longer than the interval or API rate limits.

---

#### 1.2 Incident Retrieval & Processing

**Overview:**  
Fetches encrypted incident data from PulsePoint API, decrypts it, filters out previously seen incidents using static workflow storage, maps incident and unit codes to meaningful descriptions, and outputs new incident data.

**Nodes Involved:**  
- Get alerts  

**Node Details:**

- **Get alerts**  
  - Type: Code Node (JavaScript)  
  - Role: Fetches and decrypts real-time PulsePoint emergency incidents, filters new incidents, and enriches data with human-readable mappings.  
  - Configuration:  
    - Uses Node.js `crypto` and `node-fetch` modules.  
    - Calls PulsePoint API endpoint: `https://api.pulsepoint.org/v1/webapp?resource=incidents&agencyid=19100` (agency ID hardcoded as 19100).  
    - Decrypts AES-256-CBC encrypted response with a derived key mimicking Python’s approach.  
    - Tracks seen incident IDs in workflow static data, retaining only last 100 IDs for memory efficiency.  
    - Maps incident codes (e.g., ME = Medical Emergency) and unit dispatch statuses to readable strings.  
    - Returns only new incidents not previously processed.  
  - Key Expressions:  
    - Uses `$getWorkflowStaticData('global')` for persistent incident ID tracking.  
  - Inputs: Triggered by manual or scheduled trigger.  
  - Outputs: Emits an array of new incident JSON objects with fields like ID, Type, Address, Time, Location, and Units.  
  - Edge Cases:  
    - API downtime or changes could break fetching.  
    - Decryption errors if API response format changes.  
    - Large numbers of incidents could strain static data memory.  
    - If no new incidents, outputs empty array.  
  - Version Requirements: Requires Node.js environment supporting `crypto` and `node-fetch`.  
  - Sub-workflows: None

---

#### 1.3 Incident Aggregation & Filtering

**Overview:**  
Aggregates incident objects into a single array and checks if there are any incidents to process.

**Nodes Involved:**  
- Merge all  
- If  

**Node Details:**

- **Merge all**  
  - Type: Code Node (JavaScript)  
  - Role: Consolidates multiple incident items into a single JSON object containing an array `incidents`.  
  - Configuration: Maps all input items’ JSON data into an `incidents` array within one output object.  
  - Inputs: From “Get alerts” node (array of new incidents).  
  - Outputs: Single object with `incidents` property.  
  - Edge Cases: Will produce empty array if no input items.

- **If**  
  - Type: If Node  
  - Role: Checks whether the `incidents` array length is greater than zero to decide if processing continues.  
  - Configuration: Condition tests `lengthGt 0` on `{{$json.incidents}}`.  
  - Inputs: From “Merge all”.  
  - Outputs: If True → “AI Agent”, If False → workflow ends (no further processing).  
  - Edge Cases: If empty or missing `incidents` property, condition evaluates false, preventing unnecessary AI calls.

---

#### 1.4 AI Summarization

**Overview:**  
Uses an AI agent powered by LangChain and OpenAI's chat model to generate a friendly, emoji-rich summary briefing of the new emergency incidents.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model Node  
  - Role: Provides the underlying language model interface for the AI Agent.  
  - Configuration:  
    - Model set to “o4-mini” (a compact OpenAI model).  
    - Uses stored OpenAI API credentials (`Tay - SLUG`).  
  - Inputs: None directly; connected as AI language model for “AI Agent”.  
  - Outputs: Responses passed to “AI Agent”.  
  - Edge Cases: API key limits, network errors, or model unavailability.

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Generates a concise summary of incidents with specific instructions to be brief, emoji-rich, and informative.  
  - Configuration:  
    - Input text is bound to the prior node’s output (`{{$input}}`).  
    - System message instructs the agent to report what is happening, where, time, units, and their status.  
  - Inputs: From “If” node when incidents exist.  
  - Outputs: AI-generated summary text in JSON property `output`.  
  - Edge Cases: AI model latency, unexpected input formats causing generation errors, API failures.

---

#### 1.5 Message Dispatch

**Overview:**  
Sends the AI-generated summary as a message to a configured phone number using the Blooio.com SMS API through an HTTP request.

**Nodes Involved:**  
- Send Message  

**Node Details:**

- **Send Message**  
  - Type: HTTP Request Node  
  - Role: Sends the summarized incident alert as an SMS/iMessage via Blooio.com API.  
  - Configuration:  
    - HTTP POST to `https://api.blooio.com/send-message`.  
    - Sends JSON body with two parameters: `identifier` (phone number in international format, e.g., `+11111111111`) and `message` (bound to AI Agent output `{{$json.output}}`).  
    - Headers include `Accept: application/json` and Bearer token authorization via stored credentials (`Blooio Bearer Auth account`).  
  - Inputs: From “AI Agent”.  
  - Outputs: Response from Blooio API (usually success/failure details).  
  - Edge Cases:  
    - HTTP errors, authentication failures, invalid phone number formats, API rate limits.  
  - Credentials: Requires valid Blooio API token configured in n8n credentials.

---

#### 1.6 Documentation and Guidance

**Overview:**  
Contains sticky notes with setup instructions, configuration guidance, and a visual workflow preview for user reference.

**Nodes Involved:**  
- Sticky Note (next to “Send Message”)  
- Sticky Note1 (near workflow start)  
- Sticky Note2 (near workflow start)  

**Node Details:**

- **Sticky Note** (next to Send Message)  
  - Provides detailed instructions on configuring the HTTP Request node for Blooio.com messaging, including method, URL, headers, and body setup.  
  - Includes example header content and notes on authorization.  

- **Sticky Note1**  
  - Shows a visual diagram preview of the Emergency Alerts to iMessage flow with a link to an image for quick understanding.  

- **Sticky Note2**  
  - Lists prerequisites and getting started guide: PulsePoint Agency ID, Blooio API key, OpenAI API key, and phone number formatting.  
  - Provides helpful links to external resources for each prerequisite.

---

### 3. Summary Table

| Node Name                | Node Type                        | Functional Role                          | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                         |
|--------------------------|---------------------------------|----------------------------------------|----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Manual start of the workflow            | None                       | Get alerts                 |                                                                                                                     |
| Schedule Trigger          | Schedule Trigger                | Scheduled start every 60 minutes         | None                       | Get alerts                 |                                                                                                                     |
| Get alerts               | Code Node (JavaScript)          | Fetch and decrypt PulsePoint incidents   | When clicking ‘Execute workflow’, Schedule Trigger | Merge all                  |                                                                                                                     |
| Merge all                | Code Node (JavaScript)          | Aggregate incidents into a single array  | Get alerts                 | If                         |                                                                                                                     |
| If                       | If Node                        | Check if there are new incidents          | Merge all                  | AI Agent                   |                                                                                                                     |
| AI Agent                 | LangChain Agent Node            | Generate AI summary of incidents          | If                         | Send Message               |                                                                                                                     |
| OpenAI Chat Model         | LangChain OpenAI Chat Model Node | Language model used by AI Agent          | None (connected as AI language model) | AI Agent                   |                                                                                                                     |
| Send Message             | HTTP Request                   | Send summary via Blooio.com SMS API       | AI Agent                   | None                       | "Using the n8n HTTP Request Node to Send a Blooio.com Message" with detailed setup steps and headers.                |
| Sticky Note              | Sticky Note                    | Setup instructions for Blooio HTTP Request | None                       | None                       | "Using the n8n HTTP Request Node to Send a Blooio.com Message" detailed guide.                                       |
| Sticky Note1             | Sticky Note                    | Visual workflow preview                    | None                       | None                       | "Quick Preview: Emergency Alerts → iMessage workflow in action" with image link.                                    |
| Sticky Note2             | Sticky Note                    | Getting started guide with prerequisites  | None                       | None                       | "Getting Started Guide" listing PulsePoint ID, Blooio API key, OpenAI API key, phone number, with helpful links.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Schedule Trigger Node**  
   - Name: `Schedule Trigger`  
   - Type: Schedule Trigger  
   - Set interval to every 60 minutes.

3. **Create Code Node to Fetch & Process Alerts**  
   - Name: `Get alerts`  
   - Type: Code (JavaScript)  
   - Paste the decryption and API fetch JavaScript code as provided, including:  
     - Fetching from `https://api.pulsepoint.org/v1/webapp?resource=incidents&agencyid=19100` (replace agency ID if needed).  
     - Decryption with AES-256-CBC using derived key.  
     - Static workflow data usage for filtering new incidents.  
     - Mapping incident and unit codes to readable values.  
   - Connect inputs from the Manual Trigger and Schedule Trigger nodes.

4. **Create Code Node to Merge Incidents**  
   - Name: `Merge all`  
   - Type: Code (JavaScript)  
   - Use code that aggregates all input items into one JSON with an `incidents` array.  
   - Connect output of `Get alerts` to this node.

5. **Create If Node to Check Incident Array Length**  
   - Name: `If`  
   - Type: If Node  
   - Condition: Check if `{{$json.incidents.length}} > 0` (array length greater than zero).  
   - Connect input from `Merge all`.

6. **Create OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain OpenAI Chat Model  
   - Select model `o4-mini` (or preferred OpenAI chat model).  
   - Configure with valid OpenAI API credentials.  
   - No direct input connection; used as AI language model for the Agent node.

7. **Create AI Agent Node**  
   - Name: `AI Agent`  
   - Type: LangChain Agent  
   - Set `text` parameter to `={{ $input }}` to receive incident data.  
   - Add system message instructing the agent to provide brief, emoji-rich incident summaries including what, where, time, units, and status.  
   - Set prompt type to `define`.  
   - Under AI Language Model, select the `OpenAI Chat Model` node.  
   - Connect “If” node’s True output to this node.

8. **Create HTTP Request Node to Send Message**  
   - Name: `Send Message`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.blooio.com/send-message`  
   - Authentication: HTTP Bearer with Blooio API token credential.  
   - Headers:  
     - `Accept: application/json`  
   - Body Content Type: JSON (application/json)  
   - Body parameters:  
     - `identifier`: Your phone number in international format (e.g., `+11111111111`)  
     - `message`: Bind to `{{$json.output}}` from AI Agent output.  
   - Connect input from `AI Agent`.

9. **Connect the Workflow**  
   - Manual Trigger and Schedule Trigger → Get alerts → Merge all → If  
   - If (True) → AI Agent → Send Message  
   - If (False) → End workflow

10. **Add Sticky Notes (Optional but Recommended)**  
    - Add sticky notes explaining the Blooio HTTP Request configuration, setup prerequisites, and visual workflow overview.  
    - Include links for API keys and agency IDs as per original notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                    | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| 🚨 **Using the n8n HTTP Request Node to Send a Blooio.com Message** — detailed instructions on HTTP method, URL, headers, authorization, and JSON body setup for sending SMS or email via Blooio.com.                                                                              | See Sticky Note near “Send Message” node in the workflow.                                                |
| 🟢 **Getting Started Guide:** Lists prerequisites including PulsePoint Agency ID, Blooio API key, OpenAI API key, and phone number format. Provides links to get API keys and PulsePoint agency details.                                                                           | Sticky Note near workflow start; links: [PulsePoint](https://web.pulsepoint.org), [Blooio](https://blooio.com), [OpenAI](https://platform.openai.com/account/api-keys) |
| 🚨 **Quick Preview:** Visual diagram of the Emergency Alerts → iMessage workflow showing real-time data fetching, AI summarization, and message sending. Useful for understanding workflow structure and logic.                                                                   | Image link: https://i.imgur.com/SDh1bG3.png                                                               |

---

**Disclaimer:** The provided text exclusively originates from an automated workflow created using n8n, an integration and automation tool. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible.