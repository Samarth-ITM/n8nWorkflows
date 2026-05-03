Digitize Business Cards with Thai OCR & Save to Google Sheets or CRM

https://n8nworkflows.xyz/workflows/digitize-business-cards-with-thai-ocr---save-to-google-sheets-or-crm-7978


# Digitize Business Cards with Thai OCR & Save to Google Sheets or CRM

### 1. Workflow Overview

This n8n workflow, titled **"Digitize Business Cards with Thai OCR & Save to Google Sheets or CRM"**, automates the process of digitizing business cards—supporting both English and Thai text—using Optical Character Recognition (OCR) and Language Learning Models (LLMs). It extracts contact details from uploaded images of business cards, enriches the extracted data with classification or internet search data, and saves the structured contacts into Google Sheets for CRM or sales use. Optionally, it generates and can send personalized greeting emails based on the digitized contact information.

The workflow includes **two main versions**:

- **Version 1 (without Search API):** Uses Typhoon OCR and LLMs only for extraction and enrichment.
- **Version 2 (with Search API):** Adds an internet search enrichment step using SerpAPI for richer profile data.

---

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles receiving business card images and optional messages via form submission triggers.
- **1.2 Image Preparation:** Edits and processes images for OCR compatibility.
- **1.3 OCR Processing:** Extracts raw text from the business card images using Typhoon OCR.
- **1.4 Contact Information Parsing:** Converts raw OCR text into structured JSON contact information.
- **1.5 Contact Enrichment:** Enriches contact data either by internal classification or external Search API lookup.
- **1.6 Email Drafting:** Generates personalized greeting emails using the enriched contact data.
- **1.7 Data Storage:** Saves the structured contact data and optional email content into Google Sheets.
- **1.8 Optional Email Sending:** Sends the generated greeting email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives business card images and an optional message via a web form submission to trigger the workflow.

**Nodes Involved:**  
- On form submission  
- On form submission1 (active version)  

**Node Details:**  

- **On form submission1**  
  - Type: Form Trigger  
  - Role: Entry point that accepts a single image file (business card) and an optional message from a user.  
  - Configuration: Accepts JPEG/PNG files, requires one file, optional textarea for message.  
  - Input/Output: Outputs form data including image metadata and message.  
  - Edge Cases: Missing or unsupported file type; large image file size; empty optional message.  
  - Disabled Node: "On form submission" is disabled, superseded by "On form submission1".

---

#### 2.2 Image Preparation

**Overview:**  
Prepares the received image by extracting necessary metadata and formatting it for OCR processing.

**Nodes Involved:**  
- Edit Image1 (used with On form submission1)  
- Edit Image (used with On form submission - disabled path)

**Node Details:**  

- **Edit Image1**  
  - Type: Edit Image  
  - Role: Extracts image information such as dimensions and metadata to assist OCR processing.  
  - Configuration: Operation set to 'information', data property points to the uploaded image file.  
  - Input: The image file from form submission trigger.  
  - Output: Image metadata used downstream.  
  - Edge Cases: Corrupt images, unsupported formats, or missing metadata.

---

#### 2.3 OCR Processing

**Overview:**  
Extracts raw textual content from the business card image using Typhoon OCR technology.

**Nodes Involved:**  
- Typhoon OCR1 (Version 2)  
- OCR Agent1 (Version 2)  
- Typhoon OCR (Version 1)  
- OCR Agent (Version 1)  

**Node Details:**  

- **Typhoon OCR1**  
  - Type: Language Model Chat (Typhoon OCR)  
  - Role: Sends image metadata and file to Typhoon OCR model for text extraction.  
  - Configuration: Uses ‘typhoon-ocr-preview’ model with low temperature (0.1) for deterministic output.  
  - Input: Image metadata from Edit Image node.  
  - Output: Raw text extracted from the image.  
  - Credentials: Uses Orn’s Typhoon account credentials for API access.  
  - Edge Cases: Poor image quality, unreadable text, OCR failures or timeouts.

- **OCR Agent1**  
  - Type: Langchain Agent  
  - Role: Processes image and raw text to build a markdown and JSON representation of the document layout and content.  
  - Configuration: Custom prompt instructs to reconstruct document accurately, outputting JSON with `natural_text` key.  
  - Input: Image metadata and raw text from Typhoon OCR1.  
  - Output: Structured text for parsing.  
  - Edge Cases: Partial text extraction, layout misinterpretation.

- **Typhoon OCR & OCR Agent** nodes are analogous for Version 1, with similar roles and configurations but different model versions.

---

#### 2.4 Contact Information Parsing

**Overview:**  
Transforms raw OCR textual data into a structured JSON object containing contact fields, distinguishing English and Thai where applicable.

**Nodes Involved:**  
- Contact Info Parser1 (Version 2)  
- Contact Info Parser (Version 1)  

**Node Details:**  

- **Contact Info Parser1**  
  - Type: Chain LLM  
  - Role: Extracts specified contact fields (names, job titles, organization, email, phone, website) from raw text.  
  - Configuration: Prompt instructs to differentiate English and Thai names, parse nicknames, format phone numbers, and output strictly JSON without markdown.  
  - Input: Text output from OCR Agent1.  
  - Output: JSON object with contact fields using exact field names.  
  - Edge Cases: Incomplete data, ambiguous name parsing, malformed phone numbers.  
  - Version Differences: Contact Info Parser is analogous for Version 1 with similar prompt and function.

---

#### 2.5 Contact Enrichment

**Overview:**  
Classifies and enriches the parsed contact data with additional fields such as job type, level, sector, and optionally fetches profile summaries via internet search.

**Nodes Involved:**  
- Search Agent (Version 2, with Search API)  
- Contact Info Enricher (Version 1, without Search API)  

**Node Details:**  

- **Search Agent**  
  - Type: Langchain Agent  
  - Role: Uses internet search (SerpAPI) to enrich contact JSON with marketing categories and a profile summary.  
  - Configuration: Prompt guides classification into job_type, job_level, sector, and retrieval of profile summary from web.  
  - Input: Parsed contact JSON from Contact Info Parser1.  
  - Output: Enriched JSON with additional fields including `profile_summary`.  
  - Credentials: Requires SerpAPI account credentials.  
  - Edge Cases: No search results, inconsistent data, API rate limits or failures.

- **Contact Info Enricher**  
  - Type: Chain LLM  
  - Role: Classifies job_type, job_level, and sector based on parsed contact info without internet lookup.  
  - Input: Parsed contact JSON from Contact Info Parser.  
  - Output: JSON with classification fields (job_type, job_level, sector).  
  - Edge Cases: Ambiguous or missing job titles or organization names.

---

#### 2.6 Email Drafting

**Overview:**  
Generates a short, friendly greeting email message based on the enriched contact details and optional message from the form.

**Nodes Involved:**  
- Greeting Email Composer (Version 2)  
- Greeting Email Composer1 (Version 1)  

**Node Details:**  

- **Greeting Email Composer**  
  - Type: Chain LLM  
  - Role: Creates JSON-formatted email containing `subject` and `body` fields with a personalized greeting.  
  - Configuration: Uses structured contact data and optional message; email content is in English, concise, friendly, and includes a disclaimer about automation.  
  - Input: Enriched contact JSON plus optional message from form submission.  
  - Output: JSON with email subject and body.  
  - Edge Cases: Missing first name, empty optional message, incomplete contact data.

- **Greeting Email Composer1** is used in Version 1 with similar parameters.

---

#### 2.7 Data Storage

**Overview:**  
Appends the structured contact information and optional email details into a Google Sheets spreadsheet for CRM or lead management.

**Nodes Involved:**  
- Add Contact to Google Sheet (Version 1)  
- Add Contact to Google Sheet1 (Version 2)  

**Node Details:**  

- **Add Contact to Google Sheet1**  
  - Type: Google Sheets node  
  - Role: Appends contact fields and email content to a specified Google Sheets document and sheet.  
  - Configuration: Maps all parsed and enriched fields including `profile_summary`, original message, email subject, and body to columns.  
  - Input: JSON contact object from Text to JSON2 node.  
  - Credentials: Uses Google Sheets OAuth2 credentials.  
  - Edge Cases: Sheet access errors, quota exceeded, invalid mapping.

- **Add Contact to Google Sheet** is analogous for Version 1 with fewer columns (no profile summary or email content columns).

---

#### 2.8 Optional Email Sending

**Overview:**  
Optionally sends the personalized greeting email to the contact’s email address using Gmail.

**Nodes Involved:**  
- Send a message  

**Node Details:**  

- **Send a message**  
  - Type: Gmail node  
  - Role: Sends an email using Gmail OAuth2 credentials to the extracted email address with the generated subject and body.  
  - Configuration: Dynamically sets recipient email, subject, and message content from previous nodes.  
  - Input: Email JSON from Text to JSON node.  
  - Credentials: Gmail OAuth2 credentials required.  
  - Edge Cases: Sending failures, invalid email address, quota limits.

- This node is optional and disconnected by default; users can connect it to enable email sending.

---

### 3. Summary Table

| Node Name              | Node Type                                    | Functional Role                         | Input Node(s)            | Output Node(s)                  | Sticky Note                                                                                               |
|------------------------|----------------------------------------------|---------------------------------------|--------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------|
| On form submission1     | Form Trigger                                 | Receive image and optional message    | —                        | Edit Image1                   |                                                                                                           |
| Edit Image1            | Edit Image                                   | Extract image metadata                 | On form submission1       | OCR Agent1                   |                                                                                                           |
| Typhoon OCR1           | Langchain LLM Chat (Typhoon OCR)             | Extract raw OCR text                   | Edit Image1 (indirect)    | OCR Agent1 (via ai_languageModel) |                                                                                                           |
| OCR Agent1             | Langchain Agent                              | Process OCR text and layout           | Typhoon OCR1              | Contact Info Parser1          |                                                                                                           |
| Contact Info Parser1   | Chain LLM                                    | Parse OCR text to structured contact  | OCR Agent1                | Search Agent                  |                                                                                                           |
| Search Agent           | Langchain Agent (with SerpAPI)               | Enrich contact via internet search    | Contact Info Parser1      | Text to JSON2                 |                                                                                                           |
| Text to JSON2          | Set Node                                     | Convert enriched text to JSON object  | Search Agent              | Greeting Email Composer       |                                                                                                           |
| Greeting Email Composer | Chain LLM                                    | Generate personalized greeting email  | Text to JSON2             | Text to JSON                 |                                                                                                           |
| Text to JSON           | Set Node                                     | Parse email JSON string                | Greeting Email Composer   | Add Contact to Google Sheet1  |                                                                                                           |
| Add Contact to Google Sheet1 | Google Sheets                              | Save contact and email data            | Text to JSON              | —                             |                                                                                                           |
| Send a message         | Gmail                                        | (Optional) Send greeting email        | Text to JSON              | —                             | Optional nodes to facilitate sending emails                                                              |
| Sticky Note            | Sticky Note                                  | Version 1 description and instructions| —                        | —                             | "## Version 1: Without Search API..."                                                                     |
| Sticky Note1           | Sticky Note                                  | Version 2 description and instructions| —                        | —                             | "## Version 2: With Search API..."                                                                        |
| Sticky Note2           | Sticky Note                                  | Workflow general overview, branding   | —                        | —                             | "# OCR Business Card Reader with Thai Support – Enrich & Save Contacts to CRM..."                         |
| Typhoon OCR            | Langchain LLM Chat (Typhoon OCR)             | Version 1 OCR text extraction         | Edit Image (disabled path)| OCR Agent (disabled path)     |                                                                                                           |
| OCR Agent              | Langchain Agent                              | Version 1 OCR text processing         | Typhoon OCR               | Contact Info Parser (disabled path) |                                                                                                           |
| Contact Info Parser    | Chain LLM                                    | Version 1 contact parsing              | OCR Agent                 | Contact Info Enricher         |                                                                                                           |
| Contact Info Enricher  | Chain LLM                                    | Version 1 contact enrichment           | Contact Info Parser       | Greeting Email Composer1      |                                                                                                           |
| Greeting Email Composer1 | Chain LLM                                  | Version 1 email drafting               | Contact Info Enricher     | Text to JSON 2                |                                                                                                           |
| Text to JSON 2         | Set Node                                     | Version 1 JSON parsing                 | Greeting Email Composer1  | Add Contact to Google Sheet   |                                                                                                           |
| Add Contact to Google Sheet | Google Sheets                              | Version 1 save contact data            | Text to JSON 2            | —                             |                                                                                                           |
| On form submission     | Form Trigger                                 | Disabled older trigger                 | —                        | Edit Image (disabled path)    |                                                                                                           |
| Edit Image             | Edit Image                                   | Disabled older image processing       | On form submission        | OCR Agent (disabled path)     |                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: `Form Trigger`  
   - Configure form with the title “Name card collector”  
   - Add two fields:  
     - File upload field labeled “nameCard” (accept `.jpg, .png, .jpeg`, single file, required)  
     - Textarea field labeled “Message (Optional)”, not required  
   - Save webhook URL for external form access  

2. **Add Edit Image Node**  
   - Type: `Edit Image`  
   - Operation: `information`  
   - Data Property Name: set to the file property from the form trigger (e.g., `nameCard`)  
   - Connect form trigger output to this node  

3. **Add Typhoon OCR Node**  
   - Type: `Langchain LLM Chat` (Typhoon OCR model)  
   - Model: `typhoon-ocr-preview` or `typhoon-ocr-preview-techsauce-workshop` depending on version  
   - Set temperature to 0.1 for consistent OCR results  
   - Credentials: Configure OpenAI API credentials with Typhoon account  
   - Connect output of Edit Image node to this node’s language model input  

4. **Add OCR Agent Node**  
   - Type: `Langchain Agent`  
   - Prompt: Use the provided custom prompt to reconstruct document layout and return JSON with key `natural_text`  
   - Enable passthrough of binary images  
   - Connect Typhoon OCR output to this agent node  

5. **Add Contact Info Parser Node**  
   - Type: `Chain LLM`  
   - Prompt: Configure with instructions to extract contact fields (English and Thai names, job titles, organization, email, phone, website) and to return pure JSON without markdown  
   - Connect OCR Agent output to this node  

6. **Add Contact Enrichment Node**  
   - For Version 1 (without Search API):  
     - Type: `Chain LLM`  
     - Prompt: Classify job_type, job_level, sector based on contact info fields  
     - Connect Contact Info Parser output to this node  
   - For Version 2 (with Search API):  
     - Add SerpAPI node with valid credentials  
     - Add Search Agent node (Langchain Agent) with prompt to enrich contact info and fetch profile summary using SerpAPI  
     - Connect Contact Info Parser output to Search Agent, and Search Agent to SerpAPI node  

7. **Add Set Node to Parse JSON**  
   - Type: `Set`  
   - Assign a new field `contact_obj` by parsing the output text (removing markdown code fences if present) from enrichment node  
   - Connect enrichment output to this node  

8. **Add Greeting Email Composer Node**  
   - Type: `Chain LLM`  
   - Prompt: Generate a short, friendly email with JSON output containing `subject` and `body` fields, using contact info and optional message from form  
   - Connect parsed JSON node output to this node  

9. **Add Another Set Node to Parse Email JSON**  
   - Type: `Set`  
   - Assign a new field `contact_obj` by parsing the JSON email output string  
   - Connect Greeting Email Composer output to this node  

10. **Add Google Sheets Node**  
    - Type: `Google Sheets`  
    - Operation: Append  
    - Document ID: Google Sheet where contacts will be saved  
    - Sheet Name: Specify the sheet (e.g., `gid=0`)  
    - Mapping: Map all parsed contact fields plus optional message, email subject, and email content to respective columns  
    - Credentials: Configure Google Sheets OAuth2 credentials  
    - Connect output of last Set node to this node  

11. **(Optional) Add Gmail Node to Send Email**  
    - Type: `Gmail`  
    - Parameters: `Send To` set to contact email, `Subject` and `Message` set to generated email fields  
    - Credentials: Configure Gmail OAuth2 credentials  
    - Connect parsed email JSON node output to this node  

12. **Add Sticky Notes**  
    - Add descriptive sticky notes explaining Version 1 and Version 2 flows, instructions, and usage tips as per original content  

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| The workflow supports Thai and English business cards using Typhoon OCR and LLM models.                             | Branding and core technology: [opentyphoon.ai](https://opentyphoon.ai)                                    |
| Two versions are available: a cost-free Typhoon-only version and an enriched version using SerpAPI Search API.        | Search API may incur additional costs.                                                                   |
| Includes optional greeting email generation and sending to facilitate event lead engagement or follow-up.           | Email sender disclaimer is included for demo purposes only.                                              |
| For detailed setup and API key configuration, see official Typhoon n8n integration guide.                           | [Typhoon n8n Integration Guide](https://opentyphoon.ai/blog/en/n8n-typhoon-integration-guide)              |
| To enable email sending, connect the Gmail node and configure OAuth2 credentials properly.                           | Optional step, disconnected by default.                                                                  |
| The prompt texts embedded in LLM nodes can be adjusted to customize parsing and enrichment according to business needs. | Allows flexible adaptation to domain-specific data extraction and classification.                         |

---

This documentation provides a comprehensive, structured understanding of the workflow, enabling reproduction, modification, and anticipation of integration challenges.