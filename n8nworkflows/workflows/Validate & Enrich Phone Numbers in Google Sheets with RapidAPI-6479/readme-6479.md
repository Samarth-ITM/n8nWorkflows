Validate & Enrich Phone Numbers in Google Sheets with RapidAPI

https://n8nworkflows.xyz/workflows/validate---enrich-phone-numbers-in-google-sheets-with-rapidapi-6479


# Validate & Enrich Phone Numbers in Google Sheets with RapidAPI

### 1. Workflow Overview

This workflow automates the validation and enrichment of phone numbers stored in a Google Sheet by leveraging the RapidAPI phone number validation service. It reads phone numbers from a Google Sheet, iterates over each entry to validate and enrich it via an HTTP API call, then updates the original sheet with validation results and additional information like country, location, and timezone.

**Target Use Cases:**  
- Data cleansing and verification of phone numbers in bulk.  
- Enriching phone records with geographic and timezone metadata.  
- Automating repetitive manual validation tasks.  
- Integrations where phone data quality is critical, e.g., CRM or marketing databases.

**Logical Blocks:**

- **1.1 Manual Trigger:** Initiates the workflow execution manually.  
- **1.2 Phone Numbers Retrieval:** Reads phone numbers from Google Sheets.  
- **1.3 Batch Processing Loop:** Processes each phone number individually in batches.  
- **1.4 Phone Validation API Call:** Sends phone numbers to RapidAPI for validation and enrichment.  
- **1.5 Updating Google Sheets:** Updates the sheet with the API response data keyed by phone number.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger

- **Overview:**  
  Provides a manual start point for the workflow, useful for testing or one-off runs.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Starts workflow execution on user command in the n8n editor.  
  - **Configuration:** No parameters required; defaults suffice.  
  - **Inputs:** None  
  - **Outputs:** Connects to Google Sheets node for reading data.  
  - **Failure Modes:** None expected; workflow will not run unless manually triggered.  
  - **Notes:** Used primarily during development or manual runs.

#### 1.2 Phone Numbers Retrieval

- **Overview:**  
  Reads all rows from a specified Google Sheet containing phone numbers for validation.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Type:** Google Sheets node (Read operation)  
  - **Role:** Fetches phone number data from a defined sheet and document.  
  - **Configuration:**  
    - Authentication via Service Account credentials.  
    - Document ID and Sheet Name provided as URLs (masked in JSON).  
    - Reads all rows with no filters applied.  
  - **Key Variables:** Outputs JSON objects with phone numbers and related data.  
  - **Inputs:** Connected from manual trigger.  
  - **Outputs:** Feeds data into the Loop Over Items node.  
  - **Edge Cases:**  
    - Authentication failure if service account is misconfigured.  
    - Empty or invalid Sheet/Document ID results in no data read.  
    - Large sheets may impact performance or require pagination (not configured here).  
  - **Notes:** Uses Google Docs account credential named "Google Docs account".

#### 1.3 Batch Processing Loop

- **Overview:**  
  Iterates over each phone number row individually to process API calls sequentially or in controlled batches.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)

- **Node Details:**  
  - **Type:** SplitInBatches node  
  - **Role:** Splits the list of phone numbers into single-item batches to process one at a time.  
  - **Configuration:** Default batch size (1) implied; no explicit batch size set.  
  - **Inputs:** Receives array of phone number objects from Google Sheets.  
  - **Outputs:**  
    - Main output 1: Empty (no further use).  
    - Main output 2: Sends individual phone number to HTTP Request node.  
  - **Edge Cases:**  
    - If input is empty, loop does not run.  
    - Batch size can be adjusted to prevent API rate limiting.  
  - **Notes:** Ensures API calls are isolated per phone number to handle errors per item.

#### 1.4 Phone Validation API Call

- **Overview:**  
  Sends each phone number to RapidAPI’s phone number validation endpoint, retrieving validation and enrichment data.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **Type:** HTTP Request node  
  - **Role:** Makes POST requests to the RapidAPI phone validation service.  
  - **Configuration:**  
    - URL: `https://phone-number-validator11.p.rapidapi.com/phone.php`  
    - Method: POST  
    - Content Type: multipart/form-data  
    - Body Parameter: `phone` set dynamically from current item’s phone field (`{{$json.phone}}`).  
    - Headers:  
      - `x-rapidapi-host`: `phone-number-validator11.p.rapidapi.com`  
      - `x-rapidapi-key`: placeholder `"your key"` (to be replaced with valid RapidAPI key).  
    - Error Handling: On error, continues execution with error output (does not fail workflow).  
  - **Inputs:** Receives single phone number JSON from Loop Over Items.  
  - **Outputs:**  
    - On success: Outputs validation/enrichment JSON.  
    - On error: Outputs error JSON but continues.  
  - **Edge Cases:**  
    - API key missing or invalid results in auth errors.  
    - Network failure or timeout possible; handled by continuing on error.  
    - API rate limiting may cause failures; no retry configured.  
  - **Notes:** Users must replace `"your key"` with their actual RapidAPI key.

#### 1.5 Updating Google Sheets

- **Overview:**  
  Updates the original Google Sheet rows with validation results such as validity status, country, location, and timezone.

- **Nodes Involved:**  
  - Google Sheets1

- **Node Details:**  
  - **Type:** Google Sheets node (Update operation)  
  - **Role:** Matches rows by phone number and updates enriched columns with API response data.  
  - **Configuration:**  
    - Authentication via Service Account (same as read node).  
    - Document ID and Sheet Name same as reading node.  
    - Matching column: `phone` used to find the correct row to update.  
    - Columns updated: `phone`, `is_valid`, `country`, `location`, `timezone`.  
    - Mapping uses expressions to extract fields from API response JSON.  
    - Does not convert fields to string; preserves original types.  
  - **Inputs:** Receives data from HTTP Request node, combined with original `phone` from Google Sheets.  
  - **Outputs:** Feeds back into Loop Over Items (likely to continue batch processing).  
  - **Edge Cases:**  
    - If phone number not found in sheet, update may fail silently or do nothing.  
    - Authentication or permission issues may cause update failure.  
    - Data type mismatches in columns may cause errors or invalid data.  
  - **Notes:** Uses same Google Docs account credential; requires write permission.

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                         | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                                               |
|-------------------------------|-----------------------|---------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger        | Manually triggers the workflow start  | None                        | Google Sheets               | ### 🔘 When clicking ‘Execute workflow’ Manually starts the workflow from the n8n editor. Used during development or one-time execution.   |
| Google Sheets                  | Google Sheets         | Reads phone numbers from Google Sheet | When clicking ‘Execute workflow’ | Loop Over Items              | ### 📄 Google Sheets (Read) Fetches phone numbers from a specified Google Sheet. Uses a service account for authentication and reads all rows. |
| Loop Over Items                | SplitInBatches        | Loops over each phone number individually | Google Sheets              | HTTP Request (main output 2) | ### 🔁 Loop Over Items (Split In Batches) Loops through each phone number one at a time. Ensures individual API processing per phone number. |
| HTTP Request                  | HTTP Request          | Calls RapidAPI to validate phone number | Loop Over Items            | Google Sheets1 (main output 0), Loop Over Items (main output 1 error) | ### 🌐 HTTP Request (RapidAPI) Sends a POST request to the RapidAPI phone validation endpoint. Receives validation results like `is_valid`, `country`, `location`, and `timezone`. |
| Google Sheets1                 | Google Sheets         | Updates Google Sheet rows with results | HTTP Request               | Loop Over Items              | ### 📥 Google Sheets1 (Update) Updates the original row in the Google Sheet with the API response. Matches based on the phone number and adds new data to columns. |
| Sticky Note                   | Sticky Note           | Documentation summary overview        | None                        | None                        | # 📋 Phone Number Validation with Google Sheets \n\n## 🧩 **Use Case**\nAutomatically validate phone numbers from a Google Sheet using RapidAPI and update the results back to the same sheet.\n\n## ✅ **Benefits**\n- Automates phone validation  \n- Enriches data with location, country, timezone  \n- Reduces manual effort  \n- Handles batch processing  \n- Continues on errors  \n\n---\n\n## 🧱 **Node-by-Node Explanation**\n\n- **🔘 When clicking ‘Execute workflow’** – Manually triggers the workflow.  \n- **📄 Google Sheets** – Reads phone numbers from a Google Sheet.  \n- **🔁 Loop Over Items** – Iterates over each row/phone number.  \n- **🌐 HTTP Request** – Sends phone number to RapidAPI for validation.  \n- **📥 Google Sheets1** – Updates the sheet with validation results (is_valid, country, location, timezone).  \n |
| Sticky Note1                  | Sticky Note           | Explains manual trigger node          | None                        | None                        | ### 🔘 When clicking ‘Execute workflow’ Manually starts the workflow from the n8n editor.  Used during development or one-time execution.     |
| Sticky Note2                  | Sticky Note           | Explains Google Sheets reading node   | None                        | None                        | ### 📄 Google Sheets (Read) Fetches phone numbers from a specified Google Sheet.  Uses a service account for authentication and reads all rows. |
| Sticky Note3                  | Sticky Note           | Explains loop node                    | None                        | None                        | ### 🔁 Loop Over Items (Split In Batches) Loops through each phone number one at a time.  Ensures individual API processing per phone number.  |
| Sticky Note4                  | Sticky Note           | Explains HTTP Request node            | None                        | None                        | ### 🌐 HTTP Request (RapidAPI) Sends a POST request to the RapidAPI phone validation endpoint.  Receives validation results like `is_valid`, `country`, `location`, and `timezone`. |
| Sticky Note5                  | Sticky Note           | Explains Google Sheets update node    | None                        | None                        | ### 📥 Google Sheets1 (Update) Updates the original row in the Google Sheet with the API response.  Matches based on the phone number and adds new data to columns. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`. No configuration needed. This node will start the workflow manually.

2. **Add Google Sheets Node (Read)**  
   - Add a **Google Sheets** node named `Google Sheets`.  
   - Set operation to **Read** (default).  
   - Enter the **Document ID** and **Sheet Name** of your Google Sheet containing phone numbers.  
   - Set **Authentication** to **Service Account** and select appropriate Google Docs credentials with read access.  
   - Leave other options default to read all rows.

3. **Add SplitInBatches Node**  
   - Add a **SplitInBatches** node named `Loop Over Items`.  
   - Connect the output of `Google Sheets` node to this node.  
   - Set **Batch Size** to 1 (default) to process one phone number at a time.

4. **Add HTTP Request Node**  
   - Add an **HTTP Request** node named `HTTP Request`.  
   - Connect the second output (main output index 1) of `Loop Over Items` to this node.  
   - Configure as follows:  
     - **Method:** POST  
     - **URL:** `https://phone-number-validator11.p.rapidapi.com/phone.php`  
     - **Content Type:** multipart/form-data  
     - **Body Parameters:** Add parameter:  
       - Name: `phone`  
       - Value: Expression: `{{$json.phone}}` (to dynamically insert phone number)  
     - **Header Parameters:** Add headers:  
       - Name: `x-rapidapi-host` / Value: `phone-number-validator11.p.rapidapi.com`  
       - Name: `x-rapidapi-key` / Value: Your RapidAPI key (replace `"your key"`)  
     - Set **On Error** behavior to **Continue on error** to avoid workflow halt on API errors.

5. **Add Google Sheets Node (Update)**  
   - Add another **Google Sheets** node named `Google Sheets1`.  
   - Set operation to **Update**.  
   - Provide the same **Document ID** and **Sheet Name** as before.  
   - Set **Authentication** to **Service Account** with write permission.  
   - Under **Columns**, define the mapping:  
     - `phone`: `={{ $('Google Sheets').item.json.phone }}` (original phone number from read node)  
     - `is_valid`: `={{ $json.is_valid }}` (from API response)  
     - `country`: `={{ $json.country }}`  
     - `location`: `={{ $json.location }}`  
     - `timezone`: `={{ $json.timezones[0] }}` (first timezone in array)  
   - Set **Matching Columns** to `phone` to identify the row to update.

6. **Connect Workflow**  
   - Connect `When clicking ‘Execute workflow’` → `Google Sheets`  
   - `Google Sheets` → `Loop Over Items`  
   - `Loop Over Items` (output 1) → `HTTP Request`  
   - `HTTP Request` → `Google Sheets1`  
   - `Google Sheets1` → `Loop Over Items` (main output 1) to continue batch processing.

7. **Credential Setup**  
   - Ensure Google Service Account credentials are configured and authorized for the Google Docs API with read/write access.  
   - Create and input your RapidAPI key for phone number validation in the HTTP Request node headers.

8. **Test & Execute**  
   - Save the workflow.  
   - Click **Execute Workflow** manually to start processing phone numbers.  
   - Monitor output and logs for any errors or edge cases.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                             | Context or Link                                                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates phone validation and enrichment with RapidAPI, reducing manual data cleaning efforts and improving data quality.                                                  | General workflow description                                                                                                                                     |
| Replace `"your key"` in the HTTP Request node with your actual RapidAPI key to authenticate API calls.                                                                                    | API key configuration requirement                                                                                                                               |
| Ensure Google Service Account has permissions for both reading and writing the specified Google Sheet.                                                                                     | Credential setup for Google Sheets nodes                                                                                                                        |
| The workflow handles errors gracefully by continuing on API errors, allowing batch processing to complete even if some numbers fail validation.                                          | Error handling design                                                                                                                                             |
| For more info on RapidAPI phone validation API: https://rapidapi.com/                                                                                        | RapidAPI documentation (external)                                                                                                                               |
| Use batch size tuning in SplitInBatches node to manage API rate limits or optimize throughput.                                                                                              | Performance tuning advice                                                                                                                                        |
| This workflow is suitable for manual or scheduled execution but requires manual trigger setup; to automate, add a schedule trigger node as needed.                                      | Execution mode note                                                                                                                                               |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow export. It adheres strictly to content policies with no illegal or protected material. The data processed is legal and public.