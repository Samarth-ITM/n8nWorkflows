Daily Gmail Inbox Digest to Discord with GPT-4.1-mini and PDF Conversion

https://n8nworkflows.xyz/workflows/daily-gmail-inbox-digest-to-discord-with-gpt-4-1-mini-and-pdf-conversion-8358


# Daily Gmail Inbox Digest to Discord with GPT-4.1-mini and PDF Conversion

### 1. Workflow Overview

This workflow automates the process of generating a daily digest of important Gmail inbox emails, summarizing them using AI, converting the summary into a PDF, and sending the results to a Discord channel. Its primary use case is for users who want to receive a concise, readable summary of their daily critical emails along with a downloadable PDF version, directly in Discord.

The workflow is logically structured into the following functional blocks:

- **1.1 Daily Trigger**: Initiates the workflow once per day at a specified hour.
- **1.2 Gmail Email Retrieval**: Fetches all important unread emails from the past 24 hours.
- **1.3 Email Content Extraction and Aggregation**: Extracts relevant fields (subject, sender, plain text body) from emails and aggregates them.
- **1.4 AI Summarization**: Summarizes the aggregated emails using GPT-4.1-mini into both plain text and markdown digest formats.
- **1.5 Markdown to HTML and PDF Conversion**: Converts the markdown summary to HTML and then to a PDF file using PDF.co API.
- **1.6 PDF Retrieval**: Downloads the generated PDF file via an HTTP request.
- **1.7 Discord Notification**: Sends the plain text summary and the PDF attachment to a Discord channel through a webhook.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger  
- **Overview:**  
Triggers the workflow daily at 20:00 (8 PM) to start processing the Gmail inbox.
- **Nodes Involved:**  
  - Schedule Trigger
- **Node Details:**  
  - **Schedule Trigger**  
    - Type: scheduleTrigger  
    - Configuration: Runs once every day at 20:00 hours.  
    - Inputs: None (start node)  
    - Outputs: Connects to "get mails from past 24 h" node  
    - Edge cases: Scheduler misconfiguration or system downtime causing missed runs.  
    - No sub-workflow references.

#### 1.2 Gmail Email Retrieval  
- **Overview:**  
Fetches all emails labeled as INBOX and IMPORTANT received within the last 24 hours.  
- **Nodes Involved:**  
  - get mails from past 24 h  
  - Sticky Note1 (comment)  
- **Node Details:**  
  - **get mails from past 24 h**  
    - Type: gmail  
    - Configuration: Retrieves all emails from INBOX and IMPORTANT labels received after current time minus 1 day. Returns all matching emails.  
    - Credentials: Gmail OAuth2  
    - Inputs: From Schedule Trigger  
    - Outputs: To "extract required sections from mails"  
    - Edge cases: Gmail API rate limits, OAuth token expiration, network issues.  
    - Sticky Note1: "get mails - Important and inbox"  
    - No sub-workflow references.

#### 1.3 Email Content Extraction and Aggregation  
- **Overview:**  
Extracts necessary fields from raw Gmail message objects, cleans the plain text body, and aggregates all email data fields for summarization.  
- **Nodes Involved:**  
  - extract required sections from mails (Code node)  
  - Aggregate  
  - Sticky Note2 (comment)  
- **Node Details:**  
  - **extract required sections from mails**  
    - Type: code (JavaScript)  
    - Configuration: Decodes base64 email bodies, extracts subject, from, to, and cleans plain text by removing URLs and excessive blank lines. Supports multipart email bodies and fallback text fields.  
    - Inputs: From "get mails from past 24 h"  
    - Outputs: To "Aggregate"  
    - Edge cases: Emails without plain text parts, malformed base64 data, missing headers.  
    - Sticky Note2: "extract fields and aggregate - sender, subject, body"  
  - **Aggregate**  
    - Type: aggregate  
    - Configuration: Aggregates all items including fields: subject, from, plainText.  
    - Inputs: From "extract required sections from mails"  
    - Outputs: To "Summarization Chain"  
    - Edge cases: Large email volumes causing memory issues.

#### 1.4 AI Summarization  
- **Overview:**  
Uses GPT-4.1-mini to generate a dual-format summary (plain text and markdown) of the aggregated emails with clear formatting and action flags.  
- **Nodes Involved:**  
  - OpenAI Chat Model1  
  - Summarization Chain  
  - separate text and markdown (Code node)  
  - Sticky Note3 (comment)  
- **Node Details:**  
  - **OpenAI Chat Model1**  
    - Type: langchain OpenAI Chat Model  
    - Configuration: Uses GPT-4.1-mini model without additional options.  
    - Inputs: From "Summarization Chain" via ai_languageModel input  
    - Outputs: To "Summarization Chain"  
    - Credentials: OpenAI API  
    - Edge cases: API rate limits, model unavailability, authentication errors.  
  - **Summarization Chain**  
    - Type: langchain summarization chain  
    - Configuration: Uses a custom prompt instructing summarization into two formats—plain text and markdown with clear formatting and action required flags.  
    - Inputs: From "Aggregate" node main input, from "OpenAI Chat Model1" as language model  
    - Outputs: To "separate text and markdown"  
    - Edge cases: Prompt format errors, empty inputs.  
  - **separate text and markdown**  
    - Type: code (JavaScript)  
    - Configuration: Extracts plain text and markdown summaries from the AI output using regex.  
    - Inputs: From "Summarization Chain"  
    - Outputs: To "Markdown"  
    - Edge cases: Unexpected AI output format causing regex to fail.  
  - Sticky Note3: "summarize all mails - model used gpt 4o-mini" (note: typo in 'gpt 4o-mini', intended 'gpt-4.1-mini')

#### 1.5 Markdown to HTML and PDF Conversion  
- **Overview:**  
Converts the markdown summary to HTML, then sends it to PDF.co API to generate a PDF file.  
- **Nodes Involved:**  
  - Markdown  
  - PDFco Api  
  - Sticky Note4 (comment)  
- **Node Details:**  
  - **Markdown**  
    - Type: markdown  
    - Configuration: Converts markdown input (from "separate text and markdown" node's markdown field) to HTML.  
    - Inputs: From "separate text and markdown"  
    - Outputs: To "PDFco Api"  
    - Edge cases: Invalid markdown causing conversion errors.  
  - **PDFco Api**  
    - Type: PDF.co API node  
    - Configuration: Converts HTML content (from "Markdown") to PDF.  
    - Credentials: PDF.co API credentials  
    - Inputs: From "Markdown" (HTML content)  
    - Outputs: To "HTTP Request" for PDF download  
    - Edge cases: API rate limits, invalid HTML input, network errors.  
  - Sticky Note4: "convert markdown to html and convert html to pdf using PDFco Api"

#### 1.6 PDF Retrieval  
- **Overview:**  
Downloads the PDF file generated by PDF.co using the URL provided in the API response.  
- **Nodes Involved:**  
  - HTTP Request  
  - Sticky Note5 (comment)  
- **Node Details:**  
  - **HTTP Request**  
    - Type: httpRequest  
    - Configuration: Performs a GET request to the URL provided in the previous node's JSON output (PDF download link).  
    - Inputs: From "PDFco Api"  
    - Outputs: To "Discord" node  
    - Edge cases: URL missing or expired, network errors, download failures.  
  - Sticky Note5: "Download PDF - using link generated from pdfco"

#### 1.7 Discord Notification  
- **Overview:**  
Sends the plain text summary and the PDF file as an attachment to a Discord channel using a webhook.  
- **Nodes Involved:**  
  - Discord  
  - Sticky Note6 (comment)  
- **Node Details:**  
  - **Discord**  
    - Type: discord  
    - Configuration: Sends a message to Discord webhook with content set to the plain text digest and attaches the downloaded PDF file. Uses webhook authentication.  
    - Credentials: Discord Webhook account  
    - Inputs: From "HTTP Request" (PDF data)  
    - Outputs: None (end node)  
    - Edge cases: Webhook invalid or revoked, attachment size limits, network failures.  
  - Sticky Note6: "Send discord - using webhook - result will contain summary and a pdf of summary"

---

### 3. Summary Table

| Node Name                    | Node Type                        | Functional Role                           | Input Node(s)                  | Output Node(s)                    | Sticky Note                                                                                  |
|------------------------------|---------------------------------|-----------------------------------------|-------------------------------|----------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger             | scheduleTrigger                 | Daily trigger to start workflow         | None                          | get mails from past 24 h          | ## daily trigger                                                                            |
| get mails from past 24 h     | gmail                          | Fetch emails from inbox and important   | Schedule Trigger              | extract required sections from mails | ## get mails - Important and inbox                                                          |
| extract required sections from mails | code (JavaScript)             | Extracts & cleans email fields           | get mails from past 24 h      | Aggregate                       | ## extract fields and aggregate - sender, subject, body                                    |
| Aggregate                   | aggregate                      | Aggregates email data                    | extract required sections from mails | Summarization Chain             |                                                                                              |
| OpenAI Chat Model1          | langchain OpenAI Chat Model    | GPT-4.1-mini LLM for summarization      | Summarization Chain (ai_languageModel) | Summarization Chain            | ## summarize all mails - model used gpt 4o-mini                                             |
| Summarization Chain         | langchain summarization chain  | Summarizes emails into text & markdown  | Aggregate, OpenAI Chat Model1 | separate text and markdown       | ## summarize all mails - model used gpt 4o-mini                                             |
| separate text and markdown  | code (JavaScript)              | Extracts plain text and markdown outputs | Summarization Chain           | Markdown                       |                                                                                              |
| Markdown                   | markdown                       | Converts markdown to HTML                | separate text and markdown    | PDFco Api                     | ## convert markdown to html and convert html to pdf using PDFco Api                         |
| PDFco Api                  | PDF.co API                     | Converts HTML to PDF                     | Markdown                     | HTTP Request                   | ## convert markdown to html and convert html to pdf using PDFco Api                         |
| HTTP Request               | httpRequest                    | Downloads generated PDF                  | PDFco Api                   | Discord                       | ## Download PDF - using link generated from pdfco                                           |
| Discord                   | discord                        | Sends message and PDF to Discord channel | HTTP Request                | None                          | ## Send discord - using webhook - result will contain summary and a pdf of summary          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: scheduleTrigger  
   - Set to trigger daily at 20:00 (8 PM).  
   - No credentials needed.

2. **Create Gmail Node**  
   - Type: gmail  
   - Operation: getAll  
   - Filters: LabelIds include "INBOX" and "IMPORTANT"  
   - ReceivedAfter: Set to `{{$now.minus({days:1}).toISO()}}` to fetch emails from past 24 hours  
   - ReturnAll: true  
   - Credentials: Configure Gmail OAuth2 credentials.

3. **Create Code Node to Extract Email Sections**  
   - Type: code (JavaScript)  
   - Paste the provided JS code to decode base64 email bodies, extract subject, from, to, and clean plain text.  
   - Input: connects from Gmail node.

4. **Create Aggregate Node**  
   - Type: aggregate  
   - Aggregate all item data  
   - Include fields: subject, from, plainText  
   - Input: from the Code node.

5. **Create OpenAI Chat Model Node**  
   - Type: langchain OpenAI Chat Model  
   - Model: select "gpt-4.1-mini"  
   - Credentials: configure OpenAI API key.

6. **Create Summarization Chain Node**  
   - Type: langchain summarization chain  
   - Use custom prompt instructing to summarize emails into two formats: plain text digest and markdown digest, highlighting actions required.  
   - Connect Aggregate node to main input  
   - Connect OpenAI Chat Model node to ai_languageModel input.

7. **Create Code Node to Separate Text and Markdown**  
   - Type: code (JavaScript)  
   - Paste JavaScript code that uses regex to extract plain text and markdown from AI output.  
   - Input: from Summarization Chain.

8. **Create Markdown Node**  
   - Type: markdown  
   - Mode: markdownToHtml  
   - Input: markdown field from previous Code node.

9. **Create PDF.co API Node**  
   - Type: PDFco Api  
   - Operation: URL/HTML to PDF  
   - ConvertType: htmlToPDF  
   - Input: HTML from Markdown node  
   - Credentials: configure PDF.co API credentials.

10. **Create HTTP Request Node**  
    - Type: httpRequest  
    - Method: GET  
    - URL: bind dynamically to the PDF download URL from PDFco API response (`{{$json.url}}`).  
    - Input: from PDFco API node.

11. **Create Discord Node**  
    - Type: discord  
    - Authentication: webhook  
    - Webhook ID: configure webhook URL or use webhook credentials  
    - Content: bind to plainText from the separate text and markdown node.  
    - Attachments: pass the downloaded PDF file from HTTP Request node.  
    - Input: from HTTP Request node.

12. **Connect Nodes in Sequence:**  
    Schedule Trigger → Gmail → Code (extract required sections) → Aggregate → Summarization Chain (main input) → OpenAI Chat Model (ai_languageModel input) → Summarization Chain output → separate text and markdown → Markdown → PDFco Api → HTTP Request → Discord.

13. **Configure Credentials:**  
    - Gmail OAuth2 for Gmail node  
    - OpenAI API key for OpenAI Chat Model node  
    - PDF.co API key for PDFco API node  
    - Discord Webhook for Discord node

14. **Set Sticky Notes (Optional):**  
    Add sticky notes near logical blocks for clarity, using descriptions from the original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses GPT-4.1-mini model for text summarization, requiring OpenAI API access.     | OpenAI API documentation                        |
| PDF conversion is handled by PDF.co API, which requires an account and API key.                | https://pdf.co/api                               |
| Discord messages are sent via webhook integration; ensure the webhook URL is valid and active. | Discord Webhook docs: https://discord.com/developers/docs/resources/webhook |
| Email extraction includes base64 decoding adapted for Gmail's URL-safe base64 encoding.       | Gmail API documentation                          |
| The summarization prompt splits output into plain text and markdown for flexible presentation.| Custom prompt embedded in Summarization Chain node |
| Potential failure points include API rate limits, expired OAuth tokens, malformed email data. | Monitor n8n executions for error handling       |

---

This documentation provides a thorough understanding, node-level details, and instructions to rebuild or modify the workflow independently.