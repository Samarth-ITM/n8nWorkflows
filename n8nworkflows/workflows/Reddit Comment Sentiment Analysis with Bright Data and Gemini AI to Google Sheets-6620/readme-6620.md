Reddit Comment Sentiment Analysis with Bright Data and Gemini AI to Google Sheets

https://n8nworkflows.xyz/workflows/reddit-comment-sentiment-analysis-with-bright-data-and-gemini-ai-to-google-sheets-6620


# Reddit Comment Sentiment Analysis with Bright Data and Gemini AI to Google Sheets

### 1. Workflow Overview

This workflow automates the sentiment analysis of Reddit post comments by leveraging Bright Data for web scraping, Google Gemini AI models for natural language processing, and Google Sheets for data storage. It is designed to help users extract comments from a specific Reddit post, analyze their sentiment (positive, negative, or neutral) with contextual reasoning, and save the results in a structured and accessible spreadsheet format.

The workflow is logically divided into four main blocks:

- **1.1 Trigger & Input Setup:** Manual initiation and Reddit post URL configuration.
- **1.2 Snapshot Creation & Waiting:** Initiation of Bright Data web scraping job and wait for data readiness.
- **1.3 Download & Limit Data:** Download scraped comments and limit the number for processing.
- **1.4 Sentiment Analysis & Storage:** AI-based sentiment classification with Google Gemini and saving results to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Setup

**Overview:**  
This initial block sets up the workflow trigger and defines the Reddit post URL to analyze. The workflow starts manually, giving control to the user for testing or targeted runs.

**Nodes Involved:**  
- Trigger: Manual Start  
- Set Reddit Post URL

**Node Details:**

- **Trigger: Manual Start**  
  - Type: Manual Trigger node  
  - Role: Initiates the workflow on manual user action in the n8n editor.  
  - Configuration: Default manual trigger with no additional parameters.  
  - Inputs: None  
  - Outputs: Connected to `Set Reddit Post URL`  
  - Edge Cases: None typical; user must start manually.

- **Set Reddit Post URL**  
  - Type: Set node  
  - Role: Stores the target Reddit post URL as a workflow variable for downstream use.  
  - Configuration: A single string parameter "post URL" set manually to a sample Reddit post URL.  
  - Inputs: From `Trigger: Manual Start`  
  - Outputs: Connected to `Bright Data: Get comments`  
  - Edge Cases: Invalid or malformed URLs could cause scraping failure downstream.

---

#### 1.2 Snapshot Creation & Waiting

**Overview:**  
This block uses Bright Data’s web scraper API to create a snapshot job for the given Reddit post URL and waits five minutes to allow the scraping job to complete.

**Nodes Involved:**  
- Bright Data: Get comments  
- Wait for Snapshot Processing (5 min)

**Node Details:**

- **Bright Data: Get comments**  
  - Type: Bright Data web scraper node  
  - Role: Sends the Reddit URL to Bright Data to start a new snapshot scraping job capturing structured comment data.  
  - Configuration:  
    - URLs: Dynamic expression referencing "post URL" from previous Set node.  
    - Dataset ID: Specific Bright Data dataset for Reddit comments (`gd_lvzdpsdlw09j6t702`).  
    - Credentials: Bright Data API account configured for access.  
  - Inputs: From `Set Reddit Post URL`  
  - Outputs: Connected to `Wait for Snapshot Processing (5 min)`  
  - Edge Cases: API authentication errors, invalid dataset ID, URL not reachable, or scraper limits.

- **Wait for Snapshot Processing (5 min)**  
  - Type: Wait node  
  - Role: Pauses workflow execution for 5 minutes to allow Bright Data to complete scraping.  
  - Configuration:  
    - Duration: 5 minutes  
    - Webhook ID: Used internally by n8n for delayed continuation.  
  - Inputs: From `Bright Data: Get comments`  
  - Outputs: Connected to `Bright Data: Download Comments Snapshot`  
  - Edge Cases: Timing too short or too long depending on data size; workflow stuck if webhook fails.

---

#### 1.3 Download & Limit Data

**Overview:**  
This block downloads the scraped comments from Bright Data using the snapshot ID, then limits the number of comments to five for manageable processing during testing or demonstration.

**Nodes Involved:**  
- Bright Data: Download Comments Snapshot  
- Limit to 5 Comments

**Node Details:**

- **Bright Data: Download Comments Snapshot**  
  - Type: Bright Data web scraper node  
  - Role: Downloads the scraped comment data using the snapshot ID obtained previously.  
  - Configuration:  
    - Operation: Download snapshot  
    - Snapshot ID: Dynamically obtained from previous node's JSON (`$json.snapshot_id`)  
    - Credentials: Bright Data account  
  - Inputs: From `Wait for Snapshot Processing (5 min)`  
  - Outputs: Connected to `Limit to 5 Comments`  
  - Edge Cases: Snapshot ID missing or invalid, network issues, API limits.

- **Limit to 5 Comments**  
  - Type: Limit node  
  - Role: Restricts the dataset to the first 5 comments to avoid long processing times or overload.  
  - Configuration:  
    - Max Items: 5  
  - Inputs: From `Bright Data: Download Comments Snapshot`  
  - Outputs: Connected to `AI Sentiment Classifier`  
  - Edge Cases: If fewer than 5 comments available, processes all; no failure expected.

---

#### 1.4 Sentiment Analysis & Storage

**Overview:**  
This block uses AI to classify each comment’s sentiment, applies output parsing and correction, then writes the structured sentiment data including reasoning back to Google Sheets.

**Nodes Involved:**  
- AI Sentiment Classifier  
- Google Gemini Chat Model  
- Auto-fixing Output Parser  
- Structured Output Parser  
- Save Sentiment to Google Sheets

**Node Details:**

- **AI Sentiment Classifier**  
  - Type: Langchain Agent node  
  - Role: Processes each comment to classify sentiment as positive, negative, or neutral with reasoning.  
  - Configuration:  
    - Prompt: Custom text instructing AI to evaluate sentiment based on comment text and upvotes.  
    - Has Output Parser: Enabled to structure AI response.  
    - AI Model Input: Connected to Google Gemini Chat Model as language model and to output parsers.  
  - Inputs: From `Limit to 5 Comments` (main), Google Gemini Chat Model (ai_languageModel), Auto-fixing Output Parser (ai_outputParser)  
  - Outputs: Connected to `Save Sentiment to Google Sheets`  
  - Edge Cases: AI model timeout, malformed output, bad prompt input, rate limits.

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini LM Chat node  
  - Role: Provides the AI model interface to Google Gemini (PaLM API) for natural language understanding.  
  - Configuration:  
    - Credentials: Google Palm API account with valid access.  
    - Options: Default settings.  
  - Inputs: Connected as AI language model input to `AI Sentiment Classifier`  
  - Outputs: Connected to `AI Sentiment Classifier`  
  - Edge Cases: API quota limits, authentication errors, network failures.

- **Auto-fixing Output Parser**  
  - Type: Langchain Auto-fixing Output Parser node  
  - Role: Automatically corrects AI output if it is malformed or incomplete to ensure valid downstream parsing.  
  - Configuration: Default options.  
  - Inputs: From `Google Gemini Chat Model1` (a secondary Gemini model node)  
  - Outputs: Connected to `AI Sentiment Classifier` as output parser input  
  - Edge Cases: Parsing failure if output is severely corrupted or incompatible.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser node  
  - Role: Converts the AI output into a structured JSON format containing keys: comment, sentiment, and reason.  
  - Configuration:  
    - JSON Schema Example specifies expected fields and example values for parsing.  
  - Inputs: From `Google Gemini Chat Model1` (ai_outputParser)  
  - Outputs: Connected to `Auto-fixing Output Parser`  
  - Edge Cases: Schema mismatch or invalid AI output structure.

- **Save Sentiment to Google Sheets**  
  - Type: Google Sheets node  
  - Role: Appends the structured sentiment data to a predefined Google Sheets document for persistent storage and analysis.  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet Name: Pre-configured with Google Sheets OAuth2 credentials  
    - Columns mapped: Comment, Sentiment, Reason (from AI output)  
  - Inputs: From `AI Sentiment Classifier`  
  - Outputs: None (end node)  
  - Edge Cases: Authentication failure, sheet access permission errors, quota exceeded.

---

### 3. Summary Table

| Node Name                       | Node Type                                      | Functional Role                          | Input Node(s)                       | Output Node(s)                        | Sticky Note                                                                                                                          |
|--------------------------------|------------------------------------------------|----------------------------------------|-----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Trigger: Manual Start           | manualTrigger                                  | Workflow manual initiation              | None                              | Set Reddit Post URL                  | ### 🔹 **Section 1: Trigger & Input Setup**<br>Manual trigger for controlled workflow start.                                           |
| Set Reddit Post URL             | set                                            | Sets target Reddit post URL             | Trigger: Manual Start             | Bright Data: Get comments            | ### 🔹 **Section 1: Trigger & Input Setup**<br>Manually enter Reddit post URL.                                                         |
| Bright Data: Get comments       | brightData (webScraper)                        | Starts Bright Data scraping snapshot    | Set Reddit Post URL               | Wait for Snapshot Processing (5 min) | ### 🔹 **Section 2: Snapshot Creation & Waiting**<br>Starts snapshot job for scraping comments.                                         |
| Wait for Snapshot Processing (5 min) | wait                                      | Pauses workflow for data readiness      | Bright Data: Get comments         | Bright Data: Download Comments Snapshot | ### 🔹 **Section 2: Snapshot Creation & Waiting**<br>Wait 5 minutes for scraping to complete.                                           |
| Bright Data: Download Comments Snapshot | brightData (webScraper)                    | Downloads scraped comments snapshot     | Wait for Snapshot Processing (5 min) | Limit to 5 Comments               | ### 🔹 **Section 3: Download & Limit Data**<br>Downloads scraped comments data using snapshot ID.                                      |
| Limit to 5 Comments             | limit                                          | Limits number of comments processed     | Bright Data: Download Comments Snapshot | AI Sentiment Classifier         | ### 🔹 **Section 3: Download & Limit Data**<br>Limits comments to 5 for manageable processing.                                          |
| AI Sentiment Classifier         | langchain.agent                                | Classifies comment sentiment with AI    | Limit to 5 Comments, Google Gemini Chat Model, Auto-fixing Output Parser | Save Sentiment to Google Sheets | ### 🔹 **Section 4: Sentiment Analysis & Storage**<br>AI analyzes sentiment and outputs structured data.                              |
| Google Gemini Chat Model        | langchain.lmChatGoogleGemini                   | Provides Google Gemini AI model          | AI Sentiment Classifier (ai_languageModel) | AI Sentiment Classifier (ai_languageModel) | ### 🔹 **Section 4: Sentiment Analysis & Storage**<br>Primary AI model for understanding and sentiment analysis.                      |
| Google Gemini Chat Model1       | langchain.lmChatGoogleGemini                   | Secondary Google Gemini AI model         | Structured Output Parser (ai_outputParser) | Auto-fixing Output Parser (ai_languageModel) | Part of output parsing chain in sentiment analysis block.                                                                              |
| Structured Output Parser        | langchain.outputParserStructured               | Converts AI output to structured JSON   | Google Gemini Chat Model1 (ai_outputParser) | Auto-fixing Output Parser (ai_outputParser) | Parses AI output into comment, sentiment, and reason fields.                                                                           |
| Auto-fixing Output Parser       | langchain.outputParserAutofixing                | Fixes malformed AI output                | Google Gemini Chat Model1 (ai_languageModel) | AI Sentiment Classifier (ai_outputParser) | Auto-corrects AI output to ensure valid formatting downstream.                                                                         |
| Save Sentiment to Google Sheets | googleSheets                                  | Saves sentiment data to Google Sheets   | AI Sentiment Classifier           | None                                | ### 🔹 **Section 4: Sentiment Analysis & Storage**<br>Saves results for analysis, reports, and tracking.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**
   - Name: `Trigger: Manual Start`  
   - Type: Manual trigger (default)  
   - Purpose: Start workflow manually for testing or controlled runs.

2. **Create a Set Node**
   - Name: `Set Reddit Post URL`  
   - Type: Set  
   - Purpose: Store the Reddit post URL to analyze.  
   - Configuration: Add a string field named `post URL` and set its value to your target Reddit post URL, e.g., `https://www.reddit.com/r/iphone/comments/1kl0tb5/new_ios_185_update/...`  
   - Connect: Output of `Trigger: Manual Start` → Input of this node.

3. **Add Bright Data Web Scraper Node (Create Snapshot)**
   - Name: `Bright Data: Get comments`  
   - Type: Bright Data (webScraper)  
   - Purpose: Request new scraping snapshot for the Reddit URL.  
   - Parameters:  
     - URLs: Use expression `{{ $json["post URL"] }}` to dynamically pass the URL.  
     - Dataset ID: Use Bright Data dataset ID for Reddit comments, e.g., `gd_lvzdpsdlw09j6t702`.  
   - Credentials: Connect your Bright Data API credential.  
   - Connect: Output of `Set Reddit Post URL` → Input of this node.

4. **Add Wait Node**
   - Name: `Wait for Snapshot Processing (5 min)`  
   - Type: Wait  
   - Purpose: Pause workflow to allow scraping to finish.  
   - Parameters: Set duration to 5 minutes.  
   - Connect: Output of `Bright Data: Get comments` → Input of this node.

5. **Add Bright Data Web Scraper Node (Download Snapshot)**
   - Name: `Bright Data: Download Comments Snapshot`  
   - Type: Bright Data (webScraper)  
   - Purpose: Download scraped comments using snapshot ID.  
   - Parameters:  
     - Operation: Download snapshot  
     - Snapshot ID: Use expression `{{ $json.snapshot_id }}` obtained from the previous node.  
   - Credentials: Same Bright Data API credential.  
   - Connect: Output of `Wait for Snapshot Processing (5 min)` → Input of this node.

6. **Add Limit Node**
   - Name: `Limit to 5 Comments`  
   - Type: Limit  
   - Purpose: Restrict processing to first 5 comments.  
   - Parameters: Max items = 5.  
   - Connect: Output of `Bright Data: Download Comments Snapshot` → Input of this node.

7. **Add Google Gemini Chat Model Node**
   - Name: `Google Gemini Chat Model`  
   - Type: Langchain Google Gemini LM Chat  
   - Purpose: Use Google Gemini AI for sentiment analysis.  
   - Credentials: Add Google PaLM API credentials.  
   - Configuration: Use default options.  
   - Connect: Will connect as AI language model input to next node.

8. **Add Structured Output Parser Node**
   - Name: `Structured Output Parser`  
   - Type: Langchain Structured Output Parser  
   - Purpose: Parse AI response into structured JSON.  
   - Configuration:  
     - Set JSON schema example with keys: comment, sentiment, reason.  
   - Connect: Output parser input from Gemini Chat Model 1 (created below).

9. **Add Google Gemini Chat Model1 Node**
   - Name: `Google Gemini Chat Model1`  
   - Type: Langchain Google Gemini LM Chat  
   - Purpose: Secondary AI model node for output parsing chain.  
   - Credentials: Use same Google PaLM API credentials.  
   - Connect: AI output parser input connected to `Structured Output Parser`, language model output connected to `Auto-fixing Output Parser`.

10. **Add Auto-fixing Output Parser Node**
    - Name: `Auto-fixing Output Parser`  
    - Type: Langchain Auto-fixing Output Parser  
    - Purpose: Fix any malformed AI output to ensure downstream compatibility.  
    - Connect: Output parser chain between `Structured Output Parser` and `AI Sentiment Classifier`.

11. **Add AI Sentiment Classifier Node**
    - Name: `AI Sentiment Classifier`  
    - Type: Langchain Agent  
    - Purpose: Analyze each comment’s sentiment and provide reasoned output.  
    - Configuration:  
      - Prompt (with expressions):  
        ```
        Based on the Reddit post's comment below, decide whether the sentiment is positive, negative or neutral.

        comment: {{ $json.comment }}
        Number of upvotes: {{ $json.num_upvotes }}
        ```  
      - Enable output parser.  
    - Inputs:  
      - Main input from `Limit to 5 Comments`  
      - AI language model input from `Google Gemini Chat Model`  
      - AI output parser input from `Auto-fixing Output Parser`  
    - Outputs: Connect to next node.

12. **Add Google Sheets Node**
    - Name: `Save Sentiment to Google Sheets`  
    - Type: Google Sheets  
    - Purpose: Append sentiment data to Google Sheets for record keeping.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Your Google Sheets document ID (e.g., `1WPOfGgyqbcu8RFdA_LyMvCRxqrwr-nUL5JT656ASwOA`)  
      - Sheet Name: Specify sheet (e.g., gid=0)  
      - Columns mapped:  
        - Comment: `={{ $json.output.comment }}`  
        - Sentiment: `={{ $json.output.sentiment }}`  
        - Reason: `={{ $json.output.reason }}`  
    - Credentials: Google Sheets OAuth2 credentials.  
    - Connect: Input from `AI Sentiment Classifier`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow assistance and contact provided by Yaron Been with links to YouTube and LinkedIn channels for support and tutorials.      | https://www.youtube.com/@YaronBeen/videos<br>https://www.linkedin.com/in/yaronbeen/                     |
| Bright Data affiliate link to support free content creation: https://get.brightdata.com/1tndi4600b25                                | Bright Data web scraping service                                                                         |
| Copy of the Google Sheets template used for sentiment results: [Google Sheets Template](https://docs.google.com/spreadsheets/d/1ycEjFdFK9MXQif2O_jh0Fhjlw5R9zwFUZ4qZdx8Duss/edit?usp=sharing) | Template for comment sentiment analysis results                                                        |
| Workflow emphasizes no-code automation, real AI integration, and extensibility (e.g., scheduled runs, notifications, expanded data) | Useful for beginners and advanced users looking to automate social sentiment insights                    |

---

**Disclaimer:** The provided text and workflow originate exclusively from an automated n8n workflow. All data and processes comply with current content policies and contain no illegal or protected content. All data handled are public and legal.