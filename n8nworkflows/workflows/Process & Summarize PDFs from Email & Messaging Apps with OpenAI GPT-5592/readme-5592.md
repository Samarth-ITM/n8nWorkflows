Process & Summarize PDFs from Email & Messaging Apps with OpenAI GPT

https://n8nworkflows.xyz/workflows/process---summarize-pdfs-from-email---messaging-apps-with-openai-gpt-5592


# Process & Summarize PDFs from Email & Messaging Apps with OpenAI GPT

### 1. Workflow Overview

This workflow automates the processing and summarization of PDF documents received via email and messaging platforms, leveraging OpenAI GPT through n8n LangChain nodes. It targets use cases where users want to automatically extract, analyze, and summarize PDF attachments sent by Gmail, Outlook, Telegram, and WhatsApp Business, and then send back summarized content to the original communication platform.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Monitors multiple communication sources (Gmail, Outlook, Telegram, WhatsApp) for incoming messages containing PDF attachments.
- **1.2 Data Extraction & Validation**: Extracts relevant data from incoming messages, validates the presence and type of PDF attachments, and downloads PDFs from messaging platforms.
- **1.3 PDF Content Extraction**: Extracts the textual content from the downloaded PDF files.
- **1.4 AI Summarization**: Uses OpenAI GPT models via LangChain to summarize the extracted PDF text.
- **1.5 Response Preparation & Routing**: Prepares the summarized response and routes it back to the appropriate messaging platform (WhatsApp or Telegram), also notifying about email PDF processing.
- **1.6 Auxiliary Nodes**: Sticky notes for documentation or temporary comments.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for new incoming messages from different platforms and triggers the workflow to process the content.

**Nodes Involved:**  
- Gmail Email Monitor  
- Outlook Email Monitor  
- Telegram Bot Receiver  
- WhatsApp Business Receiver  

**Node Details:**  

- **Gmail Email Monitor**  
  - Type: Gmail node (Email trigger)  
  - Configuration: Webhook-based trigger monitoring Gmail inbox for new emails.  
  - Inputs: Incoming new email events  
  - Outputs: Email data passed to "Extract Gmail Data"  
  - Failure modes: Gmail API auth failure, webhook disconnection, quota limits  

- **Outlook Email Monitor**  
  - Type: Microsoft Outlook node (Email trigger)  
  - Configuration: Webhook-based trigger for new Outlook emails.  
  - Inputs: Incoming new email events  
  - Outputs: Email data passed to "Extract Outlook Data"  
  - Failure modes: OAuth token expiration, API rate limits  

- **Telegram Bot Receiver**  
  - Type: Telegram Trigger node  
  - Configuration: Webhook listening for incoming Telegram messages directed to the bot.  
  - Inputs: New Telegram messages  
  - Outputs: Message data passed to "Extract Telegram Data"  
  - Failure modes: Bot token invalid, webhook issues  

- **WhatsApp Business Receiver**  
  - Type: WhatsApp Trigger node  
  - Configuration: Webhook listening for WhatsApp Business messages.  
  - Inputs: Incoming WhatsApp messages  
  - Outputs: Message data passed to "Extract WhatsApp Data"  
  - Failure modes: WhatsApp Business API connection, webhook failures  

---

#### 2.2 Data Extraction & Validation

**Overview:**  
Extracts relevant data fields from incoming messages/emails and validates whether PDF attachments are present and valid for further processing.

**Nodes Involved:**  
- Extract Gmail Data  
- Extract Outlook Data  
- Extract Telegram Data  
- Extract WhatsApp Data  
- Check Email Attachments1  
- Filter PDF Attachments  
- Validate Telegram PDF  
- Validate WhatsApp PDF  
- Download Telegram PDF  
- Download WhatsApp PDF  

**Node Details:**  

- **Extract Gmail Data**  
  - Type: Set node  
  - Role: Extracts and structures Gmail email fields (e.g., attachments, sender info) for downstream processing.  
  - Outputs: Data passed to "Check Email Attachments1"  
  - Edge cases: Emails without attachments, malformed data  

- **Extract Outlook Data**  
  - Type: Set node  
  - Role: Similar to Gmail Data extraction but tailored for Outlook email structure.  
  - Outputs: Data passed to "Check Email Attachments1"  
  - Edge cases: Missing attachments, API data inconsistencies  

- **Check Email Attachments1**  
  - Type: If node  
  - Role: Checks if email contains attachments eligible for processing (PDFs).  
  - Outputs: Passes items with attachments to "Filter PDF Attachments"  
  - Edge cases: Emails without attachments are filtered out  

- **Filter PDF Attachments**  
  - Type: Code node  
  - Role: Filters the attachments to include only PDFs for extraction.  
  - Key logic: JavaScript code evaluates attachment mime types or file extensions.  
  - Outputs: Valid PDF attachments sent to "Extract PDF Text Content"  
  - Edge cases: Non-PDF attachments discarded; errors if attachment metadata missing  

- **Extract Telegram Data**  
  - Type: Set node  
  - Role: Extracts Telegram message data including document attachments.  
  - Outputs: Passes data to "Validate Telegram PDF"  
  - Edge cases: Messages without documents, unsupported file types  

- **Validate Telegram PDF**  
  - Type: If node  
  - Role: Validates if the Telegram attachment is a PDF.  
  - Outputs: If true, triggers "Download Telegram PDF"  
  - Edge cases: Non-PDF files bypassed  

- **Download Telegram PDF**  
  - Type: Telegram node (Download)  
  - Role: Downloads the PDF file from Telegram servers.  
  - Outputs: Passes file to "Extract PDF Text Content"  
  - Failure modes: Download errors, network timeouts  

- **Extract WhatsApp Data**  
  - Type: Set node  
  - Role: Extracts WhatsApp message data including media attachments.  
  - Outputs: Passes data to "Validate WhatsApp PDF"  
  - Edge cases: Messages without media, unsupported types  

- **Validate WhatsApp PDF**  
  - Type: If node  
  - Role: Checks if WhatsApp media is a PDF.  
  - Outputs: If true, triggers "Download WhatsApp PDF"  
  - Edge cases: Non-PDF media ignored  

- **Download WhatsApp PDF**  
  - Type: WhatsApp node (Download)  
  - Role: Downloads the PDF file from WhatsApp servers.  
  - Outputs: Passes file to "Extract PDF Text Content"  
  - Failure modes: Download failures, API errors  

---

#### 2.3 PDF Content Extraction

**Overview:**  
This block extracts the textual content from PDF files, making it ready for AI summarization.

**Nodes Involved:**  
- Extract PDF Text Content  

**Node Details:**  

- **Extract PDF Text Content**  
  - Type: Extract From File node  
  - Role: Reads and extracts text content from PDF binary data.  
  - Inputs: PDF files from email or messaging platforms  
  - Outputs: Text content forwarded to "AI PDF Summarizer"  
  - Edge cases: Corrupted PDFs, extraction errors, unsupported PDF formats  

---

#### 2.4 AI Summarization

**Overview:**  
Summarizes extracted PDF text using OpenAI GPT models integrated via LangChain nodes.

**Nodes Involved:**  
- AI PDF Summarizer  
- Summarize PDF  

**Node Details:**  

- **AI PDF Summarizer**  
  - Type: LangChain Agent node (OpenAI GPT)  
  - Role: Orchestrates the summarization task, leveraging the Summarize PDF node as the language model.  
  - Inputs: Text content from "Extract PDF Text Content"  
  - Outputs: Summarized response passed to "Prepare Response Data"  
  - Requirements: OpenAI credentials configured; proper prompt setup essential  
  - Failure modes: API rate limits, invalid credentials, timeouts  

- **Summarize PDF**  
  - Type: LangChain LM Chat OpenAI node  
  - Role: Executes GPT model calls for summarization with custom prompts.  
  - Inputs: Text data from "Extract PDF Text Content"  
  - Outputs: Summarized text to "AI PDF Summarizer" node  
  - Configuration: OpenAI API key, model parameters (temperature, max tokens)  
  - Failure modes: API errors, prompt failures  

---

#### 2.5 Response Preparation & Routing

**Overview:**  
Prepares the summarized text for sending and routes the response back to the correct platform (WhatsApp or Telegram). Also sends a notification message about PDF processing.

**Nodes Involved:**  
- Prepare Response Data  
- Route Response by Platform  
- Send WhatsApp Summary  
- Send Telegram Summary  
- Notify Email PDF Processing  

**Node Details:**  

- **Prepare Response Data**  
  - Type: Set node  
  - Role: Formats the AI-generated summary into platform-specific message payloads.  
  - Inputs: Summarized text from "AI PDF Summarizer"  
  - Outputs: Passes formatted data to "Route Response by Platform"  
  - Edge cases: Missing summary data, formatting errors  

- **Route Response by Platform**  
  - Type: If node  
  - Role: Checks the platform origin to route responses accordingly.  
  - Outputs:  
    - True branch: Sends WhatsApp summary and notification  
    - False branch: Sends Telegram summary and notification  
  - Edge cases: Unknown platforms, missing platform data  

- **Send WhatsApp Summary**  
  - Type: WhatsApp node (Send message)  
  - Role: Sends the summarized PDF text back to the WhatsApp user.  
  - Inputs: Prepared message from "Route Response by Platform"  
  - Failure modes: WhatsApp API errors, message formatting issues  

- **Send Telegram Summary**  
  - Type: Telegram node (Send message)  
  - Role: Sends the summarized PDF text back to the Telegram user.  
  - Inputs: Prepared message from "Route Response by Platform"  
  - Failure modes: Telegram API errors, formatting issues  

- **Notify Email PDF Processing**  
  - Type: WhatsApp node (Send message)  
  - Role: Sends a notification message about the PDF processing status, presumably to a monitoring or admin WhatsApp number.  
  - Inputs: Triggered alongside platform-specific summary sending  
  - Edge cases: Notification failures  

---

#### 2.6 Auxiliary Nodes

**Overview:**  
Sticky notes positioned around the workflow presumably for future comments or documentation.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  

**Node Details:**  
- Type: Sticky Note nodes  
- Role: Visual annotations with no execution impact  
- Content: Empty in this export  

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                   | Input Node(s)             | Output Node(s)                          | Sticky Note          |
|---------------------------|--------------------------------|---------------------------------|---------------------------|----------------------------------------|----------------------|
| Gmail Email Monitor        | Gmail Trigger                  | Triggers on new Gmail emails    | -                         | Extract Gmail Data                     |                      |
| Extract Gmail Data         | Set                            | Extracts Gmail email data       | Gmail Email Monitor       | Check Email Attachments1               |                      |
| Outlook Email Monitor      | Microsoft Outlook Trigger      | Triggers on new Outlook emails  | -                         | Extract Outlook Data                   |                      |
| Extract Outlook Data       | Set                            | Extracts Outlook email data     | Outlook Email Monitor     | Check Email Attachments1               |                      |
| Check Email Attachments1   | If                             | Checks for email attachments    | Extract Gmail/Outlook Data| Filter PDF Attachments                 |                      |
| Filter PDF Attachments     | Code                           | Filters only PDF attachments    | Check Email Attachments1  | Extract PDF Text Content               |                      |
| Telegram Bot Receiver      | Telegram Trigger               | Triggers on Telegram messages   | -                         | Extract Telegram Data                  |                      |
| Extract Telegram Data      | Set                            | Extracts Telegram data          | Telegram Bot Receiver     | Validate Telegram PDF                  |                      |
| Validate Telegram PDF      | If                             | Validates Telegram PDFs         | Extract Telegram Data     | Download Telegram PDF                  |                      |
| Download Telegram PDF      | Telegram Download              | Downloads Telegram PDF files    | Validate Telegram PDF     | Extract PDF Text Content               |                      |
| WhatsApp Business Receiver | WhatsApp Trigger              | Triggers on WhatsApp messages   | -                         | Extract WhatsApp Data                  |                      |
| Extract WhatsApp Data      | Set                            | Extracts WhatsApp message data  | WhatsApp Business Receiver| Validate WhatsApp PDF                  |                      |
| Validate WhatsApp PDF      | If                             | Validates WhatsApp PDFs         | Extract WhatsApp Data     | Download WhatsApp PDF                  |                      |
| Download WhatsApp PDF      | WhatsApp Download             | Downloads WhatsApp PDF files    | Validate WhatsApp PDF     | Extract PDF Text Content               |                      |
| Extract PDF Text Content   | Extract From File              | Extracts text from PDF files    | Download Telegram/WhatsApp PDF, Filter PDF Attachments | AI PDF Summarizer         |                      |
| AI PDF Summarizer          | LangChain Agent               | Summarizes PDF text via GPT    | Extract PDF Text Content  | Prepare Response Data                  |                      |
| Summarize PDF             | LangChain LM Chat OpenAI       | GPT model invocation for summary | Extract PDF Text Content  | AI PDF Summarizer                      |                      |
| Prepare Response Data      | Set                            | Formats summary for response    | AI PDF Summarizer         | Route Response by Platform             |                      |
| Route Response by Platform | If                             | Routes response by platform     | Prepare Response Data     | Send WhatsApp Summary, Send Telegram Summary, Notify Email PDF Processing |                      |
| Send WhatsApp Summary      | WhatsApp Send message          | Sends summary to WhatsApp       | Route Response by Platform| -                                      |                      |
| Send Telegram Summary      | Telegram Send message          | Sends summary to Telegram       | Route Response by Platform| -                                      |                      |
| Notify Email PDF Processing| WhatsApp Send message          | Sends notification about processing | Route Response by Platform| -                                  |                      |
| Sticky Note               | Sticky Note                    | Visual annotation               | -                         | -                                      |                      |
| Sticky Note1              | Sticky Note                    | Visual annotation               | -                         | -                                      |                      |
| Sticky Note2              | Sticky Note                    | Visual annotation               | -                         | -                                      |                      |
| Sticky Note3              | Sticky Note                    | Visual annotation               | -                         | -                                      |                      |
| Sticky Note4              | Sticky Note                    | Visual annotation               | -                         | -                                      |                      |
| Sticky Note5              | Sticky Note                    | Visual annotation               | -                         | -                                      |                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers for Incoming Messages:**

   - Add a **Gmail Email Trigger** node: Configure with Gmail OAuth credentials, enable webhook, monitor inbox for new emails.
   - Add a **Microsoft Outlook Email Trigger** node: Configure with Outlook OAuth credentials and webhook.
   - Add a **Telegram Trigger** node: Connect your Telegram bot token, enable webhook to receive new messages.
   - Add a **WhatsApp Business Trigger** node: Connect WhatsApp Business API credentials, enable webhook.

2. **Extract Incoming Message Data:**

   - For Gmail and Outlook, add **Set** nodes named "Extract Gmail Data" and "Extract Outlook Data" respectively, to parse incoming email structure and extract attachments metadata.
   - For Telegram and WhatsApp, add **Set** nodes named "Extract Telegram Data" and "Extract WhatsApp Data" respectively, to parse message content and extract document/media info.

3. **Check for Attachments in Emails:**

   - Add an **If** node named "Check Email Attachments1" to evaluate if incoming emails contain attachments.
   - Connect outputs of "Extract Gmail Data" and "Extract Outlook Data" to this node.

4. **Filter PDF Attachments:**

   - Add a **Code** node named "Filter PDF Attachments" running JavaScript code to filter attachments array for files with PDF mime type or extension.
   - Connect "Check Email Attachments1" to this node.

5. **Validate PDF Attachments from Messaging Platforms:**

   - Add **If** nodes named "Validate Telegram PDF" and "Validate WhatsApp PDF" to check if the attachment/media is a PDF based on metadata.
   - Connect "Extract Telegram Data" and "Extract WhatsApp Data" respectively.

6. **Download PDF Files:**

   - Add a **Telegram** node "Download Telegram PDF" to download files from Telegram servers.
   - Add a **WhatsApp** node "Download WhatsApp PDF" to download files from WhatsApp servers.
   - Connect the validation nodes to their respective download nodes.

7. **Extract Text From PDFs:**

   - Add an **Extract From File** node named "Extract PDF Text Content" configured to extract text from PDF input.
   - Connect outputs from "Download Telegram PDF", "Download WhatsApp PDF", and "Filter PDF Attachments" to this node.

8. **Set Up AI Summarization:**

   - Add a **LangChain LM Chat OpenAI** node named "Summarize PDF" configured with OpenAI credentials, set model parameters for text summarization.
   - Add a **LangChain Agent** node named "AI PDF Summarizer" that uses the "Summarize PDF" node as language model.
   - Connect "Extract PDF Text Content" to "Summarize PDF", and "Summarize PDF" output to "AI PDF Summarizer".

9. **Prepare Response Data:**

   - Add a **Set** node "Prepare Response Data" to format the summary output into message payloads suitable for WhatsApp or Telegram.
   - Connect "AI PDF Summarizer" output to this node.

10. **Route Responses By Platform:**

    - Add an **If** node "Route Response by Platform" to check origin platform (WhatsApp or Telegram).
    - Connect "Prepare Response Data" output here.

11. **Send Summarized Responses:**

    - Add a **WhatsApp Send** node "Send WhatsApp Summary" configured with WhatsApp API credentials.
    - Add a **Telegram Send** node "Send Telegram Summary" configured with Telegram bot token.
    - Connect "Route Response by Platform" outputs accordingly.

12. **Send Notification of PDF Processing:**

    - Add a **WhatsApp Send** node "Notify Email PDF Processing" to send status or logs to an admin or monitoring number.
    - Connect both branches of "Route Response by Platform" to this node as well.

13. **Add Sticky Notes (Optional):**

    - Add **Sticky Note** nodes for documentation or reminders.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow automates PDF summarization from multiple platforms leveraging OpenAI GPT and n8n LangChain integration.        | Workflow purpose summary                                                                                          |
| Ensure all API credentials (Gmail, Outlook OAuth2, Telegram Bot Token, WhatsApp Business API, OpenAI API) are correctly configured. | Critical for successful execution                                                                                 |
| LangChain nodes require specific version compatibility with n8n >= 0.208.0 and OpenAI credential setup.                  | Official n8n documentation on LangChain: https://docs.n8n.io/integrations/agents/langchain/                       |
| PDF extraction may fail with encrypted or malformed PDFs; consider adding error handling or retries for robustness.     | Best practice for file handling                                                                                   |
| WhatsApp Business API and Telegram Bot API have rate limits; implement throttling or delays if processing large volumes. | API usage guidelines                                                                                              |
| For webhook triggers, ensure your n8n instance is accessible via HTTPS and webhook URLs are correctly configured.        | n8n webhook setup best practices                                                                                   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.