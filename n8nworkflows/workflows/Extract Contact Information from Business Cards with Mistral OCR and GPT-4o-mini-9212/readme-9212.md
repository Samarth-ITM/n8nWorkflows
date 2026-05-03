Extract Contact Information from Business Cards with Mistral OCR and GPT-4o-mini

https://n8nworkflows.xyz/workflows/extract-contact-information-from-business-cards-with-mistral-ocr-and-gpt-4o-mini-9212


# Extract Contact Information from Business Cards with Mistral OCR and GPT-4o-mini

---

### 1. Workflow Overview

This workflow automates the extraction of contact information from business cards uploaded as images or PDFs. It uses the Mistral OCR API to convert the uploaded document into text, then applies an AI agent powered by OpenAI’s GPT-4o-mini model to parse and structure the extracted text into defined contact fields. Finally, it upserts the structured data into an n8n Data Table keyed by the email address to create or update contact entries.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception:** Handles user input via a web form to upload business card files.
- **1.2 Document Conversion:** Converts uploaded files to base64 format for API consumption.
- **1.3 OCR Processing:** Sends the base64 document to Mistral OCR API to extract text.
- **1.4 AI Parsing:** Uses a LangChain AI Agent with GPT-4o-mini to interpret OCR text and parse structured contact information.
- **1.5 Data Persistence:** Upserts the structured contact information into an n8n Data Table keyed by email.
- **1.6 Memory Management:** Maintains session-based memory in the AI Agent to improve context handling.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures business card uploads through a web form, triggering the workflow.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note (documentation)

- **Node Details:**  

  **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point, starts workflow on user file upload  
  - Configuration:  
    - Form titled "Business card scanner"  
    - Single required file field labeled "business card" (accepts PDF or image)  
  - Inputs: External user via webhook  
  - Outputs: Binary file data for the uploaded document  
  - Edge cases: Missing file upload, unsupported file types  
  - No version-specific requirements

  **Sticky Note**  
  - Provides user-facing instructions: "# upload via a web form pdf, jpg, ...."

---

#### 2.2 Document Conversion

- **Overview:**  
  Converts the uploaded binary file into a base64 encoded string suitable for API submission.

- **Nodes Involved:**  
  - Data to base64  
  - Sticky Note (documentation)

- **Node Details:**  

  **Data to base64**  
  - Type: Extract from File  
  - Role: Convert binary file to base64 property  
  - Configuration:  
    - Operation: binaryToProperty  
    - Binary Property: "Business_card" (matching form upload)  
    - Output: base64 string stored in JSON property `$json.data`  
  - Inputs: Output from "On form submission" node  
  - Outputs: JSON with base64 data for OCR API  
  - Edge cases: File read errors, corrupt files

  **Sticky Note1**  
  - Describes the OCR step and links to Mistral OCR docs:  
    "[Docs](https://mistral.ai/news/mistral-ocr)"

---

#### 2.3 OCR Processing

- **Overview:**  
  Sends the base64 encoded document to Mistral OCR API to extract text content from the business card.

- **Nodes Involved:**  
  - Mistral OCR API  
  - JSON Parser  
  - Sticky Note (documentation)

- **Node Details:**  

  **Mistral OCR API**  
  - Type: HTTP Request  
  - Role: Call external OCR API to parse document image  
  - Configuration:  
    - POST to `https://api.mistral.ai/v1/ocr`  
    - JSON body includes model "mistral-ocr-latest" and document URL with base64 data  
    - Response expected as a file, output property `"ocr_output.json"`  
    - Authentication: HTTP Bearer token via Mistral OCR credentials  
  - Inputs: Base64 data from "Data to base64" node  
  - Outputs: Binary JSON file with OCR results  
  - Edge cases: HTTP errors, auth failures, invalid responses, timeouts

  **JSON Parser**  
  - Type: Extract from File  
  - Role: Parse binary JSON response from OCR into usable JSON object  
  - Configuration:  
    - Operation: fromJson  
    - Binary property: "ocr_output.json"  
  - Inputs: Output from "Mistral OCR API"  
  - Outputs: Parsed JSON with OCR text in `$json.data.pages[0].markdown`  

  **Sticky Note2**  
  - Notes about extracting content and OpenAI API key setup:  
    "[Setup an openAI API key](https://platform.openai.com/api-keys)"

---

#### 2.4 AI Parsing

- **Overview:**  
  Uses a LangChain AI Agent with GPT-4o-mini to interpret the OCR text and extract structured contact information according to a defined schema.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Simple Memory  
  - Sticky Note (documentation)

- **Node Details:**  

  **AI Agent**  
  - Type: LangChain Agent  
  - Role: Main AI logic node that processes OCR text and extracts structured fields  
  - Configuration:  
    - Input text template: `"Business Card: {{ $json.data.pages[0].markdown }}"`  
    - System message: "You are a business card reader that reads the data from the card."  
    - Uses OpenAI Chat Model as language model  
    - Uses Structured Output Parser for JSON schema enforcement  
    - Connects to Simple Memory for session context based on submission time  
  - Inputs: OCR parsed JSON from "JSON Parser"  
  - Outputs: JSON with structured fields (`firstname`, `name`, `company`, `web`, `email`, `street`, `postcode`, `place`, `phone`, `mobile`, `jobdescription`)  
  - Edge cases: Parsing errors, AI API failures, rate limits, malformed input

  **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat  
  - Role: Provides GPT-4o-mini language capabilities to AI Agent  
  - Configuration: Model set to "gpt-4o-mini"  
  - Credentials: OpenAI API key  
  - Inputs: AI Agent prompts  
  - Outputs: AI-generated chat completions

  **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Enforces output JSON schema for contact fields  
  - Configuration: Schema example with empty strings for all contact fields to ensure consistent output

  **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversation/session memory keyed by form submission timestamp  
  - Configuration: Session key set to `{{ $('Data to base64').item.json.submittedAt }}`  
  - Inputs: Memory attached to AI Agent for context preservation  

  **Sticky Note3**  
  - Explains database upsert using email as key

---

#### 2.5 Data Persistence

- **Overview:**  
  Saves or updates the extracted contact information into an n8n Data Table named "business_cards" using the email field as the unique key.

- **Nodes Involved:**  
  - Upsert row(s)  
  - Sticky Note (documentation)

- **Node Details:**  

  **Upsert row(s)**  
  - Type: Data Table (n8n native)  
  - Role: Insert or update contact records in Data Table  
  - Configuration:  
    - Data Table: "business_cards" (ID: WTKp1ogUGN0k0pD6)  
    - Mapping fields from AI Agent output JSON to Data Table columns (`firstname`, `name`, `company`, `jobdescription`, `email`, `phone`, `mobile`, `street`, `postcode`, `place`)  
    - Upsert operation keyed by `email` field to avoid duplicates  
  - Inputs: Structured JSON output from AI Agent  
  - Outputs: Data Table operation result  
  - Edge cases: Duplicate emails with conflicting data, missing email field (which breaks uniqueness), database permission errors

  **Sticky Note4**  
  - Provides high-level notes about the whole workflow and setup instructions including credential configuration and Data Table creation:  
  ```
  ## 🧠 Business Card Scanner
  
  This workflow automatically extracts contact details from uploaded business card images or PDFs using Mistral OCR and OpenAI GPT-4o-mini, then saves the structured data into an n8n Data Table.
  
  Setup:
  	1. Add your Mistral API key (HTTP Bearer Auth).
  	2. Add your OpenAI API key (OpenAI Credentials).
  	3. Create a Data Table named business_cards with fields for name, company, email, phone, etc.
  	4. Enable the Form Trigger to upload business cards and test the workflow.
  
  ✅ Result: business cards are instantly digitized into searchable contact data.
  ```

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                             | Input Node(s)            | Output Node(s)      | Sticky Note                                                  |
|---------------------|----------------------------------|---------------------------------------------|--------------------------|---------------------|--------------------------------------------------------------|
| On form submission   | Form Trigger                     | Entry point, receives uploaded business card | -                        | Data to base64       | # upload via a web form pdf, jpg, ....                       |
| Data to base64       | Extract from File                | Converts uploaded file to base64 string      | On form submission        | Mistral OCR API      | # OCR via Mistral OCR API [Docs](https://mistral.ai/news/mistral-ocr) |
| Mistral OCR API      | HTTP Request                    | Sends document to OCR API for text extraction | Data to base64            | JSON Parser          |                                                              |
| JSON Parser          | Extract from File               | Parses OCR JSON file response                 | Mistral OCR API           | AI Agent             | # Extract the defined content [Setup an openAI API key](https://platform.openai.com/api-keys) |
| AI Agent             | LangChain Agent                 | Parses OCR text with GPT-4o-mini to structured data | JSON Parser, Simple Memory, OpenAI Chat Model, Structured Output Parser | Upsert row(s)        | # upsert the database using the email address as key criteria |
| OpenAI Chat Model    | LangChain OpenAI Chat           | Provides GPT-4o-mini model for AI Agent      | AI Agent (ai_languageModel) | AI Agent             |                                                              |
| Structured Output Parser | LangChain Output Parser       | Enforces JSON schema for structured output   | AI Agent (ai_outputParser) | AI Agent             |                                                              |
| Simple Memory        | LangChain Memory Buffer Window  | Maintains session memory context              | -                        | AI Agent             |                                                              |
| Upsert row(s)        | Data Table                     | Upserts parsed contact data into database     | AI Agent                  | -                   |                                                              |
| Sticky Note          | Sticky Note                    | Documentation and instructions                | -                        | -                   | # upload via a web form pdf, jpg, ....                       |
| Sticky Note1         | Sticky Note                    | Documentation for OCR step                     | -                        | -                   | # OCR via Mistral OCR API [Docs](https://mistral.ai/news/mistral-ocr) |
| Sticky Note2         | Sticky Note                    | Documentation for AI parsing and OpenAI setup | -                        | -                   | # Extract the defined content [Setup an openAI API key](https://platform.openai.com/api-keys) |
| Sticky Note3         | Sticky Note                    | Documentation for database upsert              | -                        | -                   | # upsert the database using the email address as key criteria |
| Sticky Note4         | Sticky Note                    | High-level workflow overview and setup notes  | -                        | -                   | ## 🧠 Business Card Scanner ... full setup instructions       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Set form title: "Business card scanner"  
   - Add one required File field labeled "business card" (accepts PDF/image)  
   - Save and note webhook URL

2. **Add Extract from File Node (Data to base64)**  
   - Operation: binaryToProperty  
   - Binary Property Name: "Business_card" (matches upload)  
   - Output property: `data` (base64 string)  
   - Connect input from Form Trigger node

3. **Configure HTTP Request Node (Mistral OCR API)**  
   - Method: POST  
   - URL: `https://api.mistral.ai/v1/ocr`  
   - Authentication: HTTP Bearer Auth (add Mistral OCR API key credential)  
   - Body Type: JSON  
   - Body content:  
     ```json
     {
       "model": "mistral-ocr-latest",
       "document": {
         "type": "document_url",
         "document_url": "data:application/pdf;base64,{{ $json.data }}"
       },
       "include_image_base64": true
     }
     ```  
   - Response: Set response format to file, output property name "ocr_output.json"  
   - Connect input from Data to base64 node

4. **Add Extract from File Node (JSON Parser)**  
   - Operation: fromJson  
   - Binary Property Name: "ocr_output.json"  
   - Connect input from Mistral OCR API node

5. **Create LangChain OpenAI Chat Model Node**  
   - Model: Select "gpt-4o-mini"  
   - Credentials: Add OpenAI API key credential  
   - No additional options needed  
   - No direct input; connect via AI Agent node

6. **Create Structured Output Parser Node**  
   - Paste JSON schema example for contact fields with empty strings as placeholders:  
     ```json
     {
       "firstname": "",
       "name": "",
       "company": "",
       "web": "",
       "email": "",
       "street": "",
       "postcode": "",
       "place": "",
       "phone": "",
       "mobile": "",
       "jobdescription": ""
     }
     ```  
   - Connect output parser to AI Agent node

7. **Create Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Session Key: `={{ $('Data to base64').item.json.submittedAt }}` (captures submission timestamp)  
   - Session ID Type: customKey  
   - Connect ai_memory input to AI Agent node

8. **Create LangChain Agent Node**  
   - Text input: `"Business Card: {{ $json.data.pages[0].markdown }}"` (uses parsed OCR markdown text)  
   - System Message: "You are a business card reader that reads the data from the card."  
   - Prompt Type: define  
   - Enable Output Parser and attach Structured Output Parser node  
   - Attach OpenAI Chat Model and Simple Memory node as ai_languageModel and ai_memory inputs respectively  
   - Connect input from JSON Parser node

9. **Create Data Table Node (Upsert row(s))**  
   - Operation: upsert  
   - Data Table: Select or create a Data Table named "business_cards" with columns: firstname, name, company, jobdescription, email, phone, mobile, street, postcode, place  
   - Mapping Mode: Define below  
   - Map each column to corresponding field in AI Agent output JSON, e.g., `={{ $json.output.firstname }}` etc.  
   - Filtering Conditions: Use `email` field as unique key for upsert  
   - Connect input from AI Agent node

10. **Connect workflow nodes in order:**  
    On form submission → Data to base64 → Mistral OCR API → JSON Parser → AI Agent → Upsert row(s)  
    Attach OpenAI Chat Model, Structured Output Parser, and Simple Memory nodes as AI Agent internal dependencies

11. **Credential Setup:**  
    - Create HTTP Bearer Auth credential for Mistral OCR API with your API key  
    - Create OpenAI credential with your OpenAI API key

12. **Testing:**  
    - Enable the Form Trigger webhook  
    - Upload test business card files (PDF or images)  
    - Verify data is extracted and saved into the "business_cards" Data Table

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow automatically extracts contact details from uploaded business card images or PDFs using Mistral OCR and OpenAI GPT-4o-mini, then saves the structured data into an n8n Data Table. Setup includes adding API keys and creating the data table. | Workflow overview sticky note at start of workflow                                                             |
| Mistral OCR API documentation: https://mistral.ai/news/mistral-ocr                                                                                                                                                                            | Sticky Note1 linked from OCR processing block                                                                   |
| OpenAI API key setup instructions: https://platform.openai.com/api-keys                                                                                                                                                                       | Sticky Note2 linked from AI Parsing block                                                                        |
| The workflow uses the email address as the unique identifier when upserting contacts into the database to avoid duplicates.                                                                                                                   | Sticky Note3 and Upsert row(s) node settings                                                                     |
| The workflow supports PDF and common image file types for business cards uploaded via the web form.                                                                                                                                             | Sticky Note on form input                                                                                        |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It strictly adheres to content policies with no illegal or protected elements. All processed data is legal and public.