Generate & Send Spare Parts Price Quotes with Gmail, Sheets and Gemini AI

https://n8nworkflows.xyz/workflows/generate---send-spare-parts-price-quotes-with-gmail--sheets-and-gemini-ai-5492


# Generate & Send Spare Parts Price Quotes with Gmail, Sheets and Gemini AI

---

### 1. Workflow Overview

This workflow automates the process of generating and sending price quotes for spare parts requested by customers via email. It continuously monitors a Gmail inbox for incoming emails containing keywords related to spare parts in multiple languages, extracts relevant project or part information, queries Google Sheets to retrieve customer data, bill of materials, and pricing, and then uses an AI agent powered by Google Gemini to generate a complete, professional HTML quote in the same language as the request. Finally, the workflow replies to the email with the generated quote and marks the original email as read to maintain inbox hygiene.

**Target Use Cases:**
- Companies handling spare parts inquiries via email.
- Automated quote generation without manual intervention.
- Multilingual customer support with AI-driven content creation.
- Integration with Google Sheets for dynamic data retrieval.
- Use of Google Gemini AI for natural language understanding and HTML quote generation.

**Logical Blocks:**

- **1.1 Initialization & Configuration Notes:** Setup instructions and documentation stored in sticky notes for user guidance.
- **1.2 Email Monitoring & Keyword Detection:** Scheduled trigger to poll Gmail, fetch latest emails, and detect spare parts inquiries based on keywords.
- **1.3 Data Retrieval from Google Sheets:** Fetch customer info, bill of materials, and pricing data from configured Google Sheets.
- **1.4 AI Quote Generation:** Use Langchain AI Agent with Google Gemini and calculator tool to generate a fully formatted HTML quote.
- **1.5 Email Reply & Inbox Management:** Send the generated quote as a reply and mark the original email as read.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Configuration Notes

**Overview:**  
This block consists of sticky notes that provide users with necessary setup instructions, required Google Sheets structure, keyword detection details, and reminders to replace placeholder sheet IDs with actual document IDs.

**Nodes Involved:**  
- Welcome & Overview (Sticky Note)  
- Google Sheets Structure (Sticky Note)  
- Configuration Steps (Sticky Note)  
- Keyword Detection Info (Sticky Note)  
- Email Reply Flow (Sticky Note)  
- IMPORTANT: Replace Sheet IDs (Sticky Note)

**Node Details:**

- **Welcome & Overview**  
  - Type: Sticky Note  
  - Role: Introduces workflow purpose, outlines automated steps, and setup requirements.  
  - Key Info: Lists Gmail OAuth2, Google Sheets, Google Gemini API setup, and language support details.  
  - No inputs or outputs.  
  - Edge Cases: None.

- **Google Sheets Structure**  
  - Type: Sticky Note  
  - Role: Details the required columns and structure for three Google Sheets: CRM Data, Bill of Materials, and Pricing Data.  
  - Key Info: Specifies column names like `Email`, `ProjectCode`, `PartCode`, `Quantity`, `UnitPriceEUR`.  
  - No inputs or outputs.  
  - Edge Cases: Incorrect sheet structure could cause data retrieval failures.

- **Configuration Steps**  
  - Type: Sticky Note  
  - Role: Step-by-step instructions for OAuth2 credential creation and API key setup for Gmail, Sheets, and Google Gemini.  
  - Key Info: Emphasizes configuring credentials in n8n and updating sheet IDs.  
  - No inputs or outputs.  
  - Edge Cases: Misconfigured credentials will cause authentication errors.

- **Keyword Detection Info**  
  - Type: Sticky Note  
  - Role: Lists keywords in Turkish, German, and English to identify spare parts inquiries.  
  - Key Info: Example keywords “yedek parça”, “Ersatzteil”, “spare parts”.  
  - No inputs or outputs.  
  - Edge Cases: Missing keywords might cause missed inquiries.

- **Email Reply Flow**  
  - Type: Sticky Note  
  - Role: Explains the email reply and marking as read process.  
  - Key Info: Gmail node sends reply, Gmail2 node marks original email as read.  
  - No inputs or outputs.  
  - Edge Cases: Failure to mark as read could clutter inbox.

- **IMPORTANT: Replace Sheet IDs**  
  - Type: Sticky Note  
  - Role: Reminds user to replace placeholder Google Sheet IDs with actual IDs found in sheet URLs.  
  - Key Info: Exact placeholders to replace in CRM, BoM, and Price nodes.  
  - No inputs or outputs.  
  - Edge Cases: Using placeholder IDs leads to data retrieval failure.

---

#### 2.2 Email Monitoring & Keyword Detection

**Overview:**  
This block polls Gmail every minute to fetch the latest email and checks if it contains spare parts keywords in supported languages. If keywords are detected, the workflow proceeds to generate a quote.

**Nodes Involved:**  
- Schedule Trigger  
- Gmail - Get Latest Email  
- Check Spare Parts Keywords

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow every 1 minute to check for new emails.  
  - Configuration: Minute interval set to 1.  
  - Inputs: None  
  - Outputs: Triggers Gmail node.  
  - Edge Cases: If the interval is too short, it might trigger API rate limits.

- **Gmail - Get Latest Email**  
  - Type: Gmail Node  
  - Role: Retrieves the latest email received.  
  - Configuration: Limit = 1, operation = getAll (gets full email data).  
  - Credentials: Gmail OAuth2 with required scopes.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: Email JSON data.  
  - Edge Cases: Gmail API errors, authentication failure, no new email.

- **Check Spare Parts Keywords**  
  - Type: If Node  
  - Role: Checks if the email body/snippet/html contains any of the keywords (“yedek parça”, “Ersatzteil”, “spare parts”, “Ersatzteile”).  
  - Configuration: Case sensitive contains checks on `$json.snippet || $json.text || $json.html || ''`.  
  - Inputs: Email JSON from Gmail node.  
  - Outputs: Proceed if any keyword matches; otherwise, workflow ends.  
  - Edge Cases: False negatives if email uses different terminology or formatting; false positives if keywords in unrelated context.

---

#### 2.3 Data Retrieval from Google Sheets

**Overview:**  
This block fetches customer, bill of materials, and pricing data from respective Google Sheets based on email input. These data are used by the AI agent to build the quote.

**Nodes Involved:**  
- CRM - Customer Data  
- BoM - Bill of Materials  
- Price - Pricing Data  
- Calculator Tool

**Node Details:**

- **CRM - Customer Data**  
  - Type: Google Sheets Tool  
  - Role: Retrieves customer information by searching the sheet using customer email or project code.  
  - Configuration: Reads from "Sheet1" in the sheet identified by `YOUR_CRM_SHEET_ID_HERE`.  
  - Inputs: AI agent tool using email or project code as key.  
  - Outputs: Customer data JSON.  
  - Edge Cases: Missing or incorrect email/project code results in no data; incorrect sheet ID causes failure.  
  - Version: 4.6

- **BoM - Bill of Materials**  
  - Type: Google Sheets Tool  
  - Role: Retrieves list of parts for a given project code.  
  - Configuration: Reads from "Sheet1" in the sheet identified by `YOUR_BOM_SHEET_ID_HERE`.  
  - Inputs: Project code from AI agent.  
  - Outputs: List of parts JSON.  
  - Edge Cases: Missing project code or not found in sheet causes empty result; incorrect sheet ID leads to failure.  
  - Version: 4.6

- **Price - Pricing Data**  
  - Type: Google Sheets Tool  
  - Role: Retrieves unit price for a single part code.  
  - Configuration: Reads from "Sheet1" in the sheet identified by `YOUR_PRICE_SHEET_ID_HERE`.  
  - Inputs: Part code(s) from AI agent (called for each part).  
  - Outputs: Pricing data JSON.  
  - Edge Cases: Missing part code or not found causes no price; incorrect sheet ID causes failure.  
  - Version: 4.6

- **Calculator Tool**  
  - Type: Langchain Calculator Tool  
  - Role: Performs mathematical calculations such as subtotal (Quantity × Unit Price) and grand total sums.  
  - Inputs: Called internally by AI agent for calculations during quote assembly.  
  - Outputs: Calculation results.  
  - Edge Cases: Calculation errors or invalid inputs could cause wrong totals.  
  - Version: 1

---

#### 2.4 AI Quote Generation

**Overview:**  
The core AI agent processes the incoming email, extracts project or part codes, fetches data from sheets via tools, calculates prices, and generates a complete HTML quote in the sender’s language using the Google Gemini chat model.

**Nodes Involved:**  
- AI Agent - Quote Generator  
- Google Gemini Chat Model

**Node Details:**

- **AI Agent - Quote Generator**  
  - Type: Langchain Agent Node  
  - Role: Main orchestrator that uses natural language understanding to process email, calls Google Sheets tools for data, uses calculator tool for math, and composes the final HTML quote.  
  - Configuration Highlights:  
    - Input text constructed dynamically from the latest email’s subject, body snippet, and schedule trigger date.  
    - System message enforces strict rules: detect language first, respond in same language, output full HTML document using a detailed template, do not translate data from sheets, calculate totals accurately, and request real project or part codes if missing.  
    - Max iterations set to 100 to allow complex reasoning.  
  - Inputs: Email data, Schedule Trigger date, Google Sheets tools (CRM, BoM, Price), Calculator tool, Google Gemini language model.  
  - Outputs: Generated HTML quote text.  
  - Edge Cases:  
    - Missing or invalid project/part codes leads to prompt asking for correct codes instead of quote.  
    - AI model errors or timeouts.  
    - Incorrect data retrieval from sheets causes incomplete quotes.  
  - Version: 2

- **Google Gemini Chat Model**  
  - Type: Langchain Language Model Node for Google Gemini  
  - Role: Provides AI language model capabilities to the agent.  
  - Configuration: Model "models/gemini-2.0-flash-exp", temperature 0.1 for low randomness, max output tokens very high to allow full document generation.  
  - Inputs: AI agent language model calls.  
  - Outputs: Text completions for quote generation.  
  - Edge Cases: API key or quota errors, model latency.  
  - Version: 1

---

#### 2.5 Email Reply & Inbox Management

**Overview:**  
This block sends the AI-generated HTML quote as a reply to the original email and marks the original email as read to maintain inbox order.

**Nodes Involved:**  
- Gmail - Send Reply  
- Gmail - Mark as Read

**Node Details:**

- **Gmail - Send Reply**  
  - Type: Gmail Node  
  - Role: Sends the AI-generated HTML quote as a reply to the latest email.  
  - Configuration:  
    - Message content set to AI agent output (`$json.output`).  
    - Reply operation referencing original message ID from “Gmail - Get Latest Email”.  
  - Credentials: Gmail OAuth2 with send and read scopes.  
  - Inputs: AI agent output (HTML quote).  
  - Outputs: Triggers mark as read node.  
  - Edge Cases: Send failures, invalid message ID, authentication errors.  
  - Version: 2.1

- **Gmail - Mark as Read**  
  - Type: Gmail Node  
  - Role: Marks the original email as read after sending the reply.  
  - Configuration:  
    - Operation: markAsRead.  
    - Message ID obtained from “Gmail - Get Latest Email”.  
  - Credentials: Same Gmail OAuth2 credentials.  
  - Inputs: Triggered after reply sent.  
  - Outputs: End of workflow.  
  - Edge Cases: Failure to mark as read, Gmail API quota limits.  
  - Version: 2.1

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                               | Input Node(s)                  | Output Node(s)                  | Sticky Note                                              |
|---------------------------|---------------------------------|-----------------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------|
| Welcome & Overview        | Sticky Note                     | Workflow introduction and setup overview      | None                          | None                           | Overview of workflow purpose and setup instructions       |
| Google Sheets Structure   | Sticky Note                     | Details required Google Sheets structure      | None                          | None                           | Required columns and sheets for CRM, BoM, Pricing data    |
| Configuration Steps       | Sticky Note                     | Instructions for OAuth2 and API setup         | None                          | None                           | Step-by-step credential and API configuration            |
| Keyword Detection Info    | Sticky Note                     | Lists keywords for detecting spare parts      | None                          | None                           | Keywords in Turkish, German, and English                  |
| Email Reply Flow          | Sticky Note                     | Explains email reply and inbox management     | None                          | None                           | Gmail nodes send reply and mark as read                   |
| IMPORTANT: Replace Sheet IDs | Sticky Note                   | Reminds to replace placeholder Sheet IDs      | None                          | None                           | Replace placeholders with actual Google Sheet IDs        |
| Schedule Trigger          | Schedule Trigger                | Triggers workflow every 1 minute               | None                          | Gmail - Get Latest Email       |                                                          |
| Gmail - Get Latest Email  | Gmail                          | Fetches latest email from Gmail inbox          | Schedule Trigger              | Check Spare Parts Keywords     |                                                          |
| Check Spare Parts Keywords| If                             | Filters emails containing spare parts keywords | Gmail - Get Latest Email       | AI Agent - Quote Generator     | Keywords detection as per sticky note                      |
| AI Agent - Quote Generator| Langchain Agent                | Generates HTML quote using AI and data tools  | Check Spare Parts Keywords, Google Sheets Tools, Calculator Tool, Gemini Model | Gmail - Send Reply           | Core AI quote generation with strict rules                |
| Google Gemini Chat Model  | Langchain Language Model       | Provides AI language model for quote generation | AI Agent - Quote Generator (ai_languageModel) | AI Agent - Quote Generator (ai_tool) |                                                          |
| CRM - Customer Data       | Google Sheets Tool              | Retrieves customer data by email or project code | AI Agent - Quote Generator (ai_tool) | AI Agent - Quote Generator (ai_tool) | Uses user-provided CRM sheet ID                            |
| BoM - Bill of Materials   | Google Sheets Tool              | Retrieves parts list for project code          | AI Agent - Quote Generator (ai_tool) | AI Agent - Quote Generator (ai_tool) | Uses user-provided BoM sheet ID                            |
| Price - Pricing Data      | Google Sheets Tool              | Retrieves unit price for each part              | AI Agent - Quote Generator (ai_tool) | AI Agent - Quote Generator (ai_tool) | Uses user-provided Price sheet ID                          |
| Calculator Tool           | Langchain Calculator Tool       | Performs pricing calculations                   | AI Agent - Quote Generator (ai_tool) | AI Agent - Quote Generator (ai_tool) | Used for subtotal and grand total calculations            |
| Gmail - Send Reply        | Gmail                          | Sends AI-generated HTML quote as email reply  | AI Agent - Quote Generator    | Gmail - Mark as Read            | Sends reply to original email                             |
| Gmail - Mark as Read      | Gmail                          | Marks original email as read                     | Gmail - Send Reply            | None                           | Keeps inbox clean                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**

   - Add sticky notes named:
     - "Welcome & Overview" with workflow purpose and setup instructions.
     - "Google Sheets Structure" describing required sheet columns.
     - "Configuration Steps" with OAuth2 and API setup steps.
     - "Keyword Detection Info" listing spare parts keywords.
     - "Email Reply Flow" explaining email reply and mark as read process.
     - "IMPORTANT: Replace Sheet IDs" reminding to update sheet IDs.

2. **Create Schedule Trigger Node:**

   - Type: Schedule Trigger
   - Settings: Set to trigger every 1 minute (interval: 1 minute).
   - No credentials needed.

3. **Create Gmail - Get Latest Email Node:**

   - Type: Gmail
   - Operation: Get all emails
   - Limit: 1 (fetch latest email)
   - Credentials: Configure with Gmail OAuth2 credentials (with Gmail API read scopes).
   - Connect Schedule Trigger output → Gmail - Get Latest Email input.

4. **Create Check Spare Parts Keywords Node:**

   - Type: If Node
   - Set conditions to check if email content (`snippet || text || html`) contains any of the keywords:  
     - "yedek parça" (Turkish)  
     - "Ersatzteil", "Ersatzteile" (German)  
     - "spare parts" (English)  
   - Case sensitive: true
   - Connect Gmail - Get Latest Email main output → Check Spare Parts Keywords input.

5. **Create Google Sheets Tool Nodes:**

   - CRM - Customer Data:
     - Type: Google Sheets Tool
     - Sheet Name: "Sheet1"
     - Document ID: Replace placeholder with actual CRM sheet ID
     - Configuration: Search by `Email` or `ProjectCode` columns.
     - Version: Use latest supported version (4.6).

   - BoM - Bill of Materials:
     - Type: Google Sheets Tool
     - Sheet Name: "Sheet1"
     - Document ID: Replace with actual BoM sheet ID.
     - Configuration: Search by `ProjectCode`.

   - Price - Pricing Data:
     - Type: Google Sheets Tool
     - Sheet Name: "Sheet1"
     - Document ID: Replace with actual Price sheet ID.
     - Configuration: Search by `PartCode`, returns `UnitPriceEUR`.

6. **Create Calculator Tool Node:**

   - Type: Langchain Calculator Tool
   - No special configuration needed.
   - Used for calculations like subtotal and grand total.

7. **Create Google Gemini Chat Model Node:**

   - Type: Langchain LM Chat Google Gemini
   - Model Name: "models/gemini-2.0-flash-exp"
   - Temperature: 0.1
   - Max Output Tokens: Set high enough (~500000) for full HTML document generation.
   - Configure API key credential.

8. **Create AI Agent - Quote Generator Node:**

   - Type: Langchain Agent Node
   - Text Input: Use dynamic expression combining latest email To, Subject, snippet, and current date.
   - System Message: Insert detailed instructions to detect language, respond in the same language, produce full HTML quote using provided template, calculate totals, handle missing codes appropriately.
   - Tools: Add CRM, BoM, Price Google Sheets tools and Calculator tool.
   - Language Model: Set to Google Gemini Chat Model node.
   - Max iterations: 100
   - Connect Check Spare Parts Keywords "true" output → AI Agent input.
   - Connect CRM, BoM, Price, Calculator, and Gemini model nodes as tool and languageModel inputs to AI Agent.

9. **Create Gmail - Send Reply Node:**

   - Type: Gmail
   - Operation: Reply
   - Message: Set to AI Agent output (`{{$json.output}}`).
   - Message ID: Use ID of latest email from "Gmail - Get Latest Email".
   - Credentials: Gmail OAuth2 (send scopes).
   - Connect AI Agent main output → Gmail - Send Reply input.

10. **Create Gmail - Mark as Read Node:**

    - Type: Gmail
    - Operation: Mark as Read
    - Message ID: Use ID of latest email from "Gmail - Get Latest Email".
    - Credentials: Same as above.
    - Connect Gmail - Send Reply main output → Gmail - Mark as Read input.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The AI agent strictly follows an HTML template for quotes to ensure professional and consistent formatting.                             | Found in AI Agent - Quote Generator system message.                                            |
| Gmail and Google Sheets credentials require OAuth2 setup with proper API scopes for read/write operations.                              | See Configuration Steps sticky note.                                                           |
| Sheet IDs must be replaced with actual Google Sheet document IDs from URLs to enable data retrieval.                                     | See IMPORTANT: Replace Sheet IDs sticky note.                                                  |
| The AI supports multiple languages (Turkish, English, German) and responds in the detected language automatically.                      | See Welcome & Overview sticky note.                                                            |
| Keywords can be extended to support additional languages or terms by modifying the "Check Spare Parts Keywords" node conditions.        | See Keyword Detection Info sticky note.                                                        |
| The workflow polls Gmail every minute; adjust Schedule Trigger interval as needed considering API limits and business requirements.     | Adjust Schedule Trigger node settings accordingly.                                             |
| For comprehensive understanding of Langchain nodes and Google Gemini integration in n8n, refer to the n8n documentation and Google AI Studio. | n8n Docs: https://docs.n8n.io/nodes/ | Google AI Studio: https://ai.google/studio |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---