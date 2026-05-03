Extract Business Card Data from Telegram to Google Sheets with OpenRouter AI Vision

https://n8nworkflows.xyz/workflows/extract-business-card-data-from-telegram-to-google-sheets-with-openrouter-ai-vision-9754


# Extract Business Card Data from Telegram to Google Sheets with OpenRouter AI Vision

---

### 1. Workflow Overview

This workflow automates the extraction of structured contact details from business card images sent via Telegram. It leverages AI Vision capabilities through OpenRouter’s AI to analyze images, extract key business card fields, and then logs the parsed data into a Google Sheets spreadsheet. It is optimized especially for Japanese business cards to maximize OCR accuracy and data relevance.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives incoming Telegram messages, specifically filtering for images representing business cards.
- **1.2 AI Vision Processing:** Uses a LangChain AI Vision Agent with OpenRouter’s vision model to analyze the business card image and extract structured information.
- **1.3 Output Parsing:** Standardizes and cleans the AI output into a structured JSON format for consistent downstream use.
- **1.4 Data Logging:** Adds or updates the extracted contact data into a Google Sheets document, preserving existing entries and appending new ones.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon receiving a Telegram message and filters to ensure that the input contains either text or images, focusing on messages that contain business card images.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Check Input Type  

- **Node Details:**

  - **Telegram Trigger**  
    - *Type & Role:* Telegram trigger node that listens for new messages.  
    - *Configuration:* Set to capture all "message" updates; automatically downloads attached images.  
    - *Expressions/Variables:* Outputs the entire Telegram message JSON including image file IDs.  
    - *Input/Output:* No input; outputs to "Check Input Type".  
    - *Potential Failures:* Telegram API auth errors, message format changes, or missing images in messages.  
    - *Credentials:* Telegram API OAuth2 configured.

  - **Check Input Type**  
    - *Type & Role:* Conditional node (IF) that filters messages.  
    - *Configuration:* Checks if the incoming message contains text (`$json.message.text`) that does not exist (i.e., message is an image or non-text).  
    - *Expressions/Variables:* Expression `"={{ $json.message.text }}"` checked for non-existence.  
    - *Input/Output:* Input from Telegram Trigger; outputs to AI Vision Agent if condition passes.  
    - *Edge Cases:* Messages without images or text will not proceed; messages with unsupported media types fail silently.  

#### 1.2 AI Vision Processing

- **Overview:**  
  An AI Vision Agent analyzes the business card image using OpenRouter’s vision model. It extracts specified fields prioritized for Japanese text, returning structured information about the contact.

- **Nodes Involved:**  
  - AI Vision Agent  
  - OpenAI Vision Model  

- **Node Details:**

  - **AI Vision Agent**  
    - *Type & Role:* LangChain Agent designed for vision tasks.  
    - *Configuration:*  
      - Prompt includes instructions to analyze the image URL (extracted from Telegram photo file ID).  
      - System message emphasizes prioritization of Japanese text if present, fallback to romanized/English otherwise.  
      - Extracts fields: Company Name, Department, Job Title, Full Name, Postal Code, Address, Phone Number, Fax Number, Email Address, Website URL, Mobile Phone Number.  
    - *Expressions/Variables:* Uses `{{ $('Telegram Trigger').item.json.message.photo[0].file_id }}` to reference the image.  
    - *Input/Output:* Input from Check Input Type node; outputs to Ingredient Parser and finally to "Add to Google Sheet".  
    - *Version Requirements:* Requires LangChain Agent enabled with OpenRouter API credentials.  
    - *Failure Modes:* API timeouts, image download failures, OCR inaccuracies, or prompt parsing errors.

  - **OpenAI Vision Model**  
    - *Type & Role:* Language model node set to OpenRouter API with vision capabilities.  
    - *Configuration:* Temperature set to 0.3 for deterministic output.  
    - *Input/Output:* Connected as language model for AI Vision Agent.  
    - *Credentials:* OpenRouter API key configured.  
    - *Failures:* API quota exceeded, invalid credentials, or network issues.

#### 1.3 Output Parsing

- **Overview:**  
  Parses AI Vision Agent’s unstructured output into a strict JSON schema to enforce data consistency and clean formatting before logging.

- **Nodes Involved:**  
  - Ingredient Parser  

- **Node Details:**

  - **Ingredient Parser**  
    - *Type & Role:* Structured output parser for LangChain AI outputs.  
    - *Configuration:* Manual schema defining all expected fields with example data for validation. Auto-fix disabled to avoid silent corrections.  
    - *Schema Fields:* company_name, department, job_title, full_name, postal_code, address, phone_number, mobile_phone_number, fax_number, email, website_url.  
    - *Input/Output:* Takes AI Vision Agent output; feeds parsed results back as output for further use.  
    - *Failure Modes:* Parsing errors if AI output deviates from schema, missing fields, or unexpected formats.

#### 1.4 Data Logging

- **Overview:**  
  Adds or updates contact details in a Google Sheets document. Uses a conditional append or update approach to maintain consistent and up-to-date records.

- **Nodes Involved:**  
  - Add to Google Sheet  

- **Node Details:**

  - **Add to Google Sheet**  
    - *Type & Role:* Google Sheets node to append or update rows.  
    - *Configuration:*  
      - Writes to a specific Google Sheet identified by document ID and sheet name (gid=0).  
      - Mapping is defined for each contact field, including automatic date insertion (`$today`).  
      - Matching column is “company_name” to update existing entries or append new ones.  
      - Conversion to string disabled to preserve data types.  
    - *Expressions/Variables:* Uses expressions like `={{ $('AI Vision Agent').item.json.output['Full Name'] }}` to reference parsed data.  
    - *Input/Output:* Input from AI Vision Agent; no further output.  
    - *Credentials:* Google Sheets OAuth2 credentials configured.  
    - *Failures:* Authentication errors, rate limits, schema mismatches, or document access issues.

---

### 3. Summary Table

| Node Name          | Node Type                            | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                                        |
|--------------------|------------------------------------|--------------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger    | Telegram Trigger                   | Receives Telegram messages with images     | -                      | Check Input Type        | ## ①Send business card image Send a business card image to your Telegram bot                                                      |
| Check Input Type    | If Node                           | Filters messages to only process images    | Telegram Trigger        | AI Vision Agent         | ## ②Business Card Image Input Filter Checks if the incoming Telegram message contains an image or text                            |
| AI Vision Agent     | LangChain Agent (AI Vision)       | Analyzes business card image, extracts data| Check Input Type        | Add to Google Sheet     | ## ③Analyze business card image The structured output is cleaned and standardized by the parser                                   |
| OpenAI Vision Model | LangChain LM (OpenRouter API)     | Provides AI vision capabilities for agent | AI Vision Agent (LM)    | AI Vision Agent         |                                                                                                                                   |
| Ingredient Parser   | LangChain Output Parser            | Parses AI output into structured JSON      | AI Vision Agent (Output Parser) | AI Vision Agent  | ## ③Analyze business card image The structured output is cleaned and standardized by the parser                                   |
| Add to Google Sheet | Google Sheets                     | Appends or updates contact data            | AI Vision Agent         | -                      | ## ④Log the parsed business card details into Google Sheets Google Sheets node appends or updates the contact data automatically  |
| Sticky Note        | Sticky Note                       | Informational note                          | -                      | -                      | ## ①Send business card image Send a business card image to your Telegram bot                                                      |
| Sticky Note1       | Sticky Note                       | Informational note                          | -                      | -                      | ## ②Business Card Image Input Filter Checks if the incoming Telegram message contains an image or text                            |
| Sticky Note2       | Sticky Note                       | Informational note                          | -                      | -                      | ## ③Analyze business card image The structured output is cleaned and standardized by the parser                                   |
| Sticky Note3       | Sticky Note                       | Informational note                          | -                      | -                      | ## ④Log the parsed business card details into Google Sheets Google Sheets node appends or updates the contact data automatically  |
| Sticky Note4       | Sticky Note                       | Workflow description and feature summary  | -                      | -                      | ## Business Card OCR Auto-Logging (Telegram → AI → Google Sheets) Description and key features of the workflow                   |
| Sticky Note5       | Sticky Note                       | Use cases description                       | -                      | -                      | **Use Cases:** 💼Sales & CRM: Automatically build and update your client database from received business cards, etc.               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: message  
     - Enable "Download" to true to download images automatically  
   - Credentials: Telegram API OAuth2 (configured with your Telegram bot token)  
   - Position: Start node

2. **Create Check Input Type Node (If Node)**  
   - Type: If  
   - Parameters:  
     - Condition: Check if `{{$json.message.text}}` does **not** exist (string operation: "notExists")  
   - Connect output of Telegram Trigger to input of this node

3. **Create AI Vision Agent Node (LangChain Agent)**  
   - Type: LangChain Agent (AI Vision)  
   - Parameters:  
     - Text prompt:  
       ```
       Analyze the provided business card image or text and accurately identify all of the following information.List them clearly.

       Image URL: {{ $('Telegram Trigger').item.json.message.photo[0].file_id }}
       ```
     - System Message:  
       ```
       You are a business card data extraction expert. Analyze the provided business card image or text and accurately identify all of the following information. Return the results as a structured list.

       Important Rules:
       Always prioritize Japanese text when it is available.
       Only use romanized or English text if no Japanese text exists for that field.
       If both Japanese and romanized text are present, return only the Japanese version.

       Fields to extract:
       -Company Name
       -Department
       -Job Title
       -Full Name
       -Postal Code
       -Address
       -Phone Number
       -Fax Number
       -Email Address
       -Website URL
       -Mobile Phone Number
       ```
     - Output parser: Enabled (to connect next node)  
   - Connect output of Check Input Type to this node

4. **Create OpenAI Vision Model Node (LangChain LM)**  
   - Type: LangChain LM Chat OpenRouter  
   - Parameters:  
     - Temperature: 0.3  
   - Credentials: OpenRouter API key  
   - Connect as language model to AI Vision Agent node (via ai_languageModel input)

5. **Create Ingredient Parser Node (LangChain Output Parser Structured)**  
   - Type: LangChain Output Parser Structured  
   - Parameters:  
     - Auto Fix: false  
     - Schema Type: Manual  
     - Input Schema (JSON example):  
       ```
       {
         "company_name": "Example Company Ltd.",
         "department": "Sales",
         "job_title": "Sales Manager",
         "full_name": "Taro Yamada",
         "postal_code": "100-0001",
         "address": "1-1-1 Marunouchi, Chiyoda-ku, Tokyo",
         "phone_number": "+81-3-0000-0000",
         "mobile_phone_number": "+81-90-0000-0000",
         "fax_number": "+81-3-1111-1111",
         "email": "example@company.com",
         "website_url": "https://example.com"
       }
       ```
   - Connect as output parser to AI Vision Agent node (via ai_outputParser input)

6. **Create Add to Google Sheet Node**  
   - Type: Google Sheets  
   - Parameters:  
     - Operation: appendOrUpdate  
     - Document ID: Your Google Sheets document ID (e.g., `1rYf2kqfmMRUtpFAb5YdhOcsvGg8Gr1CTbO-lvrd48m4`)  
     - Sheet Name: `gid=0` (or your target sheet)  
     - Columns Mapping: Define columns matching schema fields with expressions:  
       - `date`: `={{ $today }}`  
       - `company_name`: `={{ $('AI Vision Agent').item.json.output['Company Name'] }}`  
       - `department`: `={{ $('AI Vision Agent').item.json.output['Department'] }}`  
       - `job_title`: `={{ $('AI Vision Agent').item.json.output['Job Title'] }}`  
       - `full_name`: `={{ $('AI Vision Agent').item.json.output['Full Name'] }}`  
       - `postal_code`: `={{ $('AI Vision Agent').item.json.output['Postal Code'] }}`  
       - `address`: `={{ $('AI Vision Agent').item.json.output['Address'] }}`  
       - `phone_number`: `={{ $('AI Vision Agent').item.json.output['Phone Number'] }}`  
       - `mobile_phone_number`: `={{ $('AI Vision Agent').item.json.output['Mobile Phone Number'] }}`  
       - `fax_number`: `={{ $('AI Vision Agent').item.json.output['Fax Number'] }}`  
       - `email`: `={{ $('AI Vision Agent').item.json.output['Email Address'] }}`  
       - `website_url`: `={{ $('AI Vision Agent').item.json.output['Website URL'] }}`  
     - Matching Columns: `company_name` (to update existing entries)  
     - Attempt to Convert Types: false  
     - Convert Fields to String: false  
   - Credentials: Google Sheets OAuth2 API  
   - Connect output of AI Vision Agent to this node

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow is optimized for Japanese business cards with a focus on prioritizing Japanese text over romanized or English when both are available.                                                                                                                                                                                                                                                                           | See AI Vision Agent System Message prompt in node configuration                                        |
| The workflow enables automatic logging and updating of contacts, ideal for CRM, back office digitization, marketing lead collection, and AI/OCR research.                                                                                                                                                                                                                                                                       | Sticky Note5 content                                                                                     |
| For detailed setup, ensure Google Sheets credentials have access to the target document and Telegram API credentials are correctly configured with bot permissions to receive messages. OpenRouter API credentials must have enabled vision capabilities.                                                                                                                                                                      | Credential setup recommendations                                                                         |
| Useful link for Google Sheets node documentation and OAuth2 setup: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleSheets/                                                                                                                                                                                                                                                                               | n8n Docs                                                                                               |
| OpenRouter API documentation and pricing information: https://openrouter.ai/                                                                                                                                                                                                                                                                                                                                                   | OpenRouter official site                                                                                 |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---