Write Personalised Direct Messages for Instagram with Apify, OpenAI, GSheets

https://n8nworkflows.xyz/workflows/write-personalised-direct-messages-for-instagram-with-apify--openai--gsheets-7668


# Write Personalised Direct Messages for Instagram with Apify, OpenAI, GSheets

### 1. Workflow Overview

This workflow automates the generation of personalized Instagram Direct Messages (DMs) for outreach campaigns using a combination of Google Sheets, Apify Instagram Post Scraper, and OpenAI language and vision models. It is designed for users who want to streamline personalized communication for purposes such as creator outreach, customer interviews, partnership prospecting, or market research.

**Logical Blocks:**

- **1.1 Input Reception:** Reads Instagram account data from a Google Sheet and triggers the workflow manually.
- **1.2 Batch Processing:** Splits the list of accounts into manageable batches for sequential processing.
- **1.3 Instagram Data Retrieval:** For each Instagram account, runs an Apify task to scrape recent posts and fetches detailed post data.
- **1.4 Image Analysis:** Downloads the latest post image and analyzes it using OpenAI’s Vision capabilities to extract descriptive content.
- **1.5 Personalized DM Generation:** Uses OpenAI’s GPT models to generate a customized Instagram DM based on the account metadata, image analysis, and recent post commentary.
- **1.6 Output Storage:** Writes the generated message back into the Google Sheet alongside the corresponding account for tracking and further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow manually and retrieves the list of Instagram accounts from a Google Sheet.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Instagram accounts (Google Sheets Read)
- **Node Details:**

1. **When clicking ‘Execute workflow’**  
   - **Type:** Manual Trigger  
   - **Role:** Starts workflow execution on user command.  
   - **Configuration:** Default manual trigger, no parameters.  
   - **Connections:** Output connects to "Get Instagram accounts".  
   - **Edge Cases:** None significant; user must manually trigger.

2. **Get Instagram accounts**  
   - **Type:** Google Sheets (Read)  
   - **Role:** Reads Instagram account data (including account names, full names, biographies) from a specified Google Sheet tab.  
   - **Configuration:**  
     - Document ID and Sheet Name point to a specific Google Sheet designed for Instagram accounts.  
     - Authentication via Google Service Account for read-only access.  
   - **Key Variables:** Reads columns including "Account", "Full Name", and "Biography".  
   - **Connections:** Output connects to "Loop Over Items".  
   - **Edge Cases:**  
     - Authentication errors if service account credentials are invalid or permissions missing.  
     - Empty or malformed sheet data may cause downstream nodes to fail.

#### 2.2 Batch Processing

- **Overview:** Splits the list of Instagram accounts into batches for sequential processing to manage rate limits and workload.
- **Nodes Involved:**  
  - Loop Over Items (Split in Batches)
- **Node Details:**

1. **Loop Over Items**  
   - **Type:** SplitInBatches  
   - **Role:** Divides the input list of Instagram accounts into smaller batches to process individually.  
   - **Configuration:** Default batch options; batch size unspecified (defaults to 1 item per batch).  
   - **Connections:**  
     - First output (main) connects to "Fetch Instagram Account Data".  
     - Second output is unused.  
   - **Edge Cases:**  
     - If batch size is too large, may cause timeout or API rate limit errors downstream.  
     - If no items are present, downstream nodes receive empty input.

#### 2.3 Instagram Data Retrieval

- **Overview:** For each Instagram account, this block executes an Apify task to scrape recent posts and then retrieves detailed post data from the resulting dataset.
- **Nodes Involved:**  
  - Fetch Instagram Account Data (Apify Run Task)  
  - Get Instagram Account Data (Apify Get Dataset Items)
- **Node Details:**

1. **Fetch Instagram Account Data**  
   - **Type:** Apify Node (Run Actor Task)  
   - **Role:** Executes the Instagram Post Scraper Apify task to fetch the latest posts of the given Instagram account.  
   - **Configuration:**  
     - Uses a custom JSON body with parameters: limits results to 10 posts, no skipping of pinned posts, and the username is dynamically set from the current account item.  
     - Waits up to 60 seconds for task completion.  
     - Requires Apify API credentials with access to the Instagram Post Scraper task.  
   - **Connections:** Output connects to "Get Instagram Account Data".  
   - **Edge Cases:**  
     - Task failure or timeout if Instagram blocks scraping or if Apify API quota is exceeded.  
     - Invalid or private Instagram accounts may result in empty or error datasets.

2. **Get Instagram Account Data**  
   - **Type:** Apify Node (Get Dataset Items)  
   - **Role:** Retrieves the scraped dataset items (posts data) produced by the previous Apify task.  
   - **Configuration:**  
     - Dataset ID is dynamically obtained from the previous node's output.  
     - Pulls all items in the dataset for further processing.  
   - **Connections:** Output connects to "Fetch Image of latest post".  
   - **Edge Cases:**  
     - Dataset not found or empty if scraping failed.  
     - Rate limits or API errors.

#### 2.4 Image Analysis

- **Overview:** Downloads the latest post image and analyzes it using OpenAI’s vision model for content description.
- **Nodes Involved:**  
  - Fetch Image of latest post (HTTP Request)  
  - Analyze image (OpenAI Vision)
- **Node Details:**

1. **Fetch Image of latest post**  
   - **Type:** HTTP Request  
   - **Role:** Downloads the image file of the latest Instagram post using its URL.  
   - **Configuration:**  
     - URL dynamically set from the JSON field `displayUrl` of the post dataset.  
     - Sends custom HTTP headers to mimic a browser user agent to avoid blocking.  
     - Response is expected as a file (image).  
   - **Connections:** Output connects to "Analyze image".  
   - **Edge Cases:**  
     - 404 or inaccessible image URLs.  
     - Timeout or network errors.  
     - Instagram rate limits or anti-bot measures.

2. **Analyze image**  
   - **Type:** OpenAI Node (Vision)  
   - **Role:** Uses OpenAI's GPT-4o vision model to analyze the image content and extract a concise descriptive paragraph including any text from the image.  
   - **Configuration:**  
     - Input type is base64 image data.  
     - Model set to GPT-4o for advanced image understanding.  
   - **Connections:** Output connects to "Generate Personalized DM".  
   - **Edge Cases:**  
     - API errors due to quota or credentials.  
     - Image format issues or corrupted data.

#### 2.5 Personalized DM Generation

- **Overview:** Generates a personalized Instagram DM message using OpenAI’s GPT-5 language model based on analyzed image content, Instagram account metadata, and recent post comments.
- **Nodes Involved:**  
  - Generate Personalized DM (OpenAI Chat)
- **Node Details:**

1. **Generate Personalized DM**  
   - **Type:** OpenAI Node (Chat Completion)  
   - **Role:** Produces a customized Instagram DM message optimized for engagement and authenticity, following a strict message structure and style guidelines.  
   - **Configuration:**  
     - Model: GPT-5 (latest generation).  
     - Messages include a detailed prompt instructing the AI to:  
       - Use information from the account's full name, biography, last post image analysis, caption, and recent comments.  
       - Generate a single paragraph DM with an opening connection, value demonstration, and clear ask.  
       - Follow strict style rules (greeting, no apologies, peer exchange tone, etc.).  
       - Include a provided booking link (dynamically inserted).  
     - System message sets the persona as a seasoned growth stock investor and business strategist with specific communication style guidelines.  
     - JSON output enabled to structure the response as a JSON object with a single "message" field.  
   - **Connections:** Output connects to "Add message to Account".  
   - **Edge Cases:**  
     - API errors, rate limits, or malformed prompt responses.  
     - Potential for incomplete or off-message generation if prompt or input data is missing.

#### 2.6 Output Storage

- **Overview:** Appends or updates the Google Sheet with the generated personalized DM message paired with the Instagram account.
- **Nodes Involved:**  
  - Add message to Account (Google Sheets Append/Update)
- **Node Details:**

1. **Add message to Account**  
   - **Type:** Google Sheets (Append or Update)  
   - **Role:** Writes the generated DM message into the Google Sheet alongside the corresponding Instagram account for easy tracking and sending.  
   - **Configuration:**  
     - Uses the same Google Sheet and Sheet Name as the input node.  
     - Matches rows based on the "Account" column to update existing entries or append new ones.  
     - Writes "Account" and the "Message" fields.  
     - Authentication via the same Google Service Account.  
   - **Connections:** Output loops back to "Loop Over Items" to process next batch/account.  
   - **Edge Cases:**  
     - Authentication errors or permission issues.  
     - Write conflicts if multiple instances run concurrently.  
     - Invalid data causing write errors.

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                                  | Input Node(s)                   | Output Node(s)                | Sticky Note                                                                                                  |
|----------------------------|--------------------------|-------------------------------------------------|---------------------------------|------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger           | Manual start of the workflow                     | —                               | Get Instagram accounts         |                                                                                                              |
| Get Instagram accounts      | Google Sheets             | Reads Instagram account data from Google Sheet  | When clicking ‘Execute workflow’| Loop Over Items               | Copy and configure Google Sheet with Instagram accounts; use Service Account credentials.                     |
| Loop Over Items             | Split in Batches          | Processes accounts one by one in batches         | Get Instagram accounts           | Fetch Instagram Account Data  |                                                                                                              |
| Fetch Instagram Account Data| Apify (Run Actor Task)    | Runs Apify Instagram Post Scraper task per user | Loop Over Items                  | Get Instagram Account Data    | Requires Apify API credentials and Instagram Post Scraper task setup.                                        |
| Get Instagram Account Data  | Apify (Get Dataset Items) | Retrieves scraped Instagram post data            | Fetch Instagram Account Data     | Fetch Image of latest post    |                                                                                                              |
| Fetch Image of latest post  | HTTP Request              | Downloads the latest Instagram post image        | Get Instagram Account Data       | Analyze image                | Sends browser-like headers to avoid blocking.                                                                |
| Analyze image              | OpenAI (Vision)           | Analyzes post image content                        | Fetch Image of latest post       | Generate Personalized DM      | Uses GPT-4o vision model to extract descriptive text from image.                                             |
| Generate Personalized DM    | OpenAI (Chat Completion)  | Generates personalized Instagram DM message      | Analyze image                   | Add message to Account        | Uses GPT-5 with detailed prompt and persona to craft message optimized for engagement and authenticity.      |
| Add message to Account      | Google Sheets             | Appends or updates generated DM in Google Sheet  | Generate Personalized DM          | Loop Over Items               | Writes message back to Google Sheet for tracking; requires write permission.                                  |
| Sticky Note7                | Sticky Note               | Documentation and instructions                    | —                               | —                            | Describes workflow purpose, setup instructions, and usage links: https://docs.google.com/spreadsheets/d/1ATZijpA_kFQyO8afIA-EEx_dA6TSWlyvn8jzTp1eLqs/edit#gid=842468139 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: "When clicking ‘Execute workflow’"
   - Default configuration.
   - This node starts the workflow manually.

3. **Add a Google Sheets node to read Instagram accounts:**
   - Name: "Get Instagram accounts"
   - Operation: "Read rows"
   - Configure Document ID and Sheet Name to point to your Google Sheet with Instagram accounts.
   - Authentication: Use Google Service Account credentials with read access.
   - Read columns: Include "Account", "Full Name", "Biography".
   - Connect from Manual Trigger.

4. **Add a SplitInBatches node to process accounts one by one:**
   - Name: "Loop Over Items"
   - Default batch size (1).
   - Connect from "Get Instagram accounts".

5. **Add an Apify node to run Instagram Post Scraper task:**
   - Name: "Fetch Instagram Account Data"
   - Resource: "Actor tasks"
   - Operation: "Run task"
   - Use custom JSON body:
     ```
     {
       "resultsLimit": 10,
       "skipPinnedPosts": false,
       "username": ["{{ $json.Account }}"]
     }
     ```
   - Set the Apify actor task ID for the Instagram Post Scraper.
   - Wait for finish: 60 seconds.
   - Credentials: Apify API key with access to the task.
   - Connect from "Loop Over Items".

6. **Add an Apify node to retrieve dataset items:**
   - Name: "Get Instagram Account Data"
   - Resource: "Datasets"
   - Operation: "Get items"
   - Dataset ID: Dynamically set to `{{ $json.defaultDatasetId }}`
   - Credentials: Same Apify API key.
   - Connect from "Fetch Instagram Account Data".

7. **Add an HTTP Request node to download the latest post image:**
   - Name: "Fetch Image of latest post"
   - HTTP Method: GET
   - URL: Dynamic from `{{ $json.displayUrl }}`
   - Response format: File
   - Add HTTP headers:
     - User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
     - Accept: image/webp,image/apng,image/*,*/*;q=0.8
   - Connect from "Get Instagram Account Data".

8. **Add an OpenAI node for image analysis:**
   - Name: "Analyze image"
   - Resource: "image"
   - Operation: "analyze"
   - Model: GPT-4o
   - Input type: base64 (use binary data from previous HTTP request)
   - Credentials: OpenAI API key
   - Connect from "Fetch Image of latest post".

9. **Add an OpenAI Chat Completion node to generate DM:**
   - Name: "Generate Personalized DM"
   - Model: GPT-5
   - Enable JSON output.
   - Set prompt messages carefully:
     - System message: Persona as seasoned investor and business strategist with communication style instructions.
     - User message: Detailed instructions to write a personalized Instagram DM including variables from the Google Sheet (full name, biography), image analysis, last post caption, and comments.
     - Include booking link and options for incentive as per the prompt.
   - Credentials: OpenAI API key.
   - Connect from "Analyze image".

10. **Add a Google Sheets node to append or update message:**
    - Name: "Add message to Account"
    - Operation: Append or Update
    - Document and Sheet: Same as "Get Instagram accounts"
    - Matching column: "Account"
    - Columns to write: "Account", "Message" (from OpenAI output)
    - Authentication: Google Service Account with write permission.
    - Connect from "Generate Personalized DM".

11. **Connect "Add message to Account" output back to "Loop Over Items"** to process next batch until all accounts are handled.

12. **Add a Sticky Note node (optional) to document workflow purpose and setup instructions.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| This n8n template fetches Instagram profile information with Apify and generates personalized DMs in Google Sheets. It is ideal for creator outreach, customer interviews, partnership prospecting, and streamlining direct messaging campaigns to save hours per research or marketing campaign.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note content inside workflow                                                                                           |
| Make a copy of [this Google Sheet](https://docs.google.com/spreadsheets/d/1ATZijpA_kFQyO8afIA-EEx_dA6TSWlyvn8jzTp1eLqs/edit#gid=842468139) and update all Google Sheets nodes to point to your copy.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Link provided in Sticky Note                                                                                                  |
| Use a Google Service Account for authentication to enable n8n to read and write to your Google Sheets without user interaction. See [Google Service Account credentials in n8n docs](https://docs.n8n.io/integrations/builtin/credentials/google/service-account/).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Official n8n documentation                                                                                                    |
| Add your OpenAI API key in the OpenAI nodes with sufficient quota and access to GPT-4o (vision) and GPT-5 models.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | OpenAI API Key setup documentation                                                                                            |
| Add Apify API key and configure your Instagram Post Scraper task in Apify. The task must be accessible and capable of running via API. See [Apify Instagram Post Scraper](https://apify.com/apify/instagram-post-scraper).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Apify platform and task setup                                                                                                 |
| The workflow requires manual execution and processes accounts one by one to respect API rate limits and data integrity.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Operational note                                                                                                              |

---

**Disclaimer:** The text provided here is exclusively from an automated n8n workflow. It fully complies with content policies and contains no illegal or offensive elements. All processed data is public and legal.