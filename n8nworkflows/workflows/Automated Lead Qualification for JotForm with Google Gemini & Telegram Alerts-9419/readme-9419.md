Automated Lead Qualification for JotForm with Google Gemini & Telegram Alerts

https://n8nworkflows.xyz/workflows/automated-lead-qualification-for-jotform-with-google-gemini---telegram-alerts-9419


# Automated Lead Qualification for JotForm with Google Gemini & Telegram Alerts

### 1. Workflow Overview

This workflow automates lead qualification for contact form submissions received via JotForm. Its main purpose is to classify incoming leads into three categories—Hot Lead, Cold Lead, or Garbage/Spam—using AI-powered analysis through Google Gemini. Based on the classification, it triggers corresponding actions such as sending Telegram notifications, flagging submissions, or deleting spam entries.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new form submissions from JotForm.
- **1.2 Data Extraction and Preparation:** Organizes the raw submission data for AI processing.
- **1.3 AI Lead Classification:** Uses Google Gemini via a Langchain agent to analyze and classify the lead quality.
- **1.4 Decision Routing:** Routes the submission based on AI classification into Hot, Cold, or Spam paths.
- **1.5 Post-Classification Actions:** Sends notifications, flags leads, or deletes spam submissions accordingly.
- **1.6 One-Time Setup for Telegram Chat ID:** Allows setup of Telegram chat ID via a trigger node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new submissions on a specific JotForm form and outputs the raw submission data for further processing.

**Nodes Involved:**  
- Trigger: When Form Submitted

**Node Details:**  
- **Trigger: When Form Submitted**  
  - Type: JotForm Trigger  
  - Role: Webhook trigger for new form submissions  
  - Configuration: Listens to form ID "252791935306463" (specific JotForm form)  
  - Inputs: None (trigger)  
  - Outputs: Raw submission JSON including submissionID and full form data  
  - Edge Cases: Webhook misconfiguration, form ID mismatch, network errors  
  - Notes: Requires valid webhook setup in JotForm and correct webhook URL in n8n

---

#### 1.2 Data Extraction and Preparation

**Overview:**  
Extracts key submission fields and organizes them into variables for AI processing.

**Nodes Involved:**  
- Parse AI Classification

**Node Details:**  
- **Parse AI Classification**  
  - Type: Set node  
  - Role: Extracts submissionID and raw form data into named variables  
  - Configuration: Assigns "submissionID" and "Form_Data" (full JSON) from trigger output  
  - Inputs: Output from "Trigger: When Form Submitted"  
  - Outputs: JSON object with named fields for AI consumption  
  - Edge Cases: Missing fields in submission, unexpected data format  
  - Notes: Prepares data for classification prompt

---

#### 1.3 AI Lead Classification

**Overview:**  
Analyzes the submission data using an AI agent powered by Google Gemini to classify lead quality into Hot, Cold, or Garbage/Spam.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- Classify Lead Quality  

**Node Details:**  
- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini Language Model node  
  - Role: Executes AI chat model for natural language processing  
  - Configuration: Default options, integrated as language model for agent node  
  - Inputs: Text prompt from "Classify Lead Quality" node via Langchain integration  
  - Outputs: AI-generated classification response  

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI output to ensure it matches expected JSON schema  
  - Configuration: JSON schema example expects "lead_classification" (int) and "conclusion" (string)  
  - Inputs: Raw AI output from Google Gemini  
  - Outputs: Validated JSON with classification and explanation  

- **Classify Lead Quality**  
  - Type: Langchain Agent node  
  - Role: Orchestrates the AI prompt to classify lead quality using Google Gemini and parses output  
  - Configuration:  
    - System message describes business context and detailed classification criteria for Hot (1), Cold (0), and Garbage/Spam (2) leads.  
    - Input prompt injects form data as JSON string under "contact us submission data"  
    - Output expected in strict JSON format with classification and conclusion  
  - Inputs: Data from "Parse AI Classification" node  
  - Outputs: Parsed classification result for routing  
  - Edge Cases: AI model errors, invalid JSON output, timeouts, ambiguous classifications  

---

#### 1.4 Decision Routing

**Overview:**  
Routes the classified lead to appropriate action nodes based on the AI classification result.

**Nodes Involved:**  
- Route by Lead Type

**Node Details:**  
- **Route by Lead Type**  
  - Type: Switch node  
  - Role: Directs workflow based on lead_classification value (0, 1, 2)  
  - Configuration:  
    - Condition 1: lead_classification == 0 → Cold Lead branch  
    - Condition 2: lead_classification == 1 → Hot Lead branch  
    - Condition 3: lead_classification == 2 → Garbage/Spam branch  
  - Inputs: Output from "Classify Lead Quality" node  
  - Outputs: Three branches to respective nodes  
  - Edge Cases: Unexpected classification values, missing output field  

---

#### 1.5 Post-Classification Actions

**Overview:**  
Executes follow-up actions based on lead type: ignores cold leads, notifies hot leads, flags submissions, or deletes spam.

**Nodes Involved:**  
- Ignore Cold Lead  
- Notify Hot Lead  
- Flag in JotForm  
- Delete Spam Submission

**Node Details:**  
- **Ignore Cold Lead**  
  - Type: No Operation node  
  - Role: Ends flow for cold leads with no action  
  - Inputs: Cold Lead branch from Switch  
  - Outputs: None  
  - Edge Cases: None  

- **Notify Hot Lead**  
  - Type: Telegram node  
  - Role: Sends Telegram message notification for hot leads  
  - Configuration:  
    - Text includes submission details (Name, Email, Contact no, Website, Message) extracted from "Parse AI Classification" node  
    - Includes AI classification conclusion  
    - Requires Telegram Chat ID and bot credentials  
  - Inputs: Hot Lead branch from Switch  
  - Outputs: Connects to "Flag in JotForm" node  
  - Edge Cases: Telegram API errors, incorrect chat ID, message formatting failures  

- **Flag in JotForm**  
  - Type: HTTP Request node  
  - Role: Flags submission in JotForm system (e.g., marks as important)  
  - Configuration:  
    - POST request to JotForm API endpoint for submission using submissionID  
    - Sets "flag" field to "1"  
    - Uses JotForm API key in headers  
  - Inputs: From "Notify Hot Lead" node  
  - Outputs: None  
  - Edge Cases: API key invalid, network errors, permission issues  

- **Delete Spam Submission**  
  - Type: HTTP Request node  
  - Role: Deletes identified spam submissions from JotForm  
  - Configuration:  
    - DELETE request to JotForm API with submissionID  
    - Uses JotForm API key in headers  
  - Inputs: Garbage/Spam branch from Switch  
  - Outputs: None  
  - Edge Cases: API errors, unauthorized requests, rate limits  

---

#### 1.6 One-Time Setup for Telegram Chat ID

**Overview:**  
Facilitates initial capture of Telegram chat ID for configuring notifications.

**Nodes Involved:**  
- Get Chat ID (One-Time Setup)

**Node Details:**  
- **Get Chat ID (One-Time Setup)**  
  - Type: Telegram Trigger node  
  - Role: Listens for Telegram messages to capture chat ID  
  - Configuration: Listens for "message" updates  
  - Inputs: None (trigger)  
  - Outputs: Telegram message data including chat ID  
  - Edge Cases: Telegram webhook misconfiguration, message permission errors  
  - Notes: Used once to retrieve chat ID for "Notify Hot Lead" node configuration

---

### 3. Summary Table

| Node Name                     | Node Type                               | Functional Role                              | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                                          |
|-------------------------------|---------------------------------------|----------------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Trigger: When Form Submitted   | JotForm Trigger                       | Captures new JotForm submissions             | None                         | Parse AI Classification      |                                                                                                                                      |
| Parse AI Classification        | Set                                  | Extracts submissionID and raw form data      | Trigger: When Form Submitted | Classify Lead Quality        |                                                                                                                                      |
| Google Gemini Chat Model       | Langchain Google Gemini Model         | AI model to analyze lead data                 | Classify Lead Quality (AI LM)| Classify Lead Quality (AI OP)|                                                                                                                                      |
| Structured Output Parser       | Langchain Structured Output Parser    | Parses AI output to JSON schema               | Google Gemini Chat Model      | Classify Lead Quality (main) |                                                                                                                                      |
| Classify Lead Quality          | Langchain Agent                       | Orchestrates AI classification                | Parse AI Classification      | Route by Lead Type           |                                                                                                                                      |
| Route by Lead Type             | Switch                              | Routes leads based on classification          | Classify Lead Quality         | Ignore Cold Lead, Notify Hot Lead, Delete Spam Submission |                                                                                                                                      |
| Ignore Cold Lead               | No Operation                        | Ends flow for cold leads                       | Route by Lead Type            | None                        |                                                                                                                                      |
| Notify Hot Lead               | Telegram                            | Sends Telegram notification for hot leads     | Route by Lead Type            | Flag in JotForm             |                                                                                                                                      |
| Flag in JotForm               | HTTP Request                       | Flags submission in JotForm                    | Notify Hot Lead              | None                        |                                                                                                                                      |
| Delete Spam Submission        | HTTP Request                       | Deletes spam submissions from JotForm         | Route by Lead Type            | None                        |                                                                                                                                      |
| Get Chat ID (One-Time Setup)  | Telegram Trigger                   | Captures Telegram chat ID for setup           | None                         | None                        |                                                                                                                                      |
| Sticky Note2                 | Sticky Note                        | Documentation and overview                     | None                         | None                        | # Smart Lead Qualification for JotForm Contact Forms  ... See full content in node details above                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Parameters: Select form ID "252791935306463"  
   - Webhook: Ensure webhook is activated in JotForm to point to n8n instance  
   - Connect output to next node  

2. **Create Set Node (Parse AI Classification)**  
   - Type: Set  
   - Parameters: Assign variables:  
     - `submissionID` = `{{$json["submissionID"]}}`  
     - `Form_Data` = `{{$json["rawRequest"]}}` (entire form submission JSON)  
   - Connect input from JotForm Trigger  

3. **Create Langchain Google Gemini Chat Model Node**  
   - Type: Langchain Google Gemini Model  
   - Parameters: Use default options  
   - This node is linked internally by Langchain agent node; no direct manual connection required  

4. **Create Langchain Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - Parameters: Provide JSON schema example:  
     ```json
     {
       "lead_classification": 1,
       "conclusion": "Brief explanation of why this is classified as hot/cold lead"
     }
     ```  
   - This node connects internally to Langchain agent node  

5. **Create Langchain Agent Node (Classify Lead Quality)**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text input with prompt:  
       ```
       contact us submission data :-
       {{ $json.Form_Data.toJsonString() }}
       ```  
     - System message to define AI role and classification criteria (as per detailed classification criteria in node)  
     - Enable output parser and select the Structured Output Parser node  
   - Connect input from "Parse AI Classification" set node  
   - Connect AI language model input to Google Gemini Chat Model node  
   - Connect AI output parser input to Structured Output Parser node  

6. **Create Switch Node (Route by Lead Type)**  
   - Type: Switch  
   - Parameters: Create three rules matching `{{$json.output.lead_classification}}` equal to "0", "1", or "2"  
   - Input from "Classify Lead Quality" node  

7. **Create No Operation Node (Ignore Cold Lead)**  
   - Type: NoOp  
   - Input from "Route by Lead Type" first output (cold leads)  

8. **Create Telegram Node (Notify Hot Lead)**  
   - Type: Telegram  
   - Parameters:  
     - Text: construct message with fields from "Parse AI Classification" node, e.g.  
       ```
       Submission data
       Name :- {{ $('Parse AI Classification').item.json.Form_Data.Name || '-' }}
       Email :- {{ $('Parse AI Classification').item.json.Form_Data["E-mail"] || '-' }}
       Contact no :- {{ $('Parse AI Classification').item.json.Form_Data["Contact Number"] || '-' }}
       Website :- {{ $('Parse AI Classification').item.json.Form_Data.Website || '-' }}
       Message :- {{ $('Parse AI Classification').item.json.Form_Data.Message || '-' }}

       Conclusion :-
       {{ $json.output.conclusion }}
       ```  
     - Chat ID: Set your Telegram chat ID  
     - Credentials: Set up Telegram Bot credentials with Bot Token  
   - Connect input from "Route by Lead Type" second output (hot leads)  

9. **Create HTTP Request Node (Flag in JotForm)**  
   - Type: HTTP Request  
   - Parameters:  
     - Method: POST  
     - URL: `https://api.jotform.com/submission/{{ $json.submissionID }}` (replace with dynamic submissionID from trigger)  
     - Body: JSON `{ "flag": "1" }`  
     - Headers: Add header `APIKEY` with your JotForm API key  
   - Connect input from "Notify Hot Lead" node  

10. **Create HTTP Request Node (Delete Spam Submission)**  
    - Type: HTTP Request  
    - Parameters:  
      - Method: DELETE  
      - URL: `https://api.jotform.com/submission/{{ $json.submissionID }}`  
      - Headers: Add header `APIKEY` with your JotForm API key  
    - Connect input from "Route by Lead Type" third output (spam leads)  

11. **Create Telegram Trigger Node (Get Chat ID One-Time Setup)**  
    - Type: Telegram Trigger  
    - Parameters: Set to listen for message updates  
    - This node is standalone and used to capture chat ID for Telegram notification setup  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                        | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow is designed to automate lead qualification specifically for AI and automation services businesses receiving contact form submissions via JotForm. The AI classification logic is embedded in a Langchain agent using Google Gemini to ensure nuanced and context-aware classification. | n8n + Langchain + Google Gemini integration                                                              |
| Telegram notification setup requires bot credentials and chat ID. Use the Telegram Trigger node to capture your chat ID by sending a message to your bot once.                                                                                                                                      | Telegram Bot API documentation: https://core.telegram.org/bots/api                                       |
| JotForm API key is needed for flagging and deleting submissions. Ensure correct permissions and API key security.                                                                                                                                                                                 | JotForm API docs: https://api.jotform.com/docs                                                          |
| The classification schema strictly expects JSON output from AI; malformed outputs may cause downstream failures. Consider adding error handling or fallback nodes if needed.                                                                                                                        | AI output format validation is enforced by Structured Output Parser node                                 |
| Lead classification criteria are detailed in the system message of the AI agent node to tailor AI responses specifically for business needs.                                                                                                                                                     | Tailor the system prompt for different business contexts or languages                                   |
| The sticky note in the workflow provides a comprehensive summary and quick links for external resources, including the JotForm website partner link.                                                                                                                                             | JotForm: https://www.jotform.com/?partner=roshanramanidev                                              |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, an integration and automation platform. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly available.