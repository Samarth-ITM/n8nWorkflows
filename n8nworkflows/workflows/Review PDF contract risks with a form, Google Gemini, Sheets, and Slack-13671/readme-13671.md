Review PDF contract risks with a form, Google Gemini, Sheets, and Slack

https://n8nworkflows.xyz/workflows/review-pdf-contract-risks-with-a-form--google-gemini--sheets--and-slack-13671


# Review PDF contract risks with a form, Google Gemini, Sheets, and Slack

# Workflow Analysis: Review PDF contract risks with a form, Google Gemini, Sheets, and Slack

## 1. Workflow Overview

This workflow provides an automated end-to-end solution for legal document auditing. It allows users to upload PDF contracts via a web form, uses AI to extract and analyze the legal text for potential risks (such as unfavorable terms or missing protections), logs the findings into a centralized tracking system, and sends urgent notifications for high-risk documents.

The logic is divided into the following functional blocks:
- **1.1 Input Reception:** Captures the PDF file and metadata (type, notes) via an n8n Form.
- **1.2 PDF Processing:** Converts the binary PDF data into a text format readable by AI models.
- **1.3 AI Analysis:** Utilizes Google Gemini to perform a deep legal review based on a structured prompt.
- **1.4 Data Parsing:** Standardizes the AI's JSON output and calculates internal risk scores.
- **1.5 Logging & Alerting:** Records all data in Google Sheets and triggers a Slack alert if the risk score exceeds a specific threshold.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception
- **Overview:** Serves as the user interface for the workflow, collecting the contract and context.
- **Nodes Involved:** `Upload Contract Form`.
- **Node Details:**
    - **Type:** Form Trigger.
    - **Configuration:** Title "Contract Review Upload". Fields include a required file upload (.pdf), a dropdown for "Contract Type" (NDA, Service Agreement, etc.), and a textarea for "Notes".
    - **Output:** Binary file data and JSON metadata (type and notes).

### 2.2 PDF Processing
- **Overview:** Transforms the raw binary file into strings that can be processed by the LLM.
- **Nodes Involved:** `Extract PDF Text`.
- **Node Details:**
    - **Type:** Code Node (JavaScript).
    - **Technical Role:** Manually parses PDF binary stream markers to extract text content.
    - **Key Expressions:** Uses `Buffer.from(pdfBase64, 'base64')` and Regex to isolate text within stream parentheses.
    - **Edge Cases:** If extraction results in less than 50 characters, it returns a fallback message to inform the AI that data may be minimal.

### 2.3 AI Analysis
- **Overview:** The "brain" of the workflow where the legal assessment happens.
- **Nodes Involved:** `Analyze Contract`, `Gemini Chat Model`.
- **Node Details:**
    - **Analyze Contract (Chain):** Uses a legal analyst persona. The prompt instructs the model to identify auto-renewal traps, liability caps, and IP ownership. It strictly requests a JSON response.
    - **Gemini Chat Model:** Configured with `gemini-1.5-flash` and a temperature of `0.2` to ensure consistent, non-creative, and factual analysis.
    - **Potential Failure:** Model hallucinations or refusal to answer if the PDF content is flagged by safety filters.

### 2.4 Data Parsing
- **Overview:** Cleans the AI's response to ensure it is a valid object for downstream nodes.
- **Nodes Involved:** `Parse Risk Results`.
- **Node Details:**
    - **Type:** Code Node (JavaScript).
    - **Configuration:** Uses Regex to find JSON blocks within the AI's text response.
    - **Key Logic:** Calculates `isHighRisk` (boolean) if the `overallRiskScore` is greater than 7. It also counts the number of high-risk clauses found.
    - **Edge Cases:** Includes a `try-catch` block that provides a default score of 5 if the AI's response is malformed.

### 2.5 Logging and Alerts
- **Overview:** Handles data persistence and urgent communication.
- **Nodes Involved:** `Log Review to Sheets`, `Check Risk Level`, `Send Risk Alert`.
- **Node Details:**
    - **Log Review to Sheets:** Appends or updates a row in a Google Sheet called "Contract Reviews".
    - **Check Risk Level (IF):** A logic gate that checks if `overallRiskScore > 7`.
    - **Send Risk Alert (Slack):** Sends a formatted message to a specific channel including the summary, risk score, and missing protections.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Upload Contract Form | Form Trigger | Entry Point | - | Extract PDF Text | Contract Review Workflow |
| Extract PDF Text | Code | Data Conversion | Upload Contract Form | Analyze Contract | PDF processing: Extracts raw text from the uploaded PDF for AI analysis. |
| Analyze Contract | Chain LLM | AI Orchestration | Extract PDF Text | Parse Risk Results | AI risk analysis: Gemini reviews each clause and assigns a risk score from 1-10. |
| Gemini Chat Model | Gemini Chat Model | AI Engine | - | Analyze Contract | AI risk analysis: Gemini reviews each clause and assigns a risk score from 1-10. |
| Parse Risk Results | Code | Data Structuring | Analyze Contract | Log Review to Sheets | AI risk analysis: Gemini reviews each clause and assigns a risk score from 1-10. |
| Log Review to Sheets | Google Sheets | Data Storage | Parse Risk Results | Check Risk Level | Logging and alerts: All reviews go to Sheets. High-risk contracts get a Slack notification. |
| Check Risk Level | IF | Conditional Logic | Log Review to Sheets | Send Risk Alert | Logging and alerts: All reviews go to Sheets. High-risk contracts get a Slack notification. |
| Send Risk Alert | Slack | Communication | Check Risk Level | - | Logging and alerts: All reviews go to Sheets. High-risk contracts get a Slack notification. |

---

## 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Form Trigger** node. Add three fields: `Contract PDF` (File), `Contract Type` (Dropdown), and `Notes` (Textarea).
2.  **PDF Parsing:** Connect a **Code Node**. Use JavaScript to extract the binary data from the `$input` and convert it to a string. (Note: For better results in production, consider using a dedicated "Read Binary File" or "PDF" node if available in your n8n version).
3.  **AI Integration:**
    - Add a **Basic LLM Chain** node. 
    - Attach a **Google Gemini Chat Model** node to it.
    - Set the Model to `gemini-1.5-flash` and Temperature to `0.2`.
    - In the Chain's prompt, define the JSON schema required (overall\_risk, risk\_score, key\_clauses, etc.).
4.  **Data Processing:** Connect another **Code Node** to parse the LLM's string output into a JSON object. Ensure you add logic to calculate if a contract is "High Risk" based on the numerical score.
5.  **Storage Setup:** Add a **Google Sheets** node. Configure it to `Append or Update`. You will need to create a spreadsheet with headers matching your JSON keys (e.g., summary, risk\_score, type).
6.  **Logic Gate:** Add an **IF** node. Set the condition to check if the risk score from the previous step is greater than 7.
7.  **Notification Setup:** Connect a **Slack** node to the "True" branch of the IF node. Use expressions to map the contract summary and score into a readable Slack message.
8.  **Credentials:** 
    - Configure **Google Gemini API** (via Google AI Studio).
    - Configure **Google Sheets OAuth2**.
    - Configure **Slack OAuth2** or Webhook.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Set up Google Gemini API credentials (free tier works) | [Google AI Studio](https://aistudio.google.com/) |
| Edit the prompt in the "Analyze Contract" node to focus on specific clause types | AI Customization |
| Share the form URL with your legal team | Production Deployment |
| Adjust the risk threshold in the "Check Risk Level" node (default: score > 7) | Threshold Calibration |