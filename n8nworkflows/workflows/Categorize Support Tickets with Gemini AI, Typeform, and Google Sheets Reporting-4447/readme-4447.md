Categorize Support Tickets with Gemini AI, Typeform, and Google Sheets Reporting

https://n8nworkflows.xyz/workflows/categorize-support-tickets-with-gemini-ai--typeform--and-google-sheets-reporting-4447


# Categorize Support Tickets with Gemini AI, Typeform, and Google Sheets Reporting

### 1. Workflow Overview

This workflow automates the process of categorizing support tickets submitted via Typeform, storing them in Google Sheets, and sending summarized reports via email. It leverages Google Gemini AI to classify ticket messages into predefined categories and sentiments, keeps a record in a Google Sheet, aggregates ticket counts per category, and emails a summary report.

**Logical Blocks:**

- **1.1 Input Reception:** Triggering the workflow when a new Typeform submission arrives.
- **1.2 AI Processing:** Classifying the support ticket message using Google Gemini AI.
- **1.3 Parsing AI Output:** Extracting structured category and sentiment data from the AI response.
- **1.4 Data Storage:** Appending classified ticket data into a Google Sheet.
- **1.5 Data Aggregation:** Fetching all stored tickets and counting tickets per category.
- **1.6 Reporting:** Sending an email summary of ticket counts by category.
- **1.7 Workflow Assistance:** Informational sticky note providing author and support contacts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for new support ticket submissions via Typeform and triggers the workflow.
- **Nodes Involved:**  
  - Form Trigger  
  - Sticky Note (Trigger explanation)

- **Node Details:**

  - **Form Trigger**
    - Type: Typeform Trigger
    - Role: Captures new form submissions from Typeform form with ID `lwldQPTN`.
    - Configuration: Uses webhook ID `d02525cf-b9c1-492d-866c-e188bd781155`.
    - Input: External Typeform submissions.
    - Output: JSON including fields such as "Enter your name", "Enter your email", and "Your message".
    - Edge Cases: Webhook misconfiguration, Typeform API rate limits or changes to form structure may cause failures.
    - Version: 1.1

  - **Sticky Note (Trigger explanation)**
    - Content: "Triggers the workflow to start the summary process."
    - Role: Documentation only.

#### 1.2 AI Processing

- **Overview:** Uses Google Gemini AI to classify the support message into categories and sentiment labels.
- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - AI Categorization  
  - Sticky Note2 (AI categorization explanation)

- **Node Details:**

  - **Google Gemini Chat Model**
    - Type: Google Gemini Chat Model (LangChain integration)
    - Role: Provides the AI model endpoint for classification.
    - Configuration: Uses model name `models/gemini-2.0-flash`.
    - Input: Receives prompt from AI Categorization node.
    - Output: AI model response.
    - Edge Cases: Authentication errors with Google Gemini API, rate limits, or model availability.
    - Version: 1

  - **AI Categorization**
    - Type: LangChain Chain LLM node
    - Role: Sends a prompt to the Gemini model for classification.
    - Configuration: Prompt instructs to classify the message into category and sentiment JSON with these categories: `Billing`, `Bug Report`, `Feature Request`, `How-To`, `Complaint`; sentiments: `Positive`, `Neutral`, `Negative`.
    - Key Expression: Uses `{{$json['Your message']}}` from the Typeform submission.
    - Input: Message text from Form Trigger.
    - Output: Raw Gemini model text response.
    - Edge Cases: Improper prompt formatting, unexpected AI output format, network errors.
    - Version: 1.5

  - **Sticky Note2**
    - Content: "Uses Gemini model to classify or tag each support request into predefined categories."
    - Role: Documentation.

#### 1.3 Parsing AI Output

- **Overview:** Parses the raw text response from Gemini AI to extract clean JSON fields for category and sentiment.
- **Nodes Involved:**  
  - Extract Category  
  - Sticky Note10 (Parsing explanation)

- **Node Details:**

  - **Extract Category**
    - Type: Code node (JavaScript)
    - Role: Cleans and parses the Gemini response, extracting `category` and `sentiment`.
    - Key Logic:
      - Removes code block markers (```json```) from AI output.
      - Parses JSON.
      - Returns JSON object with `category` and `sentiment`.
    - Input: Raw text from AI Categorization node.
    - Output: Structured JSON with `category` and `sentiment`.
    - Edge Cases: Malformed AI output or JSON parse errors.
    - Version: 2

  - **Sticky Note10**
    - Content: "Parses Gemini's response to extract only the relevant category label from the output."
    - Role: Documentation.

#### 1.4 Data Storage

- **Overview:** Stores each categorized ticket with user details into a Google Sheet for record-keeping.
- **Nodes Involved:**  
  - Store Data  
  - Sticky Note6 (Storage explanation)

- **Node Details:**

  - **Store Data**
    - Type: Google Sheets node
    - Role: Appends a new row to the Google Sheet with ticket data.
    - Configuration:
      - Document ID: `1SY1fCAbsvyTzIBbmwZpdktbUFN4jKNIpF4GJ9lFKvPM`
      - Sheet Name: `Sheet1` (gid=0)
      - Columns mapped to JSON:
        - Name: from Typeform "Enter your name"
        - Email: from Typeform "Enter your email"
        - Message: from Typeform "Your message"
        - Category: from Extract Category output
        - Sentiment: from Extract Category output
        - Timestamp: current time `$now`
      - Operation: Append
    - Input: JSON with extracted classification and original form data.
    - Output: Confirmation of append operation.
    - Edge Cases: Google Sheets API quota limits, permission errors, invalid sheet ID.
    - Version: 4.5

  - **Sticky Note6**
    - Content: "Appends each categorized and data ticket into a central Google Sheet for record keeping and later aggregation."
    - Role: Documentation.

#### 1.5 Data Aggregation

- **Overview:** Fetches all stored tickets from Google Sheets and counts how many tickets belong to each category.
- **Nodes Involved:**  
  - Fetch Support Ticket Data  
  - Increment Counter  
  - Sticky Note5 (Fetching explanation)  
  - Sticky Note4 (Counting explanation)

- **Node Details:**

  - **Fetch Support Ticket Data**
    - Type: Google Sheets node
    - Role: Reads all rows from the Google Sheet.
    - Configuration:
      - Document ID and Sheet same as Store Data node.
      - Operation: Read all data.
    - Input: Triggered after new data is appended.
    - Output: Array of ticket objects including categories.
    - Edge Cases: Large data volumes causing timeout, API errors.
    - Version: 4.5

  - **Increment Counter**
    - Type: Code node (JavaScript)
    - Role: Iterates over all tickets and counts occurrences per category.
    - Logic:
      - Initializes empty count object.
      - For each item, increments count for its `Category`.
      - Returns counts as summary object.
    - Input: Array of tickets from Fetch Support Ticket Data.
    - Output: JSON with ticket counts by category.
    - Edge Cases: Missing or empty Category fields.
    - Version: 2

  - **Sticky Note5**
    - Content: "Fetches all categorized entries from the Google Sheet to compute ticket counts by category."
    - Role: Documentation.

  - **Sticky Note4**
    - Content: "Groups tickets by category and counts the total per group to prepare data for reporting."
    - Role: Documentation.

#### 1.6 Reporting

- **Overview:** Sends an email containing a summary report of ticket counts by category.
- **Nodes Involved:**  
  - Email Ticket Summary  
  - Sticky Note3 (Email explanation)

- **Node Details:**

  - **Email Ticket Summary**
    - Type: Email Send node
    - Role: Sends a notification email with ticket category counts.
    - Configuration:
      - To: `notify@example.com`
      - From: `you@example.com`
      - Subject: "Support Ticket Summary"
      - Body: Plain text email with counts for each category (Billing, Bug Report, Feature Request, How-To, Complaint), defaulting to 0 if missing.
    - Input: Summary counts from Increment Counter.
    - Output: Email send status.
    - Edge Cases: SMTP configuration errors, invalid recipient email, network issues.
    - Version: 1

  - **Sticky Note3**
    - Content: "Sends an email with the total number of tickets per category from today’s run."
    - Role: Documentation.

#### 1.7 Workflow Assistance

- **Overview:** Provides contact information and helpful links for workflow support and author credits.
- **Nodes Involved:**  
  - Sticky Note9

- **Node Details:**

  - **Sticky Note9**
    - Content:
      ```
      =======================================
                  WORKFLOW ASSISTANCE
      =======================================
      For any questions or support, please contact:
          Yaron@nofluff.online

      Explore more tips and tutorials here:
         - YouTube: https://www.youtube.com/@YaronBeen/videos
         - LinkedIn: https://www.linkedin.com/in/yaronbeen/
      ======================================

      Author:
      Yaron Been
      ![Yaron Been](https://1.gravatar.com/avatar/a4e4dcaa1f76ff5266bbf80e8df86d22efda890474c68f7796e72fd82e3f2375?size=512&d=initials)
      ```
    - Role: Documentation and support contact.
    - Position: Far right, visually covering no functional nodes.

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                             | Input Node(s)           | Output Node(s)         | Sticky Note                                                  |
|-------------------------|-----------------------------------|--------------------------------------------|------------------------|------------------------|--------------------------------------------------------------|
| Form Trigger            | Typeform Trigger                  | Trigger workflow on new Typeform submission | —                      | AI Categorization      | Triggers the workflow to start the summary process.          |
| AI Categorization       | LangChain Chain LLM               | Sends prompt to Gemini AI for classification | Form Trigger           | Extract Category       | Uses Gemini model to classify or tag each support request... |
| Google Gemini Chat Model | LangChain Google Gemini Model     | Provides Gemini AI model for classification | AI Categorization (ai)  | AI Categorization      |                                                              |
| Extract Category        | Code Node (JS)                   | Parses AI response JSON to extract category and sentiment | AI Categorization       | Store Data             | Parses Gemini's response to extract only relevant category.  |
| Store Data              | Google Sheets Append              | Stores ticket data into Google Sheet        | Extract Category        | Fetch Support Ticket Data | Appends each categorized ticket into Google Sheet.           |
| Fetch Support Ticket Data| Google Sheets Read                | Fetches all ticket data from Google Sheet   | Store Data              | Increment Counter      | Fetches all categorized entries from the Google Sheet.       |
| Increment Counter       | Code Node (JS)                   | Counts tickets per category                   | Fetch Support Ticket Data | Email Ticket Summary  | Groups tickets by category and counts total per group.       |
| Email Ticket Summary    | Email Send                       | Sends email summary of ticket counts         | Increment Counter       | —                      | Sends an email with the total number of tickets per category.|
| Sticky Note             | Sticky Note                      | Documentation (Trigger explanation)          | —                      | —                      | Triggers the workflow to start the summary process.          |
| Sticky Note1            | Sticky Note                      | Documentation (Workflow purpose)              | —                      | —                      | This workflow retrieves support ticket data from Google Sheets, counts how many tickets fall into each category, and sends a summary report via email. |
| Sticky Note2            | Sticky Note                      | Documentation (AI classification)             | —                      | —                      | Uses Gemini model to classify or tag each support request... |
| Sticky Note3            | Sticky Note                      | Documentation (Email explanation)             | —                      | —                      | Sends an email with the total number of tickets per category from today’s run. |
| Sticky Note4            | Sticky Note                      | Documentation (Counting explanation)          | —                      | —                      | Groups tickets by category and counts the total per group to prepare data for reporting. |
| Sticky Note5            | Sticky Note                      | Documentation (Fetching data explanation)     | —                      | —                      | Fetches all categorized entries from the Google Sheet to compute ticket counts by category. |
| Sticky Note6            | Sticky Note                      | Documentation (Storage explanation)           | —                      | —                      | Appends each categorized and data ticket into a central Google Sheet for record keeping and later aggregation. |
| Sticky Note9            | Sticky Note                      | Documentation (Support and author info)       | —                      | —                      | See Section 1.7 for full content and links.                  |
| Sticky Note10           | Sticky Note                      | Documentation (Parsing AI output)              | —                      | —                      | Parses Gemini's response to extract only the relevant category label from the output. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Typeform Trigger node**  
   - Node Type: Typeform Trigger  
   - Configuration: Use your Typeform form ID (e.g., `lwldQPTN`).  
   - Set webhook ID automatically or configure webhook in Typeform.  
   - This node starts the workflow on new form submissions.

2. **Add the Google Gemini Chat Model node**  
   - Node Type: LangChain Google Gemini Chat Model  
   - Set `modelName` to `models/gemini-2.0-flash`.  
   - Configure Google API credentials for Gemini access.  
   - This node provides the AI model interface.

3. **Add the AI Categorization node**  
   - Node Type: LangChain Chain LLM  
   - Connect AI Categorization node’s LLM input to Google Gemini Chat Model node.  
   - Set `promptType` to `define`.  
   - Set prompt text as:  
     ```
     Classify the support request below:
     Message: {{ $json['Your message'] }}

     Return output with 'category' and 'sentiment' columns of JSON FILE:
     {
       "category": one of ["Billing", "Bug Report", "Feature Request", "How-To", "Complaint"],
       "sentiment": one of ["Positive", "Neutral", "Negative"]
     }
     ```
   - Input: Message from Typeform trigger.

4. **Add a Code node to parse AI output**  
   - Node Type: Code (JavaScript)  
   - Connect from AI Categorization node.  
   - Code:  
     ```js
     const inputString = $json.text;
     const cleaned = inputString.replace(/```json|```/g, '').trim();
     const data = JSON.parse(cleaned);
     return [{ category: data.category, sentiment: data.sentiment }];
     ```
   - This extracts structured fields from the AI response.

5. **Add Google Sheets Append node ("Store Data")**  
   - Node Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your Google Sheet ID (example: `1SY1fCAbsvyTzIBbmwZpdktbUFN4jKNIpF4GJ9lFKvPM`).  
   - Sheet Name: Use first sheet (`Sheet1` or `gid=0`).  
   - Map columns:  
     - Name: `={{ $('Form Trigger').item.json['Enter your name'] }}`  
     - Email: `={{ $('Form Trigger').item.json['Enter your email'] }}`  
     - Message: `={{ $('Form Trigger').item.json['Your message'] }}`  
     - Category: `={{ $json.category }}`  
     - Sentiment: `={{ $json.sentiment }}`  
     - Timestamp: `={{ $now }}`  
   - Connect input from Extract Category node.

6. **Add Google Sheets Read node ("Fetch Support Ticket Data")**  
   - Node Type: Google Sheets  
   - Operation: Read all rows  
   - Use same document ID and sheet as Store Data node.  
   - Connect input from Store Data node.

7. **Add Code node ("Increment Counter")**  
   - Node Type: Code (JavaScript)  
   - Connect from Fetch Support Ticket Data node.  
   - Code:  
     ```js
     const items = $input.all();
     const counts = {};
     items.forEach(item => {
       const cat = item.json.Category;
       if (cat) {
         counts[cat] = (counts[cat] || 0) + 1;
       }
     });
     return [{ json: { summary: counts } }];
     ```
   - This counts tickets per category.

8. **Add Email Send node ("Email Ticket Summary")**  
   - Node Type: Email Send  
   - Connect from Increment Counter node.  
   - Configure SMTP credentials or use n8n's default email settings.  
   - Set To: `notify@example.com` (replace with actual recipient).  
   - From: `you@example.com` (replace with actual sender).  
   - Subject: `"Support Ticket Summary"`  
   - Body (Text):  
     ```
     Hello,

     Here is the updated count of categorized support tickets:

     Billing: {{$json.summary.Billing || 0}}
     Bug Report: {{$json.summary['Bug Report'] || 0}}
     Feature Request: {{$json.summary['Feature Request'] || 0}}
     How-To: {{$json.summary['How-To'] || 0}}
     Complaint: {{$json.summary.Complaint || 0}}

     Best regards,
     Support Tracker
     ```
9. **Connect all nodes in the following sequence:**  
   - Form Trigger → AI Categorization (LLM input: Google Gemini Chat Model)  
   - AI Categorization → Extract Category → Store Data → Fetch Support Ticket Data → Increment Counter → Email Ticket Summary

10. **Add sticky notes for documentation at appropriate positions (optional but recommended).**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow author and support contact: Yaron Been, email: Yaron@nofluff.online                                                                                                                                                                                              | Author contact                                                |
| YouTube channel with tips and tutorials: https://www.youtube.com/@YaronBeen/videos                                                                                                                                                                                        | Video resource                                                |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                   | Author professional profile                                   |
| Workflow purpose: Retrieves support tickets from Google Sheets, categorizes using Gemini AI, aggregates counts by category, and emails a report.                                                                                                                           | Workflow description                                         |
| Ensure Google Gemini API credentials and Google Sheets API access are properly configured and permissions granted.                                                                                                                                                         | Credential setup requirement                                  |
| Validate Typeform webhook setup and form ID correctness; monitor API usage quotas for Typeform, Google Gemini, and Google Sheets to avoid disruptions.                                                                                                                    | Integration considerations                                    |
| This workflow assumes consistent AI output formatting; changes in Gemini model output format may require updates to the parsing code node.                                                                                                                                 | AI output format dependency                                   |
| Gmail or any SMTP credentials are required for Email Send node; ensure SMTP server allows sending to the configured recipient address.                                                                                                                                      | Email configuration notes                                    |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.