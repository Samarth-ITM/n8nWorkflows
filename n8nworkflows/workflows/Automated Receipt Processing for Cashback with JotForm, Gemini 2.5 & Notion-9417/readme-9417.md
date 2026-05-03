Automated Receipt Processing for Cashback with JotForm, Gemini 2.5 & Notion

https://n8nworkflows.xyz/workflows/automated-receipt-processing-for-cashback-with-jotform--gemini-2-5---notion-9417


# Automated Receipt Processing for Cashback with JotForm, Gemini 2.5 & Notion

---

## 1. Workflow Overview

This workflow automates the processing of purchase receipts submitted via a JotForm form to calculate cashback rewards for XQ Pharma customers. It extracts receipt images, performs OCR to convert images to text, uses Google Gemini 2.5 AI to clean and structure the extracted text, identifies eligible products, calculates cashback amounts, logs the transaction in Notion, and sends notification emails to both the customer and marketing team.

### Logical Blocks

- **1.1 Input Reception:** Triggered by JotForm form submission, retrieves all uploaded receipt files, and selects the latest receipt.
- **1.2 Receipt File Handling:** Downloads the latest receipt image and converts it to a base64 string for OCR processing.
- **1.3 OCR Processing:** Sends the base64 image to OCR.Space API to extract raw text from the receipt image.
- **1.4 AI Processing & Data Extraction:** Uses Google Gemini 2.5 to clean, structure, and extract key receipt data such as store name, purchase date, product details, and cashback calculations.
- **1.5 Conditional Cashback Check:** Proceeds only if cashback calculated is greater than zero.
- **1.6 Data Logging:** Stores customer and cashback details in a Notion database.
- **1.7 Notification Emails:** Sends a personalized cashback confirmation email to the customer and an internal notification email to the marketing team.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
Initiates the workflow on form submission and retrieves all receipt files submitted by the customer, extracting the most recent receipt's URL.

**Nodes Involved:**  
- JotForm Trigger  
- Fetch All Receipts  
- Gets Last Receipt

**Node Details:**  

- **JotForm Trigger**  
  - *Type:* Trigger node for JotForm submissions  
  - *Config:* Listens to submissions on a specific JotForm ID with a webhook  
  - *Expressions:* Uses form ID and webhook ID placeholders  
  - *Input:* HTTP webhook from JotForm upon form submission  
  - *Output:* JSON with form submission data including customer info and uploaded files  
  - *Failures:* Webhook misconfiguration, invalid form ID, network errors  
  - *Sticky Note:* "Starts the workflow upon form submission; provides customer and initial purchase data."

- **Fetch All Receipts**  
  - *Type:* HTTP Request  
  - *Config:* GET request to JotForm API endpoint that lists all files for the form, using API key for authentication  
  - *Expressions:* Uses form ID and API key placeholders  
  - *Input:* Trigger from JotForm Trigger  
  - *Output:* JSON list of all uploaded receipt files for the form  
  - *Failures:* API key invalid or insufficient permissions, network issues, rate limiting  
  - *Sticky Note:* "Requests the list of all uploaded receipt files from the JotForm API."

- **Gets Last Receipt**  
  - *Type:* Code (JavaScript)  
  - *Config:* Processes array of files from previous node, extracts last uploaded receipt URL, handles nested JSON structures from JotForm responses  
  - *Expressions:* Accesses `$json.files`, `$json.answers`, `$json.content` depending on structure  
  - *Input:* JSON file list from Fetch All Receipts  
  - *Output:* JSON object with `lastFile` containing the latest receipt file URL  
  - *Failures:* Unexpected data format, empty file list  
  - *Sticky Note:* "Extracts the URL for the most recently uploaded receipt from the file list."

---

### 2.2 Receipt File Handling

**Overview:**  
Downloads the latest receipt image file via HTTPS and converts it into a base64 string for use in OCR processing.

**Nodes Involved:**  
- Fetch Receipt File  
- Convert Image to base64

**Node Details:**  

- **Fetch Receipt File**  
  - *Type:* HTTP Request  
  - *Config:* Downloads receipt image file from the extracted URL (force HTTPS)  
  - *Expressions:* URL uses expression to replace http with https for secure download  
  - *Input:* Output of Gets Last Receipt node  
  - *Output:* Binary file content of the receipt image  
  - *Failures:* URL invalid or expired, network failure, file not found  
  - *Sticky Note:* "Downloads the actual receipt image file using the extracted URL."

- **Convert Image to base64**  
  - *Type:* Extract From File  
  - *Config:* Converts binary image data to base64 string stored in JSON under `imageBase64`  
  - *Input:* Binary file from Fetch Receipt File  
  - *Output:* JSON with base64 string of the image for OCR input  
  - *Failures:* Corrupted file data, conversion errors  
  - *Sticky Note:* "Encodes the downloaded image into a base64 string, preparing it for OCR."

---

### 2.3 OCR Processing

**Overview:**  
Sends the base64 encoded receipt image to OCR.Space API to extract raw textual content from the image.

**Nodes Involved:**  
- OCR.Space

**Node Details:**  

- **OCR.Space**  
  - *Type:* HTTP Request  
  - *Config:* POST multipart/form-data request to https://api.ocr.space/parse/image with parameters:  
    - `base64Image` from previous node  
    - Language set to Arabic (`ara`)  
    - OCR Engine version 1  
    - Filetype jpg  
  - *Headers:* API key authorization header  
  - *Input:* JSON with `imageBase64` from Convert Image to base64  
  - *Output:* JSON with raw OCR text under `ParsedResults[0].ParsedText`  
  - *Failures:* API key invalid, quota exceeded, malformed request, OCR engine errors  
  - *Sticky Note:* "Converts the receipt image into plain, readable text."

---

### 2.4 AI Processing & Data Extraction

**Overview:**  
Processes the raw OCR text using Google Gemini 2.5 to clean and organize the receipt data, identify XQ Pharma products, extract structured details, and compute cashback amounts.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Structured Output Parser  
- AI Agent

**Node Details:**  

- **Google Gemini Chat Model**  
  - *Type:* LangChain LM Chat Google Gemini  
  - *Config:* Uses model `models/gemini-2.5-flash-lite` with default options  
  - *Input:* Connected to AI Agent as language model backend  
  - *Output:* Passes processed text to AI Agent with output parser attached  
  - *Failures:* API credential issues, rate limiting, model unavailability

- **Structured Output Parser**  
  - *Type:* LangChain Output Parser (Structured JSON)  
  - *Config:* JSON schema example specifying receipt data structure with store name, purchase date, products array, and cashback totals  
  - *Input:* Output from Google Gemini Chat Model  
  - *Output:* Structured JSON data ready for downstream processing  
  - *Failures:* Parsing errors if AI output does not conform to schema

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Config:* Multi-step prompt including cleaning and organizing OCR text, identifying XQ Pharma products, extracting detailed data, and outputting valid JSON only  
  - *Expressions:* Uses OCR text input from OCR.Space node's `ParsedResults[0].ParsedText`  
  - *Input:* OCR text JSON and language model/structured parser nodes  
  - *Output:* JSON object with extracted receipt data (store_name, purchase_date, products, total_cashback)  
  - *Failures:* AI prompt misinterpretation, missing or ambiguous OCR data, API errors  
  - *Sticky Note:* "Uses Gemini to clean the raw OCR text, identify XQ products, and extract structured data like prices and cashback."

---

### 2.5 Conditional Cashback Check

**Overview:**  
Checks whether the total cashback amount calculated by the AI Agent is greater than zero to decide if the workflow should continue.

**Nodes Involved:**  
- Is There's cashback (If node)

**Node Details:**  

- **Is There's cashback**  
  - *Type:* If node (conditional)  
  - *Config:* Condition checks if `output.total_cashback` from AI Agent is greater than 0  
  - *Input:* Output JSON from AI Agent  
  - *Output:* Continues only if cashback > 0, otherwise stops workflow for this item  
  - *Failures:* Missing or malformed cashback data  
  - *Sticky Note:* "A conditional check; it only allows the workflow to proceed if the calculated cashback is greater than zero."

---

### 2.6 Data Logging

**Overview:**  
Logs the customer and cashback transaction details into a specified Notion database for record-keeping and campaign tracking.

**Nodes Involved:**  
- Add info to Database (Notion node)

**Node Details:**  

- **Add info to Database**  
  - *Type:* Notion node to create new database page  
  - *Config:*  
    - Uses Notion database ID placeholder  
    - Maps form fields and AI output fields into Notion properties:  
      - Customer name (title) from form Full Name  
      - Customer Email (email)  
      - Store Type (rich text) from form Purchase Channel  
      - Store Name (rich text) from form Store Name  
      - Cashback Amount (number) from AI Agent's total cashback  
  - *Input:* JSON with customer and cashback data  
  - *Output:* Confirmation and page details from Notion  
  - *Failures:* Invalid database ID, permission issues, API rate limits  
  - *Sticky Note:* "Logs the customer details and the calculated cashback amount to a Notion database."

---

### 2.7 Notification Emails

**Overview:**  
Sends two emails: one to the customer confirming their cashback reward, and one internal marketing notification with customer and transaction details.

**Nodes Involved:**  
- Customer Email (Gmail node)  
- Marketing Email (Gmail node)

**Node Details:**  

- **Customer Email**  
  - *Type:* Gmail node (send email)  
  - *Config:*  
    - Sends to customer email address from JotForm submission  
    - Subject: "Cashback"  
    - Message: Personalized cashback details including purchase date, store, products, cashback values  
  - *Input:* Data from AI Agent and JotForm Trigger  
  - *Failures:* Gmail OAuth2 token issues, invalid email address, sending limits  
  - *Sticky Note:* "Sends a confirmation email to the customer detailing their earned cashback reward."

- **Marketing Email**  
  - *Type:* Gmail node (send email)  
  - *Config:*  
    - Sends to marketing team email address (Marketing@yourcompany.com)  
    - Subject: "Cashback Notification - New Reward"  
    - Message: Detailed transaction info including customer name, email, phone, products, total cashback, and status notes  
  - *Input:* Data from AI Agent and JotForm Trigger  
  - *Failures:* Same as Customer Email node  
  - *Sticky Note:* "Sends an internal email notification with all transaction details to the marketing team for tracking."

---

## 3. Summary Table

| Node Name              | Node Type                          | Functional Role                               | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                   |
|------------------------|----------------------------------|-----------------------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger        | JotForm Trigger                  | Starts workflow on form submission             | -                       | Fetch All Receipts       | Starts the workflow upon form submission; provides customer and initial purchase data.                                         |
| Fetch All Receipts      | HTTP Request                    | Retrieves all receipt files from JotForm API  | JotForm Trigger         | Gets Last Receipt        | Requests the list of all uploaded receipt files from the JotForm API.                                                         |
| Gets Last Receipt       | Code (JavaScript)                | Extracts latest receipt URL                     | Fetch All Receipts       | Fetch Receipt File       | Extracts the URL for the most recently uploaded receipt from the file list.                                                   |
| Fetch Receipt File      | HTTP Request                    | Downloads receipt image file                     | Gets Last Receipt        | Convert Image to base64  | Downloads the actual receipt image file using the extracted URL.                                                              |
| Convert Image to base64 | Extract From File               | Converts image binary to base64 string          | Fetch Receipt File       | OCR.Space                | Encodes the downloaded image into a base64 string, preparing it for OCR.                                                     |
| OCR.Space              | HTTP Request                    | OCR processing to extract text from image       | Convert Image to base64  | AI Agent                | Converts the receipt image into plain, readable text.                                                                          |
| Google Gemini Chat Model| LangChain LM Chat Google Gemini | AI model to clean and process OCR text          | AI Agent (ai_languageModel) | AI Agent                | Uses Gemini to clean the raw OCR text, identify XQ products, and extract structured data like prices and cashback.             |
| Structured Output Parser| LangChain Output Parser (Structured JSON) | Parses AI output into structured JSON      | AI Agent (ai_outputParser) | AI Agent                |                                                                                                                              |
| AI Agent               | LangChain Agent                 | Orchestrates AI processing and data extraction | OCR.Space, Gemini Model, Parser | Is There's cashback    | Uses Gemini to clean the raw OCR text, identify XQ products, and extract structured data like prices and cashback.             |
| Is There's cashback    | If node                         | Checks if cashback > 0, conditional continuation | AI Agent                | Add info to Database     | A conditional check; it only allows the workflow to proceed if the calculated cashback is greater than zero.                   |
| Add info to Database    | Notion                         | Logs cashback and customer data to Notion DB  | Is There's cashback      | Customer Email, Marketing Email | Logs the customer details and the calculated cashback amount to a Notion database.                                              |
| Customer Email          | Gmail                         | Sends cashback confirmation email to customer | Add info to Database     | -                       | Sends a confirmation email to the customer detailing their earned cashback reward.                                             |
| Marketing Email         | Gmail                         | Sends internal marketing notification email   | Add info to Database     | -                       | Sends an internal email notification with all transaction details to the marketing team for tracking.                         |
| Sticky Note             | Sticky Note                   | Various explanatory notes                       | -                       | -                       | Multiple notes associated with various workflow steps (see detailed section 2).                                               |

---

## 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger Node**  
   - Type: JotForm Trigger  
   - Configure webhook with your JotForm form webhook ID  
   - Set Form ID to your Cashback form's ID  
   - This node listens to new form submissions containing receipt and customer data.

2. **Create Fetch All Receipts Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.jotform.com/form/YOUR_FORM_ID/files`  
   - Query parameter: `apiKey` = your JotForm API Key with full access  
   - Connect input from JotForm Trigger node  
   - Retrieves all uploaded receipt files for the form.

3. **Create Gets Last Receipt Node (Code Node)**  
   - Type: Code (JavaScript)  
   - Paste code that extracts the last file URL from the JotForm files response (handle arrays or nested structures)  
   - Input: from Fetch All Receipts  
   - Output: JSON with field `lastFile.url` containing latest receipt file URL.

4. **Create Fetch Receipt File Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: expression `={{ $json.lastFile.url.replace('http://','https://') }}` to enforce HTTPS  
   - Response format: file  
   - Input: Gets Last Receipt node  
   - Downloads the actual receipt image file.

5. **Create Convert Image to base64 Node**  
   - Type: Extract From File  
   - Operation: binaryToProperty  
   - Destination key: `imageBase64`  
   - Input: Fetch Receipt File node  
   - Converts downloaded image binary to base64 string.

6. **Create OCR.Space Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.ocr.space/parse/image`  
   - Content type: multipart/form-data  
   - Body parameters:  
     - `base64Image`: `=data:image/jpeg;base64,{{$json.imageBase64}}`  
     - `language`: `ara`  
     - `OCREngine`: `1`  
     - `filetype`: `jpg`  
   - Header parameters: add API key header with your OCR.Space API key  
   - Input: Convert Image to base64 node  
   - Extracts raw text from receipt image.

7. **Create Google Gemini Chat Model Node**  
   - Type: LangChain LM Chat Google Gemini  
   - Model Name: `models/gemini-2.5-flash-lite`  
   - Input: connected as `ai_languageModel` to AI Agent node (see step 9)  
   - No extra parameters needed.

8. **Create Structured Output Parser Node**  
   - Type: LangChain Output Parser (Structured JSON)  
   - Paste JSON schema example for receipt data structure (store_name, purchase_date, products, total_cashback)  
   - Connect as `ai_outputParser` to AI Agent node.

9. **Create AI Agent Node**  
   - Type: LangChain Agent  
   - Prompt: Multi-step instructions to clean OCR text, identify XQ Pharma products, extract structured data including prices and cashback calculation, output valid JSON only  
   - Input expression for OCR text: `{{ $json.ParsedResults[0].ParsedText }}` from OCR.Space node  
   - Connect `ai_languageModel` input to Google Gemini Chat Model node  
   - Connect `ai_outputParser` input to Structured Output Parser node

10. **Create If Node "Is There's cashback"**  
    - Type: If node  
    - Condition: `={{ $json.output.total_cashback }}` greater than 0  
    - Input: AI Agent node  
    - Output: true branch proceeds, false branch stops processing.

11. **Create Add info to Database Node (Notion)**  
    - Type: Notion  
    - Resource: databasePage  
    - Database ID: your Notion cashback database ID  
    - Map properties:  
      - Customer name (title): `={{ $('JotForm Trigger').item.json['Full Name'].first }} {{ $('JotForm Trigger').item.json['Full Name'].last }}`  
      - Customer Email (email): `={{ $('JotForm Trigger').item.json['E-mail'] }}`  
      - Store Type (rich_text): `={{ $('JotForm Trigger').item.json['Purchase Channel'] }}`  
      - Store Name (rich_text): `={{ $('JotForm Trigger').item.json['Store Name'] }}`  
      - Cashback Amount (number): `={{ $json.output.total_cashback }}`  
    - Connect input from If node's true branch.

12. **Create Customer Email Node (Gmail)**  
    - Type: Gmail Send  
    - Send To: `={{ $('JotForm Trigger').item.json['E-mail'] }}`  
    - Subject: "Cashback"  
    - Message: Personalized message with purchase date, store, products, and cashback totals using expressions referencing AI Agent output and JotForm Trigger data  
    - Connect input from Add info to Database node.

13. **Create Marketing Email Node (Gmail)**  
    - Type: Gmail Send  
    - Send To: `Marketing@yourcompany.com`  
    - Subject: "Cashback Notification - New Reward"  
    - Message: Detailed notification including customer name, email, phone number, products, total cashback, and status  
    - Connect input from Add info to Database node.

14. **Connect Nodes Sequentially**  
    - JotForm Trigger → Fetch All Receipts → Gets Last Receipt → Fetch Receipt File → Convert Image to base64 → OCR.Space → AI Agent → Is There's cashback (If) → Add info to Database → Customer Email & Marketing Email

15. **Credentials Setup**  
    - JotForm API Key with Full Access for Fetch All Receipts node  
    - OCR.Space API Key for OCR.Space node  
    - Google Gemini AI credentials as required by LangChain Google Gemini node  
    - Notion API token with access to the specified database  
    - Gmail OAuth2 credentials for sending emails

16. **Testing and Validation**  
    - Verify JotForm webhook is properly configured (see Sticky Note 12)  
    - Test with sample receipt submissions including Arabic and English text  
    - Confirm OCR accuracy and AI parsing output  
    - Confirm Notion entries and email deliveries

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| **JotForm Setup Guide:** Instructions to link JotForm form with n8n webhook, generate full access API key for file downloads, and configure node placeholders with actual form IDs and keys.                                                                                                                                                                                                                                                                                                                                   | See Sticky Note12 content in workflow for details   |
| The workflow handles receipts that may contain Arabic and English text, with possible OCR imperfections such as missing spaces and repeated lines. The AI Agent prompt is designed to normalize and clean this data before extraction.                                                                                                                                                                                                                                                                                           | AI Agent node prompt details                         |
| Cashback is calculated as 10% of the total price per identified XQ Pharma product. If quantity or unit price is missing, logical inference is applied (e.g., default quantity = 1).                                                                                                                                                                                                                                                                                                                                             | AI Agent prompt and structured output schema        |
| Emails sent via Gmail nodes require proper OAuth2 configuration with valid tokens to avoid sending failures.                                                                                                                                                                                                                                                                                                                                                                                                                    | Gmail node configuration                             |
| Notion node requires a database with properties matching the mapped fields: Customer name (title), Customer Email (email), Store Type, Store Name, Cashback Amount.                                                                                                                                                                                                                                                                                                                                                                | Notion database setup                                |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---