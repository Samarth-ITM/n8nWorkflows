Generate Personalized Sales Emails with MadKudu Research & OpenAI for Outreach.io Sequences

https://n8nworkflows.xyz/workflows/generate-personalized-sales-emails-with-madkudu-research---openai-for-outreach-io-sequences-7149


# Generate Personalized Sales Emails with MadKudu Research & OpenAI for Outreach.io Sequences

---

## 1. Workflow Overview

This workflow automates the generation and synchronization of personalized sales outreach emails using MadKudu research data and OpenAI's language models, integrating directly with Outreach.io CRM sequences for sales engagement.

**Target Use Cases:**  
- Sales Development Representatives (SDRs), Business Development Representatives (BDRs), and sales teams aiming to scale personalized email outreach efficiently.  
- Automating prospect research, email drafting, CRM updating, and sequence enrollment with minimal manual intervention.

**Logical Blocks:**

- **1.1 Input Reception & Email Extraction**  
  Receives a prospect’s email address via chat trigger and extracts/validates it.

- **1.2 Account & Contact Research**  
  Uses MadKudu MCP and OpenAI models to research the prospect’s company and contact information, generating a comprehensive account brief.

- **1.3 Personalized Email Generation**  
  Produces five tailored email drafts based on research angles, selecting the best one for outreach.

- **1.4 Outreach.io Prospect Verification & Sync**  
  Checks if the prospect already exists in Outreach.io; updates existing prospect or creates a new one, syncing the personalized email content into a custom field.

- **1.5 Sequence Enrollment & Confirmation**  
  Adds the prospect to a predefined Outreach.io email sequence, waits for enrollment confirmation, and handles success or no-operation paths.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Email Extraction

**Overview:**  
This initial block captures the prospect’s email address through a chat trigger, then extracts and validates the email for use downstream.

**Nodes Involved:**  
- When chat message received  
- OpenAI Chat Model  
- Information Extractor

**Node Details:**  

- **When chat message received**  
  - Type: Langchain chatTrigger (Webhook)  
  - Role: Entry point for workflow; triggers on incoming chat messages containing prospect information  
  - Config: Default webhook, no special conditions  
  - Inputs: External chat message (expects email address)  
  - Outputs: Raw chat input JSON  
  - Failures: Missing or malformed chat input may block extraction

- **OpenAI Chat Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Processes chat input text for downstream extraction  
  - Config: GPT-4.1-mini model selected for advanced natural language understanding  
  - Inputs: Raw chat input text  
  - Outputs: Processed text for extraction  
  - Credentials: Requires valid OpenAI API key  
  - Failures: API timeouts, invalid API keys, or rate limits

- **Information Extractor**  
  - Type: Langchain Information Extractor  
  - Role: Extracts the prospect’s email address from the processed chat input  
  - Config: Attribute defined as `email` (required)  
  - Inputs: Text containing email  
  - Outputs: JSON with extracted email under `.output.email`  
  - Failures: No valid email found in input leads to workflow halt or error

---

### 2.2 Account & Contact Research

**Overview:**  
Conducts in-depth company research via MadKudu MCP and OpenAI to generate an account brief outlining company insights and context relevant to personalized outreach.

**Nodes Involved:**  
- MadKudu MCP  
- OpenAI Model  
- MadKudu: Generate Account Brief  
- Structured Output Parser

**Node Details:**  

- **MadKudu MCP**  
  - Type: MadKudu MCP Client Tool (Langchain)  
  - Role: Connects to MadKudu’s API to retrieve streaming company data and signals  
  - Config: SSE endpoint dynamically composed using environment variable `madkudu_api_key`  
  - Inputs: Prospect email domain or contact info  
  - Outputs: Streaming research data for use in account briefing  
  - Failures: API key invalid, network errors, SSE stream interruptions

- **OpenAI Model**  
  - Type: Langchain LM Chat OpenAI  
  - Role: Processes MadKudu data to generate natural language summaries  
  - Config: GPT-4.1-mini, no special options  
  - Inputs: Raw data from MadKudu MCP  
  - Outputs: Textual synthesis for account brief  
  - Credentials: Requires OpenAI API key  
  - Failures: API errors or rate limits

- **MadKudu: Generate Account Brief**  
  - Type: Langchain Agent  
  - Role: Uses MadKudu-specific instructions and domain data to generate an insightful account brief including Intro, Why, Why now, Why us, Risks  
  - Config: Prompt set to use MadKudu account brief instructions with prospect’s company domain  
  - Inputs: Company domain from extracted email  
  - Outputs: Structured account brief for email generation  
  - Failures: Incomplete data, prompt errors, or API failures

- **Structured Output Parser**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses JSON output structures for downstream nodes  
  - Config: JSON schema example includes email drafts and reasoning  
  - Inputs: Raw AI-generated JSON text  
  - Outputs: Parsed JSON for email drafting  
  - Failures: Malformed JSON, parsing errors

---

### 2.3 Personalized Email Generation

**Overview:**  
Generates five personalized email drafts using research data and AI, then selects the best email and rationale for sending.

**Nodes Involved:**  
- Generate outreach email

**Node Details:**  

- **Generate outreach email**  
  - Type: Langchain Agent  
  - Role: Combines MadKudu research and OpenAI to create multiple tailored outreach angles, selecting the best to send  
  - Config: Complex prompt including context, research steps, and guidelines for tone and personalization; uses MadKudu MCP outputs and extracted contact info  
  - Inputs: Account brief data and contact email  
  - Outputs: JSON with email drafts (five angles), selected email, and reasoning  
  - Failures: Prompt failures, AI generation errors, incomplete input data

---

### 2.4 Outreach.io Prospect Verification & Sync

**Overview:**  
Checks if the prospect exists in Outreach.io CRM by email, then either updates the existing prospect’s custom field with the generated email draft or creates a new prospect with the email included.

**Nodes Involved:**  
- Outreach: Check Existing Contact  
- Contact Exists? (If node)  
- Update Existing Prospect  
- Outreach: Create New Prospect  
- Set Contact ID  
- Set Contact ID1  
- Merge Outreach Prospect ID  
- Set Outreach Mailbox and Sequence

**Node Details:**  

- **Outreach: Check Existing Contact**  
  - Type: HTTP Request  
  - Role: Queries Outreach.io API for prospects matching the provided email  
  - Config: GET request with email filter, OAuth2 authentication  
  - Inputs: Extracted email from chat input  
  - Outputs: JSON array of prospect data  
  - Failures: API auth errors, no results, network issues

- **Contact Exists?**  
  - Type: If Node  
  - Role: Checks if the returned prospects array length is greater than zero (strict number comparison)  
  - Inputs: Length of data array from previous node  
  - Outputs: True branch if prospect exists, False branch if not

- **Update Existing Prospect**  
  - Type: HTTP Request (PATCH)  
  - Role: Updates the custom field (`custom49`) of the existing prospect with the personalized email draft  
  - Config: PATCH to `/prospects/{id}`, JSON body with custom field update, OAuth2 credentials  
  - Inputs: Prospect ID, email draft from "Generate outreach email" node  
  - Outputs: Updated prospect data  
  - Failures: API errors, invalid ID, permission issues

- **Outreach: Create New Prospect**  
  - Type: HTTP Request (POST)  
  - Role: Creates a new prospect in Outreach.io with email and custom field containing the email draft  
  - Config: POST to `/prospects`, JSON body with email and custom field, OAuth2 credentials  
  - Inputs: Email from chat input, email draft  
  - Outputs: New prospect data  
  - Failures: API errors, duplicate emails, permission issues

- **Set Contact ID** & **Set Contact ID1**  
  - Type: Set Node  
  - Role: Extracts and assigns `Outreach Prospect ID` from API response to JSON for downstream use  
  - Config: Assign string from parsed JSON response data ID  
  - Inputs: API response data from either update or create nodes  
  - Outputs: JSON with prospect ID

- **Merge Outreach Prospect ID**  
  - Type: Merge Node  
  - Role: Merges branches from "Set Contact ID" and "Set Contact ID1" into a unified stream for sequence addition  
  - Inputs: Prospect ID from either update or create path  
  - Outputs: Merged data for sequence enrollment

- **Set Outreach Mailbox and Sequence**  
  - Type: Set Node  
  - Role: Sets static Outreach Mailbox ID and Sequence ID variables used for enrollment  
  - Config: Numbers assigned (Mailbox ID = 51, Sequence ID = 781 by default)  
  - Inputs: None (preset values)  
  - Outputs: JSON with mailbox and sequence IDs  
  - Failures: Must be updated to match user Outreach account setup

---

### 2.5 Sequence Enrollment & Confirmation

**Overview:**  
Enrolls the prospect into the specified Outreach.io sequence, waits 30 seconds, then confirms enrollment status before ending or no-op.

**Nodes Involved:**  
- Outreach Submit batch Add to Sequence  
- Wait  
- Outreach: Get confirmation  
- Outreach: Get confirmation1  
- If (check enrollment status)  
- No Operation, do nothing

**Node Details:**  

- **Outreach Submit batch Add to Sequence**  
  - Type: HTTP Request (POST)  
  - Role: Adds prospect ID to Outreach sequence using mailbox and sequence IDs  
  - Config: POST to `/batches/actions/prospectsAddToSequence`, JSON body with IDs, OAuth2 authentication  
  - Inputs: Prospect ID, mailbox ID, sequence ID  
  - Outputs: Batch action response with batch ID

- **Wait**  
  - Type: Wait Node  
  - Role: Pauses workflow for 30 seconds to allow sequence enrollment processing  
  - Config: Wait time set to 30 seconds

- **Outreach: Get confirmation** & **Outreach: Get confirmation1**  
  - Type: HTTP Request  
  - Role: Polls Outreach API to verify batch enrollment status using batch ID  
  - Config: GET requests to `/batches/{batchId}` and `/prospects/{prospectId}`, OAuth2 authentication  
  - Inputs: Batch ID and Prospect ID  
  - Outputs: Enrollment state information

- **If (check enrollment status)**  
  - Type: If Node  
  - Role: Checks if enrollment state is `"finished"`  
  - Inputs: State attribute from batch confirmation response  
  - Outputs: True branch proceeds, False branch triggers no-op

- **No Operation, do nothing**  
  - Type: NoOp Node  
  - Role: Ends workflow path cleanly when enrollment not finished or no action needed

---

## 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                            | Input Node(s)                            | Output Node(s)                            | Sticky Note                                                                                                  |
|--------------------------------|----------------------------------|--------------------------------------------|------------------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When chat message received      | Langchain chatTrigger             | Entry point to receive prospect email      | —                                        | OpenAI Chat Model                           |                                                                                                              |
| OpenAI Chat Model               | Langchain LM Chat OpenAI          | Processes raw chat input                    | When chat message received                | Information Extractor                       |                                                                                                              |
| Information Extractor           | Langchain Information Extractor   | Extracts validated email                    | OpenAI Chat Model                         | MadKudu MCP                                |                                                                                                              |
| MadKudu MCP                    | MadKudu MCP Client Tool           | Obtains company research data               | Information Extractor                     | MadKudu: Generate Account Brief, Generate outreach email | Sticky Note3: Explains MadKudu MCP use for account briefing (developers.madkudu.com)                          |
| OpenAI Model                   | Langchain LM Chat OpenAI          | AI processing for account brief             | MadKudu MCP                              | MadKudu: Generate Account Brief, Generate outreach email |                                                                                                              |
| MadKudu: Generate Account Brief| Langchain Agent                   | Creates company brief from MadKudu data     | MadKudu MCP, OpenAI Model                 | Generate outreach email                     |                                                                                                              |
| Structured Output Parser       | Langchain Output Parser Structured| Parses AI JSON output                        | MadKudu: Generate Account Brief           | Generate outreach email                     |                                                                                                              |
| Generate outreach email        | Langchain Agent                   | Generates 5 personalized emails & selects best | MadKudu: Generate Account Brief          | Outreach: Check Existing Contact            | Sticky Note4: Details on generating 5 email approaches and selecting best                                    |
| Outreach: Check Existing Contact| HTTP Request                    | Checks if prospect exists in Outreach.io    | Generate outreach email                   | Set Outreach Mailbox and Sequence           |                                                                                                              |
| Set Outreach Mailbox and Sequence| Set Node                       | Sets Outreach mailbox and sequence IDs      | Outreach: Check Existing Contact          | Contact Exists?                             | Sticky Note: Explains syncing email to Outreach, editing custom field ID (custom49)                         |
| Contact Exists?                | If Node                          | Branches based on prospect existence        | Set Outreach Mailbox and Sequence         | Update Existing Prospect (true), Outreach: Create New Prospect (false) |                                                                                                              |
| Update Existing Prospect       | HTTP Request                    | Updates prospect custom field with email    | Contact Exists? (true)                    | Set Contact ID                              |                                                                                                              |
| Outreach: Create New Prospect  | HTTP Request                    | Creates new prospect with email and custom field | Contact Exists? (false)                   | Set Contact ID1                             |                                                                                                              |
| Set Contact ID                 | Set Node                        | Stores updated prospect ID                   | Update Existing Prospect                  | Merge Outreach Prospect ID                   |                                                                                                              |
| Set Contact ID1                | Set Node                        | Stores created prospect ID                   | Outreach: Create New Prospect             | Merge Outreach Prospect ID                   |                                                                                                              |
| Merge Outreach Prospect ID     | Merge Node                     | Merges prospect ID from create/update paths | Set Contact ID, Set Contact ID1            | Outreach Submit batch Add to Sequence        |                                                                                                              |
| Outreach Submit batch Add to Sequence | HTTP Request             | Adds prospect to Outreach sequence           | Merge Outreach Prospect ID                 | Wait                                        | Sticky Note1: Details on sequence enrollment, config mailbox and sequence IDs, 30s wait                     |
| Wait                          | Wait Node                     | Waits 30 seconds to allow enrollment processing | Outreach Submit batch Add to Sequence       | Outreach: Get confirmation                   |                                                                                                              |
| Outreach: Get confirmation     | HTTP Request                    | Checks batch enrollment status               | Wait                                     | Outreach: Get confirmation1                  |                                                                                                              |
| Outreach: Get confirmation1    | HTTP Request                    | Retrieves prospect enrollment state          | Outreach: Get confirmation                | If                                           |                                                                                                              |
| If                            | If Node                         | Confirms enrollment finished state           | Outreach: Get confirmation1                | No Operation (false branch)                  |                                                                                                              |
| No Operation, do nothing       | NoOp Node                      | Ends workflow cleanly when no enrollment     | If (false branch)                         | —                                            |                                                                                                              |
| Workflow Start Guide           | Sticky Note                    | Explains input email process                  | —                                        | —                                            |                                                                                                              |
| Complete Setup Guide           | Sticky Note                    | Lists required credentials and configuration | —                                        | —                                            |                                                                                                              |
| Template Overview             | Sticky Note                    | Summarizes workflow purpose and instructions | —                                        | —                                            | Includes contact info: Reach out to [MadKudu](product@madkudu.com)                                          |
| Sticky Note                   | Sticky Note                    | Explains syncing personalized email to Outreach | —                                        | —                                            |                                                                                                              |
| Sticky Note1                  | Sticky Note                    | Explains sequence enrollment setup and verification | —                                        | —                                            |                                                                                                              |
| Sticky Note3                  | Sticky Note                    | MadKudu MCP research explanation              | —                                        | —                                            | Link: developers.madkudu.com                                                                                 |
| Sticky Note4                  | Sticky Note                    | Personalized email generation explanation     | —                                        | —                                            |                                                                                                              |

---

## 4. Reproducing the Workflow from Scratch

1. **Create “When chat message received” node**  
   - Type: Langchain chatTrigger (Webhook)  
   - No special parameters; this is the entry point for receiving the prospect email.

2. **Add “OpenAI Chat Model” node**  
   - Type: Langchain LM Chat OpenAI  
   - Model: Select `gpt-4.1-mini`  
   - Connect input from “When chat message received” node  
   - Credentials: Configure with valid OpenAI API key.

3. **Add “Information Extractor” node**  
   - Type: Langchain Information Extractor  
   - Set attribute: `email` (required) to extract email from chat input  
   - Connect input from “OpenAI Chat Model” node.

4. **Add “MadKudu MCP” node**  
   - Type: MadKudu MCP Client Tool  
   - Set SSE endpoint: `https://mcp.madkudu.com/{{$vars.madkudu_api_key}}/sse`  
   - Connect input from “Information Extractor” node  
   - Ensure environment variable `madkudu_api_key` is set with valid API key.

5. **Add “OpenAI Model” node**  
   - Type: Langchain LM Chat OpenAI  
   - Model: `gpt-4.1-mini`  
   - Connect input from “MadKudu MCP” node  
   - Credentials: OpenAI API key.

6. **Add “MadKudu: Generate Account Brief” node**  
   - Type: Langchain Agent  
   - Configure prompt to use MadKudu account brief instructions with domain extracted from the email in previous step  
   - Connect inputs from both “MadKudu MCP” and “OpenAI Model” nodes.

7. **Add “Structured Output Parser” node**  
   - Type: Langchain Output Parser Structured  
   - Configure with JSON schema example to parse email drafts and reasoning  
   - Connect input from “MadKudu: Generate Account Brief” node.

8. **Add “Generate outreach email” node**  
   - Type: Langchain Agent  
   - Configure prompt with detailed instructions to generate 5 email angles and select the best  
   - Connect input from “Structured Output Parser”  
   - Use variables for company domain and contact email extracted earlier.

9. **Add “Outreach: Check Existing Contact” node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.outreach.io/api/v2/prospects?filter[emails]={{ extracted_email }}`  
   - Authentication: OAuth2 (Outreach OAuth)  
   - Connect input from “Generate outreach email”.

10. **Add “Set Outreach Mailbox and Sequence” node**  
    - Type: Set Node  
    - Assign variables:  
      - Outreach Mailbox ID = 51 (update as per your Outreach account)  
      - Outreach Sequence ID = 781 (update accordingly)  
    - Connect input from “Outreach: Check Existing Contact”.

11. **Add “Contact Exists?” node**  
    - Type: If Node  
    - Condition: Check if length of `data` array from “Outreach: Check Existing Contact” > 0  
    - Connect input from “Set Outreach Mailbox and Sequence”.

12. **Add “Update Existing Prospect” node**  
    - Type: HTTP Request (PATCH)  
    - URL: `https://api.outreach.io/api/v2/prospects/{{existing_prospect_id}}`  
    - Body: JSON patch updating `custom49` with the selected email draft from “Generate outreach email”  
    - Authentication: OAuth2  
    - Connect from Contact Exists? (true branch).

13. **Add “Outreach: Create New Prospect” node**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.outreach.io/api/v2/prospects`  
    - Body: JSON creating new prospect with email and `custom49` field containing selected email draft  
    - Authentication: OAuth2  
    - Connect from Contact Exists? (false branch).

14. **Add “Set Contact ID” and “Set Contact ID1” nodes**  
    - Type: Set Nodes  
    - Extract and assign `Outreach Prospect ID` from response JSON ID from update or create nodes respectively  
    - Connect from “Update Existing Prospect” and “Outreach: Create New Prospect”.

15. **Add “Merge Outreach Prospect ID” node**  
    - Type: Merge Node  
    - Merge outputs from “Set Contact ID” and “Set Contact ID1”  
    - Connect output to next step.

16. **Add “Outreach Submit batch Add to Sequence” node**  
    - Type: HTTP Request (POST)  
    - URL: `https://api.outreach.io/api/v2/batches/actions/prospectsAddToSequence`  
    - Body: JSON with prospect ID, mailbox ID, and sequence ID to enroll prospect  
    - Authentication: OAuth2  
    - Connect input from “Merge Outreach Prospect ID”.

17. **Add “Wait” node**  
    - Type: Wait Node  
    - Duration: 30 seconds  
    - Connect input from “Outreach Submit batch Add to Sequence”.

18. **Add “Outreach: Get confirmation” and “Outreach: Get confirmation1” nodes**  
    - Type: HTTP Request (GET)  
    - URLs:  
      - `/batches/{batchId}` to get batch status  
      - `/prospects/{prospectId}` to get prospect status  
    - Authentication: OAuth2  
    - Connect “Outreach: Get confirmation” from “Wait” node, then chain to “Outreach: Get confirmation1”.

19. **Add “If” node**  
    - Type: If Node  
    - Condition: Check if `state` attribute in batch confirmation response equals `"finished"`  
    - Connect input from “Outreach: Get confirmation1”.

20. **Add “No Operation, do nothing” node**  
    - Type: NoOp Node  
    - Connect from “If” node false branch to cleanly end workflow if enrollment not finished.

21. **Final Configuration:**  
    - Set environment variables (e.g., `madkudu_api_key`)  
    - Configure OAuth2 credentials for Outreach and API keys for OpenAI  
    - Update the `custom49` field name if your Outreach custom field differs  
    - Update Outreach Mailbox ID and Sequence ID to match your account settings.

---

## 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                              |
|----------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow generates personalized outreach emails using MadKudu research and OpenAI, syncing to Outreach CRM and enrolling in sequences. | Overview and purpose of the workflow                          |
| Requires environment variable `madkudu_api_key` for MadKudu API authentication                                                         | Environment setup                                            |
| Outreach OAuth2 credentials needed for API interactions                                                                                 | OAuth2 setup in n8n for API requests                         |
| OpenAI API key required for language model usage                                                                                       | OpenAI account setup                                        |
| Customize Outreach custom field (default `custom49`) in update and create prospect nodes                                                | Outreach CRM customization                                   |
| Update Outreach Mailbox ID and Sequence ID for sequence enrollment                                                                     | Outreach account configuration                               |
| Workflow input example: entering `john@company.com` triggers research and email generation                                             | Input example                                                |
| Contact MadKudu for support: product@madkudu.com                                                                                       | Support contact                                              |
| MadKudu developer docs: https://developers.madkudu.com                                                                                 | MadKudu API and MCP documentation                            |
| Tone guidelines for generated emails: professional, helpful, neutral; avoid excessive flattery or sounding like a chatbot              | Email generation best practices                              |

---

**Disclaimer:** The provided analysis is based solely on the supplied n8n workflow JSON, respecting all applicable content policies. The workflow manipulates only legal and public data.