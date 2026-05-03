Red Team Phishing Simulation with Redirect Tracking and Google Sheets

https://n8nworkflows.xyz/workflows/red-team-phishing-simulation-with-redirect-tracking-and-google-sheets-6509


# Red Team Phishing Simulation with Redirect Tracking and Google Sheets

### 1. Workflow Overview

This workflow titled **"Red Team Phishing Simulation with Redirect Tracking and Google Sheets"** is designed to simulate phishing redirect link hits and log these events for analysis. It is intended primarily for Red Team operators or security professionals conducting phishing simulation campaigns. The workflow automates the generation of redirect cloak URLs, simulates redirect hits, and appends these events as logs into a Google Sheets document for tracking and further analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to initiate the workflow.
- **1.2 Data Retrieval:** Reading source data from Google Sheets to generate redirect URLs.
- **1.3 Redirect Link Generation:** Creating redirect cloak links based on the input data.
- **1.4 Redirect Hit Simulation:** Simulating a user clicking on the redirect link.
- **1.5 Logging:** Appending the redirect hit event details back into Google Sheets for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives the initial manual trigger to start the phishing redirect simulation workflow.

- **Nodes Involved:**  
  - `⚡ Trigger RedirectCloak`

- **Node Details:**  
  - **Node:** ⚡ Trigger RedirectCloak  
    - **Type:** Manual Trigger  
    - **Role:** Entry point to manually start the workflow.  
    - **Configuration:** No special parameters; trigger is manually activated by the user.  
    - **Inputs:** None  
    - **Outputs:** Passes control to the `Google Sheets` node to retrieve data.  
    - **Edge Cases:** Workflow will not run unless manually triggered; no input validation needed here.  
    - **Version:** Compatible with n8n v1 and above.  

#### 2.2 Data Retrieval

- **Overview:**  
  This block reads data from a Google Sheets spreadsheet, presumably containing information necessary to generate redirect links (e.g., URLs, user identifiers).

- **Nodes Involved:**  
  - `Google Sheets`

- **Node Details:**  
  - **Node:** Google Sheets  
    - **Type:** Google Sheets node (v4.6)  
    - **Role:** Reads rows or data from a specified Google Sheets document.  
    - **Configuration:**  
      - Authenticated via OAuth2 credentials configured for Google Sheets API.  
      - Parameters likely set to "Read Rows" operation with a specific spreadsheet ID and sheet name/range.  
    - **Inputs:** Receives trigger from manual trigger node.  
    - **Outputs:** Passes data to the `🔗 Generate Redirect Link` node for processing.  
    - **Edge Cases:**  
      - Authentication failure (expired or misconfigured credentials).  
      - Empty or missing data in the sheet.  
      - API rate limits or quota issues.  
    - **Version:** Requires Google Sheets node version 4.6 or compatible.  

#### 2.3 Redirect Link Generation

- **Overview:**  
  This block transforms the retrieved data to generate redirect cloak URLs. It sets or computes new fields based on input data.

- **Nodes Involved:**  
  - `🔗 Generate Redirect Link`

- **Node Details:**  
  - **Node:** 🔗 Generate Redirect Link  
    - **Type:** Set node (v3.4)  
    - **Role:** Constructs the redirect cloak URL for each entry by setting new fields or modifying existing ones.  
    - **Configuration:**  
      - Uses expressions to build URLs dynamically, possibly concatenating base redirect URLs with parameters from input data.  
      - May create or overwrite fields such as `redirect_url` or `cloak_link`.  
    - **Inputs:** Receives data from `Google Sheets` node.  
    - **Outputs:** Passes updated data to `🛰️ Simulate Redirect Hit`.  
    - **Edge Cases:**  
      - Expression evaluation failures if expected fields are missing.  
      - Malformed URLs if input data contains unexpected characters.  
    - **Version:** Compatible with Set node v3.4.  

#### 2.4 Redirect Hit Simulation

- **Overview:**  
  This block simulates a user clicking on the redirect link generated previously, potentially to test tracking or functionality.

- **Nodes Involved:**  
  - `🛰️ Simulate Redirect Hit`

- **Node Details:**  
  - **Node:** 🛰️ Simulate Redirect Hit  
    - **Type:** Set node (v3.4)  
    - **Role:** Simulates the redirect click event by setting or modifying data to represent a hit.  
    - **Configuration:**  
      - Likely sets fields such as timestamp, hit status, or other metadata for logging purposes.  
    - **Inputs:** Receives redirect URL data from `🔗 Generate Redirect Link`.  
    - **Outputs:** Passes the simulated hit data to `📄 Append Redirect Logs`.  
    - **Edge Cases:**  
      - Data consistency issues if input data is incomplete.  
      - Potential for missing fields for logging if previous steps fail.  
    - **Version:** Compatible with Set node v3.4.  

#### 2.5 Logging

- **Overview:**  
  This block appends the simulated redirect hit data into a Google Sheets document, maintaining a persistent log of all redirect events.

- **Nodes Involved:**  
  - `📄 Append Redirect Logs`

- **Node Details:**  
  - **Node:** 📄 Append Redirect Logs  
    - **Type:** Google Sheets node (v4.6)  
    - **Role:** Writes one or multiple rows into a specified Google Sheets spreadsheet to log redirect hit events.  
    - **Configuration:**  
      - Uses "Append" operation targeting a specific sheet and range.  
      - Authenticated with Google Sheets OAuth2 credentials.  
    - **Inputs:** Receives simulated hit data from `🛰️ Simulate Redirect Hit`.  
    - **Outputs:** None (end of workflow).  
    - **Edge Cases:**  
      - Authentication or permission errors.  
      - API rate limits or quota exceeded.  
      - Data formatting issues causing append failure.  
    - **Version:** Requires Google Sheets node v4.6 or compatible.  

---

### 3. Summary Table

| Node Name               | Node Type               | Functional Role                  | Input Node(s)           | Output Node(s)                    | Sticky Note |
|-------------------------|-------------------------|---------------------------------|------------------------|----------------------------------|-------------|
| Sticky Note             | Sticky Note             | Annotation / Comment             | None                   | None                             |             |
| ⚡ Trigger RedirectCloak | Manual Trigger          | Workflow manual start trigger   | None                   | Google Sheets                    |             |
| Google Sheets           | Google Sheets (v4.6)    | Read source data from spreadsheet | ⚡ Trigger RedirectCloak | 🔗 Generate Redirect Link        |             |
| 🔗 Generate Redirect Link | Set (v3.4)             | Generate redirect cloak URLs     | Google Sheets          | 🛰️ Simulate Redirect Hit         |             |
| 🛰️ Simulate Redirect Hit | Set (v3.4)             | Simulate redirect click event    | 🔗 Generate Redirect Link | 📄 Append Redirect Logs          |             |
| 📄 Append Redirect Logs | Google Sheets (v4.6)    | Log redirect hit data            | 🛰️ Simulate Redirect Hit | None                            |             |
| Sticky Note1             | Sticky Note             | Annotation / Comment             | None                   | None                             |             |
| Sticky Note2             | Sticky Note             | Annotation / Comment             | None                   | None                             |             |
| Sticky Note3             | Sticky Note             | Annotation / Comment             | None                   | None                             |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name it `⚡ Trigger RedirectCloak`.  
   - No special parameters needed. This node starts the workflow manually.

3. **Add a Google Sheets node to read data:**  
   - Name it `Google Sheets`.  
   - Set operation to "Read Rows".  
   - Configure credentials with Google OAuth2 (ensure correct scopes for Sheets API).  
   - Specify the spreadsheet ID and the sheet name or range where source phishing data is stored (e.g., user emails, original URLs).  
   - Connect `⚡ Trigger RedirectCloak` output to `Google Sheets` input.

4. **Add a Set node to generate redirect URLs:**  
   - Name it `🔗 Generate Redirect Link`.  
   - Use this node to add or modify fields:  
     - For example, create a new field `redirect_url` by concatenating a base redirect cloak URL with parameters from the Google Sheets data (e.g., `https://redirect.example.com?target={{ $json["original_url"] }}&user={{ $json["user_id"] }}`).  
   - Connect `Google Sheets` output to `🔗 Generate Redirect Link` input.

5. **Add another Set node to simulate redirect hit:**  
   - Name it `🛰️ Simulate Redirect Hit`.  
   - Set fields to simulate a redirect event, e.g.:  
     - Add timestamp field for the simulated hit (`hit_timestamp` = current ISO datetime).  
     - Add status field (`hit_status` = "simulated").  
   - Connect `🔗 Generate Redirect Link` output to `🛰️ Simulate Redirect Hit` input.

6. **Add a Google Sheets node to append logs:**  
   - Name it `📄 Append Redirect Logs`.  
   - Set operation to "Append Row(s)".  
   - Configure credentials with Google OAuth2.  
   - Specify the spreadsheet ID and sheet name used for logging redirects.  
   - Map fields from the previous node (`🛰️ Simulate Redirect Hit`) to columns in the log sheet (e.g., redirect_url, hit_timestamp, hit_status).  
   - Connect `🛰️ Simulate Redirect Hit` output to `📄 Append Redirect Logs` input.

7. **Add any Sticky Notes as needed for annotations or instructions (optional).**

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                         |
|-------------------------------------------------------------------------------|---------------------------------------|
| The workflow requires properly configured Google Sheets OAuth2 credentials.   | Google Cloud Console for API access.  |
| Ensure the Google Sheets have correct column headers to match mapped fields.  | Spreadsheet setup instructions.       |
| Test manual triggering before deploying in automated environments.            | Manual Trigger node usage.             |
| This workflow is part of a Red Team phishing simulation toolkit.              | Internal RedOps documentation or blog.|

---

**Disclaimer:** The text provided is exclusively from an automated n8n workflow designed for integration and automation. It strictly respects content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.