Cold Email Icebreakers from Local Business Search with GPT-4 and Dumpling AI

https://n8nworkflows.xyz/workflows/cold-email-icebreakers-from-local-business-search-with-gpt-4-and-dumpling-ai-6156


# Cold Email Icebreakers from Local Business Search with GPT-4 and Dumpling AI

---
### 1. Workflow Overview

This workflow automates the generation of personalized cold email icebreakers for local businesses based on a keyword search. Its primary use case is digital marketing outreach, where a user inputs a business-related keyword (e.g., "dentist in New York"), and the workflow:

- Searches for local businesses matching the keyword via Dumpling AI’s Google Maps API.
- Extracts contact details and website summaries for each business.
- Uses GPT-4 to craft a warm, customized icebreaker email referencing the business’s website content.
- Logs the results in Google Sheets for record-keeping.
- Optionally adds the lead to an Instantly.ai email campaign for outreach.

The main logical blocks include:

- **1.1 Input Reception:** Collects the keyword from a form trigger.
- **1.2 Local Business Search:** Calls Dumpling AI to search Google Maps for businesses matching the keyword.
- **1.3 Business Data Processing:** Splits the list of businesses, loops through each, and extracts email and website summary.
- **1.4 AI-Powered Email Generation:** Uses GPT-4 to write personalized icebreaker emails based on the extracted data.
- **1.5 Conditional Lead Handling:** Filters businesses with valid emails, logs data to Google Sheets, and optionally adds to an Instantly.ai campaign.
- **1.6 Workflow Metadata and Notes:** Contains a sticky note summarizing the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures the search keyword from a user via a form submission, triggering the workflow.
- **Nodes Involved:**  
  - *Form: Submit Search Keyword*

- **Node Details:**

  - **Form: Submit Search Keyword**
    - Type: Form Trigger (Webhook-based)
    - Configuration: Presents a single input field labeled "Keyword" for users to enter a search term.
    - Key Variables: `{{$json.Keyword}}` used downstream as the search query.
    - Inputs: None (trigger node)
    - Outputs: Connects to the Dumpling AI Google Maps search node.
    - Edge Cases: Missing or empty keyword submissions may cause Dumpling AI search to fail or return empty results.
    - Version-specific: v2.2

#### 1.2 Local Business Search

- **Overview:** Calls Dumpling AI’s API to search Google Maps businesses matching the keyword.
- **Nodes Involved:**  
  - *Dumpling AI: Search Google Maps for Businesses*  
  - *🧮 Split: Extract Individual Places*

- **Node Details:**

  - **Dumpling AI: Search Google Maps for Businesses**
    - Type: HTTP Request
    - Configuration: POST request to Dumpling AI endpoint `/api/v1/search-maps` with JSON body containing the query keyword.
    - Authentication: HTTP header with Dumpling AI API key.
    - Input: Receives search keyword from form trigger.
    - Output: JSON with a `places` array of businesses.
    - Edge Cases: API rate limits, network errors, invalid or empty query input.
    - Version-specific: v4.2
  
  - **🧮 Split: Extract Individual Places**
    - Type: Split Out
    - Configuration: Splits the `places` array from Dumpling AI response into individual items for processing.
    - Input: Connects from Dumpling AI search node.
    - Output: Passes each business individually downstream.
    - Edge Cases: Empty or malformed `places` array.
    - Version-specific: v1

#### 1.3 Business Data Processing

- **Overview:** Processes each business individually in manageable batches; for each business, extracts email and website summary using Dumpling AI.
- **Nodes Involved:**  
  - *🔁 Loop: Process Each Business*  
  - *🧠 Dumpling AI: Extract Email + Website Summary*

- **Node Details:**

  - **🔁 Loop: Process Each Business**
    - Type: Split In Batches
    - Configuration: Processes businesses in batches of 2 to control rate and resource usage.
    - Input: Receives individual business items from Split Out node.
    - Output: Sends each batch to both the email/website extraction and Instantly API nodes (parallel branches).
    - Edge Cases: Large batch sizes may cause API throttling or timeouts.
    - Version-specific: v3
  
  - **🧠 Dumpling AI: Extract Email + Website Summary**
    - Type: HTTP Request
    - Configuration: POST request to Dumpling AI `/api/v1/extract` endpoint, sending the business’s website URL and requesting extraction of `email` and `websiteSummary`.
    - Authentication: HTTP header with Dumpling AI API key.
    - Input: Receives each business’s website URL.
    - Output: JSON with extracted email and website summary.
    - On error: Continues workflow without halting (important to handle missing emails gracefully).
    - Edge Cases: Websites without emails, extraction failures, invalid URLs, API errors.
    - Version-specific: v4.2

#### 1.4 AI-Powered Email Generation

- **Overview:** Generates a personalized icebreaker email using GPT-4 based on the business name, keywords, and website summary.
- **Nodes Involved:**  
  - *✍️ GPT-4: Write Personalized Icebreaker Email*

- **Node Details:**

  - **✍️ GPT-4: Write Personalized Icebreaker Email**
    - Type: Langchain OpenAI Node (GPT-4 model)
    - Configuration: Uses GPT-4o model with a system prompt instructing it to write a warm, customized, non-salesy icebreaker email referencing business specifics.
    - Input Variables:
      - Business Name: from Split node item (`title`)
      - Keywords: from Split node item (`types`)
      - Website Summary: from Dumpling AI extraction results
    - Output: A 4-6 sentence icebreaker email.
    - Execution: Runs once per business item.
    - Edge Cases: GPT API timeouts, malformed input data, inappropriate output (requires monitoring).
    - Version-specific: v1.8

#### 1.5 Conditional Lead Handling

- **Overview:** Filters businesses to ensure an email exists, logs the data to Google Sheets, and optionally adds leads to an Instantly.ai campaign.
- **Nodes Involved:**  
  - *✅ IF: Email Exists*  
  - *📄 Log to Google Sheets*  
  - *📤 Instantly API: Add to Campaign*

- **Node Details:**

  - **✅ IF: Email Exists**
    - Type: Filter
    - Configuration: Checks if the extracted email field exists and is non-empty.
    - Input: Receives from GPT-4 node.
    - Output: Passes only businesses with valid emails to Google Sheets logging.
    - Edge Cases: Some businesses may lack emails, causing filtering out.
    - Version-specific: v2.2
  
  - **📄 Log to Google Sheets**
    - Type: Google Sheets
    - Configuration: Appends or updates a row with business details and the generated icebreaker email.
    - Sheet: Specified by document ID and sheet name (gid=0).
    - Columns: Email, Phone, Title, Ice Breaker, Website URL, Website Summary.
    - Matching: Uses Website URL to prevent duplicates.
    - Credentials: OAuth2 Google Sheets account.
    - Edge Cases: API quota limits, sheet permission errors.
    - Version-specific: v4.6
  
  - **📤 Instantly API: Add to Campaign**  
    - Type: HTTP Request  
    - Configuration: POST to Instantly.ai API to add lead with email, personalization (icebreaker message), phone, and website to a specific campaign.  
    - Authentication: HTTP Header with Instantly API key.  
    - Note: Uses fixed "CampaignID" placeholder (must be replaced with actual campaign ID).  
    - Input: Receives business data from Loop node directly (parallel branch).  
    - Edge Cases: API errors, invalid campaign ID, invalid email format.  
    - Version-specific: v4.2  

#### 1.6 Workflow Metadata and Notes

- **Overview:** Provides a sticky note summarizing the workflow’s purpose, tools, and high-level logic.
- **Nodes Involved:**  
  - *Sticky Note*

- **Node Details:**

  - **Sticky Note**
    - Type: Sticky Note display
    - Content: Summary describing the workflow steps, tools used, and optional Instantly.ai integration.
    - Visual aid for users to understand workflow context.
    - Version-specific: v1

---

### 3. Summary Table

| Node Name                             | Node Type                  | Functional Role                                    | Input Node(s)                        | Output Node(s)                             | Sticky Note                                                                                   |
|-------------------------------------|----------------------------|---------------------------------------------------|------------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------|
| Form: Submit Search Keyword          | Form Trigger               | Receives user keyword input                        | None                               | Dumpling AI: Search Google Maps            |                                                                                              |
| Dumpling AI: Search Google Maps for Businesses | HTTP Request              | Searches Google Maps businesses via Dumpling AI  | Form: Submit Search Keyword         | 🧮 Split: Extract Individual Places         |                                                                                              |
| 🧮 Split: Extract Individual Places  | Split Out                  | Splits business list into individual items        | Dumpling AI: Search Google Maps    | 🔁 Loop: Process Each Business               |                                                                                              |
| 🔁 Loop: Process Each Business       | Split In Batches           | Processes businesses in batches (batch size=2)    | 🧮 Split: Extract Individual Places | 🧠 Dumpling AI: Extract Email + Website Summary, 📤 Instantly API: Add to Campaign             |                                                                                              |
| 🧠 Dumpling AI: Extract Email + Website Summary | HTTP Request              | Extracts email and website summary from URL       | 🔁 Loop: Process Each Business      | ✍️ GPT-4: Write Personalized Icebreaker Email   |                                                                                              |
| ✍️ GPT-4: Write Personalized Icebreaker Email | Langchain OpenAI (GPT-4)  | Generates personalized icebreaker email            | 🧠 Dumpling AI: Extract Email + Website Summary | ✅ IF: Email Exists                          |                                                                                              |
| ✅ IF: Email Exists                  | Filter                     | Checks if extracted email exists                    | ✍️ GPT-4: Write Personalized Icebreaker Email | 📄 Log to Google Sheets                       |                                                                                              |
| 📄 Log to Google Sheets              | Google Sheets              | Logs business data and icebreaker email            | ✅ IF: Email Exists                 | 🔁 Loop: Process Each Business               |                                                                                              |
| 📤 Instantly API: Add to Campaign    | HTTP Request               | Adds lead to Instantly.ai campaign                  | 🔁 Loop: Process Each Business      | None                                       |                                                                                              |
| Sticky Note                        | Sticky Note                | Workflow summary and notes                          | None                               | None                                       | ### ✉️ Cold Email Icebreaker Generator\n\nThis workflow:\n- Accepts a keyword (e.g. "dentist in New York")\n- Searches local businesses using Dumpling AI\n- Extracts website summaries and emails\n- Uses GPT-4 to write short, friendly icebreaker emails\n- Logs results to Google Sheets\n- Optionally adds leads to Instantly.ai campaigns\n\n✅ Tools Used:\n- Dumpling AI\n- GPT-4\n- Google Sheets\n- Instantly.ai (optional) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: *Form Trigger*  
   - Configuration: Add a single form field labeled "Keyword" (text input).  
   - Set webhook ID (auto-generated).  
   - This node will trigger the workflow when a user submits a search keyword.

2. **Add HTTP Request Node for Dumpling AI Google Maps Search:**  
   - Name: "Dumpling AI: Search Google Maps for Businesses"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/search-maps`  
   - Body Type: JSON  
   - Body Content: `{ "query": "{{$json.Keyword}}", "page": "" }`  
   - Authentication: HTTP Header Auth with Dumpling AI API key (credential setup required).  
   - Connect Form Trigger output to this node’s input.

3. **Add Split Out Node to Extract Individual Places:**  
   - Name: "🧮 Split: Extract Individual Places"  
   - Type: Split Out  
   - Field to Split Out: `places` (from Dumpling AI response)  
   - Connect Dumpling AI Search node output to this node’s input.

4. **Add Split In Batches Node for Processing Businesses:**  
   - Name: "🔁 Loop: Process Each Business"  
   - Type: Split In Batches  
   - Batch Size: 2  
   - Connect Split Out node output to this node’s input.

5. **Add HTTP Request Node to Extract Email and Website Summary:**  
   - Name: "🧠 Dumpling AI: Extract Email + Website Summary"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/extract`  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "url": "{{ $json.website }}",
       "schema": {
         "email": "string",
         "websiteSummary": "string"
       }
     }
     ```  
   - Authentication: HTTP Header Auth with Dumpling AI API key.  
   - Set Error Handling to continue on failure.  
   - Connect Loop node output to this node’s input.

6. **Add GPT-4 Node to Write Personalized Icebreaker Email:**  
   - Name: "✍️ GPT-4: Write Personalized Icebreaker Email"  
   - Type: Langchain OpenAI (OpenAI API)  
   - Model: GPT-4o (or equivalent GPT-4 model)  
   - Messages: Set system prompt instructing to write a warm, personalized icebreaker based on:  
     - Business Name (`$('🧮 Split: Extract Individual Places').item.json.title`)  
     - Keywords (`$('🧮 Split: Extract Individual Places').item.json.types`)  
     - Website Summary (`$json.results.websiteSummary`)  
   - Connect Dumpling AI Extract Email node output to this node’s input.

7. **Add Filter Node to Check if Email Exists:**  
   - Name: "✅ IF: Email Exists"  
   - Type: Filter  
   - Condition: Check if `$('🧠 Dumpling AI: Extract Email + Website Summary').item.json.results.email` exists and is not empty.  
   - Connect GPT-4 node output to this node’s input.

8. **Add Google Sheets Node to Log Results:**  
   - Name: "📄 Log to Google Sheets"  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Document ID: Set the Google Sheets document ID where data will be stored.  
   - Sheet Name: Use appropriate sheet (e.g., "Sheet1" or `gid=0`).  
   - Columns to map:  
     - Email: Extracted email  
     - Phone: `$('🧮 Split: Extract Individual Places').item.json.phoneNumber`  
     - Title: Business name  
     - Ice Breaker: GPT-4 generated message  
     - Website URL: Business website URL  
     - Website Summary: Extracted summary  
   - Matching Columns: Use Website URL to avoid duplicates.  
   - Set Google OAuth2 credentials.  
   - Connect Email Exists filter output to this node.

9. **Add Instantly.ai HTTP Request Node (Optional):**  
   - Name: "📤 Instantly API: Add to Campaign"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.instantly.ai/api/v1/lead/add`  
   - Body Parameters:  
     - campaign_id: Replace `"CampaignID"` with actual campaign ID  
     - email: Business email  
     - personalization: GPT-4 icebreaker content  
     - phone: Business phone  
     - website: Business website  
   - Authentication: HTTP Header Auth with Instantly API key.  
   - Connect the Loop node output (parallel branch) to this node.

10. **Add Sticky Note for Documentation (Optional):**  
    - Name: "Sticky Note"  
    - Type: Sticky Note  
    - Content: Summary of the workflow purpose, tools used, and key steps.

11. **Connect Nodes Appropriately:**  
    - Form Trigger → Dumpling AI Search → Split Out → Loop  
    - Loop → Dumpling AI Extract & Instantly API (parallel)  
    - Dumpling AI Extract → GPT-4 → Email Exists Filter → Google Sheets  
    - Google Sheets → Loop (for next batch iteration)

12. **Test Workflow:**  
    - Submit a keyword via the form (e.g., "dentist in New York")  
    - Verify that businesses are fetched, emails and summaries extracted, icebreakers generated, and data logged.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This cold email icebreaker generator uses Dumpling AI for local business data extraction and GPT-4 to craft personalized outreach messages.                                                                                   | Workflow purpose summary                                                                                         |
| The workflow includes optional integration with Instantly.ai for campaign management; ensure to replace `"CampaignID"` with your actual campaign identifier.                                                                   | Instantly.ai API integration detail                                                                              |
| Google Sheets is used as a central log to track businesses contacted, their emails, and the generated icebreaker messages to avoid duplicate outreach.                                                                           | Google Sheets logging                                                                                              |
| Dumpling AI API requires API key authentication via HTTP header; monitor API usage and limits.                                                                                                                                 | Dumpling AI credentials requirement                                                                               |
| GPT-4 API usage may incur costs; prompt engineering ensures friendly, human-like personalized messages that avoid generic sales pitches.                                                                                        | GPT-4 prompt design                                                                                               |
| Workflow handles missing emails gracefully by filtering out businesses without valid emails before logging and outreach.                                                                                                        | Email existence filter                                                                                            |
| Sticky Note in workflow provides a user-friendly summary and can be used as documentation for operators or future maintainers.                                                                                                | Sticky Note content                                                                                               |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.