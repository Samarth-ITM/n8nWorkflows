Generate Accessible Alt Text with Claude AI from Google Sheets to WordPress

https://n8nworkflows.xyz/workflows/generate-accessible-alt-text-with-claude-ai-from-google-sheets-to-wordpress-7548


# Generate Accessible Alt Text with Claude AI from Google Sheets to WordPress

### 1. Workflow Overview

This workflow automates the generation and updating of accessible alternative (alt) text for images hosted on a WordPress site by leveraging AI analysis via Claude AI and synchronizing data with Google Sheets. It is designed for WordPress site owners, content managers, and accessibility advocates seeking to efficiently manage alt text for large media libraries to improve SEO and accessibility compliance.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Authentication**: Receives Google Sheets URL input via a webhook, fetches WordPress credentials and site information from a dedicated Google Sheet, and encodes credentials for authentication.

- **1.2 Data Retrieval and Batch Processing**: Retrieves a list of image URLs and media IDs from a Google Sheet, then processes them in batches to handle multiple images efficiently.

- **1.3 AI-powered Image Analysis**: For each image URL, Claude AI analyzes the image to generate a concise, accessibility-compliant alt text description.

- **1.4 Conditional Flow and Data Updating**: Checks for errors in AI output, updates the Google Sheet with the generated alt text, and then updates the corresponding WordPress media item’s alt text via REST API.

- **1.5 Workflow Termination**: Sends a final response confirming completion of alt text generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Authentication

- **Overview:**  
  This block handles the reception of the Google Sheets URL input via a webhook, retrieves WordPress admin credentials and domain information from a secondary Google Sheet, and prepares the Base64 encoded authentication header needed for WordPress REST API calls.

- **Nodes Involved:**  
  - Send Sheets URL  
  - Get WP Key + website URL WITHOUT https://  
  - Get Base64 key  

- **Node Details:**

  - **Send Sheets URL**  
    - *Type:* Langchain chatTrigger (webhook)  
    - *Role:* Receives Google Sheets URL as chat input via a public webhook, enabling external trigger of the workflow.  
    - *Configuration:* Public webhook with response mode set to respondNode.  
    - *Input:* Triggered by external request containing the Sheets URL.  
    - *Output:* Passes chatInput (Sheets URL) to the next node.  
    - *Failure Modes:* Webhook errors if malformed or missing input.  
    - *Notes:* Entry point of the workflow.

  - **Get WP Key + website URL WITHOUT https://**  
    - *Type:* Google Sheets (read)  
    - *Role:* Reads WordPress admin username, application password (KEY), and domain (without https://) from a sheet named "Infos client" inside the provided Google Sheets URL.  
    - *Configuration:* Uses OAuth2 credentials; document ID dynamically set from webhook input.  
    - *Input:* Receives Sheets URL from previous node.  
    - *Output:* Provides JSON with keys: Admin Name, KEY, Domaine.  
    - *Failure Modes:* Authentication failure, missing or malformed sheet or data, incorrect sheet name.  

  - **Get Base64 key**  
    - *Type:* Code (JavaScript)  
    - *Role:* Concatenates WordPress admin username and application password, encodes in Base64, and constructs the HTTP Basic Authentication header.  
    - *Configuration:* Uses Node.js Buffer to encode `${username}:${password}`.  
    - *Input:* Reads credentials from "Get WP Key + website URL WITHOUT https://".  
    - *Output:* JSON containing username, password, raw credentials string, and full authHeader for HTTP requests.  
    - *Failure Modes:* Missing credential fields cause undefined errors; encoding failures unlikely but possible if inputs are empty.  

---

#### 2.2 Data Retrieval and Batch Processing

- **Overview:**  
  Retrieves the list of WordPress media items (image URLs and IDs) from the "Export media" sheet and splits the data into manageable batches for sequential processing.

- **Nodes Involved:**  
  - Get URLs  
  - Loop Over Items  

- **Node Details:**

  - **Get URLs**  
    - *Type:* Google Sheets (read)  
    - *Role:* Reads the "Export media" sheet which contains WordPress media IDs, image URLs, and alt text placeholders.  
    - *Configuration:* Uses OAuth2 credentials; document ID dynamically from webhook input.  
    - *Input:* Google Sheets URL from the authentication block.  
    - *Output:* JSON array of media entries with ID, URL, and existing alt text (if any).  
    - *Failure Modes:* Authentication errors, missing sheet, or malformed data.  

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes media entries one at a time (default batch size 1) for sequential handling by downstream nodes.  
    - *Configuration:* No custom batch size specified; defaults to 1 for serial processing.  
    - *Input:* Receives media list from "Get URLs".  
    - *Output:* Outputs one media item per iteration.  
    - *Failure Modes:* If batch size is too large, may cause memory or timeout issues.

---

#### 2.3 AI-powered Image Analysis

- **Overview:**  
  For each image URL, this block sends a prompt to Claude AI (Anthropic API) to generate an accessible alt text description following strict guidelines on length, content, and style.

- **Nodes Involved:**  
  - Analyze image  

- **Node Details:**

  - **Analyze image**  
    - *Type:* Anthropic (Claude AI) node via Langchain  
    - *Role:* AI-based image analysis generating concise alt text limited to 125 characters, neutral tone, starting with the main element.  
    - *Configuration:* Uses model "claude-sonnet-4-20250514". The prompt is in French, specifying detailed criteria for alt text.  
    - *Key Expression:* Image URL passed dynamically from current item `$json.URL`.  
    - *Input:* Image URL from current batch item.  
    - *Output:* AI-generated alt text string in JSON content array at `content[0].text`.  
    - *Error Handling:* On error, continues with regular output to avoid workflow halting.  
    - *Failure Modes:* API key issues, rate limits, malformed URLs, unsupported media, or AI response errors.

---

#### 2.4 Conditional Flow and Data Updating

- **Overview:**  
  This block verifies AI output for errors, then updates the Google Sheet with the generated alt text and subsequently updates WordPress media items via REST API.

- **Nodes Involved:**  
  - If  
  - Update row in sheet  
  - Update WP image alt  

- **Node Details:**

  - **If**  
    - *Type:* If node (logical condition)  
    - *Role:* Checks if the AI analysis result contains an error key (`$json.error`).  
    - *Configuration:* Condition tests if error key does *not* exist. If no error, proceeds to update data; else skips to next batch.  
    - *Input:* AI analysis node output.  
    - *Output:* Two branches — success (no error) and failure (error present).  
    - *Failure Modes:* Expression evaluation errors if `$json.error` is undefined or improperly formatted.

  - **Update row in sheet**  
    - *Type:* Google Sheets (update)  
    - *Role:* Updates the "Export media" sheet’s Alt text column for the current media row with the newly generated alt text.  
    - *Configuration:* Matches rows by URL; updates "Alt text" column using AI output. Document ID dynamically set.  
    - *Key Expression:* Alt text set to `{{$json.content[0].text}}` from AI output.  
    - *Input:* AI analysis with successful alt text.  
    - *Output:* Updated sheet data confirmation.  
    - *Failure Modes:* Authentication issues, row matching failures, concurrent update conflicts.  

  - **Update WP image alt**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to WordPress REST API to update the media item’s alt_text field.  
    - *Configuration:*  
      - URL constructed from domain and media ID dynamically: `https://{domain}/wp-json/wp/v2/media/{ID}`  
      - Method: POST  
      - Headers include Content-Type: application/json and Authorization with Basic Auth header from credentials block  
      - Body: JSON with `alt_text` field set from AI output  
    - *Input:* Updated alt text and media ID from loop iteration.  
    - *Output:* WordPress API response confirming update.  
    - *Failure Modes:* Authentication failure, invalid media ID, API endpoint inaccessible, network timeout, malformed JSON body.

---

#### 2.5 Workflow Termination

- **Overview:**  
  Sends a final JSON response indicating that alt text generation is complete.

- **Nodes Involved:**  
  - Fin  

- **Node Details:**

  - **Fin**  
    - *Type:* Respond to Webhook  
    - *Role:* Sends JSON response to the initial webhook caller confirming the end of processing.  
    - *Configuration:* Response body contains a simple message: `"Fin de la rédaction des alt description"` (End of alt descriptions writing).  
    - *Input:* Triggered after all batch processing is complete.  
    - *Output:* HTTP JSON response.  
    - *Failure Modes:* Response failure if webhook connection lost or timeout.

---

### 3. Summary Table

| Node Name                                      | Node Type                          | Functional Role                                      | Input Node(s)                         | Output Node(s)                     | Sticky Note                                                                                                  |
|------------------------------------------------|----------------------------------|-----------------------------------------------------|-------------------------------------|----------------------------------|--------------------------------------------------------------------------------------------------------------|
| Send Sheets URL                                 | Langchain chatTrigger (Webhook)  | Receives Google Sheets URL input                     | —                                   | Get WP Key + website URL WITHOUT https:// | See detailed workflow description and setup instructions in sticky note.                                     |
| Get WP Key + website URL WITHOUT https://      | Google Sheets                    | Retrieves WordPress admin credentials and domain    | Send Sheets URL                     | Get Base64 key                   | See detailed workflow description and setup instructions in sticky note.                                     |
| Get Base64 key                                  | Code                            | Encodes WordPress credentials for Basic Auth header | Get WP Key + website URL WITHOUT https:// | Get URLs                        | See detailed workflow description and setup instructions in sticky note.                                     |
| Get URLs                                        | Google Sheets                    | Retrieves media list with IDs and URLs               | Get Base64 key                     | Loop Over Items                 | See detailed workflow description and setup instructions in sticky note.                                     |
| Loop Over Items                                 | Split In Batches                 | Processes media items sequentially in batches        | Get URLs                          | Fin, Analyze image               | See detailed workflow description and setup instructions in sticky note.                                     |
| Analyze image                                   | Anthropic (Claude AI)            | Generates alt text description from image URL       | Loop Over Items                   | If                             | See detailed workflow description and setup instructions in sticky note.                                     |
| If                                              | If                             | Checks for errors in AI response                      | Analyze image                    | Update row in sheet, Loop Over Items | See detailed workflow description and setup instructions in sticky note.                                     |
| Update row in sheet                             | Google Sheets                   | Updates alt text in Google Sheet                      | If                              | Update WP image alt              | See detailed workflow description and setup instructions in sticky note.                                     |
| Update WP image alt                             | HTTP Request                   | Updates WordPress media alt text via REST API        | Update row in sheet              | Loop Over Items                 | See detailed workflow description and setup instructions in sticky note.                                     |
| Fin                                             | Respond to Webhook               | Sends final completion message                        | Loop Over Items                 | —                              | See detailed workflow description and setup instructions in sticky note.                                     |
| Sticky Note                                     | Sticky Note                    | Documentation and instructions for the workflow      | —                               | —                              | Contains full workflow description, setup instructions, and usage notes.                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Input Node**  
   - Create a **Langchain chatTrigger** node named `Send Sheets URL`.  
   - Set mode to `webhook`, public access enabled, response mode `responseNode`.  
   - This node receives the Google Sheets URL as chat input.

2. **Create Google Sheets Node to Retrieve WordPress Credentials**  
   - Add a **Google Sheets** node named `Get WP Key + website URL WITHOUT https://`.  
   - Set operation to read from sheet named "Infos client".  
   - Use OAuth2 credentials connected to your Google Sheets account.  
   - Set document ID dynamically using expression: `{{$json.chatInput}}` from `Send Sheets URL`.  
   - Retrieve columns: Admin Name (username), KEY (application password), Domaine (site domain without https://).

3. **Create Code Node to Generate Basic Auth Header**  
   - Add a **Code** node named `Get Base64 key`.  
   - Use JavaScript to concatenate username and password from previous node and encode in Base64:  
     ```js
     const username = $('Get WP Key + website URL WITHOUT https://').first().json['Admin Name'];
     const password = $('Get WP Key + website URL WITHOUT https://').first().json.KEY;
     const credentials = username + ':' + password;
     const encodedCredentials = Buffer.from(credentials).toString('base64');
     const authHeader = 'Basic ' + encodedCredentials;
     return { json: { username, password, credentials, authHeader } };
     ```
   - Connect output of `Get WP Key + website URL WITHOUT https://` to this node.

4. **Create Google Sheets Node to Retrieve Media List**  
   - Add a **Google Sheets** node named `Get URLs`.  
   - Set to read from sheet named "Export media".  
   - Use same OAuth2 credentials.  
   - Set document ID dynamically from `Send Sheets URL` chatInput.  
   - Retrieve rows containing columns: ID (WordPress media ID), URL (image URL), Alt text.

5. **Create Split In Batches Node**  
   - Add a **Split In Batches** node named `Loop Over Items`.  
   - Connect input from `Get URLs`.  
   - Default batch size is 1 to process one image at a time.

6. **Create Anthropic AI Node for Image Analysis**  
   - Add an **Anthropic (Claude AI)** node named `Analyze image`.  
   - Set model to `"claude-sonnet-4-20250514"`.  
   - Configure prompt to French text specifying alt text generation rules, including maximum 125 characters, neutral tone, essential visual elements only.  
   - Set resource type to `image`.  
   - Pass image URL dynamically using expression: `{{$json.URL}}`.  
   - Provide Anthropic API credentials.

7. **Create If Node to Check AI Errors**  
   - Add an **If** node named `If`.  
   - Configure condition type: string operation `notExists` on `$json.error`.  
   - If no error, proceed to update sheet; otherwise, loop back to next item.

8. **Create Google Sheets Update Node**  
   - Add a **Google Sheets** node named `Update row in sheet`.  
   - Set operation to `update` on sheet "Export media".  
   - Match rows by `URL`.  
   - Update `Alt text` column with AI-generated text: `{{$json.content[0].text}}`.  
   - Set document ID dynamically from webhook input.

9. **Create HTTP Request Node to Update WordPress Media Alt Text**  
   - Add an **HTTP Request** node named `Update WP image alt`.  
   - Method: POST  
   - URL constructed as: `https://{{ $json.Domaine }}/wp-json/wp/v2/media/{{ $json.ID }}` (use expression referencing domain and media ID from appropriate nodes).  
   - Set headers:  
     - `Content-Type: application/json`  
     - `Authorization: Basic {Base64 key from code node}`  
   - Body parameters (JSON): `{ "alt_text": "{{ AI alt text }}" }`  
   - Connect input from `Update row in sheet`.

10. **Create Respond to Webhook Node**  
    - Add a **Respond to Webhook** node named `Fin`.  
    - Respond with JSON: `{ "text": "Fin de la rédaction des alt description" }`.  
    - Connect this node from the batch loop exit path.

11. **Connect Nodes in Order**  
    - Webhook `Send Sheets URL` → `Get WP Key + website URL WITHOUT https://` → `Get Base64 key` → `Get URLs` → `Loop Over Items` → `Analyze image` → `If` → success branch → `Update row in sheet` → `Update WP image alt` → loop back to `Loop Over Items`.  
    - On batch completion, `Loop Over Items` → `Fin`.

12. **Set Credentials**  
    - Provide Google Sheets OAuth2 credentials for all Google Sheets nodes.  
    - Provide Anthropic API credentials for the Claude AI node.  
    - WordPress authentication handled dynamically via encoded credentials from Google Sheets.

13. **Test with Valid Sheets and WordPress Setup**  
    - Prepare Google Sheets with two sheets:  
      - "Export media" with columns: ID, URL, Alt text (empty initially)  
      - "Infos client" with Admin Name, KEY (WordPress application password), Domaine (site domain without https://)  
    - Ensure WordPress site has Application Passwords enabled and Export Media URLs plugin installed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                                                                                                                                                                                                                                                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates AI-powered alt text generation from Google Sheets to WordPress media library. It is ideal for WordPress site owners and accessibility advocates needing scalable, consistent alt text for SEO and compliance. The AI prompt is in French but can be modified for other languages. Setup requires Google Sheets with media data and WordPress credentials, plus Anthropic API access. The workflow includes error handling to skip unsupported formats.                                                                                                                                                 | Workflow general description and purpose.                                                                                                                                                                                                                                                                                                                                 |
| To export WordPress media URLs, install the "Export Media URLs" plugin, enable ID and URL columns for export, then export media list into Google Sheets.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | WordPress plugin link: https://wordpress.org/plugins/export-media-urls/                                                                                                                                                                                                                                                                                                   |
| Configure WordPress Application Passwords by navigating to Users > Your Profile > Application Passwords, create a new password (e.g., "n8n API"), and copy it immediately.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | WordPress admin dashboard instructions.                                                                                                                                                                                                                                                                                                                                   |
| Google Sheets template with correct structure can be duplicated from: https://docs.google.com/spreadsheets/d/1BKGQRx_xDiuh3QD3ACOOTJomsWFuPBjCHYMQX8UzTBE/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Template sheet for "Export media" and "Infos client".                                                                                                                                                                                                                                                                                                                     |
| The AI prompt can be customized in the "Analyze image" node to alter language, character limit, or style. Batch size can be adjusted in "Loop Over Items" node to balance speed and API limits.                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Customization options for AI and batching.                                                                                                                                                                                                                                                                                                                                |
| This workflow requires reliable internet and API connectivity; ensure Google Sheets and WordPress credentials remain valid. Monitor API usage quotas for Anthropic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Operational considerations and maintenance.                                                                                                                                                                                                                                                                                                                              |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.