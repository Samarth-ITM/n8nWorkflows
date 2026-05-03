Simulate Phishing Attacks with Google Sheets for Security Awareness & SOC Testing

https://n8nworkflows.xyz/workflows/simulate-phishing-attacks-with-google-sheets-for-security-awareness---soc-testing-6510


# Simulate Phishing Attacks with Google Sheets for Security Awareness & SOC Testing

---

### 1. Workflow Overview

This workflow, titled **"Simulate Phishing Attacks with Google Sheets for Security Awareness & SOC Testing"**, is designed to automate the process of simulating phishing credential trap attacks targeting a list of users maintained in Google Sheets. It is aimed at security operations centers (SOC) and security awareness teams to test and improve organizational resilience against phishing.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception**: Manual trigger to start the simulation.
- **1.2 Target Acquisition**: Retrieval of target user data from Google Sheets.
- **1.3 Target Validation**: Filtering out invalid or incomplete target entries.
- **1.4 Trap Link Generation**: Creating personalized phishing trap links for valid targets.
- **1.5 Credential Submission Simulation**: Simulating the action of credential submission by the target.
- **1.6 Logging**: Appending the results of the simulation to a Google Sheets log for tracking and analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually. It allows an operator to start the phishing simulation process on demand.

- **Nodes Involved:**  
  - `⚡ Trigger TrapSim`

- **Node Details:**  

  - **Node Name:** ⚡ Trigger TrapSim  
  - **Type:** Manual Trigger  
  - **Technical Role:** Entry point to manually start the workflow  
  - **Configuration:** Default manual trigger with no parameters; waits for user to start execution  
  - **Expressions/Variables:** None  
  - **Input Connections:** None (start node)  
  - **Output Connections:** Connects to `📄 Get Trap Targets`  
  - **Version Requirements:** None specific, supported in all recent n8n versions  
  - **Edge Cases / Failures:** None expected; manual trigger is reliable  
  - **Sub-workflow Reference:** None

---

#### 2.2 Target Acquisition

- **Overview:**  
  This block retrieves the list of phishing simulation targets from a Google Sheets document. The data typically includes user identifiers and contact information.

- **Nodes Involved:**  
  - `📄 Get Trap Targets`

- **Node Details:**  

  - **Node Name:** 📄 Get Trap Targets  
  - **Type:** Google Sheets  
  - **Technical Role:** Reads rows from a configured Google Sheets spreadsheet containing target information  
  - **Configuration:**  
    - Operation: Read rows  
    - Spreadsheet ID and Sheet Name: Configured to the target list sheet  
    - Authentication: Requires Google Sheets OAuth2 credentials  
  - **Expressions/Variables:** Data retrieved becomes input for filtering  
  - **Input Connections:** From `⚡ Trigger TrapSim`  
  - **Output Connections:** To `🧹 Filter Valid Targets`  
  - **Version Requirements:** Google Sheets node version 4.6 or later recommended for stability and features  
  - **Edge Cases / Failures:**  
    - Authentication errors if credentials are invalid or expired  
    - Empty or malformed data in sheet could cause downstream issues  
    - API rate limits if sheet is very large or accessed frequently  
  - **Sub-workflow Reference:** None

---

#### 2.3 Target Validation

- **Overview:**  
  This block filters the retrieved targets to ensure only valid and actionable entries proceed. For example, it may exclude rows lacking email addresses or with invalid formats.

- **Nodes Involved:**  
  - `🧹 Filter Valid Targets`

- **Node Details:**  

  - **Node Name:** 🧹 Filter Valid Targets  
  - **Type:** If (Conditional) Node  
  - **Technical Role:** Applies conditional logic to filter out invalid targets  
  - **Configuration:**  
    - Condition checks presence and possibly format validity of critical fields (e.g., email address not empty)  
    - Likely configured with simple expressions such as `{{$json["email"] !== ""}}`  
  - **Expressions/Variables:** Uses JSON data fields from `📄 Get Trap Targets`  
  - **Input Connections:** From `📄 Get Trap Targets`  
  - **Output Connections:** To `🎯 Generate Trap Link` for valid entries; others discarded or ignored  
  - **Version Requirements:** If node version 2.2 or later recommended for expression support  
  - **Edge Cases / Failures:**  
    - Expression errors if expected fields missing  
    - No valid targets case leads to no further execution downstream  
  - **Sub-workflow Reference:** None

---

#### 2.4 Trap Link Generation

- **Overview:**  
  This block generates personalized phishing trap URLs for each validated target, simulating a credential trap link that might be sent in a phishing email.

- **Nodes Involved:**  
  - `🎯 Generate Trap Link`

- **Node Details:**  

  - **Node Name:** 🎯 Generate Trap Link  
  - **Type:** Set Node  
  - **Technical Role:** Constructs custom URL strings with embedded target information  
  - **Configuration:**  
    - Uses expressions to build URLs dynamically, e.g. concatenating base phishing URL with user-specific parameters  
    - Sets new fields such as `trapLink` in the workflow data  
  - **Expressions/Variables:** Uses input JSON fields like user email or ID to create personalized links  
  - **Input Connections:** From `🧹 Filter Valid Targets`  
  - **Output Connections:** To `🪤 Simulate Credential Submission`  
  - **Version Requirements:** Set node v3.4 or higher recommended to support complex expressions  
  - **Edge Cases / Failures:**  
    - Invalid data may lead to malformed URLs  
    - Expression evaluation errors if input data missing expected fields  
  - **Sub-workflow Reference:** None

---

#### 2.5 Credential Submission Simulation

- **Overview:**  
  This block simulates the victim submitting credentials through the trap link, potentially generating fake credential data or marking the trap as triggered.

- **Nodes Involved:**  
  - `🪤 Simulate Credential Submission`

- **Node Details:**  

  - **Node Name:** 🪤 Simulate Credential Submission  
  - **Type:** Set Node  
  - **Technical Role:** Adds or modifies data to simulate credential capture (e.g., timestamp, fake credentials)  
  - **Configuration:**  
    - Sets simulated credential fields such as username, password, or submission timestamps  
  - **Expressions/Variables:** May use timestamp functions or randomized data generators  
  - **Input Connections:** From `🎯 Generate Trap Link`  
  - **Output Connections:** To `📄 Append Trap Log`  
  - **Version Requirements:** Set node v3.4 or higher recommended for expression support  
  - **Edge Cases / Failures:**  
    - Lack of input data can cause missing simulated credentials  
    - Expression failures if randomization or date functions misconfigured  
  - **Sub-workflow Reference:** None

---

#### 2.6 Logging

- **Overview:**  
  This block appends the simulated trap interaction data to a Google Sheets log for record-keeping, analysis, and auditing.

- **Nodes Involved:**  
  - `📄 Append Trap Log`

- **Node Details:**  

  - **Node Name:** 📄 Append Trap Log  
  - **Type:** Google Sheets  
  - **Technical Role:** Appends a new row to a specified Google Sheets log sheet  
  - **Configuration:**  
    - Operation: Append row  
    - Spreadsheet ID and Sheet Name: Configured to the trap log sheet  
    - Maps fields like timestamp, target email, trap link, and simulated credentials to columns  
    - Uses Google Sheets OAuth2 credentials for authentication  
  - **Expressions/Variables:** Uses data fields set by previous nodes  
  - **Input Connections:** From `🪤 Simulate Credential Submission`  
  - **Output Connections:** None (end of workflow)  
  - **Version Requirements:** Google Sheets node v4.6 or later recommended  
  - **Edge Cases / Failures:**  
    - Authentication errors if credentials invalid  
    - API rate limits or quota exceeded errors  
    - Data mapping errors if fields mismatch sheet columns  
  - **Sub-workflow Reference:** None

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                        | Input Node(s)          | Output Node(s)                | Sticky Note |
|---------------------------|--------------------|-------------------------------------|-----------------------|------------------------------|-------------|
| Sticky Note               | Sticky Note        | Annotation / Comment                 | None                  | None                         |             |
| ⚡ Trigger TrapSim         | Manual Trigger     | Workflow start trigger               | None                  | 📄 Get Trap Targets           |             |
| 📄 Get Trap Targets       | Google Sheets      | Retrieve list of trap targets        | ⚡ Trigger TrapSim     | 🧹 Filter Valid Targets       |             |
| 🧹 Filter Valid Targets    | If (Conditional)   | Filter out invalid targets           | 📄 Get Trap Targets   | 🎯 Generate Trap Link         |             |
| 🎯 Generate Trap Link      | Set                | Generate personalized trap links    | 🧹 Filter Valid Targets| 🪤 Simulate Credential Submission |             |
| 🪤 Simulate Credential Submission | Set         | Simulate victim credential submission| 🎯 Generate Trap Link  | 📄 Append Trap Log            |             |
| 📄 Append Trap Log        | Google Sheets      | Log simulation data to sheet         | 🪤 Simulate Credential Submission | None                    |             |
| Sticky Note2              | Sticky Note        | Annotation / Comment                 | None                  | None                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `⚡ Trigger TrapSim`  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually. No additional configuration needed.

2. **Create Google Sheets Node to Get Trap Targets**  
   - Name: `📄 Get Trap Targets`  
   - Type: Google Sheets  
   - Operation: Read Rows  
   - Configure:  
     - Enter Spreadsheet ID containing target users.  
     - Select appropriate Sheet Name (e.g., "Targets").  
     - Provide Google Sheets OAuth2 credentials.  
   - Connect output from `⚡ Trigger TrapSim` to this node.

3. **Create If Node to Filter Valid Targets**  
   - Name: `🧹 Filter Valid Targets`  
   - Type: If (Conditional)  
   - Condition: Check that critical fields exist, for example:  
     - Expression: `{{$json["email"] !== undefined && $json["email"] !== ""}}`  
   - Connect input from `📄 Get Trap Targets`.  
   - Connect the "true" output to `🎯 Generate Trap Link`.

4. **Create Set Node to Generate Trap Link**  
   - Name: `🎯 Generate Trap Link`  
   - Type: Set  
   - Purpose: Build personalized trap URL for each valid target.  
   - Configuration:  
     - Add a new field, e.g., `trapLink`, with an expression such as:  
       `https://phishing.example.com/trap?user={{$json["email"]}}`  
   - Connect input from the "true" output of `🧹 Filter Valid Targets`.  
   - Connect output to `🪤 Simulate Credential Submission`.

5. **Create Set Node to Simulate Credential Submission**  
   - Name: `🪤 Simulate Credential Submission`  
   - Type: Set  
   - Purpose: Add simulated credential data such as fake username, password, or timestamp.  
   - Configuration:  
     - Add fields like `submittedUsername` (e.g., same as email), `submittedPassword` (e.g. fixed string or random), `submissionTime` (use expression `{{$now}}`).  
   - Connect input from `🎯 Generate Trap Link`.  
   - Connect output to `📄 Append Trap Log`.

6. **Create Google Sheets Node to Append Trap Log**  
   - Name: `📄 Append Trap Log`  
   - Type: Google Sheets  
   - Operation: Append Row  
   - Configure:  
     - Provide Spreadsheet ID for the log spreadsheet.  
     - Select Sheet Name for logging (e.g., "Trap Log").  
     - Map workflow fields to columns, e.g., email, trapLink, submittedUsername, submissionTime.  
     - Use Google Sheets OAuth2 credentials.  
   - Connect input from `🪤 Simulate Credential Submission`.

7. **Optional: Add Sticky Notes**  
   - Add `Sticky Note` nodes to annotate the workflow for clarity as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                    |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow supports security awareness by simulating phishing credential traps automatically. | Security Awareness / SOC Testing                   |
| Requires Google Sheets OAuth2 credentials with read/write access to the target and log sheets.   | Credential Setup                                   |
| Use meaningful spreadsheet and sheet names to avoid confusion (e.g., "Targets" and "Trap Log").  | Best Practices                                    |
| Ensure your Google Sheets API quota allows frequent reads and writes as this workflow may be run multiple times. | API Quota Management                               |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---