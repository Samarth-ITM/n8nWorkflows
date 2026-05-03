Extract Business Card Data from Slack to Google Sheets with GPT-4o OCR

https://n8nworkflows.xyz/workflows/extract-business-card-data-from-slack-to-google-sheets-with-gpt-4o-ocr-8367


# Extract Business Card Data from Slack to Google Sheets with GPT-4o OCR

### 1. Workflow Overview

This n8n workflow automates the extraction of business card data from images uploaded to a specific Slack channel and appends the structured contact information to a Google Sheets document. It harnesses OCR and AI (GPT-4o) capabilities to parse images, identify key contact fields, and streamline contact management without manual data entry.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Detects new messages with uploaded images in Slack.
- **1.2 Image Retrieval:** Downloads the uploaded business card image from Slack.
- **1.3 AI/OCR Parsing:** Uses GPT-4o to analyze the image and extract structured contact details.
- **1.4 Data Transformation:** Cleans and prepares the extracted data for insertion.
- **1.5 Data Storage:** Appends the cleaned contact data as a new row in Google Sheets.
- **1.6 Confirmation Notification:** Sends a confirmation message back to Slack with the parsed contact details.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a new message containing an uploaded business card image is posted in a designated Slack channel.

- **Nodes Involved:**  
  - Slack Trigger

- **Node Details:**  

  - **Slack Trigger**  
    - Type: Slack Trigger (Webhook-based event listener)  
    - Configuration: Listens for "message" events on Slack channel ID `C09DW6Q03T8` (named "card")  
    - Credentials: Connected to Slack account with OAuth token  
    - Input/Output: Receives Slack event data including message content and file metadata  
    - Edge Cases:  
      - Messages without file uploads trigger no further action  
      - Slack API rate limits or connectivity issues may cause missed triggers  
      - Permissions errors if app lacks channel or message read access

#### 1.2 Image Retrieval

- **Overview:**  
  Downloads the uploaded business card image from Slack using the file URL provided in the message event.

- **Nodes Involved:**  
  - Fetch images (HTTP Request node)

- **Node Details:**  

  - **Fetch images**  
    - Type: HTTP Request  
    - Configuration: Uses URL from Slack event `$json.files[0].url_private_download` to fetch the image file  
    - Authentication: Slack OAuth credentials for authorized file download  
    - Output: Binary file data representing the business card image  
    - Edge Cases:  
      - File URL may expire or be inaccessible if permissions are insufficient  
      - Network timeouts or Slack API errors may interrupt file retrieval  
      - Multiple files in one message are not explicitly handled; only the first file is fetched

#### 1.3 AI/OCR Parsing

- **Overview:**  
  Analyzes the downloaded image using GPT-4o to identify and extract structured contact information such as full names, job titles, company names, phone numbers, and email addresses.

- **Nodes Involved:**  
  - AI model (OpenAI GPT-4o LM Chat node)  
  - Scan Contact Information (LangChain Agent node)  
  - Structure Output (Output Parser Structured node)

- **Node Details:**  

  - **AI model**  
    - Type: LangChain OpenAI Chat Language Model (GPT-4o)  
    - Configuration: Model set to "gpt-4o" for advanced OCR and language understanding  
    - Input: Receives prompt and image data (via connected nodes)  
    - Credentials: OpenAI API key with GPT-4o access  
    - Edge Cases:  
      - API rate limits or quota exhaustion may cause failures  
      - Prompt or data formatting issues may affect output accuracy

  - **Scan Contact Information**  
    - Type: LangChain Agent node, specialized for OCR and structured extraction  
    - Configuration:  
      - System Message: Guides the AI to extract professional contact info from images with multiple business cards  
      - Prompt: Requests extraction of full names, job titles, company names, phone numbers, and emails  
    - Output Parser: Enabled to parse AI output into structured JSON  
    - Input: Image data from Fetch images node and AI model node output  
    - Output: JSON with extracted contact fields  
    - Edge Cases:  
      - Ambiguous or low-quality images may yield incomplete or inaccurate data  
      - Multiple cards in one image are supported but may be limited by AI parsing quality

  - **Structure Output**  
    - Type: LangChain Output Parser Structured  
    - Configuration: JSON schema example specifies expected fields to match AI output  
    - Input: AI output text from Scan Contact Information node  
    - Output: Fully structured JSON object with contact fields normalized  
    - Edge Cases:  
      - Parsing may fail if AI output deviates significantly from expected schema  
      - Missing or malformed fields could cause downstream errors

#### 1.4 Data Transformation

- **Overview:**  
  Prepares and reformats the structured contact data for insertion by splitting out individual contact entries if multiple exist.

- **Nodes Involved:**  
  - Transforming data (Split Out node)

- **Node Details:**  

  - **Transforming data**  
    - Type: Split Out  
    - Configuration: Splits the `output` field, which contains an array of contact objects, into individual items for sequential processing  
    - Input: Structured JSON array from Structure Output node  
    - Output: Individual contact objects, one per execution cycle  
    - Edge Cases:  
      - Empty arrays result in no downstream data  
      - Non-array input will cause node failure

#### 1.5 Data Storage

- **Overview:**  
  Inserts each extracted contact as a new row into a target Google Sheets spreadsheet, mapping fields to columns.

- **Nodes Involved:**  
  - Append row in sheet (Google Sheets node)

- **Node Details:**  

  - **Append row in sheet**  
    - Type: Google Sheets  
    - Configuration:  
      - Operation: Append a row  
      - Document ID: Google Sheet ID `1NEmgb1BU706kR4k-H2e3L6T8AnUPxFsNzkQZNhVAP90`  
      - Sheet Name: `gid=0` (default first sheet)  
      - Columns mapped with expressions from JSON contact object fields:  
        - Name: `{{$json.output['full names']}}`  
        - Email: `{{$json.output.email}}`  
        - Phone: `{{$json.output['phone numbers']}}`  
        - Company: `{{$json.output['company names']}}`  
        - Job Title: `{{$json.output['job titles']}}`  
    - Credentials: OAuth2 Google Sheets account with write access  
    - Edge Cases:  
      - API quota limits or permission issues may block row insertion  
      - Mismatched or missing fields may cause incomplete rows  
      - Concurrent appends could cause race conditions if running multiple instances

#### 1.6 Confirmation Notification

- **Overview:**  
  Sends a formatted confirmation message back to the Slack channel to notify users that the contact data has been successfully saved.

- **Nodes Involved:**  
  - Send a message (Slack node)

- **Node Details:**  

  - **Send a message**  
    - Type: Slack message sender node  
    - Configuration:  
      - Channel: Slack channel ID `C09DW6Q03T8` (same as trigger)  
      - Message Text Template:  
        ```
        ---
        Name: {{ $json.Name }}
        Title: {{ $json['Job Title'] }}
        Company: {{ $json.Company }}
        Phone: {{ $json.Phone }}
        Email: {{ $json.Email }}
        ```  
      - Uses data from appended Google Sheets row to confirm saved contact  
    - Credentials: Slack OAuth token same as trigger node  
    - Edge Cases:  
      - Slack API errors or connectivity issues may prevent message delivery  
      - Missing contact fields may cause incomplete message formatting

---

### 3. Summary Table

| Node Name             | Node Type                        | Functional Role                   | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                      |
|-----------------------|---------------------------------|---------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| Slack Trigger         | Slack Trigger                   | Start workflow on Slack message | —                         | Fetch images             |                                                                                                |
| Fetch images          | HTTP Request                   | Download business card image    | Slack Trigger             | Scan Contact Information |                                                                                                |
| AI model              | LangChain OpenAI Chat          | AI language model GPT-4o         | —                         | Scan Contact Information |                                                                                                |
| Scan Contact Information | LangChain Agent                | Extract contact info from image | Fetch images, AI model    | Transforming data        |                                                                                                |
| Structure Output      | LangChain Output Parser Structured | Parse AI output into JSON       | Scan Contact Information  | Transforming data        |                                                                                                |
| Transforming data     | Split Out                      | Split array of contacts         | Structure Output          | Append row in sheet      |                                                                                                |
| Append row in sheet   | Google Sheets                  | Append contact to spreadsheet   | Transforming data         | Send a message           |                                                                                                |
| Send a message        | Slack                         | Notify Slack channel             | Append row in sheet       | —                        |                                                                                                |
| Sticky Note           | Sticky Note                   | Describes workflow overview     | —                         | —                        | ## How it works ... (Full workflow explanation and usage instructions)                         |
| Sticky Note1          | Sticky Note                   | Marketing and summary note       | —                         | —                        | ## Scan Business Cards in Slack to Google Sheets ... (Summary of workflow benefits)             |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a **Slack Trigger** node  
- Type: Slack Trigger  
- Parameters: Trigger on "message" event  
- Channel ID: Set to the Slack channel where business card photos will be uploaded (e.g., `C09DW6Q03T8`)  
- Connect your Slack OAuth credentials with read message and file access.

**Step 2:** Add an **HTTP Request** node named "Fetch images"  
- Method: GET (default)  
- URL: Use expression `{{$json.files[0].url_private_download}}` to get the file URL from Slack event data  
- Authentication: Use Slack OAuth credentials for authorized file download  
- Response Format: File (binary)  
- Connect Slack Trigger output to this node input.

**Step 3:** Add a **LangChain OpenAI Chat** node named "AI model"  
- Model: Select GPT-4o (or your available advanced OCR-capable model)  
- Credentials: Enter your OpenAI API key with GPT-4o access  
- No additional options required  
- This node will feed into the next agent node.

**Step 4:** Add a **LangChain Agent** node named "Scan Contact Information"  
- Text/Prompt:  
  ```
  Please identify and extract all professional contact information from the image containing several business cards. You have to include details that are full names, job titles, company names, phone numbers, and email addresses.
  ```  
- System Message:  
  ```
  You assist sales/BD teams by parsing images with several business cards. Identify every card and pull the essentials—full names, job titles, companies, phone numbers, and emails.
  ```  
- Enable Output Parser  
- Connect "Fetch images" and "AI model" nodes as inputs to this node.

**Step 5:** Add a **LangChain Output Parser Structured** node named "Structure Output"  
- JSON Schema Example:  
  ```json
  [{
    "full names": "Toshiki Hirao",
    "job titles": "CEO",
    "company names": "dTosh",
    "phone numbers": "012-3456-938",
    "email": "xxx@yyy.jp"
  }]
  ```  
- Connect output from "Scan Contact Information."

**Step 6:** Add a **Split Out** node named "Transforming data"  
- Field to split out: `output` (the array of contact objects)  
- Include all other fields  
- Connect from "Structure Output" node.

**Step 7:** Add a **Google Sheets** node named "Append row in sheet"  
- Operation: Append  
- Document ID: Set to your target Google Sheets spreadsheet ID  
- Sheet Name: Use sheet's GID or name (e.g., `gid=0`)  
- Columns Mapping:  
  - Name: `={{ $json.output['full names'] }}`  
  - Email: `={{ $json.output.email }}`  
  - Phone: `={{ $json.output['phone numbers'] }}`  
  - Company: `={{ $json.output['company names'] }}`  
  - Job Title: `={{ $json.output['job titles'] }}`  
- Connect Google Sheets OAuth2 credentials with write permissions  
- Connect "Transforming data" output.

**Step 8:** Add a **Slack** node named "Send a message"  
- Operation: Send Message  
- Channel: Same Slack channel as trigger (`C09DW6Q03T8`)  
- Message Text:  
  ```
  ---
  Name: {{ $json.Name }}
  Title: {{ $json['Job Title'] }}
  Company: {{ $json.Company }}
  Phone: {{ $json.Phone }}
  Email: {{ $json.Email }}
  ```  
- Use Slack OAuth2 credentials  
- Connect from "Append row in sheet."

**Step 9:** Connect the nodes following the sequence:  
Slack Trigger → Fetch images → Scan Contact Information → Structure Output → Transforming data → Append row in sheet → Send a message

**Step 10:** (Optional) Add Sticky Note nodes with the provided content for user guidance and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow automatically converts messy business card photos into structured contact data, eliminating manual entry. Upload photos to Slack and have contacts saved to Google Sheets instantly.                                                                                                                                                                                                                                                                                                                                 | Marketing summary for workflow purpose                                                             |
| Requirements include n8n cloud instance, Slack and Google Sheets accounts with proper access, and AI/OCR capability enabled in n8n (OpenAI GPT-4o recommended).                                                                                                                                                                                                                                                                                                                                                                   | Usage prerequisites                                                                                 |
| Workflow includes detailed instructions on connecting Slack and Google Sheets credentials, setting channel IDs, and mapping extracted data fields for customization.                                                                                                                                                                                                                                                                                                                                                            | Setup instructions                                                                                  |
| For AI model, GPT-4o is used for superior OCR and language understanding. Alternative OCR providers can be integrated if configured accordingly.                                                                                                                                                                                                                                                                                                                                                                               | AI/OCR provider note                                                                               |
| Slack messages confirm contact addition with formatted details for immediate feedback.                                                                                                                                                                                                                                                                                                                                                                                                                                            | Slack notification behavior                                                                         |
| Workflow tested with Slack API version supporting file download and message triggers; Google Sheets API version 4.7 or higher recommended.                                                                                                                                                                                                                                                                                                                                                                                        | API version recommendations                                                                         |

---

**Disclaimer:**  
The text provided is derived exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal or offensive elements. All data processed is legal and public.