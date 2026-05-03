Track and Visualize Daily Moods with GPT-4o, Wolfram Alpha, Sheets, and Slack

https://n8nworkflows.xyz/workflows/track-and-visualize-daily-moods-with-gpt-4o--wolfram-alpha--sheets--and-slack-10922


# Track and Visualize Daily Moods with GPT-4o, Wolfram Alpha, Sheets, and Slack

### 1. Workflow Overview

This workflow is designed to **track and visualize daily user moods** by analyzing mood text input, quantifying emotional parameters, generating visual graphs, providing personalized feedback, and logging data to Google Sheets. It also supports retrieving and visualizing mood history over time.

The workflow logically splits into two main flows:

- **1.1 Mood Input and Analysis Flow:**  
  Handles incoming mood submissions via webhook, processes mood text with AI to extract emotional metrics (valence and energy), generates a mood graph using Wolfram Alpha, produces personalized feedback, logs all data into Google Sheets, and returns a complete JSON response including a graph URL.

- **1.2 Mood History Retrieval and Visualization Flow:**  
  Triggered by a separate webhook, this flow fetches historical mood entries from Google Sheets, builds a Wolfram Alpha query to plot time-series mood data, generates a graph image, and uploads this visualization to Slack.

Supporting logic blocks include data preparation, Wolfram Alpha query creation, AI feedback generation, and error-safe data handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Mood Input and Analysis Flow

**Overview:**  
This block receives user mood input via a webhook, uses AI to quantify mood into numerical valence and energy scores, prepares data fields, generates a Wolfram Alpha plot query and image, produces personalized feedback, logs all data into Google Sheets, and returns a full JSON API response.

**Nodes Involved:**  
- Webhook: Mood Input  
- AI: Analyze Mood to Numbers  
- Parse AI Response  
- Prepare Data Fields  
- Create Wolfram Query  
- AI: Generate Feedback  
- Generate Mood Graph  
- Log Mood to Google Sheet  
- Return API Response

---

**Node Details:**

- **Webhook: Mood Input**  
  - *Type:* Webhook (POST)  
  - *Role:* Entry point for receiving mood text and userId from external clients.  
  - *Config:* Path `/mood`, HTTP method POST, response mode set to respond with node output.  
  - *Input/Output:* Receives JSON body with fields like `mood` and `userId`. Outputs to "AI: Analyze Mood to Numbers".  
  - *Edge Cases:* Possible malformed JSON or missing fields; no explicit validation shown.  
  - *Version:* 2.1

- **AI: Analyze Mood to Numbers**  
  - *Type:* Langchain Agent (OpenAI GPT-4o-mini)  
  - *Role:* Analyze mood text to extract numeric `valence` (-1 to 1) and `energy` (0 to 1).  
  - *Config:* Instruction prompt forcing JSON-only output with valence and energy. Input text sourced from webhook body or prior node.  
  - *Input/Output:* Input from webhook data or test node; outputs JSON string with valence and energy fields. Connects to "Parse AI Response".  
  - *Edge Cases:* AI response may be malformed JSON, which is handled by the next node’s JSON.parse. AI model credentials required.  
  - *Version:* 3

- **Parse AI Response**  
  - *Type:* Set  
  - *Role:* Parses AI JSON string output into numeric `valence` and `energy` fields for use downstream.  
  - *Config:* Uses expression `JSON.parse($json.output).valence` and `.energy`.  
  - *Input/Output:* Input from AI node JSON string; outputs structured numeric data to "Prepare Data Fields".  
  - *Edge Cases:* If AI output is invalid JSON, parsing will fail, possibly breaking workflow. No fallback shown.  
  - *Version:* 3.4

- **Prepare Data Fields**  
  - *Type:* Set  
  - *Role:* Consolidates all relevant data fields: mood text, userId, valence, energy, timestamp `createdAt`.  
  - *Config:* Fields pulled from either input JSON or webhook, valence and energy from parsed AI data. Timestamp uses current ISO time.  
  - *Input/Output:* Inputs from parsed AI response and webhook; outputs to "Create Wolfram Query".  
  - *Edge Cases:* Missing userId or mood defaults to webhook body values where possible.  
  - *Version:* 3.4

- **Create Wolfram Query**  
  - *Type:* Set  
  - *Role:* Constructs a simple Wolfram Alpha query string for plotting valence over a fixed range.  
  - *Config:* Query string format: `plot y = <valence> * x from x = 0 to 10` using current valence value.  
  - *Input/Output:* Input from prepared data fields; outputs query string to "AI: Generate Feedback".  
  - *Edge Cases:* If valence is missing or invalid, query may be malformed.  
  - *Version:* 3.4

- **AI: Generate Feedback**  
  - *Type:* Langchain Agent (OpenAI GPT-4o-mini)  
  - *Role:* Generates a short positive advice sentence based on mood text and quantified valence/energy.  
  - *Config:* Prompt injects moodText, valence, and energy with instructions for brief English advice.  
  - *Input/Output:* Input from "Create Wolfram Query"; outputs textual feedback to "Generate Mood Graph".  
  - *Edge Cases:* AI may produce unexpected output; relies on valid input params.  
  - *Version:* 3

- **Generate Mood Graph**  
  - *Type:* HTTP Request  
  - *Role:* Sends Wolfram Alpha simple API request to generate a PNG graph based on the query string.  
  - *Config:* URL `https://api.wolframalpha.com/v1/simple`, query parameters include user AppID and query string from prior node. Response expected as file (image).  
  - *Input/Output:* Input from feedback generation node; outputs binary image data to "Log Mood to Google Sheet".  
  - *Credentials:* Requires Wolfram Alpha AppID credential.  
  - *Edge Cases:* API errors, invalid query, or network issues can cause failure. Placeholders must be replaced.  
  - *Version:* 4.3

- **Log Mood to Google Sheet**  
  - *Type:* Google Sheets Append Row  
  - *Role:* Logs all consolidated data fields (userId, moodText, valence, energy, createdAt, Wolfram query string, AI feedback) as a new row.  
  - *Config:* Appends to `Sheet1` of a Google Sheets document with specific column names matching those in data.  
  - *Credentials:* Requires Google Sheets OAuth2 credentials.  
  - *Edge Cases:* Sheet ID must be correct; headers must exist exactly; API quota or auth failures possible.  
  - *Version:* 4.2

- **Return API Response**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends a JSON response to the original webhook caller including mood text, valence, energy, Wolfram query, AI feedback, and URL of the generated graph image.  
  - *Config:* Uses expressions to pull data from prior nodes, including binary data URL of the graph image.  
  - *Input/Output:* Input from Google Sheets logging node; outputs HTTP JSON response.  
  - *Edge Cases:* If any prior node fails or data missing, response may be incomplete.  
  - *Version:* 1.4

---

#### 2.2 Mood History Retrieval and Visualization Flow

**Overview:**  
This block handles requests to retrieve mood history for a user, fetches historical data from Google Sheets, constructs a Wolfram Alpha time-series plot query, generates a graph of mood valence over time, and uploads the graph image to Slack.

**Nodes Involved:**  
- Webhook: History Request  
- Get History from Sheet  
- Create Time Series Query  
- Generate History Graph  
- Upload a file (Slack)

---

**Node Details:**

- **Webhook: History Request**  
  - *Type:* Webhook (POST)  
  - *Role:* Entry point for fetching mood history data.  
  - *Config:* Path `/history`, HTTP method POST, no direct response node (async flow).  
  - *Input/Output:* Receives JSON with `userId` and optional `limit` for number of records. Outputs to "Get History from Sheet".  
  - *Edge Cases:* Missing userId or limit; no validation shown.  
  - *Version:* 2.1

- **Get History from Sheet**  
  - *Type:* Google Sheets Read  
  - *Role:* Reads entire `Sheet1` from Google Sheets document to retrieve all mood records.  
  - *Config:* Document ID placeholder must be replaced.  
  - *Credentials:* Requires Google Sheets OAuth2 credentials.  
  - *Input/Output:* Input from webhook; outputs all rows to "Create Time Series Query".  
  - *Edge Cases:* Large sheets may slow down; missing or incorrect Sheet ID.  
  - *Version:* 4.2

- **Create Time Series Query**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filters, sorts, and limits mood entries by userId and date; formats data points as Wolfram Alpha time series query string.  
  - *Config:*  
    - Filters entries by `userId`.  
    - Sorts descending by `createdAt` to limit to recent entries.  
    - Sorts ascending to prepare time series.  
    - Maps entries to `{date, valence}` format required by Wolfram Alpha.  
    - Constructs query string `time series plot { ... }`.  
  - *Input/Output:* Input is full sheet data; output is the Wolfram query string and filtered history array.  
  - *Edge Cases:* Date parsing errors, missing valence values, malformed sheet data, inconsistent userId fields.  
  - *Version:* 2

- **Generate History Graph**  
  - *Type:* HTTP Request  
  - *Role:* Sends time series query to Wolfram Alpha simple API to generate a PNG image graph of mood over time.  
  - *Config:* Similar to single mood graph generation; uses Wolfram AppID and query string from code node.  
  - *Credentials:* Wolfram Alpha AppID needed.  
  - *Input/Output:* Input from code node; outputs binary image data to Slack upload.  
  - *Edge Cases:* API failures or invalid queries.  
  - *Version:* 4.3

- **Upload a file (Slack)**  
  - *Type:* Slack File Upload  
  - *Role:* Uploads generated mood history graph image to Slack workspace/channel.  
  - *Config:* Uploads file resource, no channel specified in JSON (likely configured in node).  
  - *Credentials:* Requires Slack OAuth2 credentials with file upload scope.  
  - *Input/Output:* Input is binary image from previous node; output is Slack message object.  
  - *Edge Cases:* Slack API rate limits, missing permissions, invalid token.  
  - *Version:* 2.3

---

#### 2.3 Supporting Nodes and Notes

- **Manual Trigger & Set Test Data**  
  - Used for local testing; sets sample mood text and userId to simulate webhook input.

- **OpenAI Chat Model and OpenAI Chat Model1**  
  - Underlying language model nodes supporting the Langchain Agent nodes above.

- **Sticky Notes:**  
  - Provide detailed explanations of workflow purpose, setup instructions, and block descriptions for user clarity.

---

### 3. Summary Table

| Node Name                | Node Type                   | Functional Role                              | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                                                                                             |
|--------------------------|-----------------------------|----------------------------------------------|------------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Webhook: Mood Input      | Webhook                     | Receives mood text and userId input           | -                                  | AI: Analyze Mood to Numbers         | ## 1. How to Start For testing, edit `Set Test Data` and hit 'Execute Workflow'. For real use, send a POST request to the `/mood` webhook. The workflow is smart enough to handle both! |
| AI: Analyze Mood to Numbers | Langchain Agent (OpenAI)   | Analyze mood text into valence and energy     | Webhook: Mood Input, Set Test Data | Parse AI Response                   | ## 2. Analyze & Prep Here's where the magic starts. An AI quantifies your mood into scores. We then parse the response and gather all the data needed, like your `userId` and a timestamp. |
| Parse AI Response         | Set                         | Parses AI JSON string into numeric fields     | AI: Analyze Mood to Numbers         | Prepare Data Fields                | ## 2. Analyze & Prep                                                                                                                     |
| Prepare Data Fields       | Set                         | Consolidates mood data and metadata           | Parse AI Response                   | Create Wolfram Query               | ## 2. Analyze & Prep                                                                                                                     |
| Create Wolfram Query      | Set                         | Builds Wolfram Alpha plot query string        | Prepare Data Fields                 | AI: Generate Feedback              | ## 3. Generate Output A second AI writes you an encouraging message. At the same time, we build a query and send it to Wolfram Alpha, which returns your cool new mood graph.               |
| AI: Generate Feedback     | Langchain Agent (OpenAI)     | Generates personalized advice based on mood   | Create Wolfram Query                | Generate Mood Graph                | ## 3. Generate Output                                                                                                                    |
| Generate Mood Graph       | HTTP Request                | Requests Wolfram Alpha to produce mood graph  | AI: Generate Feedback               | Log Mood to Google Sheet           | ## 3. Generate Output                                                                                                                    |
| Log Mood to Google Sheet  | Google Sheets Append Row    | Logs mood and analysis data to Google Sheets  | Generate Mood Graph                 | Return API Response                | ## 4. Log & Respond Time to save our work. We log the complete record to your Google Sheet. If you used the webhook, we also send a JSON response back with all the analysis and a link to your graph. |
| Return API Response       | Respond to Webhook          | Returns JSON API response to client           | Log Mood to Google Sheet            | -                                | ## 4. Log & Respond                                                                                                                      |
| Webhook: History Request  | Webhook                    | Receives request for mood history retrieval   | -                                  | Get History from Sheet             | ## 5. History Graph Flow This separate flow looks back at your mood history. Triggered by a POST request to `/history`, it fetches your data, builds a query, and posts a time-series graph to Slack. |
| Get History from Sheet    | Google Sheets Read          | Reads all mood history data from Google Sheets| Webhook: History Request            | Create Time Series Query           | ## 5. History Graph Flow                                                                                                                 |
| Create Time Series Query  | Code (JavaScript)           | Filters, sorts, and formats history data for Wolfram Alpha plotting | Get History from Sheet              | Generate History Graph             | ## 5. History Graph Flow                                                                                                                 |
| Generate History Graph    | HTTP Request                | Requests Wolfram Alpha to generate time series graph | Create Time Series Query            | Upload a file (Slack)              | ## 5. History Graph Flow                                                                                                                 |
| Upload a file             | Slack File Upload           | Uploads mood history graph image to Slack     | Generate History Graph              | -                                | ## 5. History Graph Flow                                                                                                                 |
| Manual Trigger            | Manual Trigger              | For local testing, triggers test data flow    | -                                  | Set Test Data                     |                                                                                                                                         |
| Set Test Data             | Set                         | Sets sample mood text and userId for testing  | Manual Trigger                     | AI: Analyze Mood to Numbers         |                                                                                                                                         |
| OpenAI Chat Model         | LM Chat OpenAI              | Underlying model for AI analysis               | AI: Analyze Mood to Numbers         | AI: Analyze Mood to Numbers         |                                                                                                                                         |
| OpenAI Chat Model1        | LM Chat OpenAI              | Underlying model for AI feedback generation    | AI: Generate Feedback              | AI: Generate Feedback              |                                                                                                                                         |
| Mood Graph Studio         | Sticky Note                 | Workflow overview and instructions             | -                                  | -                                | ## Track and Visualize Daily Moods This workflow analyzes your daily mood using OpenAI, visualizes it with Wolfram Alpha, and logs everything to Google Sheets. It also includes a feature to retrieve and graph your mood history. |
| Sticky Note              | Sticky Note                 | Instructions for starting the workflow         | -                                  | -                                | ## 1. How to Start For testing, edit `Set Test Data` and hit 'Execute Workflow'. For real use, send a POST request to the `/mood` webhook. The workflow is smart enough to handle both!        |
| Sticky Note1             | Sticky Note                 | Explanation of analysis and preparation step   | -                                  | -                                | ## 2. Analyze & Prep Here's where the magic starts. An AI quantifies your mood into scores. We then parse the response and gather all the data needed, like your `userId` and a timestamp.        |
| Sticky Note2             | Sticky Note                 | Explanation of output generation step           | -                                  | -                                | ## 3. Generate Output A second AI writes you an encouraging message. At the same time, we build a query and send it to Wolfram Alpha, which returns your cool new mood graph.                   |
| Sticky Note3             | Sticky Note                 | Explanation of logging and response step        | -                                  | -                                | ## 4. Log & Respond Time to save our work. We log the complete record to your Google Sheet. If you used the webhook, we also send a JSON response back with all the analysis and a link to your graph. |
| Sticky Note4             | Sticky Note                 | Explanation of history graph flow                | -                                  | -                                | ## 5. History Graph Flow This separate flow looks back at your mood history. Triggered by a POST request to `/history`, it fetches your data, builds a query, and posts a time-series graph to Slack. |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create the Mood Input Webhook**  
- Add a Webhook node named `Webhook: Mood Input`.  
- Set HTTP Method to POST.  
- Set path to `mood`.  
- Set Response Mode to `Response Node`.  
- This node receives JSON with `mood` (string) and `userId` (string).

**Step 2: Create AI Node to Analyze Mood**  
- Add a Langchain Agent node named `AI: Analyze Mood to Numbers`.  
- Use OpenAI GPT-4o-mini model.  
- Prompt: Instruct to analyze input text and output JSON with numeric valence (-1 to 1) and energy (0 to 1).  
- Input text: Use expression `{{ $json.mood || $('Webhook: Mood Input').first().json.body.mood }}` to get mood text.  
- Connect `Webhook: Mood Input` → `AI: Analyze Mood to Numbers`.  
- Configure OpenAI credentials.

**Step 3: Parse AI JSON Response**  
- Add a Set node named `Parse AI Response`.  
- Assign `valence = JSON.parse($json.output).valence` (number).  
- Assign `energy = JSON.parse($json.output).energy` (number).  
- Connect `AI: Analyze Mood to Numbers` → `Parse AI Response`.

**Step 4: Prepare Data Fields**  
- Add a Set node named `Prepare Data Fields`.  
- Assign:  
  - `moodText` = `{{ $json.mood || $('Webhook: Mood Input').first().json.body.mood }}`  
  - `userId` = `{{ $json.userId || $('Webhook: Mood Input').first().json.body.userId }}`  
  - `valence` = `{{ $json.valence }}`  
  - `energy` = `{{ $json.energy }}`  
  - `createdAt` = `{{ $now.toISO() }}` (current timestamp ISO format)  
- Connect `Parse AI Response` → `Prepare Data Fields`.

**Step 5: Create Wolfram Alpha Query**  
- Add a Set node named `Create Wolfram Query`.  
- Assign `wolframQuery` string: `plot y = {{ $json.valence }} * x from x = 0 to 10`.  
- Include all other fields (copy from input).  
- Connect `Prepare Data Fields` → `Create Wolfram Query`.

**Step 6: Generate AI Feedback**  
- Add a Langchain Agent node named `AI: Generate Feedback`.  
- Use OpenAI GPT-4o-mini.  
- Prompt: Provide brief positive advice based on moodText, valence, and energy from `Prepare Data Fields`.  
- Connect `Create Wolfram Query` → `AI: Generate Feedback`.  
- Configure OpenAI credentials.

**Step 7: Generate Mood Graph via Wolfram Alpha**  
- Add an HTTP Request node named `Generate Mood Graph`.  
- Set URL to `https://api.wolframalpha.com/v1/simple`.  
- Set method to GET (default).  
- Add query parameters:  
  - `appid` = Your Wolfram Alpha AppID (replace placeholder).  
  - `i` = `{{ $('Create Wolfram Query').first().json.wolframQuery }}`.  
- Set Response Format to `File` (binary).  
- Connect `AI: Generate Feedback` → `Generate Mood Graph`.  
- Configure Wolfram Alpha credentials.

**Step 8: Log Data to Google Sheets**  
- Add a Google Sheets node named `Log Mood to Google Sheet`.  
- Operation: Append.  
- Document ID: Your Google Sheet ID (replace placeholder).  
- Sheet Name: `Sheet1`.  
- Map columns:  
  - `userId` = `{{ $('Prepare Data Fields').first().json.userId }}`  
  - `moodText` = `{{ $('Prepare Data Fields').first().json.moodText }}`  
  - `valence` = `{{ $('Prepare Data Fields').first().json.valence }}`  
  - `energy` = `{{ $('Prepare Data Fields').first().json.energy }}`  
  - `createdAt` = `{{ $('Prepare Data Fields').first().json.createdAt }}`  
  - `wolframQuery` = `{{ $('Create Wolfram Query').first().json.wolframQuery }}`  
  - `feedback` = `{{ $('AI: Generate Feedback').first().json.response }}`  
- Connect `Generate Mood Graph` → `Log Mood to Google Sheet`.  
- Configure Google Sheets OAuth2 credentials.

**Step 9: Return API Response**  
- Add a Respond to Webhook node named `Return API Response`.  
- Set response type to JSON.  
- Response body fields:  
  - `mood` = `{{ $('Prepare Data Fields').first().json.moodText }}`  
  - `valence` = `{{ $('Prepare Data Fields').first().json.valence }}`  
  - `energy` = `{{ $('Prepare Data Fields').first().json.energy }}`  
  - `wolframQuery` = `{{ $('Create Wolfram Query').first().json.wolframQuery }}`  
  - `feedback` = `{{ $('AI: Generate Feedback').first().json.response }}`  
  - `graph` = `{{ $('Generate Mood Graph').first().binary.data.dataUrl }}` (base64 URL)  
- Connect `Log Mood to Google Sheet` → `Return API Response`.

---

**Step 10: Create the History Retrieval Webhook**  
- Add a Webhook node named `Webhook: History Request`.  
- Set HTTP Method to POST.  
- Set path to `history`.  
- No response node configured (async flow).  

**Step 11: Get History from Google Sheets**  
- Add a Google Sheets node named `Get History from Sheet`.  
- Operation: Read all rows from `Sheet1`.  
- Document ID: Same Google Sheet ID as above.  
- Connect `Webhook: History Request` → `Get History from Sheet`.  
- Configure Google Sheets credentials.

**Step 12: Filter and Format Time Series Query**  
- Add a Code node named `Create Time Series Query`.  
- Paste the provided JavaScript code filtering by userId, sorting, limiting, and formatting Wolfram Alpha time series query.  
- Connect `Get History from Sheet` → `Create Time Series Query`.

**Step 13: Generate History Graph**  
- Add an HTTP Request node named `Generate History Graph`.  
- Similar config as mood graph generation node:  
  - URL: `https://api.wolframalpha.com/v1/simple`  
  - Query parameters: `appid` (your Wolfram Alpha AppID), `i` (time series query from previous node).  
  - Response format: File (image).  
- Connect `Create Time Series Query` → `Generate History Graph`.  
- Configure Wolfram Alpha credentials.

**Step 14: Upload Graph to Slack**  
- Add Slack node named `Upload a file`.  
- Resource: File.  
- Configure credentials with Slack OAuth2, enabling file upload scope.  
- Connect `Generate History Graph` → `Upload a file`.  
- Configure Slack channel as needed.

---

**Step 15: (Optional) Add Manual Trigger and Test Data**  
- Add Manual Trigger node named `Manual Trigger`.  
- Add Set node named `Set Test Data` with:  
  - `mood` = `"What a great day to build a workflow!"`  
  - `userId` = `"n8n-user-123"`  
- Connect `Manual Trigger` → `Set Test Data` → `AI: Analyze Mood to Numbers`.  
- Useful for local testing without external POST requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                      | Context or Link                                                                                                                                                                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow analyzes your daily mood using OpenAI, visualizes it with Wolfram Alpha, and logs everything to Google Sheets. It also includes a feature to retrieve and graph your mood history. Setup requires configuring OpenAI, Google Sheets, Slack, and Wolfram Alpha credentials. | Workflow Overview (Sticky Note "Mood Graph Studio")                                                                                                                                             |
| The first row of your Google Sheet **must** have the headers: `userId`, `moodText`, `valence`, `energy`, `createdAt`, `wolframQuery`, `feedback`.                                                                                                               | Setup instructions (Sticky Note "Mood Graph Studio")                                                                                                                                             |
| Wolfram Alpha requires a free AppID from their Developer Portal. Paste this AppID in the HTTP Request nodes for graph generation.                                                                                                                                  | Setup instructions (Sticky Note "Mood Graph Studio")                                                                                                                                             |
| For testing, use the "Set Test Data" node and then execute the workflow manually. For production, send HTTP POST requests to `/mood` and `/history` webhook URLs.                                                                                                  | Usage instructions (Sticky Note "Sticky Note")                                                                                                                                                   |
| The AI prompts enforce strict JSON output for valence and energy, and concise positive advice, avoiding explanations or markdown formatting.                                                                                                                      | AI prompt design (nodes `AI: Analyze Mood to Numbers` and `AI: Generate Feedback`)                                                                                                               |
| Edge cases include possible malformed AI responses, missing required fields in input, API failures (Wolfram Alpha, Google Sheets, Slack), and date parsing issues in history queries. Proper error handling is recommended for production.                          | General remarks based on node analysis                                                                                                                                                          |
| Slack file upload requires appropriate OAuth2 scope (`files:write`). Set the target Slack channel in the Slack upload node configuration.                                                                                                                        | Slack integration requirements                                                                                                                         |

---

This completes the detailed technical documentation and reference for the "Track and Visualize Daily Moods with GPT-4o, Wolfram Alpha, Sheets, and Slack" n8n workflow. It enables advanced users and automation agents to fully understand, reproduce, and safely extend the workflow.