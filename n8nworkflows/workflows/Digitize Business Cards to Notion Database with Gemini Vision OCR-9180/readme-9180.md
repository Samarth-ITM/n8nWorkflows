Digitize Business Cards to Notion Database with Gemini Vision OCR

https://n8nworkflows.xyz/workflows/digitize-business-cards-to-notion-database-with-gemini-vision-ocr-9180


# Digitize Business Cards to Notion Database with Gemini Vision OCR

### 1. Workflow Overview

This workflow automates the digitization of business cards by extracting contact information from an uploaded image and storing it in a Notion database. It targets users who want to quickly capture and organize business card data without manual entry. The workflow logically divides into four main blocks:

- **1.1 Input Reception:** Captures the business card image and category from a web form submission.
- **1.2 AI Processing:** Uses Google Gemini Vision OCR to analyze the uploaded image and extract textual contact details.
- **1.3 JSON Parsing:** Cleans and parses the raw JSON-like text output from the OCR into a structured JSON object.
- **1.4 Data Storage:** Creates a new page in a Notion database with the extracted contact details.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Waits for a user to submit a form containing a business card image and category. This initiates the workflow.
- **Nodes Involved:**  
  - On form submission
- **Node Details:**  
  - **Node Name:** On form submission  
  - **Type:** Form Trigger  
  - **Role:** Entry point; listens for HTTP form submissions with file upload  
  - **Configuration:**  
    - Form title: "business_card"  
    - Fields:  
      - File input named "business_card" (accepts .jpg, .png, .jpeg, single file, required)  
      - Text input named "category" (required)  
  - **Key Expressions:** None (trigger node)  
  - **Input:** HTTP request with form data  
  - **Output:** Passes binary image data and category text to next node  
  - **Edge Cases / Failures:**  
    - User uploads unsupported file type or no file → form validation error  
    - Network or webhook connectivity issues  
    - Missing required fields or incomplete form submission

#### 2.2 AI Processing (Gemini Vision OCR)

- **Overview:** Processes the uploaded business card image using Google Gemini Vision OCR to extract structured text data representing contact information.
- **Nodes Involved:**  
  - Analyze image
- **Node Details:**  
  - **Node Name:** Analyze image  
  - **Type:** Langchain Google Gemini node (custom integration with Google PaLM API)  
  - **Role:** OCR analysis and text extraction from binary image data  
  - **Configuration:**  
    - Input type: binary, property name: "business_card" (from form upload)  
    - Operation: analyze image  
    - Model used: "models/gemini-1.5-flash-latest" (Google Gemini Vision OCR latest flash model)  
    - Static example text provided as fallback template (example contact JSON)  
    - Credentials: Google PALM API credentials configured  
  - **Key Expressions:** None dynamic; input is binary property "business_card"  
  - **Input:** Binary image data from form submission  
  - **Output:** JSON-like text containing extracted contact fields  
  - **Edge Cases / Failures:**  
    - API authentication failure or quota exceeded  
    - Image too low quality or unsupported format → incomplete or incorrect text extraction  
    - Timeout or network errors with Google API  
    - Unexpected output format that could break parsing downstream

#### 2.3 JSON Parsing

- **Overview:** Cleans the raw text output from the Gemini OCR node and safely parses it into a structured JSON object.
- **Nodes Involved:**  
  - Parse to Json
- **Node Details:**  
  - **Node Name:** Parse to Json  
  - **Type:** Code node (JavaScript)  
  - **Role:** Extracts and parses the raw JSON string embedded in the OCR output  
  - **Configuration:**  
    - Reads the first text part from the OCR output JSON path `$input.first().json.content.parts[0].text`  
    - Trims and removes Markdown code block notation (```json ... ```) if present  
    - Uses try-catch JSON.parse to safely convert string to object  
    - Returns parsed JSON object or null if parsing fails  
  - **Key Expressions:** Custom JS function `safeJsonToObject()`  
  - **Input:** Raw OCR output text from "Analyze image" node  
  - **Output:** Parsed JSON object containing contact details (Name, Phone, Email, etc.)  
  - **Edge Cases / Failures:**  
    - OCR output not containing valid JSON → returns null, downstream nodes may error or produce empty entries  
    - Unexpected text formatting or missing fields  
    - Null result handling needs to be considered downstream

#### 2.4 Data Storage in Notion

- **Overview:** Creates a new page in a specified Notion database with the parsed contact details mapped to corresponding Notion properties.
- **Nodes Involved:**  
  - Create a database page
- **Node Details:**  
  - **Node Name:** Create a database page  
  - **Type:** Notion node  
  - **Role:** Inserts a new database entry for the business card contact data  
  - **Configuration:**  
    - Database ID: preconfigured to a Notion database titled "고객 명함 기록" (Customer Business Cards)  
    - Title property mapped to `$json.Name`  
    - Mappings for properties:  
      - 연락처|phone_number ← $json.Phone  
      - 이름|title ← $json.Name  
      - 이메일|email ← $json.Email  
      - 주소|rich_text ← $json.Address  
      - 회사명|rich_text ← $json.Company  
      - 휴대폰|phone_number ← $json.Mobile  
      - 홈페이지|rich_text ← $json.Website  
    - Credentials: Notion API OAuth2 credentials configured  
  - **Key Expressions:** Uses JSON path expressions like `={{ $json.Name }}` for dynamic property mapping  
  - **Input:** Parsed JSON contact object from "Parse to Json" node  
  - **Output:** Notion page creation response  
  - **Edge Cases / Failures:**  
    - Invalid or incomplete data fields leading to partial page creation  
    - Notion API authentication failure or permission errors  
    - Rate limiting or network timeouts  
    - Database schema mismatch (e.g., property keys changed in Notion)

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role              | Input Node(s)      | Output Node(s)      | Sticky Note                                                                                                                             |
|---------------------|-------------------------------|-----------------------------|--------------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                  | Input reception             | —                  | Analyze image       | —                                                                                                                                       |
| Analyze image       | Langchain Google Gemini       | OCR analyze business card   | On form submission | Parse to Json       | —                                                                                                                                       |
| Parse to Json        | Code (JavaScript)             | Clean & parse OCR output    | Analyze image      | Create a database page | —                                                                                                                                       |
| Create a database page | Notion                      | Store contact info in Notion| Parse to Json      | —                   | —                                                                                                                                       |
| Sticky Note          | Sticky Note                   | Documentation               | —                  | —                   | # 📇 Business Card OCR → Notion Database<br><br>This workflow automates the process of extracting key information from a **business card image** and storing it into a **Notion database**.<br><br>### ⚙️ Workflow Steps<br>1. **Form Submission**: Upload a business card image (`.jpg`, `.png`, `.jpeg`) and select a category.<br>2. **Gemini Vision Analysis**: The uploaded image is processed by Google Gemini to extract text information (Name, Position, Phone, Email, etc.).<br>3. **Parse JSON**: Extracted text is cleaned and parsed into structured JSON.<br>4. **Save to Notion**: The parsed contact details are stored in your Notion database (`Customer Business Cards`).<br><br>### 📌 Example Extracted Data<br>```json<br>{<br>  "Name": "Jin Park",<br>  "Position": "Head of Development",<br>  "Phone": "021231234",<br>  "Mobile": "0101231234",<br>  "Email": "abc@dc.com",<br>  "Company": "",<br>  "Address": "6F, Donga Building, 212, Yeoksam-ro, Gangnam-gu, Seoul",<br>  "Website": "www.tov.com"<br>}``` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("On form submission"):**  
   - Set form title to "business_card".  
   - Add two fields:  
     - File upload field labeled "business_card", accept `.jpg`, `.png`, `.jpeg` formats, single file, required.  
     - Text input field labeled "category", required.  
   - This node triggers the workflow on form submit and outputs binary image data and category text.

2. **Add a Google Gemini Vision node ("Analyze image"):**  
   - Select operation "analyze" with resource "image".  
   - Set input type to "binary" and binary property name to "business_card" (from form upload).  
   - Choose model "models/gemini-1.5-flash-latest" (latest Gemini Vision OCR).  
   - Configure credentials with a valid Google PaLM API account enabling Gemini Vision.  
   - Connect output of "On form submission" to this node.

3. **Add a Code node ("Parse to Json"):**  
   - Use JavaScript to extract and clean the raw JSON-like string from the Gemini output:  
     ```js
     const rawText = $input.first().json.content.parts[0].text;

     const result = safeJsonToObject(rawText);

     return [result];

     function safeJsonToObject(jsonString) {
       const trimmed = jsonString.trim();
       const cleaned = trimmed.replace(/^```json\s*/i, '').replace(/```$/i, '');
       try {
         return JSON.parse(cleaned);
       } catch (err) {
         return null;
       }
     }
     ```  
   - Connect output of "Analyze image" to this node.

4. **Add a Notion node ("Create a database page"):**  
   - Set resource to "databasePage".  
   - Provide the Notion database ID where business card data will be stored.  
   - Map properties as follows:  
     - Title property: `={{ $json.Name }}`  
     - 연락처|phone_number: `={{ $json.Phone }}`  
     - 이름|title: `={{ $json.Name }}`  
     - 이메일|email: `={{ $json.Email }}`  
     - 주소|rich_text: `={{ $json.Address }}`  
     - 회사명|rich_text: `={{ $json.Company }}`  
     - 휴대폰|phone_number: `={{ $json.Mobile }}`  
     - 홈페이지|rich_text: `={{ $json.Website }}`  
   - Configure Notion API OAuth2 credentials with appropriate permissions.  
   - Connect output of "Parse to Json" to this node.

5. **Add optional Sticky Note for documentation:**  
   - Add a sticky note describing the workflow purpose, steps, and an example JSON output for users.

6. **Set workflow execution order:**  
   - Ensure the nodes are connected in sequence: Form submission → Analyze image → Parse to Json → Create a database page.

7. **Test the workflow:**  
   - Submit a business card image and category via the form.  
   - Verify the extracted data appears correctly in the Notion database.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                          |
|------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| This workflow requires a Google PaLM API account enabled for Gemini Vision OCR, and a Notion integration with database write access. | API and credential setup instructions at https://developers.google.com/palm and https://developers.notion.com/ |
| Ensure business card images are clear and legible to optimize OCR accuracy.                                                        | General best practice for OCR workflows                                |
| The Notion database schema must have the exact property keys as mapped, e.g., "연락처|phone_number", "이름|title".                 | Notion database schema configuration                                   |
| Example extracted JSON used for debugging and fallback:                                                                              | ```json { "Name": "Jung Hyun Park", "Position": "Head of Development", "Phone": "021231234", "Mobile": "0101231234", "Email": "abc@dc.com", "Company": "TOV", "Address": "6F, Donga Building, 212, Yeoksam-ro, Gangnam-gu, Seoul", "Website": "www.tov.com" }``` | Provided in workflow sticky note                                       |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, adhering strictly to content policies and containing no illegal or protected content. All processed data is legal and public.