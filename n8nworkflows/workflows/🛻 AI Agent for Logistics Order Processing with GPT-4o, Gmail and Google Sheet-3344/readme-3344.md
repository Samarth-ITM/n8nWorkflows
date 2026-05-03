🛻 AI Agent for Logistics Order Processing with GPT-4o, Gmail and Google Sheet

https://n8nworkflows.xyz/workflows/---ai-agent-for-logistics-order-processing-with-gpt-4o--gmail-and-google-sheet-3344


# 🛻 AI Agent for Logistics Order Processing with GPT-4o, Gmail and Google Sheet

### 1. Workflow Overview

This workflow automates the processing of inbound logistics orders received via email. It targets logistics or manufacturing operations that receive purchase orders by email and need to extract structured order data to load into a Google Sheet for downstream warehouse processing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Watches a Gmail inbox for new emails with subject lines containing "Inbound Order" to trigger processing.
- **1.2 AI Processing:** Uses an AI Agent powered by OpenAI GPT-4o-mini to parse the email subject and body, extracting structured data including purchase order number, expected delivery date, and multiple order lines (SKU and quantity).
- **1.3 Data Formatting:** Transforms the AI output JSON into individual records suitable for Google Sheets insertion.
- **1.4 Data Storage:** Appends the parsed order lines into a Google Sheet with predefined columns for further logistics handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow whenever a new email arrives in the configured Gmail inbox. It filters emails to only continue processing if the subject contains the phrase "Inbound Order".

- **Nodes Involved:**  
  - Email Received (Gmail Trigger)  
  - Is PO? (If node)

- **Node Details:**

  - **Email Received**  
    - Type: Gmail Trigger  
    - Role: Watches Gmail inbox for new emails.  
    - Configuration: Polls every minute; no filters at trigger level.  
    - Input/Output: No input; outputs full email JSON including subject and body.  
    - Edge Cases: Gmail API auth errors, rate limits, or no new emails.  
    - Notes: Requires Gmail API credentials configured in n8n.

  - **Is PO?**  
    - Type: If  
    - Role: Filters emails to only those with "Inbound Order" in the subject line.  
    - Configuration: Checks if `$json.subject` contains "Inbound Order" (case-sensitive).  
    - Input: Email JSON from Gmail Trigger.  
    - Output: True branch continues processing; false branch stops workflow.  
    - Edge Cases: Emails with missing or malformed subject lines may fail condition.

---

#### 2.2 AI Processing

- **Overview:**  
  This block sends the email subject and body to an AI Agent that uses OpenAI GPT-4o-mini to extract structured order data as JSON.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Coordinates the AI prompt and output parsing.  
    - Configuration:  
      - Input text combines email subject and body.  
      - System message instructs the AI to extract purchase order number, expected delivery date (in ISO format), and order lines (SKU and quantity) strictly as JSON.  
      - Output parser enabled to enforce JSON format.  
    - Input: Email JSON from "Is PO?" node.  
    - Output: Parsed JSON object with fields: `purchase_order`, `expected_delivery_date`, and `lines` (array of SKU/quantity).  
    - Edge Cases: AI may fail to parse malformed emails or unexpected formats; API rate limits or auth errors possible.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini model for AI Agent.  
    - Configuration: Model set to "gpt-4o-mini".  
    - Input/Output: Connected as language model provider to AI Agent.  
    - Edge Cases: API key invalid or quota exceeded.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Validates and parses AI output to JSON schema.  
    - Configuration: Example JSON schema provided to enforce expected fields and structure.  
    - Input/Output: Connected as output parser to AI Agent.  
    - Edge Cases: Parsing fails if AI output is not valid JSON or does not match schema.

---

#### 2.3 Data Formatting

- **Overview:**  
  This block transforms the AI Agent’s JSON output into multiple individual records, each representing one order line with associated purchase order and delivery date, suitable for Google Sheets insertion.

- **Nodes Involved:**  
  - Format Purchase Order Lines (Code node)

- **Node Details:**

  - **Format Purchase Order Lines**  
    - Type: Code (JavaScript)  
    - Role: Maps AI output JSON to an array of objects, each containing `purchase_order`, `expected_delivery_date`, `sku`, and `quantity`.  
    - Configuration:  
      - Extracts `purchase_order`, `expected_delivery_date`, and `lines` from AI output.  
      - Returns one JSON object per order line for downstream processing.  
    - Input: AI Agent output JSON.  
    - Output: Array of JSON objects, one per order line.  
    - Edge Cases: If AI output is missing fields or empty lines array, the code may return empty or error.

---

#### 2.4 Data Storage

- **Overview:**  
  This block appends the formatted order lines into a Google Sheet with columns for PO number, expected delivery date, SKU, and quantity.

- **Nodes Involved:**  
  - Store Purchase Order Lines (Google Sheets node)

- **Node Details:**

  - **Store Purchase Order Lines**  
    - Type: Google Sheets  
    - Role: Appends rows to a Google Sheet.  
    - Configuration:  
      - Document ID set to a specific Google Sheet file.  
      - Sheet name set to the first sheet (`gid=0`).  
      - Columns mapped explicitly:  
        - PO_NUMBER ← purchase_order  
        - EXPECTED_DELIVERY DATE ← expected_delivery_date  
        - SKU_ID ← sku  
        - QUANTITY ← quantity  
      - Operation: Append rows.  
    - Input: Array of order line objects from code node.  
    - Output: Confirmation of rows appended.  
    - Edge Cases: Google API auth errors, sheet not found, column mismatch, or quota limits.

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                         | Input Node(s)        | Output Node(s)             | Sticky Note                                                                                              |
|---------------------------|--------------------------------|---------------------------------------|----------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| Email Received            | Gmail Trigger                  | Triggers workflow on new inbound order emails | None                 | Is PO?                     | ### 1. Workflow Trigger with Gmail Trigger<br>The workflow is triggered by a new email received in your Gmail mailbox. If the subject includes the string "Inbound Order" we proceed, if not we do nothing.<br>Setup Gmail API credentials.<br>[Learn more about the Gmail Trigger Node](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger) |
| Is PO?                   | If                            | Filters emails to those with "Inbound Order" in subject | Email Received       | AI Agent                   |                                                                                                        |
| AI Agent                 | LangChain Agent               | Parses email text with AI to extract structured order data | Is PO?               | Format Purchase Order Lines | ### 2. AI Agent equipped with the query tool<br>Email body and subject sent to AI Agent for parsing PO Number, delivery date, SKU, quantity.<br>Setup OpenAI GPT-4o-mini credentials.<br>[Learn more about the AI Agent Node](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| OpenAI Chat Model        | LangChain OpenAI Chat Model   | Provides GPT-4o-mini model to AI Agent | None (linked internally) | AI Agent                   |                                                                                                        |
| Structured Output Parser | LangChain Structured Output Parser | Validates AI output JSON format       | None (linked internally) | AI Agent                   |                                                                                                        |
| Format Purchase Order Lines | Code                         | Converts AI JSON output to array of order line objects | AI Agent             | Store Purchase Order Lines  |                                                                                                        |
| Store Purchase Order Lines | Google Sheets                 | Appends order lines to Google Sheet   | Format Purchase Order Lines | None                      | ### 3. Store the orderlines in a Google Sheet<br>Configure Google Sheets API credentials.<br>Select file and sheet.<br>Create columns: PO_NUMBER, EXPECTED_DELIVERY DATE, SKU_ID, QUANTITY.<br>[Learn more about the Google Sheet Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Sticky Note               | Sticky Note                   | Provides test email example            | None                 | None                       | ### Test the workflow with this email!<br>Send the example email to your Gmail box and test workflow. |
| Sticky Note1              | Sticky Note                   | Explains Gmail Trigger setup           | None                 | None                       | ### 1. Workflow Trigger with Gmail Trigger<br>Setup instructions and link.                             |
| Sticky Note2              | Sticky Note                   | Explains AI Agent setup                 | None                 | None                       | ### 2. AI Agent equipped with the query tool<br>Setup instructions and link.                          |
| Sticky Note3              | Sticky Note                   | Provides tutorial link for more details | None                 | None                       | ### 4. Do you need more details?<br>[🎥 Watch My Tutorial](https://www.youtube.com/watch?v=kQ8dO_30SB0) |
| Sticky Note4              | Sticky Note                   | Explains Google Sheets node setup      | None                 | None                       | ### 3. Store the orderlines in a Google Sheet<br>Setup instructions and link.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Credentials: Configure Gmail API OAuth2 credentials.  
   - Settings: Poll every minute.  
   - No filters at trigger level.

2. **Add If Node "Is PO?"**  
   - Condition: Check if `{{$json["subject"]}}` contains the string "Inbound Order" (case-sensitive).  
   - Connect Gmail Trigger output to this node input.

3. **Add LangChain OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Credentials: Configure OpenAI API key with access to GPT-4o-mini.  
   - Model: Select "gpt-4o-mini".

4. **Add LangChain Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - Configuration: Provide example JSON schema with fields:  
     ```json
     {
       "purchase_order": "PO45231",
       "expected_delivery_date": "2025-03-27",
       "lines": [
         { "sku": "HERM-SHOE-001", "quantity": 120 }
       ]
     }
     ```

5. **Add LangChain Agent Node "AI Agent"**  
   - Type: LangChain Agent  
   - Input: Set text to:  
     ```
     Email Subject: {{$json.subject}}
     Email Body: 
     {{$json.text}}
     ```  
   - System Message:  
     ```
     You are an assistant that processes emails related to inbound orders from Hermas.

     Each email has the subject line containing a purchase order reference (e.g., "PO45231").

     In the email body, you will find:

     An expected delivery date, typically in formats like 27/03/2025 or 2025-03-27.

     One or more order lines, where each line contains:

     An SKU (e.g., HERM-SHOE-001)

     A quantity (e.g., 120)

     Your goal is to extract the following fields:

     purchase_order: The PO number from the subject line (e.g., PO45231)

     expected_delivery_date: In ISO format (e.g., 2025-03-27)

     lines: A list of objects with sku and quantity for each order line

     Return your output strictly as a valid JSON object using the format below.
     ```  
   - Connect the OpenAI Chat Model node as the language model.  
   - Connect the Structured Output Parser node as the output parser.

6. **Connect "Is PO?" True Output to "AI Agent" Input**

7. **Add Code Node "Format Purchase Order Lines"**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const {purchase_order, expected_delivery_date, lines} = $input.first().json.output;

     return lines.map(line => ({
       json: {
         purchase_order,
         expected_delivery_date,
         sku: line.sku,
         quantity: line.quantity
       }
     }));
     ```  
   - Connect AI Agent output to this node.

8. **Add Google Sheets Node "Store Purchase Order Lines"**  
   - Type: Google Sheets  
   - Credentials: Configure Google Sheets API credentials.  
   - Operation: Append rows.  
   - Document ID: Set to your Google Sheet file ID.  
   - Sheet Name: Set to the target sheet (e.g., `gid=0`).  
   - Columns Mapping:  
     - PO_NUMBER ← `{{$json.purchase_order}}`  
     - EXPECTED_DELIVERY DATE ← `{{$json.expected_delivery_date}}`  
     - SKU_ID ← `{{$json.sku}}`  
     - QUANTITY ← `{{$json.quantity}}`  
   - Connect Code node output to this node.

9. **Connect "Format Purchase Order Lines" output to "Store Purchase Order Lines" input**

10. **Optional: Add Sticky Notes**  
    - Add notes for setup instructions, testing email example, and tutorial links as per user preference.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow designed by Samir, Supply Chain Data Scientist and founder of LogiGreen Consulting.                   | [LogiGreen Consulting](https://logi-green.com)                                                  |
| Workflow uses n8n version 1.82.1.                                                                              |                                                                                                |
| Tutorial video explaining workflow construction and usage.                                                    | [🎥 Step-by-Step Tutorial](https://www.youtube.com/watch?v=kQ8dO_30SB0)                         |
| Example inbound order email format included for testing.                                                     | See Sticky Note with example email content in workflow.                                        |
| LinkedIn contact for further logistics automation discussions.                                                | [Samir Saci LinkedIn](https://www.linkedin.com/in/samir-saci)                                  |
| Gmail Trigger Node documentation.                                                                              | [Gmail Trigger Node Docs](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger) |
| AI Agent Node documentation.                                                                                   | [AI Agent Node Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Google Sheets Node documentation.                                                                               | [Google Sheets Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |

---

This document provides a complete, structured reference to understand, reproduce, and maintain the "AI Agent for Logistics Order Processing" workflow, including all nodes, configurations, and integration points.