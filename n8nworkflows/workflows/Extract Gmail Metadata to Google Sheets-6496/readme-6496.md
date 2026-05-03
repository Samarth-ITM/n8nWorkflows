Extract Gmail Metadata to Google Sheets

https://n8nworkflows.xyz/workflows/extract-gmail-metadata-to-google-sheets-6496


# Extract Gmail Metadata to Google Sheets

### 1. Workflow Overview

This workflow automates the extraction of metadata from incoming Gmail emails and appends the structured data into a Google Sheets spreadsheet. It is designed primarily for scenarios such as capturing customer inquiries, support requests, or contact form submissions sent via email, enabling seamless data collection without manual intervention. The workflow processes incoming emails, extracts key details (sender name, email, subject, message body, timestamp), cleans and structures this data via custom logic, and updates a Google Sheet, which can serve as a CRM or data repository.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Watches for new incoming Gmail messages in real-time.
- **1.2 Data Extraction and Transformation:** Uses custom JavaScript logic to extract and normalize email metadata, handling variable email formats robustly.
- **1.3 Data Structuring:** Maps extracted fields into user-friendly column names preparing for storage.
- **1.4 Data Storage:** Appends or updates rows in a Google Sheet to keep a running log of message data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block triggers the workflow for every new Gmail email received, polling every minute to detect new messages.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Gmail Trigger node (integration trigger node)  
    - Configuration: Polls Gmail account every minute; no additional filters applied, so triggers on all incoming emails.  
    - Credentials: Uses OAuth2 credential for Gmail account access.  
    - Input: None (trigger node)  
    - Output: Emits JSON data representing the new email message, including headers, body, sender, subject, snippet, and other metadata.  
    - Edge Cases:  
      - OAuth token expiry or invalid credentials can cause authentication failures.  
      - Gmail API rate limits or quota may cause missed triggers or errors.  
      - Emails with unusual or missing fields may require fallback handling downstream.  
    - Version: 1.2

---

#### 2.2 Data Extraction and Transformation

- **Overview:**  
This block processes raw email JSON data to extract and normalize key fields (name, email, subject, message body, timestamp). It handles various formats of input data and applies fallback logic to ensure data completeness.

- **Nodes Involved:**  
  - Code (JavaScript) node  

- **Node Details:**

  - **Code**  
    - Type: Code node (executes custom JavaScript)  
    - Configuration:  
      - Extracts subject from multiple possible JSON fields (`subject`, `Subject`, `headers.subject`). Defaults to "No Subject" if missing.  
      - Extracts message body from `body`, `text`, or `snippet` fields. Defaults to "No message found." if missing.  
      - Extracts "from" field from `from`, `From`, or `headers.from`.  
      - Parses "from" field to separate sender name and email if in format "Name <email>". Otherwise treats as email only.  
      - Attempts to extract a name from message body text matching pattern "I am [Name] from ...".  
      - Chooses final name with priority: extracted from body > extracted from header > "Unknown".  
      - Adds ISO timestamp of processing time.  
    - Key expressions:  
      - Regex for parsing name and email from sender string.  
      - Regex for extracting name phrase from message body.  
      - Conditional fallbacks for missing data.  
    - Input: JSON from Gmail Trigger node.  
    - Output: JSON containing `{ name, email, subject, message, timestamp }`.  
    - Edge Cases:  
      - Messages missing typical fields fallback gracefully to default values.  
      - Complex or malformed "from" headers could cause regex failure or incorrect extraction.  
      - Unexpected email body formats may fail name extraction from text.  
      - Timezone considerations for timestamp are not explicitly handled (uses ISO string of current time).  
    - Version: 2

---

#### 2.3 Data Structuring

- **Overview:**  
Transforms the extracted data fields into a user-friendly format with descriptive field names suitable for spreadsheet columns.

- **Nodes Involved:**  
  - Edit Fields (Set node)  

- **Node Details:**

  - **Edit Fields**  
    - Type: Set node (field assignment)  
    - Configuration: Maps raw extracted JSON properties to new fields with clear labels:  
      - `Full Name` = extracted `name`  
      - `Email Address` = extracted `email`  
      - `Subject` = extracted `subject`  
      - `Body of the email` = extracted `message`  
      - `Time` = extracted `timestamp`  
    - Input: JSON from Code node.  
    - Output: JSON with renamed and structured fields for downstream use.  
    - Edge Cases: None significant; purely field mapping.  
    - Version: 3.4

---

#### 2.4 Data Storage

- **Overview:**  
Appends or updates a row in a Google Sheets document representing the incoming email metadata, ensuring persistent storage and easy access.

- **Nodes Involved:**  
  - Append row in sheet (Google Sheets node)  

- **Node Details:**

  - **Append row in sheet**  
    - Type: Google Sheets node (integration node)  
    - Operation: `appendOrUpdate` — appends a new row or updates existing row matched by `Email Address`.  
    - Configuration:  
      - Spreadsheet ID and Sheet name (Sheet1, gid=0) specified explicitly.  
      - Columns mapped from structured fields:  
        - `Tme` (note misspelling, probably meant to be "Time") ← `Time`  
        - `Name` ← `Full Name`  
        - `Subject` ← `Subject`  
        - `Email Address` ← `Email Address`  
        - `Body of the email` ← `Body of the email`  
      - Matching column for update: `Email Address`  
      - No type conversion attempted; all fields treated as strings.  
    - Input: Structured JSON from Edit Fields node.  
    - Output: Confirmation of successful append/update operation.  
    - Credentials: OAuth2 credential for Google Sheets account.  
    - Edge Cases:  
      - Google API quota or permission errors if credentials are invalid or limited.  
      - Misspelled column name `Tme` may cause confusion or errors in sheet schema.  
      - Duplicate email addresses update existing rows; if duplicates are valid, data may be overwritten unintentionally.  
      - Network errors or API rate limits may cause failures.  
    - Version: 4.5

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                     | Input Node(s)    | Output Node(s)         | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
|--------------------|-------------------------|-----------------------------------|------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger      | Gmail Trigger           | Detect new incoming Gmail emails   | None             | Code                   | ## Gmail Triggers when the new email has came                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Code               | Code (JavaScript)       | Extract and normalize email data   | Gmail Trigger    | Edit Fields            | ## It extracts useful details (like name, email, subject, and message) from incoming emails or form submissions — even if the data format varies. 🧩 Step-by-Step Explanation: ✅ 1. Get the Subject: Looks for a subject line in multiple possible fields: $json.subject, $json.Subject, $json.headers.subject If none found → sets it to "No Subject" ✅ 2. Get the Message Body: Looks for the main message in common fields: $json.body, $json.text, $json.snippet If none found → "No message found." ✅ 3. Get the "From" Information: Checks where the message came from: $json.from, $json.From, $json.headers.from ✅ 4. Extract Name & Email: If the sender is in format like: John Doe <john@example.com> It will: senderName = "John Doe" email = "john@example.com" If only an email is provided (like john@example.com), it just sets the email. ✅ 5. Try to Extract Name from the Message: If the message body has something like: Hi, I am Alice Johnson from XYZ Agency. It will extract "Alice Johnson" as the name. ✅ 6. Choose the Final Name: Order of priority: Name from message body ("I am ___ from...") Name from the email header (John Doe) If not found → "Unknown" ✅ 7. Return Structured Data: The final output is: { name: "Alice Johnson", email: "john@example.com", subject: "Website Inquiry", message: "Hi, I am Alice Johnson from XYZ...", timestamp: "2025-07-20T09:22:10.121Z" } |
| Edit Fields        | Set                     | Rename and structure extracted data | Code             | Append row in sheet    |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Append row in sheet | Google Sheets           | Append or update email data row    | Edit Fields      | None                   |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note        | Sticky Note             | Explanation of code logic           | None             | None                   | ## It extracts useful details (like name, email, subject, and message) from incoming emails or form submissions — even if the data format varies. 🧩 Step-by-Step Explanation: ✅ 1. Get the Subject: Looks for a subject line in multiple possible fields: $json.subject, $json.Subject, $json.headers.subject If none found → sets it to "No Subject" ✅ 2. Get the Message Body: Looks for the main message in common fields: $json.body, $json.text, $json.snippet If none found → "No message found." ✅ 3. Get the "From" Information: Checks where the message came from: $json.from, $json.From, $json.headers.from ✅ 4. Extract Name & Email: If the sender is in format like: John Doe <john@example.com> It will: senderName = "John Doe" email = "john@example.com" If only an email is provided (like john@example.com), it just sets the email. ✅ 5. Try to Extract Name from the Message: If the message body has something like: Hi, I am Alice Johnson from XYZ Agency. It will extract "Alice Johnson" as the name. ✅ 6. Choose the Final Name: Order of priority: Name from message body ("I am ___ from...") Name from the email header (John Doe) If not found → "Unknown" ✅ 7. Return Structured Data: The final output is: { name: "Alice Johnson", email: "john@example.com", subject: "Website Inquiry", message: "Hi, I am Alice Johnson from XYZ...", timestamp: "2025-07-20T09:22:10.121Z" } |
| Sticky Note1       | Sticky Note             | Gmail trigger explanation          | None             | None                   | ## Gmail Triggers when the new email has came                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| Sticky Note2       | Sticky Note             | Overview of the workflow and value | None             | None                   | ## What This Automation Flow Does (in Simple Terms) This automation is designed to process incoming customer emails, extract the important details, and store them in Airtable or any system you like — automatically, without any manual copy-pasting or data cleaning. --- ⚙ Tools Used n8n: Automation platform where the entire workflow is built. Airtable: Used as a database to store all the extracted customer data in a structured table format. --- 📦 Complete Flow Breakdown (Step-by-Step) 1. Trigger - New Email Received The flow starts when a new email arrives. This could be: A contact form submission from your Shopify store A customer sending you a question or feedback A support request Node Used: IMAP/Email Trigger --- 2. Custom JavaScript Code - Smart Data Extraction This is the core logic where we: Extract the sender's email address and name (even if hidden inside angled brackets like John <john@email.com>) Clean the subject line and message body Use fallback values like "No Subject" or "No message found" when content is missing Extract names from phrases like "Hi, I’m Alex" if available in the message Add a timestamp to track when the message came in Node Used: Function Node Purpose: Makes the data clean, structured, and usable — no junk text or broken formatting. --- 3. Send to Airtable (or any CRM) Once the data is extracted and cleaned: It is sent directly to your Airtable base (or CRM/Sheet/Database) One row per message, including Name, Email, Subject, Message, and Timestamp Node Used: Airtable - Create Record (You can also add filters or conditional routing if needed) --- 💡 Why This Is Valuable to You as a Store Owner ✅ Saves hours of manual work: No need to check emails, copy details, and paste them into spreadsheets or CRMs ✅ Never miss a lead: Every message is captured and stored in one place ✅ Clean, structured data: No more messy email threads — just clear info you can act on ✅ Scalable: Works whether you get 10 messages a day or 1,000 ✅ Expandable: Later you can auto-send replies, tag messages, or forward to your team --- 🧠 Bonus: Why the Code Logic Matters The JavaScript in the Function node is like a smart assistant: It understands where to find data, even if email formats are different It removes guesswork, keeps things clean, and ensures nothing breaks downstream It’s future-proof — you don’t have to update every time someone sends an email slightly differently --- 📈 Final Result You get a real-time dashboard of every incoming customer message stored neatly — ready for follow-up, reporting, or automation.  |


---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set to poll every minute (default polling frequency).  
   - Configure OAuth2 credentials for your Gmail account in n8n.  
   - No filters configured to trigger on every incoming email.  
   - Position: Start of the workflow.

2. **Create Code Node to Extract Email Metadata**  
   - Type: Code (JavaScript) node.  
   - Connect Gmail Trigger main output to this node’s input.  
   - Paste the following JavaScript logic:  
     - Extract `subject` from `subject`, `Subject`, or `headers.subject` fields, fallback: `"No Subject"`.  
     - Extract `message` body from `body`, `text`, or `snippet`, fallback: `"No message found."`.  
     - Extract `from` field from `from`, `From`, or `headers.from`.  
     - Parse `from` string to separate `senderName` and `email` if in format "Name <email>".  
     - Extract name from message body matching pattern `"I am (.*?) from"`.  
     - Select final name priority: extracted from body → from header → `"Unknown"`.  
     - Add current ISO timestamp as `timestamp`.  
     - Return a JSON object with keys: `name`, `email`, `subject`, `message`, `timestamp`.

3. **Create Set Node to Structure Fields**  
   - Type: Set node.  
   - Connect Code node output to this node input.  
   - Add the following field assignments:  
     - `Full Name` = `{{$json["name"]}}`  
     - `Email Address` = `{{$json["email"]}}`  
     - `Subject` = `{{$json["subject"]}}`  
     - `Body of the email` = `{{$json["message"]}}`  
     - `Time` = `{{$json["timestamp"]}}`

4. **Create Google Sheets Node to Append or Update Row**  
   - Type: Google Sheets node.  
   - Connect Set node output to this node input.  
   - Set operation to `appendOrUpdate`.  
   - Set spreadsheet document ID to your target Google Sheets document ID.  
   - Set sheet name to the appropriate sheet (e.g., `"Sheet1"` or use gid=0).  
   - Define columns mapping:  
     - `Tme` → `Time` (note: consider correcting spelling to `Time` in sheet schema for clarity)  
     - `Name` → `Full Name`  
     - `Email Address` → `Email Address`  
     - `Subject` → `Subject`  
     - `Body of the email` → `Body of the email`  
   - Set matching columns to `Email Address` for update detection.  
   - Configure OAuth2 credentials for Google Sheets access.

5. **Connect all nodes in sequence:**  
   Gmail Trigger → Code → Set (Edit Fields) → Google Sheets (Append row in sheet)

6. **Optional:**  
   - Add sticky notes for documentation inside n8n to explain logic at key steps.  
   - Test with sample emails to verify extraction and appending works as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                               |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This workflow automates customer email data extraction and storage to reduce manual copy-paste work, ensuring no leads are missed and data is clean and structured. It’s scalable and expandable for future automation like tagging or auto-replies. The core JavaScript is robust against variable email formats and missing fields, reducing maintenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | Overview and value proposition of the workflow (Sticky Note2 content)                                        |
| The JavaScript code extracts subject, body, sender name, and email from multiple possible JSON fields to handle variations in Gmail message payloads. It also tries to intelligently extract a name from message body text using a regex pattern. The final fallback for missing names is "Unknown".                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Explanation of code logic and regex-based extraction (Sticky Note content)                                   |
| Gmail Trigger node requires OAuth2 credentials with proper Gmail API access scopes. Be mindful of API rate limits and token refresh settings.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Gmail Trigger node credential requirements                                                                    |
| Google Sheets node requires OAuth2 credentials with edit access to the target sheet. The spreadsheet ID and sheet name must be exact. Be cautious about column name consistency; the workflow uses `Tme` as a column which appears to be a typo for `Time`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Google Sheets node credential and configuration notes                                                         |
| For error handling, consider adding error workflows or catch nodes to handle API failures, authentication errors, or unexpected data formats. Logging and alerting can improve maintenance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | General best practices for error handling in n8n workflows                                                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. The processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.