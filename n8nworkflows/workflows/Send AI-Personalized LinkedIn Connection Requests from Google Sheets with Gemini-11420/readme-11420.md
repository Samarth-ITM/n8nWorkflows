Send AI-Personalized LinkedIn Connection Requests from Google Sheets with Gemini

https://n8nworkflows.xyz/workflows/send-ai-personalized-linkedin-connection-requests-from-google-sheets-with-gemini-11420


# Send AI-Personalized LinkedIn Connection Requests from Google Sheets with Gemini

### 1. Workflow Overview

This workflow automates sending AI-personalized LinkedIn connection requests based on prospect data stored in a Google Sheet. It is designed for sales professionals, recruiters, and founders who want to scale genuine, non-robotic LinkedIn outreach.

The logical flow is organized into the following blocks:

- **1.1 Triggers**: Initiates workflow execution either on a schedule or manually, incorporating a randomized wait to simulate human behavior.
- **1.2 Google Sheets Interaction**: Reads one pending prospect from the Google Sheet and marks them as in-progress to prevent duplicate processing.
- **1.3 LinkedIn Profile Retrieval**: Fetches detailed LinkedIn profile data for the prospect using the ConnectSafely.AI API.
- **1.4 AI Message Generation**: Uses Google Gemini AI to craft a personalized LinkedIn connection message based on the retrieved profile.
- **1.5 Sending Connection Requests & Updating Sheet**: Sends the connection request with the AI-generated message via ConnectSafely.AI and updates the Google Sheet to mark the prospect as completed, recording the message sent.

---

### 2. Block-by-Block Analysis

#### 2.1 Triggers

- **Overview:**  
  This block initiates the workflow. It supports two start methods: a scheduled trigger running every minute and a manual trigger for testing. A random waiting period of 1 to 5 minutes is introduced to mimic human behavior and avoid detection as a bot.

- **Nodes Involved:**  
  - `Run Every Minute` (Schedule Trigger)  
  - `Manual Trigger (for testing)` (Manual Trigger)  
  - `Random Delay (1-5 min)` (Wait Node)

- **Node Details:**

  - **Run Every Minute**  
    - *Type:* Schedule Trigger  
    - *Role:* Automatically triggers the workflow every minute.  
    - *Configuration:* Interval set to every 1 minute.  
    - *Input:* None  
    - *Output:* Connects to `Random Delay (1-5 min)`  
    - *Edge Cases:* Scheduling delays if the workflow runs long; ensure idempotency.

  - **Manual Trigger (for testing)**  
    - *Type:* Manual Trigger  
    - *Role:* Allows manual start for testing the workflow.  
    - *Input:* None  
    - *Output:* Connects directly to `Get Pending Prospect` node.  
    - *Edge Cases:* None significant; for testing only.

  - **Random Delay (1-5 min)**  
    - *Type:* Wait Node  
    - *Role:* Adds a random delay between 1 to 5 minutes before proceeding.  
    - *Configuration:* Uses expression `={{ Math.floor(Math.random() * 4) + 1 }}` for delay in minutes.  
    - *Input:* From `Run Every Minute`  
    - *Output:* Connects to `Get Pending Prospect`  
    - *Edge Cases:* Timing issues if workflow is long-running; ensures variability in request timing.

---

#### 2.2 Google Sheets Interaction

- **Overview:**  
  This block reads one prospect with status "PENDING" from the Google Sheet, then marks that prospect as "IN PROGRESS" to prevent duplicate handling by concurrent workflow runs.

- **Nodes Involved:**  
  - `Get Pending Prospect` (Google Sheets Read)  
  - `Mark as In Progress` (Google Sheets Update)

- **Node Details:**

  - **Get Pending Prospect**  
    - *Type:* Google Sheets  
    - *Role:* Reads one prospect row marked as "PENDING" to process.  
    - *Configuration:* Points to a specified Google Sheet and sheet tab; filters or queries configured to fetch only pending prospects.  
    - *Credentials:* Uses OAuth2 to access Google Sheets.  
    - *Input:* From `Random Delay (1-5 min)` or `Manual Trigger`  
    - *Output:* To `Mark as In Progress`  
    - *Edge Cases:* Empty sheet or no pending prospects; should handle gracefully by stopping workflow or waiting.

  - **Mark as In Progress**  
    - *Type:* Google Sheets  
    - *Role:* Updates the prospect’s status to "IN PROGRESS" to lock it for processing.  
    - *Configuration:* Updates the same row with new status.  
    - *Credentials:* OAuth2 Google Sheets  
    - *Input:* From `Get Pending Prospect`  
    - *Output:* To `Fetch LinkedIn Profile`  
    - *Edge Cases:* Update failures due to permissions or sheet structure changes.

---

#### 2.3 LinkedIn Profile Retrieval

- **Overview:**  
  Retrieves the full LinkedIn profile data of the prospect from ConnectSafely.AI using their LinkedIn URL, enriching the data to aid personalized message generation.

- **Nodes Involved:**  
  - `Fetch LinkedIn Profile` (HTTP Request)

- **Node Details:**

  - **Fetch LinkedIn Profile**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to ConnectSafely.AI API to retrieve prospect profile data.  
    - *Configuration:*  
      - URL: `https://api.connectsafely.ai/linkedin/profile`  
      - Method: POST  
      - Body includes `profileId` extracted from the prospect’s LinkedIn URL field in Google Sheets.  
      - Auth: Bearer token via HTTP Bearer Auth credential using ConnectSafely.AI API key.  
    - *Input:* From `Mark as In Progress`  
    - *Output:* To `Generate Personalized Message`  
    - *Edge Cases:* API rate limits, invalid LinkedIn URLs, network failures, or authentication errors.

---

#### 2.4 AI Message Generation

- **Overview:**  
  Uses Google Gemini AI via n8n LangChain integration to generate a personalized, authentic LinkedIn connection message based on the fetched profile data. The system prompt is highly customized to produce natural, human-like messages.

- **Nodes Involved:**  
  - `Generate Personalized Message` (LangChain Agent)  
  - `Google Gemini` (Language Model)  
  - `Extract Message` (Structured Output Parser)

- **Node Details:**

  - **Generate Personalized Message**  
    - *Type:* LangChain Agent  
    - *Role:* Defines prompt and passes profile data to AI for message generation.  
    - *Configuration:*  
      - Input text is the JSON profile data from the previous node.  
      - Uses a detailed system prompt instructing the AI to create a conversational, 200-250 character LinkedIn invitation message avoiding corporate jargon and sounding authentic.  
      - Specifies message structure and style, including sign-off with user’s name (customizable).  
    - *Input:* From `Fetch LinkedIn Profile`  
    - *Output:* To `Send Connection Request` (via AI language model output)  
    - *Edge Cases:* AI timeouts, prompt errors, or unexpected output structure.

  - **Google Gemini**  
    - *Type:* Language Model Chat Node  
    - *Role:* Executes the AI generation request.  
    - *Configuration:* Uses Google Gemini model with default options.  
    - *Input:* From `Generate Personalized Message` (ai_languageModel)  
    - *Output:* To `Generate Personalized Message` (ai_languageModel output)  
    - *Edge Cases:* API limits, invalid credentials.

  - **Extract Message**  
    - *Type:* Structured Output Parser  
    - *Role:* Parses AI output JSON to extract the “message” field containing the personalized LinkedIn message.  
    - *Configuration:* Defines a JSON schema expecting an object with a required string property "message".  
    - *Input:* From `Generate Personalized Message` (ai_outputParser)  
    - *Output:* Used internally by `Generate Personalized Message` node.  
    - *Edge Cases:* Parsing errors if AI output is malformed.

---

#### 2.5 Sending Connection Requests & Updating Sheet

- **Overview:**  
  Sends the LinkedIn connection request with the personalized message via ConnectSafely.AI API, then updates the Google Sheet marking the prospect as "DONE" with the message sent.

- **Nodes Involved:**  
  - `Send Connection Request` (HTTP Request)  
  - `Mark as Complete` (Google Sheets Update)

- **Node Details:**

  - **Send Connection Request**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to ConnectSafely.AI to send the LinkedIn connection request.  
    - *Configuration:*  
      - URL: `https://api.connectsafely.ai/linkedin/connect`  
      - Method: POST  
      - Body includes:  
        - `profileId`: prospect LinkedIn URL from sheet  
        - `customMessage`: personalized message from AI output  
      - Auth: Bearer token via HTTP Bearer Auth credential (ConnectSafely API key)  
    - *Input:* From `Generate Personalized Message` node output (AI message)  
    - *Output:* To `Mark as Complete`  
    - *Edge Cases:* API failures, message length limits, invalid profile IDs, auth errors.

  - **Mark as Complete**  
    - *Type:* Google Sheets  
    - *Role:* Updates the prospect’s row to "DONE" status and records the personalized message sent.  
    - *Configuration:* Updates status and message columns accordingly.  
    - *Credentials:* Google Sheets OAuth2  
    - *Input:* From `Send Connection Request`  
    - *Output:* End of workflow  
    - *Edge Cases:* Update failures, sheet permission issues.

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                                      | Input Node(s)                        | Output Node(s)                     | Sticky Note                                                                                                       |
|----------------------------|---------------------------------------------|-----------------------------------------------------|------------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Main          | Sticky Note                                 | Documentation and overview                           | -                                  | -                                | ## 🔗 Send AI-Personalized LinkedIn Connection Requests from Google Sheets... (full content in overview)         |
| Sticky Note - Triggers      | Sticky Note                                 | Explains trigger options                             | -                                  | -                                | ### ⚡ Triggers: Two ways to start: Schedule or Manual, with random delay for human-like behavior                 |
| Sticky Note - Sheets        | Sticky Note                                 | Explains Google Sheets prospect fetching logic      | -                                  | -                                | ### 📊 Google Sheets: Fetches one PENDING prospect, marks as IN PROGRESS to prevent duplicates                    |
| Sticky Note - Profile       | Sticky Note                                 | Explains LinkedIn profile fetch                      | -                                  | -                                | ### 🔍 Profile Lookup: Fetches full LinkedIn profile data from ConnectSafely.AI                                  |
| Sticky Note - AI            | Sticky Note                                 | Explains AI message generation using Gemini         | -                                  | -                                | ### 🤖 AI Message Generation: Gemini creates personalized messages based on profile                              |
| Sticky Note - Send          | Sticky Note                                 | Explains sending connection request and updating    | -                                  | -                                | ### ✉️ Send & Update: Sends connection request and updates sheet with DONE status and message                   |
| Manual Trigger (for testing)| Manual Trigger                              | Manual workflow start for testing                    | -                                  | Get Pending Prospect             | ### ⚡ Triggers                                                                                            |
| Run Every Minute            | Schedule Trigger                            | Scheduled automatic workflow start every minute     | -                                  | Random Delay (1-5 min)           | ### ⚡ Triggers                                                                                            |
| Random Delay (1-5 min)      | Wait                                        | Adds random delay between 1-5 minutes                | Run Every Minute                   | Get Pending Prospect             | ### ⚡ Triggers                                                                                            |
| Get Pending Prospect        | Google Sheets                               | Reads one pending prospect row                        | Random Delay (1-5 min), Manual Trigger | Mark as In Progress             | ### 📊 Google Sheets                                                                                       |
| Mark as In Progress         | Google Sheets                               | Updates prospect status to IN PROGRESS               | Get Pending Prospect              | Fetch LinkedIn Profile           | ### 📊 Google Sheets                                                                                       |
| Fetch LinkedIn Profile      | HTTP Request                               | Retrieves LinkedIn profile data from ConnectSafely  | Mark as In Progress               | Generate Personalized Message    | ### 🔍 Profile Lookup                                                                                      |
| Generate Personalized Message| LangChain Agent                           | Generates personalized LinkedIn message via AI      | Fetch LinkedIn Profile            | Send Connection Request          | ### 🤖 AI Message Generation                                                                               |
| Google Gemini              | LangChain LM Chat Google Gemini            | Executes AI language model request                   | Generate Personalized Message (ai_languageModel) | Generate Personalized Message   | ### 🤖 AI Message Generation                                                                               |
| Extract Message            | LangChain Output Parser Structured          | Parses AI output to extract message field            | Generate Personalized Message (ai_outputParser) | Generate Personalized Message   | ### 🤖 AI Message Generation                                                                               |
| Send Connection Request    | HTTP Request                               | Sends connection request with personalized message  | Generate Personalized Message     | Mark as Complete                 | ### ✉️ Send & Update                                                                                       |
| Mark as Complete           | Google Sheets                               | Updates prospect status to DONE and logs message     | Send Connection Request           | -                                | ### ✉️ Send & Update                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers:**

   - Add a **Schedule Trigger** node named `Run Every Minute`:  
     - Set interval to every 1 minute.

   - Add a **Manual Trigger** node named `Manual Trigger (for testing)`.

2. **Add Random Delay:**

   - Add a **Wait** node named `Random Delay (1-5 min)`.  
   - Set Unit to "minutes".  
   - Set Amount to expression: `={{ Math.floor(Math.random() * 4) + 1 }}`.

3. **Configure Google Sheets Nodes:**

   - Add a **Google Sheets** node named `Get Pending Prospect`:  
     - Operation: Read rows.  
     - Set Document ID and Sheet Name to your target Google Sheet and tab.  
     - Filter or query to fetch only rows where `Status` is "PENDING".  
     - Limit to 1 row.

   - Add a **Google Sheets** node named `Mark as In Progress`:  
     - Operation: Update row.  
     - Use same Document ID and Sheet Name.  
     - Update the `Status` column for the fetched row to "IN PROGRESS".

   - Add a **Google Sheets** node named `Mark as Complete`:  
     - Operation: Update row.  
     - Update the `Status` column to "DONE" and the `Message` column with the personalized message.

   - **Credentials:** Configure Google Sheets OAuth2 credentials for all Google Sheets nodes.

4. **Add HTTP Request to Fetch LinkedIn Profile:**

   - Add an **HTTP Request** node named `Fetch LinkedIn Profile`:  
     - Method: POST  
     - URL: `https://api.connectsafely.ai/linkedin/profile`  
     - Authentication: HTTP Bearer Auth using your ConnectSafely.AI API key credential.  
     - Body Parameters:  
       - `profileId`: Expression extracting `LinkedIn Url` from `Get Pending Prospect` node, e.g., `={{ $('Get Pending Prospect').item.json['LinkedIn Url'] }}`.

5. **Add AI Message Generation Nodes:**

   - Add a **LangChain Agent** node named `Generate Personalized Message`:  
     - Input Text: Expression using the profile JSON from `Fetch LinkedIn Profile`, e.g., `={{ $json.profile }}` or adjust accordingly.  
     - Use a detailed system prompt (see below) instructing to generate a conversational LinkedIn connection message between 200-250 characters, personalized on the profile data, avoiding corporate language, signed off with your name.  
     - Set prompt type to "define" and enable output parser.

   - Add a **LangChain LM Chat Google Gemini** node named `Google Gemini`:  
     - Connect as AI language model for `Generate Personalized Message`.  
     - Set default options or configure as needed for Gemini API key.

   - Add an **Output Parser Structured** node named `Extract Message`:  
     - Schema: JSON object with required string property `message`.  
     - Connect as output parser for `Generate Personalized Message`.

   - **Credentials:** Configure Google Gemini API credentials accordingly.

6. **Add HTTP Request to Send Connection Request:**

   - Add an **HTTP Request** node named `Send Connection Request`:  
     - Method: POST  
     - URL: `https://api.connectsafely.ai/linkedin/connect`  
     - Authentication: HTTP Bearer Auth using ConnectSafely.AI API key.  
     - Body Parameters:  
       - `profileId`: same expression as before for LinkedIn URL.  
       - `customMessage`: expression extracting the AI-generated message, e.g., `={{ $json.message }}`.

7. **Connect Nodes:**

   - From `Run Every Minute` → `Random Delay (1-5 min)` → `Get Pending Prospect`  
   - From `Manual Trigger (for testing)` → `Get Pending Prospect`  
   - `Get Pending Prospect` → `Mark as In Progress` → `Fetch LinkedIn Profile` → `Generate Personalized Message`  
   - `Generate Personalized Message` ai_languageModel output → `Google Gemini`  
   - `Generate Personalized Message` ai_outputParser output → `Extract Message`  
   - `Generate Personalized Message` main output → `Send Connection Request` → `Mark as Complete`

8. **Customize Prompts:**

   - Edit the system prompt in `Generate Personalized Message` to include your name, role, and desired connection context.

9. **Final Checks:**

   - Ensure Google Sheet columns: `First Name`, `LinkedIn Url`, `Tagline`, `Status`, and `Message` exist.  
   - Test manual trigger first to verify all nodes work correctly.  
   - Adjust schedule trigger interval as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Workflow video demo available on YouTube: [https://youtu.be/ZCnXr3-W8xs](https://youtu.be/ZCnXr3-W8xs) | Workflow overview and usage guide video                                                                       |
| Requires ConnectSafely.AI account and API key for LinkedIn profile data and connection requests | https://connectsafely.ai                                                                                      |
| Google Sheets must have columns: `First Name`, `LinkedIn Url`, `Tagline`, `Status`, `Message`    | Data structure requirement                                                                                   |
| Customize AI prompt with your name and connection context for best personalization               | System prompt editable in `Generate Personalized Message` node                                               |
| Random delay simulates human behavior to reduce detection risk                                  | Implemented in `Random Delay (1-5 min)` wait node                                                            |

---

**Disclaimer:**  
The provided description and workflow are generated from the n8n automation platform. The workflow adheres strictly to content policies and handles only legal and public data. No illegal or protected content is involved.