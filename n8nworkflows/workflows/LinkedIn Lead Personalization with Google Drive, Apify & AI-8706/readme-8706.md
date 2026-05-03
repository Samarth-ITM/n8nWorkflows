LinkedIn Lead Personalization with Google Drive, Apify & AI

https://n8nworkflows.xyz/workflows/linkedin-lead-personalization-with-google-drive--apify---ai-8706


# LinkedIn Lead Personalization with Google Drive, Apify & AI

### 1. Workflow Overview

This workflow automates personalized lead enrichment and outreach message generation for LinkedIn leads by integrating Google Drive, Apify LinkedIn scraping services, and AI language models (Anthropic Claude and OpenAI GPT). It is designed to process new lead files added to a specific Google Drive folder, extract LinkedIn profile data, analyze recent LinkedIn posts, generate personalized outreach sentences, and update a Google Sheet with this personalization data. The workflow also sends a summary notification to a Telegram bot when enrichment completes.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception and File Processing**: Triggered by new Google Drive files, downloads and parses lead data from spreadsheets.
- **1.2 Data Preparation and Looping**: Processes each lead’s LinkedIn URL, iterating through items.
- **1.3 LinkedIn Profile Post Scraping and Analysis**: Scrapes the most recent LinkedIn post and uses AI to determine if the post is recent (from 2025).
- **1.4 AI-Based Personalization Generation**: Depending on recency of posts, generates personalized outreach sentences either referencing the post or fallback messaging.
- **1.5 Updating Google Sheet with Personalization**: Writes back personalized messages to the Google Sheet.
- **1.6 Notification and Summary Preparation**: Prepares and sends a Telegram message summarizing the enrichment results.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and File Processing

**Overview:**  
This block watches a specific Google Drive folder for new files (expected to be lead lists in spreadsheet format). Once a file is detected, it downloads and parses it into usable JSON for further processing.

**Nodes Involved:**  
- Google Drive Trigger1  
- Google Drive_DownLoad1  
- Google Sheet Parse  
- Code1

**Node Details:**

- **Google Drive Trigger1**  
  - Type: Google Drive Trigger (event-based)  
  - Configuration: Watches folder ID `1-b1yzCQA8RzQmG4vOaGKzYY8iZQxKpvf` for new files every minute  
  - Inputs: None (trigger)  
  - Outputs: Emits file metadata when a new file is created  
  - Edge Cases: Failure if folder ID is invalid or credentials expire; missed events if polling interval is too long

- **Google Drive_DownLoad1**  
  - Type: Google Drive node (download operation)  
  - Configuration: Downloads the file using the ID from the trigger node  
  - Inputs: File ID from Google Drive Trigger1  
  - Outputs: Binary file data (spreadsheet)  
  - Edge Cases: Download failure if file deleted/moved; permission errors

- **Google Sheet Parse**  
  - Type: Spreadsheet File (parse)  
  - Configuration: Parses the downloaded spreadsheet with header row enabled  
  - Inputs: Binary spreadsheet file from Google Drive_DownLoad1  
  - Outputs: JSON array of rows representing leads  
  - Edge Cases: Parsing errors if file format invalid or corrupted

- **Code1**  
  - Type: Code (JavaScript)  
  - Configuration: Transforms parsed spreadsheet rows into structured lead objects with fields for first name, last name, company, LinkedIn URL, email, website, etc. Intended to normalize input data.  
  - Inputs: JSON rows from Google Sheet Parse  
  - Outputs: Reformatted JSON lead items  
  - Edge Cases: Missing expected columns in input data; empty or malformed rows; variable naming typos (note: in the code snippet, `limitedItems` is returned but not defined—potential bug here)

---

#### 2.2 Data Preparation and Looping

**Overview:**  
Processes each lead individually by splitting the dataset into batches and looping over each item for enrichment.

**Nodes Involved:**  
- Loop Over Items1  
- End of Loops

**Node Details:**

- **Loop Over Items1**  
  - Type: Split In Batches  
  - Configuration: Splits lead items one by one (default batch size)  
  - Inputs: Reformatted lead JSON from Code1  
  - Outputs: Emits one lead item per execution cycle  
  - Edge Cases: Large datasets may slow down processing; batch size can be tuned

- **End of Loops**  
  - Type: Code (JavaScript)  
  - Configuration: Aggregates all processed lead items and creates a formatted summary message for Telegram including Google Sheet name, URL, and enrichment results (score, status if available)  
  - Inputs: All items from downstream processing (personalization results)  
  - Outputs: JSON object with `message` field for Telegram  
  - Edge Cases: Missing fields in items; errors in accessing upstream node data; logging helps debug

---

#### 2.3 LinkedIn Profile Post Scraping and Analysis

**Overview:**  
Scrapes the most recent LinkedIn post of each lead’s profile using Apify and uses OpenAI to determine if the post is from the current year (2025). This affects whether personalized messaging can include recent activity references.

**Nodes Involved:**  
- Scrape LinkedIn Profile Post  
- Date & Time  
- OpenAI  
- If the LN post is in the last 30 days

**Node Details:**

- **Scrape LinkedIn Profile Post**  
  - Type: HTTP Request  
  - Configuration: Calls Apify LinkedIn Profile Posts API with lead's LinkedIn username to fetch the most recent post (limit=1)  
  - Headers include Bearer token for authentication  
  - Inputs: LinkedIn URL username from lead item  
  - Outputs: JSON with post text and posted_at date  
  - Edge Cases: Invalid or private profiles; API quota limits; invalid token; network timeouts

- **Date & Time**  
  - Type: DateTime  
  - Configuration: Provides current date/time for comparison in downstream nodes  
  - Inputs: None (triggered per item)  
  - Outputs: Current timestamp in workflow context  
  - Edge Cases: None significant

- **OpenAI**  
  - Type: OpenAI (GPT-4.1-mini)  
  - Configuration: Checks if the LinkedIn post date is from 2025 using a tightly bound system prompt and user prompt with provided post data and current date  
  - Outputs boolean string `"true"` or `"false"`  
  - Edge Cases: API errors, rate limits, unexpected output format; expression errors in prompt construction

- **If the LN post is in the last 30 days**  
  - Type: If  
  - Configuration: Checks if OpenAI’s output equals `"true"`  
  - On true: proceeds to personalization with LinkedIn post data  
  - On false: proceeds to alternate personalization  
  - Edge Cases: Unexpected or missing input may cause branch failures

---

#### 2.4 AI-Based Personalization Generation

**Overview:**  
Generates personalized outreach sentences differently based on whether a recent LinkedIn post exists. Uses Anthropic Claude and Langchain Agent nodes to craft natural, engaging messages referencing the lead’s LinkedIn activity or fallback information.

**Nodes Involved:**  
- AI Agent1  
- Anthropic Chat Model3  
- Colect LinkedIn Data  
- AI Agent_Personalization_no LN post  
- Anthropic Chat Model1  
- Personalization (Set node)

**Node Details:**

- **AI Agent1**  
  - Type: Langchain Agent  
  - Configuration: Uses recent LinkedIn post text as input prompt to generate a single personalized sentence for outreach, with detailed system instructions to ensure natural, non-intrusive messaging  
  - Inputs: LinkedIn post text from Scrape LinkedIn Profile Post  
  - Outputs: Personalized sentence string in `output` field  
  - Edge Cases: Model timeout, incomplete output, or incoherent responses

- **Anthropic Chat Model3**  
  - Type: Anthropic LM Chat (Claude Sonnet 4)  
  - Configuration: Language model supporting AI Agent1 as backend  
  - Credentials: Anthropic API key  
  - Edge Cases: API key invalidation, quota exhaustion

- **Colect LinkedIn Data**  
  - Type: HTTP Request  
  - Configuration: Calls Apify LinkedIn Profile Scraper API to collect detailed profile data for fallback personalization when no recent post exists  
  - Inputs: LinkedIn URL from lead  
  - Outputs: JSON enriched profile data  
  - Edge Cases: API token issues, rate limits, private profiles

- **AI Agent_Personalization_no LN post**  
  - Type: Langchain Agent  
  - Configuration: Generates one personalized cold email opening sentence based on lead’s profile fields (name, headline, about, company) focusing on lead generation pain points and product benefits without referencing LinkedIn posts  
  - Inputs: Profile data from Colect LinkedIn Data  
  - Outputs: Single personalized sentence  
  - Edge Cases: Model response quality, unexpected input formats

- **Anthropic Chat Model1**  
  - Type: Anthropic LM Chat  
  - Configuration: Supports AI Agent_Personalization_no LN post node with Claude Sonnet 4 model  
  - Credentials: Anthropic API key  
  - Edge Cases: Same as above

- **Personalization (Set)**  
  - Type: Set  
  - Configuration: Assigns the generated personalized sentence to `personalization` field; also captures Google Sheet ID and file name for updating  
  - Inputs: Output from AI Agent1 or AI Agent_Personalization_no LN post  
  - Outputs: JSON with personalization and metadata  
  - Edge Cases: Expression errors if upstream data missing

---

#### 2.5 Updating Google Sheet with Personalization

**Overview:**  
Updates the original Google Sheet with the generated personalized messages, matching lead rows by email.

**Nodes Involved:**  
- Google Sheets Update with Personalization

**Node Details:**

- **Google Sheets Update with Personalization**  
  - Type: Google Sheets (update operation)  
  - Configuration: Updates rows matching lead email with new personalization text  
  - Inputs: personalization string, sheetId, and email from Personalization node  
  - Outputs: Success or error response from Google Sheets API  
  - Edge Cases: Mismatched emails; permission errors; sheet locked or deleted; API quota limits

---

#### 2.6 Notification and Summary Preparation

**Overview:**  
After processing all leads, aggregates results and sends a formatted Telegram message reporting enrichment completion and dataset summary.

**Nodes Involved:**  
- End of Loops  
- Lead Enrichment is ready  
- Sticky Note nodes (for documentation)

**Node Details:**

- **End of Loops**  
  - (See details above in 2.2)  

- **Lead Enrichment is ready**  
  - Type: Telegram  
  - Configuration: Sends the aggregated message from End of Loops to a configured Telegram Bot chat ID  
  - Inputs: Message JSON from End of Loops  
  - Outputs: Telegram API response  
  - Edge Cases: Invalid chat ID, bot token revoked, network issues

- **Sticky Note nodes**  
  - Type: Sticky Note  
  - Purpose: Provide visual documentation and explanations within the workflow editor; no functional role  

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                          | Input Node(s)                    | Output Node(s)                    | Sticky Note                                                                                  |
|-----------------------------------|----------------------------------|----------------------------------------|---------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Google Drive Trigger1              | Google Drive Trigger             | Detect new lead files in Google Drive  | None                            | Google Drive_DownLoad1            |                                                                                              |
| Google Drive_DownLoad1             | Google Drive                    | Download new lead file                  | Google Drive Trigger1           | Google Sheet Parse                |                                                                                              |
| Google Sheet Parse                | Spreadsheet File                | Parse spreadsheet to JSON               | Google Drive_DownLoad1          | Code1                           |                                                                                              |
| Code1                            | Code (JavaScript)               | Normalize lead data fields              | Google Sheet Parse             | Loop Over Items1                 |                                                                                              |
| Loop Over Items1                  | Split In Batches                | Process leads one by one                 | Code1                         | End of Loops, Scrape LinkedIn Profile Post | Sticky Note11: "## Scrape LinkedIn Recent Post"                                              |
| Scrape LinkedIn Profile Post     | HTTP Request                   | Get most recent LinkedIn post           | Loop Over Items1                | Date & Time                     | Sticky Note11: "## Scrape LinkedIn Recent Post"                                              |
| Date & Time                      | DateTime                      | Provide current date/time                | Scrape LinkedIn Profile Post    | OpenAI                         | Sticky Note4: "## Determines if the Linked Profile has a recent post"                        |
| OpenAI                          | OpenAI (GPT-4.1-mini)          | Check if recent post is from 2025       | Date & Time                    | If the LN post is in the last 30 days | Sticky Note4: "## Determines if the Linked Profile has a recent post"                        |
| If the LN post is in the last 30 days | If                            | Branch based on recent LinkedIn post    | OpenAI                        | AI Agent1, Colect LinkedIn Data  |                                                                                              |
| AI Agent1                       | Langchain Agent                | Generate personalization referencing LN post | If the LN post is in the last 30 days | Personalization                 | Sticky Note12: "## Personalization based on recent LN post"                                 |
| Anthropic Chat Model3            | Anthropic LM Chat              | AI model backend for AI Agent1          | AI Agent1                     | AI Agent1                      |                                                                                              |
| Colect LinkedIn Data             | HTTP Request                   | Get LinkedIn profile data for fallback  | If the LN post is in the last 30 days | AI Agent_Personalization_no LN post | Sticky Note10: "## Personalization based on LinkedIn Profile"                             |
| AI Agent_Personalization_no LN post | Langchain Agent              | Generate fallback personalization        | Colect LinkedIn Data           | Personalization                 | Sticky Note10: "## Personalization based on LinkedIn Profile"                               |
| Anthropic Chat Model1            | Anthropic LM Chat              | AI model backend for fallback agent     | AI Agent_Personalization_no LN post | AI Agent_Personalization_no LN post |                                                                                              |
| Personalization                 | Set                           | Store personalization and metadata      | AI Agent1, AI Agent_Personalization_no LN post | Google Sheets Update with Personalization |                                                                                              |
| Google Sheets Update with Personalization | Google Sheets               | Update sheet with personalization        | Personalization               | Loop Over Items1                | Sticky Note: "## Update Google Sheet"                                                       |
| End of Loops                    | Code (JavaScript)              | Aggregate results for Telegram message  | Loop Over Items1               | Lead Enrichment is ready        | Sticky Note3: "## Send Message to Telegram Bot"                                             |
| Lead Enrichment is ready         | Telegram                      | Send enrichment summary to Telegram Bot | End of Loops                  | None                          |                                                                                              |
| Sticky Note (multiple)           | Sticky Note                   | Documentation                           | None                         | None                          | Various notes explaining blocks and logic                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Drive Trigger node:**  
   - Type: Google Drive Trigger  
   - Setup: Watch for `fileCreated` event in folder ID `1-b1yzCQA8RzQmG4vOaGKzYY8iZQxKpvf`  
   - Poll every 1 minute  
   - Use Google Drive OAuth2 credentials

2. **Add Google Drive node to download new file:**  
   - Operation: Download  
   - File ID: Use expression from trigger node: `{{$json["id"]}}`  
   - Connect from Google Drive Trigger1  
   - Use same Google credentials

3. **Add Spreadsheet File node to parse downloaded file:**  
   - Enable header row option  
   - Input: Binary data from Google Drive_DownLoad1

4. **Add Code node to normalize spreadsheet data:**  
   - JavaScript code to map spreadsheet rows to objects with fields: firstName, lastName, company, linkedinUrl, email, website  
   - Connect from Google Sheet Parse

5. **Add Split In Batches node:**  
   - Default batch size (1)  
   - Connect from Code1

6. **Add HTTP Request node to scrape most recent LinkedIn post:**  
   - URL: Apify LinkedIn Profile Posts API endpoint with POST method  
   - JSON Body: `{ "limit": 1, "username": "{{ $json.linkedinUrl }}" }`  
   - Headers: Content-Type, Accept application/json, Authorization Bearer token  
   - Connect from Loop Over Items1

7. **Add Date & Time node:**  
   - No special config, outputs current date/time  
   - Connect from Scrape LinkedIn Profile Post

8. **Add OpenAI node:**  
   - Model: GPT-4.1-mini  
   - Messages: System prompt to check if any post date is from 2025, user prompt includes scraped post data and current date  
   - Output: JSON with `"true"` or `"false"` string  
   - Connect from Date & Time

9. **Add If node:**  
   - Condition: Check if OpenAI output equals `"true"`  
   - Connect from OpenAI

10. **Add Langchain Agent node for personalization with LN post:**  
    - Prompt includes recent LinkedIn post text with instructions to create a natural personalized sentence  
    - Connect from If (true branch)  
    - Use Anthropic Chat Model3 as language model node (Claude Sonnet 4) with Anthropic API credentials

11. **Add HTTP Request node to collect LinkedIn profile data:**  
    - Calls Apify LinkedIn Profile Scraper API with LinkedIn URL  
    - Connect from If (false branch)

12. **Add Langchain Agent node for fallback personalization:**  
    - Prompt uses profile data (name, headline, about, company) to create a personalized sales opening sentence  
    - Connect from Colect LinkedIn Data  
    - Use Anthropic Chat Model1 as language model node with Anthropic API credentials

13. **Add Set node to assign personalization and sheet metadata:**  
    - Fields: personalization (from AI Agent outputs), sheetId, fileName, etc.  
    - Connect from AI Agent1 and AI Agent_Personalization_no LN post

14. **Add Google Sheets node to update sheet with personalization:**  
    - Operation: Update  
    - Document ID: from sheetId field  
    - Sheet name: "Sheet1"  
    - Matching column: "email"  
    - Update personalization column for matching rows  
    - Connect from Personalization  
    - Use Google Sheets OAuth2 credentials

15. **Connect Google Sheets Update node output back to Loop Over Items1**  
    - To continue processing next batch

16. **Add Code node to aggregate all processed items:**  
    - Prepare Telegram message summarizing lead enrichment results and include Google Sheet name and link  
    - Connect from Loop Over Items1

17. **Add Telegram node to send the summary message:**  
    - Message text from aggregation node  
    - Chat ID: Enter your Telegram Bot Chat ID  
    - Credentials: Telegram API credentials  
    - Connect from End of Loops node

18. **Add Sticky Note nodes in the workflow at key locations for documentation**  
    - Optional but recommended for clarity

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                   |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| The workflow relies heavily on Apify APIs for LinkedIn scraping; valid API tokens are required.    | Apify LinkedIn Scraper API docs                                  |
| Anthropic Claude Sonnet 4 model is used as the AI backend for Langchain Agents for personalized text generation. | Anthropic API documentation                                      |
| OpenAI GPT-4.1-mini model is used for date validation logic of LinkedIn posts.                      | OpenAI API documentation                                         |
| Telegram bot must be configured with a valid chat ID and bot token for notifications.              | Telegram Bot API documentation                                   |
| The Google Drive folder ID and Google Sheets document ID must be correctly configured for triggers and updates. | Google Drive and Sheets API documentation                        |
| Potential error points include API rate limits, invalid credentials, missing data fields, and network timeouts. | Monitor logs and set up error handling in production             |
| Sticky notes inside the workflow provide context and explanations for each logical block.           | Visible in n8n editor interface                                  |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.