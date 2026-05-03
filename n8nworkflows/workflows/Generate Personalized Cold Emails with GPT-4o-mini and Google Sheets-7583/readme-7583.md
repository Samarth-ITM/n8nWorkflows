Generate Personalized Cold Emails with GPT-4o-mini and Google Sheets

https://n8nworkflows.xyz/workflows/generate-personalized-cold-emails-with-gpt-4o-mini-and-google-sheets-7583


# Generate Personalized Cold Emails with GPT-4o-mini and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of personalized cold email messages targeted at dental clinics, leveraging GPT-4o-mini for AI-driven content creation and Google Sheets for data management. It is designed to process a filtered list of prospects, generate customized email subject lines and bodies crafted to engage recipients effectively, and then update the prospect data in Google Sheets with the generated email content.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval and Filtering:** Fetch and filter prospect data from a Google Sheets document.
- **1.3 Batch Processing:** Process prospects individually or in batches.
- **1.4 AI Email Generation:** Use GPT-4o-mini with a custom prompt to create personalized cold email content.
- **1.5 Email Content Parsing:** Extract and structure the AI-generated JSON content into subject and body.
- **1.6 Data Update:** Update the Google Sheet rows with the generated email content and mark emails as created.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually via a trigger node.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**  
  - **When clicking ‘Test workflow’:**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow on user command.  
    - Configuration: Default manual trigger, no parameters.  
    - Inputs: None (trigger only)  
    - Outputs: Triggers the next node ("Add Row1").  
    - Edge Cases: None significant; workflow does not run without manual start.

#### 2.2 Data Retrieval and Filtering

- **Overview:** Retrieves rows from a Google Sheets document filtering for prospects qualifying for email creation.
- **Nodes Involved:**  
  - Add Row1 (Google Sheets node)
- **Node Details:**  
  - **Add Row1:**  
    - Type: Google Sheets  
    - Role: Reads rows from a specific sheet "california" in the "verified emails" spreadsheet.  
    - Configuration:  
      - Filters rows where `category` equals "Good" and `email created` equals "no".  
      - Returns all matching rows (no limit on first match).  
      - Uses OAuth2 credentials for Google Sheets access.  
    - Inputs: Trigger from manual node  
    - Outputs: Data passed to "Loop Over Items".  
    - Edge Cases:  
      - Authentication failure if OAuth2 token expires.  
      - No rows matching filter results in empty processing downstream.  
      - Google Sheets API quotas or rate limits could cause failures.

#### 2.3 Batch Processing

- **Overview:** Processes filtered prospect rows in sequence or batches to handle rate limits or large data sets.
- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches node)
- **Node Details:**  
  - **Loop Over Items:**  
    - Type: SplitInBatches  
    - Role: Controls the flow to process one or a few prospects at a time.  
    - Configuration: Default batch size (not explicitly set, so defaults apply).  
    - Inputs: Rows from "Add Row1"  
    - Outputs:  
      - Main output 0: processed batch (empty here, no direct output)  
      - Main output 1: individual items sent to "email writer".  
    - Edge Cases:  
      - Empty input leads to no batches processed.  
      - Batch size adjustments may be needed for API rate limits.

#### 2.4 AI Email Generation

- **Overview:** Uses OpenAI GPT-4o-mini model to generate personalized cold email subject and body based on prospect data.
- **Nodes Involved:**  
  - email writer (OpenAI node)
- **Node Details:**  
  - **email writer:**  
    - Type: OpenAI (Langchain)  
    - Role: Generates cold email content using a carefully crafted system and user prompt.  
    - Configuration:  
      - Model: gpt-4o-mini  
      - System prompt: Defines the role as a cold email writer focused on dental clinics with a friendly, persuasive tone. Provides instructions on format and variable usage.  
      - User prompt: Injects dynamic prospect data (email, company name, description, website) to personalize the output.  
      - Expected output: JSON with fields `subject` and `body` formatted precisely.  
    - Inputs: Prospect data from "Loop Over Items"  
    - Outputs: Raw AI response to "Code1".  
    - Credentials: OpenAI API key configured.  
    - Edge Cases:  
      - API failures (e.g., rate limits, network errors)  
      - Invalid or unexpected AI output format (non-JSON or malformed JSON)  
      - Latency or timeout issues.

#### 2.5 Email Content Parsing

- **Overview:** Parses the raw JSON string from the AI node to extract the email subject and body fields safely.
- **Nodes Involved:**  
  - Code1 (Code node)
- **Node Details:**  
  - **Code1:**  
    - Type: Code (JavaScript)  
    - Role: Cleans and parses the AI response JSON string to separate `subject` and `body`.  
    - Configuration:  
      - Removes backticks and "json" labels from the response.  
      - Tries to parse JSON; throws error if invalid.  
      - Returns JSON with `subject` and `body` for downstream use.  
    - Inputs: Raw AI response from "email writer".  
    - Outputs: Structured object for "Update row in sheet".  
    - Edge Cases:  
      - Malformed AI output causing parse errors.  
      - Empty or missing response fields.  
      - Errors stop workflow or require error handling.

#### 2.6 Data Update

- **Overview:** Updates the original Google Sheets row with generated email content and flags the email as created.
- **Nodes Involved:**  
  - Update row in sheet (Google Sheets node)
- **Node Details:**  
  - **Update row in sheet:**  
    - Type: Google Sheets  
    - Role: Updates existing prospect row with:  
      - `email subject` from parsed AI output  
      - `body` from parsed AI output  
      - Sets `email created` to "yes"  
      - Keeps `EMAIL verified` from the original row for matching.  
    - Configuration:  
      - Uses `EMAIL verified` column as the matching key to update the correct row.  
      - Uses the same sheet and document as "Add Row1".  
      - Uses OAuth2 credentials.  
    - Inputs: Parsed email content from "Code1".  
    - Outputs: Feeds back to "Loop Over Items" for further batch processing.  
    - Edge Cases:  
      - Update failures due to mismatched keys or API errors.  
      - Concurrent updates causing race conditions.  
      - Google Sheets API limits or errors.

---

### 3. Summary Table

| Node Name                | Node Type                      | Functional Role                                   | Input Node(s)                 | Output Node(s)            | Sticky Note                                                                                                 |
|--------------------------|--------------------------------|-------------------------------------------------|------------------------------|---------------------------|-------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Starts the workflow manually                      | None                         | Add Row1                  |                                                                                                             |
| Add Row1                 | Google Sheets                  | Retrieves filtered prospect rows from sheet      | When clicking ‘Test workflow’ | Loop Over Items            | ## add google sheet with prospect email name description of there business also create credentials [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Loop Over Items          | SplitInBatches                 | Processes prospect rows in batches                | Add Row1                     | email writer (batch output), Loop Over Items (empty batch output) |                                                                                                             |
| email writer             | OpenAI (Langchain)             | Generates personalized cold email content         | Loop Over Items              | Code1                     | ## this email writer AI node will write the cold email with subject and body you just have to change the prompet on behalf of your business [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Code1                    | Code                          | Parses AI response JSON into subject and body    | email writer                 | Update row in sheet        | ## this code node will separate the body and subject [Guide](https://docs.n8n.io/workflows/sticky-notes/)  |
| Update row in sheet      | Google Sheets                  | Updates prospect row with generated email content | Code1                        | Loop Over Items            |                                                                                                             |
| Sticky Note              | Sticky Note                   | Instructional note                                | None                         | None                      | ## add google sheet with prospect email name description of there business also create credentials [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Sticky Note1             | Sticky Note                   | Instructional note about AI email writer node    | None                         | None                      | ## this email writer AI node will write the cold email with subject and body you just have to change the prompet on behalf of your business [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Sticky Note2             | Sticky Note                   | Instructional note about code node parsing JSON   | None                         | None                      | ## this code node will separate the body and subject [Guide](https://docs.n8n.io/workflows/sticky-notes/)  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: When clicking ‘Test workflow’  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Sheets Node to Read Prospect Data**
   - Name: Add Row1  
   - Type: Google Sheets  
   - Operation: Read  
   - Configure:  
     - Document ID: `14JjbOiXXakwQrWpjdudiXrNDOVNfVKUGcNoBccgIJxg` (verified emails spreadsheet)  
     - Sheet Name: `california` (gid 2062707747)  
     - Filters:  
       - category = "Good"  
       - email created = "no"  
     - Return all matching rows, no limit on first match.  
   - Credentials: Google Sheets OAuth2 account setup.  
   - Connect output to "Loop Over Items".

3. **Add SplitInBatches Node for Batch Processing**
   - Name: Loop Over Items  
   - Type: SplitInBatches  
   - Configuration: Use default batch size or specify if needed (e.g., 1 for single processing).  
   - Input from "Add Row1".  
   - Connect batch output (main output 1) to "email writer".  
   - Connect empty batch output (main output 0) back to "Loop Over Items" for next batch.

4. **Add OpenAI Node for Email Generation**
   - Name: email writer  
   - Type: OpenAI (Langchain)  
   - Credentials: Configure OpenAI API key.  
   - Model: Select `gpt-4o-mini`.  
   - Prompt Setup:  
     - System Message: Define role as cold email writer for dentists, friendly and persuasive tone, with detailed instructions on JSON format, shortening company names, avoiding clichés, and using input variables: email, company name, description, website.  
     - User Message: Inject prospect data using expressions:  
       - `email:{{ $json['EMAIL verified'] }}`  
       - `company name:{{ $json.companyName }}`  
       - `description:{{ $json[' description'] }}`  
       - `website:{{ $json.website }}`  
   - Connect output to "Code1".

5. **Add Code Node to Parse AI Response**
   - Name: Code1  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const input = $input.first();
     let rawJson = input.json.message.content;
     rawJson = rawJson.replace(/```json|```/g, '').trim();
     let parsed;
     try {
       parsed = JSON.parse(rawJson);
     } catch (e) {
       throw new Error("Invalid JSON format: " + e.message);
     }
     return [{
       json: {
         subject: parsed.subject,
         body: parsed.body
       }
     }];
     ```
   - Input from "email writer".  
   - Output to "Update row in sheet".

6. **Add Google Sheets Node to Update Rows**
   - Name: Update row in sheet  
   - Type: Google Sheets  
   - Operation: Update  
   - Configure:  
     - Document ID and Sheet Name same as "Add Row1".  
     - Matching Column: `EMAIL verified` (used to locate the row).  
     - Columns to update:  
       - `email subject`: set to `={{ $json.subject }}` from Code1.  
       - `body`: set to `={{ $json.body }}` from Code1.  
       - `email created`: set to "yes".  
       - `EMAIL verified`: set to value from original item (`{{$('Add Row1').item.json['EMAIL verified']}}`).  
   - Credentials: Google Sheets OAuth2 account.  
   - Connect output back to "Loop Over Items" to continue batch processing.

7. **Add Sticky Notes for Documentation (Optional)**
   - Add notes near nodes for instructions and references as in the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Add Google Sheet with prospect email, name, description of their business; also create credentials for Google Sheets access.    | [n8n Sticky Notes Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| The AI email writer node uses a custom prompt tailored for dental clinics; edit prompt to fit your business needs.              | [n8n Sticky Notes Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| The code node safely parses AI JSON responses to separate email subject and body.                                               | [n8n Sticky Notes Guide](https://docs.n8n.io/workflows/sticky-notes/) |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.