Filter the feedback from Typeform and store in Google Sheets

https://n8nworkflows.xyz/workflows/filter-the-feedback-from-typeform-and-store-in-google-sheets-1075


# Filter the feedback from Typeform and store in Google Sheets

### 1. Workflow Overview

This workflow automates the collection, filtering, and storage of feedback submitted via a Typeform survey. It targets use cases where qualitative and quantitative feedback needs to be categorized (positive vs. negative) based on a rating scale and stored separately for further analysis or reporting.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Trigger on new Typeform submissions.
- **1.2 Data Extraction:** Extract relevant feedback fields from the raw submission data.
- **1.3 Feedback Filtering:** Classify feedback as positive or negative based on a rating threshold.
- **1.4 Data Storage:** Append the filtered feedback to separate Google Sheets sheets for positive and negative responses.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow when a new response is submitted via the specified Typeform survey.

- **Nodes Involved:**  
  - Typeform Trigger

- **Node Details:**  
  - **Node Name:** Typeform Trigger  
  - **Type:** Trigger node (typeformTrigger)  
  - **Configuration:**  
    - Triggered by new submissions on a specific Typeform identified by `formId: yxcvbnm`.  
    - Uses stored Typeform API credentials (`typeformApi`).  
    - Operates in webhook mode, awaiting live submissions.  
  - **Expressions/Variables:** None (trigger node).  
  - **Inputs:** None (starting node).  
  - **Outputs:** Passes the entire submission JSON to the next node (`Set`).  
  - **Version Requirements:** Compatible with n8n version supporting Typeform Trigger v1.  
  - **Potential Failures:**  
    - Authentication errors if credentials expire or are invalid.  
    - Webhook setup errors if the webhook URL is not properly registered in Typeform.  
    - Network or API downtime issues.  
  - **Sub-workflow:** None.

#### 2.2 Data Extraction

- **Overview:**  
  This block extracts specific fields from the Typeform submission data to simplify downstream processing.

- **Nodes Involved:**  
  - Set

- **Node Details:**  
  - **Node Name:** Set  
  - **Type:** Set node (data transformation)  
  - **Configuration:**  
    - Extracts two fields:  
      - `usefulness` from the question "How useful was the course?" (numerical).  
      - `opinion` from the question "Your opinion on the course:" (string).  
    - Uses expressions to reference the incoming JSON data from Typeform:  
      - `={{$json["How useful was the course?"]}}` for usefulness rating.  
      - `={{$json["Your opinion on the course:"]}}` for opinion text.  
    - Sets `keepOnlySet` to true, so only these two fields are forwarded.  
  - **Inputs:** Receives full Typeform submission JSON from `Typeform Trigger`.  
  - **Outputs:** Sends simplified JSON with only `usefulness` and `opinion` to the `IF` node.  
  - **Version Requirements:** None specific beyond Set node v1.  
  - **Potential Failures:**  
    - Expression failures if question keys do not exactly match the incoming JSON keys (case sensitivity or renaming in Typeform).  
    - Missing or null fields if respondents skip questions.  
  - **Sub-workflow:** None.

#### 2.3 Feedback Filtering

- **Overview:**  
  This block evaluates the usefulness rating and routes feedback to positive or negative branches.

- **Nodes Involved:**  
  - IF

- **Node Details:**  
  - **Node Name:** IF  
  - **Type:** Conditional logic node (if)  
  - **Configuration:**  
    - Condition: `usefulness >= 3` (numeric comparison).  
    - If true: feedback considered positive.  
    - If false: feedback considered negative.  
  - **Inputs:** Receives JSON with `usefulness` and `opinion` from `Set`.  
  - **Outputs:**  
    - Output 1 (true branch): routed to `Google Sheets` node for positive feedback.  
    - Output 2 (false branch): routed to `Google Sheets1` node for negative feedback.  
  - **Version Requirements:** None specific.  
  - **Potential Failures:**  
    - Expression errors if `usefulness` is missing or non-numeric.  
    - Logical errors if threshold criteria changes but node not updated.  
  - **Sub-workflow:** None.

#### 2.4 Data Storage

- **Overview:**  
  This block stores the filtered feedback into two separate Google Sheets sheets, segregating positive and negative feedback.

- **Nodes Involved:**  
  - Google Sheets (positive feedback)  
  - Google Sheets1 (negative feedback)

- **Node Details:**  

  - **Node Name:** Google Sheets  
    - **Type:** Google Sheets node  
    - **Configuration:**  
      - Operation: Append rows.  
      - Sheet ID: `asdfghjklöä` (specific Google Sheet for positives).  
      - Target range: `positive_feedback!A:C`.  
      - OAuth2 authentication using `google_sheets_oauth` credentials.  
    - **Inputs:** Receives positive feedback JSON from IF node (true branch).  
    - **Outputs:** None (final node).  
    - **Potential Failures:**  
      - OAuth2 token expiration or invalidation.  
      - API quota limits.  
      - Sheet ID or range misconfiguration.  
    - **Sub-workflow:** None.

  - **Node Name:** Google Sheets1  
    - **Type:** Google Sheets node  
    - **Configuration:**  
      - Operation: Append rows.  
      - Sheet ID: `qwertzuiop` (specific Google Sheet for negatives).  
      - Target range: `negative_feedback!A:C`.  
      - OAuth2 authentication using `google_sheets_oauth` credentials.  
      - `keyRow` set to 1 (indicating first row is header).  
    - **Inputs:** Receives negative feedback JSON from IF node (false branch).  
    - **Outputs:** None (final node).  
    - **Potential Failures:**  
      - OAuth2 authentication errors.  
      - Sheet range or ID misconfiguration.  
      - API limits.  
    - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name        | Node Type              | Functional Role            | Input Node(s)      | Output Node(s)         | Sticky Note                                          |
|------------------|------------------------|----------------------------|--------------------|------------------------|------------------------------------------------------|
| Typeform Trigger | typeformTrigger          | Start on new Typeform submission | None               | Set                    | course feedback                                      |
| Set              | set                      | Extract relevant fields from submission | Typeform Trigger   | IF                     | capture typeform data                                |
| IF               | if                       | Filter feedback by rating threshold | Set                | Google Sheets, Google Sheets1 | filter feedback                                     |
| Google Sheets    | googleSheets             | Store positive feedback    | IF (true branch)   | None                   | positive feedback                                    |
| Google Sheets1   | googleSheets             | Store negative feedback    | IF (false branch)  | None                   | negative feedback                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Typeform Trigger node:**
   - Node Type: `Typeform Trigger`
   - Set `formId` to your Typeform survey ID (e.g., `yxcvbnm`).
   - Assign the Typeform API credentials (`typeformApi`).
   - Position node (optional): x=450, y=300.
   - Confirm webhook is active and registered in Typeform.

2. **Create the Set node:**
   - Node Type: `Set`
   - Set parameters:  
     - Number field: `usefulness` with expression `={{$json["How useful was the course?"]}}`
     - String field: `opinion` with expression `={{$json["Your opinion on the course:"]}}`
   - Enable "Keep Only Set" option to forward only these fields.
   - Connect output of `Typeform Trigger` to input of `Set`.
   - Position node: x=650, y=300.

3. **Create the IF node:**
   - Node Type: `IF`
   - Add condition:  
     - Type: Number  
     - Variable: `usefulness` (from incoming JSON)  
     - Operation: Greater or equal  
     - Value: 3  
   - Connect output of `Set` to input of `IF`.
   - Position node: x=850, y=300.

4. **Create Google Sheets node for positive feedback:**
   - Node Type: `Google Sheets`
   - Operation: Append
   - Sheet ID: `asdfghjklöä` (replace with your actual Sheet ID)
   - Range: `positive_feedback!A:C`
   - Authentication: OAuth2 with `google_sheets_oauth` credentials
   - Connect `IF` node’s true output (index 0) to this node.
   - Position node: x=1050, y=200.

5. **Create Google Sheets node for negative feedback:**
   - Node Type: `Google Sheets`
   - Operation: Append
   - Sheet ID: `qwertzuiop` (replace with your actual Sheet ID)
   - Range: `negative_feedback!A:C`
   - Key Row: 1 (indicating headers present)
   - Authentication: OAuth2 with `google_sheets_oauth` credentials
   - Connect `IF` node’s false output (index 1) to this node.
   - Position node: x=1050, y=400.

6. **Activate the workflow:**
   - Ensure all credentials are valid and active.
   - Test by submitting a new Typeform entry.
   - Verify data appears in the correct Google Sheets tabs.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow screenshot referenced is stored as fileId:496.                                              | Visual reference for workflow layout             |
| This workflow assumes Typeform questions are exactly named "How useful was the course?" and "Your opinion on the course:". Any changes require Set node adjustments. | Field naming conventions in Typeform             |
| Google Sheets OAuth2 credentials need to be set up properly in n8n with appropriate scopes (spreadsheets). | Credential setup instructions                     |
| For troubleshooting webhook failures, check Typeform webhook registrations and n8n webhook URL settings. | Typeform webhook integration guidelines          |
| The numeric threshold for filtering feedback is currently fixed at 3; change this in the IF node for different criteria. | Customization point                                |

---

This detailed reference document enables full understanding, reproduction, and modification of the feedback filtering workflow with Typeform and Google Sheets in n8n.