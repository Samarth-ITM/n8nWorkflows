Verify WhatsApp Numbers in Bulk using Google Sheets & WasenderAPI

https://n8nworkflows.xyz/workflows/verify-whatsapp-numbers-in-bulk-using-google-sheets---wasenderapi-6183


# Verify WhatsApp Numbers in Bulk using Google Sheets & WasenderAPI

### 1. Workflow Overview

This workflow automates the bulk verification of WhatsApp numbers stored in a Google Sheet by leveraging an unofficial WhatsApp API service called WasenderAPI. It is designed for users who want to check if a list of phone numbers are registered on WhatsApp and update their verification status back into the Google Sheet. The workflow operates in batches with delays to respect API rate limits.

Logical blocks included:

- **1.1 Scheduled Trigger & Data Fetch**: Periodically triggers the workflow and fetches pending (unverified) WhatsApp numbers from Google Sheets.
- **1.2 Batch Processing & Rate Limiting**: Limits and splits the data into manageable batches to process sequentially with enforced wait times.
- **1.3 WhatsApp Number Verification**: Uses the WasenderAPI HTTP endpoint to verify if each number is registered on WhatsApp.
- **1.4 Conditional Status Setting**: Based on the API response, sets the status as "Verified" or "Unverified".
- **1.5 Google Sheets Update**: Writes back the verification results and changes the row status to "Checked".
- **1.6 Loop and Continue**: Waits between batches and loops to process the next batch until all pending numbers are checked.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Fetch

- **Overview:**  
  This block triggers the workflow every 5 minutes and fetches all rows from a Google Sheet where the `Status` column is empty, indicating pending verification.

- **Nodes Involved:**  
  - Trigger Every 5 Minute (Schedule Trigger)  
  - Fetch All Pending Contacts for Verifing (Google Sheets)  
  - Limit  

- **Node Details:**

  - **Trigger Every 5 Minute**  
    - Type: Schedule Trigger  
    - Configuration: Triggers every 1 minute (interval in minutes field), effectively used as a periodic scheduler.  
    - Input: None (initiates workflow)  
    - Output: Triggers downstream nodes.  
    - Edge cases: Trigger misconfiguration or downtime could delay workflow execution.

  - **Fetch All Pending Contacts for Verifing**  
    - Type: Google Sheets  
    - Configuration: Reads rows from a specific Google Sheet document and sheet tab named "Contacts". Filters rows where the `Status` column is empty (pending verification).  
    - Credentials: Google Sheets OAuth2 configured.  
    - Input: Trigger node output  
    - Output: List of rows matching the filter for further processing.  
    - Edge cases: API quota limits, authentication failure, or sheet structure change might cause failure.

  - **Limit**  
    - Type: Limit  
    - Configuration: Limits the number of rows processed to a maximum of 60 per workflow execution to avoid overloading the API or exceeding rate limits.  
    - Input: Output from Google Sheets node  
    - Output: Limited subset of rows passed on to batch processing.  
    - Edge cases: If there are more than 60 pending rows, multiple workflow runs will be needed.

---

#### 1.2 Batch Processing & Rate Limiting

- **Overview:**  
  This block splits the limited dataset into individual items or small batches and ensures a wait time between processing each batch to comply with API rate limits.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Wait  

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Configuration: Splits the input data (max 60 rows) into batches; default batch size is 1 (processes one item at a time).  
    - Input: Output from Limit node  
    - Output: Single item per iteration, enabling sequential processing.  
    - Edge cases: If batch size is too small or too big, it could affect throughput or cause rate limit errors.

  - **Wait**  
    - Type: Wait  
    - Configuration: Introduces a pause (configured in node but default not specified here; commonly 5 seconds) after processing each batch iteration to prevent API throttling.  
    - Input: After updating Google Sheets for each item  
    - Output: Triggers next batch iteration  
    - Edge cases: Excessive wait time increases total execution time; too short may cause API rate limit errors.

---

#### 1.3 WhatsApp Number Verification

- **Overview:**  
  This block sends an HTTP request to the WasenderAPI to check if a given WhatsApp number is registered.

- **Nodes Involved:**  
  - Verify WhatsApp Number Using HTTP Request  

- **Node Details:**

  - **Verify WhatsApp Number Using HTTP Request**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET (implied by URL usage)  
      - URL: `https://www.wasenderapi.com/api/on-whatsapp/{{ $json['WhatsApp No'] }}@s.whatsapp.net` (dynamic URL where phone number is inserted)  
      - Authentication: HTTP Bearer Token using WasenderAPI credentials  
    - Input: Single WhatsApp number from batch iteration  
    - Output: API response indicating if the number exists on WhatsApp (boolean in `data.exists`)  
    - Edge cases: API downtime, invalid API key, malformed phone numbers, network timeouts.

---

#### 1.4 Conditional Status Setting

- **Overview:**  
  This block evaluates the API response and sets the verification status accordingly.

- **Nodes Involved:**  
  - If  
  - Set Status  
  - Set Status1  

- **Node Details:**

  - **If**  
    - Type: If  
    - Configuration: Checks if `data.exists` in the API response is `true`.  
    - Input: HTTP Request node output  
    - Output: Routes to Set Status (true) or Set Status1 (false) nodes.  
    - Edge cases: Unexpected API response structure or missing fields.

  - **Set Status**  
    - Type: Code  
    - Configuration: Returns object `{ txt: "Verified" }`  
    - Input: If node (true branch)  
    - Output: Passes "Verified" status downstream  
    - Edge cases: Code execution failure (unlikely).

  - **Set Status1**  
    - Type: Code  
    - Configuration: Returns object `{ txt: "Unverified" }`  
    - Input: If node (false branch)  
    - Output: Passes "Unverified" status downstream  
    - Edge cases: Code execution failure (unlikely).

---

#### 1.5 Google Sheets Update

- **Overview:**  
  This block updates the Google Sheet row with the verification result and marks the row as "Checked" to avoid re-processing.

- **Nodes Involved:**  
  - Change State of Rows in Checked (Google Sheets)  

- **Node Details:**

  - **Change State of Rows in Checked**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Update  
      - Sheet and document same as fetch node  
      - Updates columns:  
        - `Verified/Unverified` column with the value from the status code nodes (`txt`)  
        - `Status` column set to `"Checked"`  
      - Matches rows by `row_number` from the batch iteration context  
    - Credentials: Google Sheets OAuth2  
    - Input: Status code node output (either "Verified" or "Unverified")  
    - Output: Confirmation of update, triggers wait node  
    - Edge cases: API quota limits, row matching failure, permission issues.

---

#### 1.6 Loop and Continue

- **Overview:**  
  This block waits a configured time before looping back to process the next batch item.

- **Nodes Involved:**  
  - Wait  
  - Loop Over Items  

- **Node Details:**

  - **Wait**  
    - (Details as above)

  - **Loop Over Items**  
    - Loops to next batch item after wait completes until all items processed.

---

### 3. Summary Table

| Node Name                             | Node Type             | Functional Role                           | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
|-------------------------------------|-----------------------|-----------------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger Every 5 Minute               | Schedule Trigger      | Periodic trigger to start workflow      | None                              | Fetch All Pending Contacts for Verifing |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Fetch All Pending Contacts for Verifing | Google Sheets        | Fetch Google Sheet rows with empty Status | Trigger Every 5 Minute            | Limit                             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Limit                               | Limit                 | Limit number of rows to 60 per run      | Fetch All Pending Contacts for Verifing | Loop Over Items                  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Loop Over Items                     | SplitInBatches        | Process rows one at a time in batches   | Limit                            | Verify WhatsApp Number Using HTTP Request (main), also empty first output |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Verify WhatsApp Number Using HTTP Request | HTTP Request         | Verify if number exists on WhatsApp via WasenderAPI | Loop Over Items                  | If                              |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| If                                  | If                    | Branch based on verification response   | Verify WhatsApp Number Using HTTP Request | Set Status (true branch), Set Status1 (false branch) |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Set Status                         | Code                  | Set status to "Verified"                 | If (true)                       | Change State of Rows in Checked   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Set Status1                        | Code                  | Set status to "Unverified"               | If (false)                      | Change State of Rows in Checked   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Change State of Rows in Checked    | Google Sheets         | Update Google Sheet row with results     | Set Status, Set Status1          | Wait                             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Wait                              | Wait                  | Delay between batch processing           | Change State of Rows in Checked  | Loop Over Items                   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note1                      | Sticky Note           | Documentation and setup instructions     | None                           | None                            | # 📘 Bulk WhatsApp Number Verification via Unofficial API\n\n## 📌 Overview\n\nThis **n8n workflow** verifies **unlimited WhatsApp numbers** from a **Google Sheet** using your **own WhatsApp number** through the [WasenderAPI.com](https://www.wasenderapi.com/) platform.\n\n---\n\n## 🔍 What It Does\n\n- ✅ Fetches real-time data from Google Sheets  \n- 🔍 Verifies if a number is registered on WhatsApp  \n- 🛡 Implements delay and batch processing to maintain rate limits  \n- ✏️ Updates verification status back to the Google Sheet  \n\n---\n\n## 🧰 Requirements\n\nBefore setting up the workflow, ensure you have:\n\n- ✅ An active **WhatsApp Account** (Personal or Business)  \n- ✅ **Google Sheets API** configured within your n8n instance  \n- ✅ A properly structured [Google Sheet](https://docs.google.com/spreadsheets/d/1Ui4TzzI-Gq-bsEsrZELwW1Kyddw0IU9L1wxlHikktqw/edit?usp=sharing)  \n- ✅ A **WasenderAPI.com** subscription (starting from ~$6/month)  \n\n---\n\n## 📄 Google Sheet Format\n\n| WhatsApp No   | Verified/Unverified                         | Status     |\n|---------------|----------------------------------|------------|\n| +8801XXXXXXX  |  *(empty)* | *(empty)*  |\n\n> ℹ️ Ensure the `Status` column is initially blank for unverified rows.\n\n---\n\n## ⚙️ How to Set Up in n8n\n\n### Step 1: 🔗 Connect to Google Sheets\n- Add a **Google Sheets** node.\n- Authenticate with your Google account.\n- Select your target document and worksheet.\n- Apply a filter to only select rows where `Status` is empty.\n\n### Step 2: 🔄 Loop Through Rows with Delay\n- Add a **SplitInBatches** or **Code** node to process 5 rows at a time.\n- Add a **Wait** node (set to 5 seconds) between each request to respect API limits.\n\n### Step 3: 🌐 Verify Number via HTTP Request\n- Add an **HTTP Request** node\n- **Method**: `POST`\n- **URL**: `https://app.wasenderapi.com/api/send-message`\n- **Headers**:\n  - `Content-Type`: `application/json`\n  - `Authorization`: `Bearer YOUR_API_KEY`\n- **JSON Body**:\n```json\n{\n  \"number\": \"{{ $json['WhatsApp No'] }}\",\n  \"message\": \"{{ $json['Message'] }}\"\n}``` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to trigger every 5 minutes (interval: minutes → 5).  
   - No credentials needed.

2. **Create Google Sheets Node (Fetch Pending Rows):**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2 (authenticate with your Google account).  
   - Operation: Read rows.  
   - Document ID: Select your specific Google Sheet.  
   - Sheet Name: The tab where numbers are stored (e.g., "Contacts").  
   - Add filter to select only rows where `Status` column is empty (blank).  
   - No limit on rows here (limit applied next).

3. **Create Limit Node:**  
   - Type: Limit  
   - Set `maxItems` to 60 to avoid processing too many rows per run.

4. **Create SplitInBatches Node (Loop Over Items):**  
   - Type: SplitInBatches  
   - Batch size: 1 (process one WhatsApp number at a time).  
   - Connect input from Limit node.

5. **Create HTTP Request Node (WhatsApp Verification):**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://www.wasenderapi.com/api/on-whatsapp/{{ $json['WhatsApp No'] }}@s.whatsapp.net` (use expression to insert WhatsApp number).  
   - Authentication: HTTP Bearer Auth (configure WasenderAPI credentials with your API key).  
   - Connect input from SplitInBatches node.

6. **Create If Node:**  
   - Type: If  
   - Condition: Check if `data.exists` property in HTTP response is `true` (use expression `{{$json.data.exists}}`).  
   - Connect input from HTTP Request node.

7. **Create Two Code Nodes (Set Status and Set Status1):**  
   - Set Status: Return `{ txt: "Verified" }`  
   - Set Status1: Return `{ txt: "Unverified" }`  
   - Connect `true` output of If node to Set Status.  
   - Connect `false` output of If node to Set Status1.

8. **Create Google Sheets Node (Update Row):**  
   - Type: Google Sheets  
   - Credentials: Same Google Sheets OAuth2.  
   - Operation: Update  
   - Document and Sheet: Same as the fetch node.  
   - Mapping:  
     - Update column `Status` to `"Checked"`  
     - Update column `Verified/Unverified` with value from previous Code node (`txt`)  
     - Match row using `row_number` from the batch item’s JSON data.  
   - Connect input from both Set Status and Set Status1 nodes.

9. **Create Wait Node:**  
   - Type: Wait  
   - Set duration to 5 seconds (or suitable delay to respect API rate limits).  
   - Connect input from Google Sheets update node.

10. **Connect Wait node back to SplitInBatches node:**  
    - This creates a loop to process the next batch item after waiting.

11. **Validate all connections and credentials:**  
    - Ensure Google Sheets OAuth2 credentials are active and have proper access.  
    - Ensure WasenderAPI credentials include a valid Bearer token.  
    - Test the HTTP Request node separately to verify API key and endpoint correctness.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                                                                                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow requires a properly structured Google Sheet where the `Status` column is initially empty for unverified numbers. The sheet should have columns: `WhatsApp No`, `Verified/Unverified`, and `Status`.                                                       | Google Sheet format example: https://docs.google.com/spreadsheets/d/1Ui4TzzI-Gq-bsEsrZELwW1Kyddw0IU9L1wxlHikktqw/edit?usp=sharing                                                                 |
| WasenderAPI is an unofficial platform for WhatsApp automation and verification; ensure you have an active subscription and valid API key before running this workflow.                                                                                                | https://www.wasenderapi.com/                                                                                                                                                                |
| Rate limiting is important to avoid API blocks; this workflow processes 60 rows per run, one by one with a 5-second wait between each request to comply with limits. Adjust batch size and wait duration as needed.                                                    |                                                                                                                                                                                              |
| For further customization, the HTTP Request node can be modified to send custom messages or integrate with other WhatsApp automation features offered by WasenderAPI.                                                                                                |                                                                                                                                                                                              |
| The workflow includes a sticky note with detailed setup instructions, overview, and links for reference.                                                                                                                                                               | Sticky Note within the workflow UI                                                                                                                                                           |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.