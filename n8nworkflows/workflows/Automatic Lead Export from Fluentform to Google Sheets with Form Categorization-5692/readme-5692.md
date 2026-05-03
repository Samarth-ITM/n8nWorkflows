Automatic Lead Export from Fluentform to Google Sheets with Form Categorization

https://n8nworkflows.xyz/workflows/automatic-lead-export-from-fluentform-to-google-sheets-with-form-categorization-5692


# Automatic Lead Export from Fluentform to Google Sheets with Form Categorization

### 1. Workflow Overview

This workflow automates the process of exporting leads submitted via the Fluentform form plugin into Google Sheets, with a clear categorization between newsletter signups and general form submissions. It targets scenarios where an organization collects leads or contacts through multiple form types on a website and wants to organize them automatically in different Google Sheets tabs or documents.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Receive incoming POST requests from Fluentform submissions via a webhook.
- **1.2 Form Type Categorization:** Distinguish between newsletter signups and general form submissions.
- **1.3 Data Extraction and Formatting:** Prepare and extract relevant data fields from the incoming payload for each form type.
- **1.4 Google Sheets Export:** Append or update lead data into dedicated Google Sheets tabs depending on form type.
- **1.5 Response Handling:** Send an acknowledgment response back to the sender confirming successful processing.
- **1.6 Documentation and Guidance:** Sticky notes provide user guidance and project credit links.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming POST requests from Fluentform submissions via a webhook node, initiating the workflow.

- **Nodes Involved:**  
  - POST - Retrieve Leads

- **Node Details:**  

  - **POST - Retrieve Leads**  
    - Type: Webhook  
    - Role: Entry point that listens for POST HTTP requests on a specified path (`your-path`).  
    - Key Configurations:  
      - HTTP method: POST  
      - Path: `your-path` (to be configured specifically for the Fluentform webhook)  
      - Response mode: Delegates response to a downstream node (Respond to Webhook node).  
    - Inputs: External HTTP POST requests from Fluentform containing submission data.  
    - Outputs: Passes JSON payload downstream for processing.  
    - Edge cases:  
      - Incorrect webhook path or HTTP method may cause missed submissions.  
      - Payload format changes in Fluentform could break downstream parsing.  
      - Potential security risk if webhook URL is public without validation.  
    - Version: 2

---

#### 2.2 Form Type Categorization

- **Overview:**  
  Determines whether the incoming submission is from the newsletter form or a general form based on a specific field value in the payload.

- **Nodes Involved:**  
  - Form or newsletter? (IF node)

- **Node Details:**  

  - **Form or newsletter?**  
    - Type: IF  
    - Role: Conditional branching based on the presence and value of the Fluentform nonce (`_fluentform_2_fluentformnonce`).  
    - Configuration:  
      - Condition: Checks if `{{$json.body._fluentform_2_fluentformnonce}}` equals `"c40c9119f6"`  
      - True path leads to the newsletter processing branch.  
      - False path leads to the general form processing branch.  
    - Inputs: JSON from "POST - Retrieve Leads" node.  
    - Outputs: Two branches — newsletter and form.  
    - Edge cases:  
      - If the nonce field is missing or altered, submissions may be misrouted.  
      - No fallback for unknown form types (non-matching nonce).  
    - Version: 2.2

---

#### 2.3 Data Extraction and Formatting

- **Overview:**  
  This block extracts and formats relevant fields for each form type into a simplified data structure suitable for Google Sheets.

- **Nodes Involved:**  
  - newsletter (Set node)  
  - form (Set node)

- **Node Details:**  

  - **newsletter**  
    - Type: Set  
    - Role: Extracts the newsletter subscriber's email from the submission payload.  
    - Configuration:  
      - Assigns a single property `email` from `{{$json.body.__submission.user_inputs.email}}`.  
    - Inputs: JSON from IF node (newsletter branch).  
    - Outputs: JSON with `email` field only.  
    - Edge cases:  
      - Missing or malformed email field may cause incorrect entries.  
    - Version: 3.4

  - **form**  
    - Type: Set  
    - Role: Extracts multiple fields from a general form submission: full name, email, phone number (`no hp`), subject, and message.  
    - Configuration:  
      - Constructs `fullname` by concatenating `first_name` and `last_name` from `body.names`.  
      - Maps `email`, `no hp` (phone number), `subject`, and `message` from respective fields in the payload.  
    - Inputs: JSON from IF node (form branch).  
    - Outputs: Structured JSON with all key fields for Google Sheets.  
    - Edge cases:  
      - Missing any of these fields may result in incomplete data rows.  
      - Assumes specific JSON paths; changes in payload will break mapping.  
    - Version: 3.4

---

#### 2.4 Google Sheets Export

- **Overview:**  
  Appends or updates the extracted lead data into specified Google Sheets tabs, one for newsletter emails and another for general form leads.

- **Nodes Involved:**  
  - Append newsletter (Google Sheets node)  
  - append form (Google Sheets node)

- **Node Details:**  

  - **Append newsletter**  
    - Type: Google Sheets  
    - Role: Adds or updates subscriber emails into the "News Letter" sheet tab.  
    - Configuration:  
      - Operation: `appendOrUpdate` by matching on `Email` column.  
      - Maps only the `Email` field from input JSON.  
      - Document ID and Sheet Name reference a specific Google Sheet and tab (gid=0).  
      - Uses OAuth2 credentials named "khaisa sheet".  
    - Inputs: JSON with `email` field from newsletter Set node.  
    - Outputs: The appended or updated row information.  
    - Edge cases:  
      - Authentication failure if credentials expire.  
      - API rate limits with Google Sheets.  
      - Duplicate emails handled by update operation.  
    - Version: 4.6

  - **append form**  
    - Type: Google Sheets  
    - Role: Adds or updates full form submission data into the "form" sheet tab.  
    - Configuration:  
      - Operation: `appendOrUpdate` by matching on `email`.  
      - Maps `full name`, `email`, `no hp`, `subject`, and `message` fields.  
      - Document ID and Sheet Name specify the same spreadsheet with a different gid (799661662).  
      - Uses the same OAuth2 credentials as above.  
    - Inputs: JSON from form Set node.  
    - Outputs: Updated sheet data.  
    - Edge cases:  
      - Same as Append newsletter regarding credentials and API limits.  
      - May overwrite existing rows matching email.  
    - Version: 4.6

---

#### 2.5 Response Handling

- **Overview:**  
  Sends a JSON success response back to the HTTP POST sender to confirm successful lead processing.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**  

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Returns a JSON response `{"status": "success"}` to the incoming HTTP request.  
    - Configuration:  
      - Response Body is static JSON confirming success.  
    - Inputs: Outputs from Append newsletter or append form nodes.  
    - Outputs: HTTP response sent to the client.  
    - Edge cases:  
      - If previous nodes fail, no response or error response might be sent.  
    - Version: 1.1

---

#### 2.6 Documentation and Guidance

- **Overview:**  
  Sticky Notes provide user instructions and project credits.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky - Parser1

- **Node Details:**  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content:  
      ```
      ## Guideline

      - adjust your `webhook url` on fluent form
      - configure your spreadsheet credentials
      ```  
    - Position: Near webhook node for visibility.

  - **Sticky - Parser1**  
    - Type: Sticky Note  
    - Content:  
      ```
      ### This Workflow is free!
      Got ideas, feedback, or just wanna chat? Hit me up [here](https://khmuhtadin.com)  

      Feeling generous? Buy me a  [coffee](coff.ee/khmuhtadin) to keep the buzz going! ☕
      ```  
    - Position: Near Google Sheets nodes.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                         | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                    |
|---------------------|---------------------|---------------------------------------|----------------------|----------------------|---------------------------------------------------------------------------------------------------------------|
| POST - Retreive Leads| Webhook             | Receive HTTP POST data from Fluentform| (External HTTP POST)  | Form or newsletter?   | ## Guideline  - adjust your `webhook url` on fluent form  - configure your spreadsheet credentials             |
| Form or newsletter?  | IF                  | Branch based on form type nonce        | POST - Retreive Leads | newsletter, form      |                                                                                                               |
| newsletter           | Set                 | Extract email for newsletter signup    | Form or newsletter? (true) | Append newsletter |                                                                                                               |
| form                | Set                 | Extract lead details from form submission | Form or newsletter? (false) | append form         |                                                                                                               |
| Append newsletter    | Google Sheets       | Append or update newsletter emails     | newsletter            | Respond to Webhook    | ### This Workflow is free! Got ideas, feedback, or just wanna chat? Hit me up [here](https://khmuhtadin.com)   |
|                     |                     |                                       |                      |                      | Feeling generous? Buy me a  [coffee](coff.ee/khmuhtadin) to keep the buzz going! ☕                             |
| append form          | Google Sheets       | Append or update general form leads    | form                  | Respond to Webhook    | ### This Workflow is free! Got ideas, feedback, or just wanna chat? Hit me up [here](https://khmuhtadin.com)   |
|                     |                     |                                       |                      |                      | Feeling generous? Buy me a  [coffee](coff.ee/khmuhtadin) to keep the buzz going! ☕                             |
| Respond to Webhook   | Respond to Webhook  | Send success response to HTTP sender   | Append newsletter, append form | (HTTP Response)   |                                                                                                               |
| Sticky Note          | Sticky Note         | User guideline on webhook and credentials | None                 | None                 | ## Guideline  - adjust your `webhook url` on fluent form  - configure your spreadsheet credentials             |
| Sticky - Parser1     | Sticky Note         | Project credits and feedback links     | None                  | None                 | ### This Workflow is free! Got ideas, feedback, or just wanna chat? Hit me up [here](https://khmuhtadin.com)   |
|                     |                     |                                       |                      |                      | Feeling generous? Buy me a  [coffee](coff.ee/khmuhtadin) to keep the buzz going! ☕                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Add a Webhook node named `POST - Retrieve Leads`.  
   - Set HTTP Method to POST.  
   - Set Path (e.g., `your-path`) as desired and configure Fluentform to send submissions to this URL.  
   - Set Response Mode to `Response Node`.

2. **Add IF Node for Form Type:**  
   - Add an IF node named `Form or newsletter?`.  
   - Add condition: Check if `{{$json.body._fluentform_2_fluentformnonce}}` equals `"c40c9119f6"`.  
   - This will split into two branches: True = newsletter, False = general form.

3. **Create Set Node for Newsletter:**  
   - Add a Set node named `newsletter`.  
   - Assign one field `email` with value from `{{$json.body.__submission.user_inputs.email}}`.

4. **Create Set Node for General Form:**  
   - Add a Set node named `form`.  
   - Assign fields:  
     - `fullname`: `{{$json.body.names.first_name}} {{$json.body.names.last_name}}`  
     - `email`: `{{$json.body.email}}`  
     - `no hp`: `{{$json.body.numeric_field}}`  
     - `subject`: `{{$json.body.subject}}`  
     - `message`: `{{$json.body.message}}`

5. **Configure Google Sheets Credentials:**  
   - Setup OAuth2 credentials for Google Sheets API access (named e.g., "khaisa sheet").

6. **Add Google Sheets Node for Newsletter:**  
   - Add Google Sheets node named `Append newsletter`.  
   - Operation: `appendOrUpdate`.  
   - Sheet Name: The tab for newsletter leads (gid=0).  
   - Document ID: Use your target Google Sheet ID.  
   - Columns: Map `Email` to `{{$json.email}}`.  
   - Matching Columns: `Email`.  
   - Use Google Sheets OAuth2 credentials.

7. **Add Google Sheets Node for General Form:**  
   - Add Google Sheets node named `append form`.  
   - Operation: `appendOrUpdate`.  
   - Sheet Name: The tab for form leads (gid=799661662).  
   - Document ID: Same Google Sheet ID.  
   - Columns: Map fields `full name`, `email`, `no hp`, `subject`, `message` accordingly.  
   - Matching Columns: `email`.  
   - Use Google Sheets OAuth2 credentials.

8. **Add Respond to Webhook Node:**  
   - Add a node named `Respond to Webhook`.  
   - Response Body: Static JSON `{ "status": "success" }`.  
   - Connect outputs of both Google Sheets nodes to this node.

9. **Connect Workflow:**  
   - Connect: `POST - Retrieve Leads` → `Form or newsletter?`  
   - Connect True branch of IF to `newsletter` → `Append newsletter` → `Respond to Webhook`  
   - Connect False branch of IF to `form` → `append form` → `Respond to Webhook`

10. **Add Sticky Notes:**  
    - Add a Sticky Note near the webhook node with instructions on adjusting webhook URL and credentials.  
    - Add a Sticky Note near Google Sheets nodes with project credits and feedback links.

11. **Activate Workflow:**  
    - Save and activate the workflow.  
    - Ensure Fluentform is configured to POST to the webhook URL.  
    - Test with form submissions.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                          |
|------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Adjust your webhook URL in Fluentform to match the webhook node path to ensure data is received correctly.        | Sticky Note near webhook node            |
| Configure Google Sheets OAuth2 credentials for authorized access to your spreadsheet.                             | Sticky Note near webhook node            |
| Workflow is free and open for feedback: [https://khmuhtadin.com](https://khmuhtadin.com)                          | Sticky - Parser1 node                    |
| Support the project by buying a coffee: [coff.ee/khmuhtadin](coff.ee/khmuhtadin)                                  | Sticky - Parser1 node                    |

---

**Disclaimer:**  
The provided text comes exclusively from an automated workflow constructed with n8n, a tool for integration and automation. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All data handled is legal and public.