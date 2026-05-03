Extract Specific Website Data with Form Input, Gemini 2.5 flash and Gmail

https://n8nworkflows.xyz/workflows/extract-specific-website-data-with-form-input--gemini-2-5-flash-and-gmail-7754


# Extract Specific Website Data with Form Input, Gemini 2.5 flash and Gmail

### 1. Workflow Overview

This workflow enables users to submit a web scraping request via a form, specifying a source website URL and the exact data they want to extract. It then fetches the HTML content of the specified website, extracts relevant HTML body content, and uses Google Gemini 2.5 flash AI model to precisely extract the requested data. Finally, the extracted data is formatted into structured JSON and sent to a specified email address via Gmail. The workflow is designed for flexible, user-driven web data extraction with automated AI processing and email delivery.

Logical blocks:

- **1.1 Input Reception:** Captures user input through a web form (URL and data extraction request).  
- **1.2 Data Retrieval:** Fetches raw HTML from the user-supplied URL and extracts the HTML body content.  
- **1.3 AI Processing:** Uses Google Gemini 2.5 flash model to extract the exact requested data from the HTML content, enforcing strict output format rules.  
- **1.4 Output Structuring:** Parses AI output into a standardized JSON format.  
- **1.5 Result Delivery:** Sends the extraction result via Gmail with detailed context.  
- **1.6 Documentation and Setup Notes:** Provides user guidance and configuration notes via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving user inputs from a web form. Users submit a source URL and specify the exact data they want to extract from that website.

- **Nodes Involved:**  
  - Web Scraper form submission

- **Node Details:**

  - **Web Scraper form submission**  
    - Type & Role: Form Trigger node; triggers workflow upon form submission.  
    - Configuration:  
      - Form Title: "Web Scraper Form"  
      - Two input fields:  
        1. "Source URL" (URL of the website to scrape)  
        2. "Data to extract" (text describing the data to extract)  
      - Webhook ID set for external form integration.  
    - Expressions/Variables: Outputs user input fields as JSON accessible downstream.  
    - Input: External HTTP form submission.  
    - Output: Passes form data to next node (Get HTML from source url).  
    - Edge Cases:  
      - Missing or invalid URL input may cause HTTP request failure downstream.  
      - Empty or ambiguous "Data to extract" may result in AI extraction failure or irrelevant output.

---

#### 2.2 Data Retrieval

- **Overview:**  
  Fetches the complete HTML content of the user-specified URL and extracts the body HTML element to isolate the main content for AI processing.

- **Nodes Involved:**  
  - Get HTML from source url  
  - HTML Extractor

- **Node Details:**

  - **Get HTML from source url**  
    - Type & Role: HTTP Request node; fetches website HTML.  
    - Configuration:  
      - URL dynamically set using the form input expression: `={{ $json['Source URL'] }}`  
      - Default GET method, no additional options.  
    - Input: Receives JSON containing "Source URL" from form submission.  
    - Output: Returns full HTTP response body (HTML content).  
    - Edge Cases:  
      - HTTP errors: 404, 500, timeouts, network errors.  
      - Invalid or malformed URLs causing request failure.  
      - Sites with anti-scraping measures or JavaScript-rendered content may return incomplete HTML.

  - **HTML Extractor**  
    - Type & Role: HTML node; extracts specific HTML segments.  
    - Configuration:  
      - Operation: extractHtmlContent  
      - Extraction values: CSS selector "body" — extracts content inside `<body>` tag only.  
    - Input: Receives raw HTML from HTTP Request node.  
    - Output: JSON containing extracted body HTML.  
    - Edge Cases:  
      - If HTML is malformed or body tag missing, extraction may fail or return empty.  
      - Large HTML body content may impact performance downstream.

---

#### 2.3 AI Processing

- **Overview:**  
  Uses the Google Gemini 2.5 flash language model to analyze the extracted HTML body and extract exactly the data requested by the user, following strict formatting rules.

- **Nodes Involved:**  
  - Data Extractor LLM Chain  
  - Google Gemini Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type & Role: Language Model node; runs Google Gemini 2.5 flash for AI processing.  
    - Configuration:  
      - Model: "models/gemini-2.5-flash"  
      - No additional options set (default).  
    - Credentials: Google PaLM API key (Google Gemini API).  
    - Input: Receives prompt and text from Data Extractor LLM Chain as AI input.  
    - Output: AI-generated response containing extracted data.  
    - Edge Cases:  
      - API quota limits or authentication errors.  
      - Latency or timeout issues.  
      - Unexpected AI output if prompt is ambiguous.

  - **Data Extractor LLM Chain**  
    - Type & Role: Chain LLM node; constructs the AI prompt and manages AI interaction.  
    - Configuration:  
      - Prompt template uses user’s extraction request from form input (`{{ $('Web Scraper form submission').item.json['Data to extract'] }}`) and the extracted HTML body (`{{ $json.body }}`).  
      - Enforces strict extraction rules: only requested data, combine multiple matches with commas, no explanations, maintain original formatting, return JSON with "result" field or "No data found".  
      - Has output parser enabled.  
    - Input: Receives extracted HTML body JSON from HTML Extractor.  
    - AI Language Model connection: connected to Google Gemini Chat Model.  
    - AI Output Parser connection: connected to Structured Output Parser.  
    - Output: Passes parsed AI extraction results downstream.  
    - Edge Cases:  
      - Failure to parse user input correctly may produce incorrect extraction.  
      - AI hallucinations or misinterpretation of HTML.  
      - Output format errors mitigated by structured output parser.

  - **Structured Output Parser**  
    - Type & Role: Langchain output parser node; parses AI raw output into structured JSON.  
    - Configuration:  
      - JSON schema example: `{ "result": "extracted value(s)" }`  
      - Ensures AI output conforms to expected JSON format.  
    - Input: Receives raw AI output from Google Gemini Chat Model via Data Extractor LLM Chain.  
    - Output: Provides clean JSON with extracted results for email node.  
    - Edge Cases:  
      - If AI output is malformed, parser may throw errors or return empty.  
      - Schema may require adjustment for different extraction result formats.

---

#### 2.4 Result Delivery

- **Overview:**  
  Sends the structured extraction result to a specified email address via Gmail, including details of the original request and extracted data.

- **Nodes Involved:**  
  - Gmail - Send Result

- **Node Details:**

  - **Gmail - Send Result**  
    - Type & Role: Gmail node; sends email with scraping results.  
    - Configuration:  
      - Recipient email address: currently set to `template_data_extactor_replace_me@yopmail.com` (must be updated).  
      - Subject line includes the source URL dynamically.  
      - Message body includes: Source URL, Data requested, Extracted result, and a thank-you note.  
      - Email type: plain text.  
      - Append attribution disabled.  
    - Credentials: OAuth2 Gmail credential configured for "Billy Email 2" account.  
    - Input: Receives structured extraction results from Data Extractor LLM Chain.  
    - Output: Sends email; no output data.  
    - Edge Cases:  
      - Authentication failures or expired tokens in Gmail OAuth2.  
      - Incorrect recipient email causes delivery failure.  
      - Large email body or invalid characters could cause issues.

---

#### 2.5 Documentation and Setup Notes

- **Overview:**  
  Provides several sticky notes for user guidance and configuration instructions, including setup requirements, workflow process summary, and contact information.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5

- **Node Details:**

  - **Sticky Note** (Setup Required)  
    - Provides instructions to update email recipient, adjust JSON schema, modify AI prompt, and required credentials (Google Gemini API, Gmail OAuth2).  
  - **Sticky Note1** (Workflow Description)  
    - Describes overall workflow purpose, features, and capabilities.  
  - **Sticky Note2** (Process Overview)  
    - Step-by-step logical outline of workflow operation.  
  - **Sticky Note3** (Data Extractor LLM Chain Notes)  
    - Notes on AI prompt customization and model adjustment.  
  - **Sticky Note4** (Contact Info)  
    - Contact details for the workflow creator Billy, including email and websites.  
  - **Sticky Note5** (Gmail Node Notes)  
    - Reminds to update email target, subject, and message.

---

### 3. Summary Table

| Node Name                  | Node Type                           | Functional Role                     | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                                  |
|----------------------------|-----------------------------------|-----------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Web Scraper form submission | Form Trigger                      | Input Reception                   | External HTTP Form           | Get HTML from source url     |                                                                                                              |
| Get HTML from source url    | HTTP Request                     | Fetch raw HTML content            | Web Scraper form submission | HTML Extractor              |                                                                                                              |
| HTML Extractor             | HTML Extractor                   | Extract HTML body content         | Get HTML from source url     | Data Extractor LLM Chain    |                                                                                                              |
| Data Extractor LLM Chain    | Chain LLM (Langchain)            | AI Data Extraction                | HTML Extractor              | Gmail - Send Result          | Connected to Google Gemini Chat Model and Structured Output Parser; see Sticky Note3 for configuration notes |
| Google Gemini Chat Model    | LLM Chat Google Gemini           | AI Language Model                 | Data Extractor LLM Chain (AI input) | Data Extractor LLM Chain (AI output) |                                                                                                              |
| Structured Output Parser    | Langchain Output Parser          | Parse AI output into JSON         | Google Gemini Chat Model (AI output) | Data Extractor LLM Chain (Parsed output) |                                                                                                              |
| Gmail - Send Result         | Gmail                           | Send extraction result by email  | Data Extractor LLM Chain    | None                        | See Sticky Note5 for email setup details                                                                     |
| Sticky Note                | Sticky Note                      | Setup instructions                | None                        | None                        | See content in Sticky Note                                                                                   |
| Sticky Note1               | Sticky Note                      | Workflow description              | None                        | None                        | See content in Sticky Note1                                                                                  |
| Sticky Note2               | Sticky Note                      | Workflow process overview         | None                        | None                        | See content in Sticky Note2                                                                                  |
| Sticky Note3               | Sticky Note                      | AI prompt & model customization   | None                        | None                        | See content in Sticky Note3                                                                                   |
| Sticky Note4               | Sticky Note                      | Contact and support info          | None                        | None                        | See content in Sticky Note4                                                                                   |
| Sticky Note5               | Sticky Note                      | Gmail node configuration notes   | None                        | None                        | See content in Sticky Note5                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Web Scraper form submission node:**  
   - Type: Form Trigger  
   - Set form title: "Web Scraper Form"  
   - Add two form fields:  
     - "Source URL" (text input)  
     - "Data to extract" (text input)  
   - Copy the webhook URL generated for external form integration.

2. **Add Get HTML from source url node:**  
   - Type: HTTP Request  
   - Set URL parameter to dynamic expression: `={{ $json['Source URL'] }}` (to read from form input)  
   - Leave method as GET and default options.

3. **Add HTML Extractor node:**  
   - Type: HTML  
   - Operation: extractHtmlContent  
   - Extraction values: Add one extraction with:  
     - Key: "body"  
     - CSS selector: "body"  
   - Connect output of HTTP Request node to this node.

4. **Add Google Gemini Chat Model node:**  
   - Type: Langchain LLM Chat Google Gemini  
   - Model Name: "models/gemini-2.5-flash"  
   - Configure credentials with Google PaLM API key (Google Gemini API account).

5. **Add Structured Output Parser node:**  
   - Type: Langchain Output Parser Structured  
   - Provide JSON schema example:  
     ```json
     {
       "result": "extracted value(s)"
     }
     ```

6. **Add Data Extractor LLM Chain node:**  
   - Type: Langchain Chain LLM  
   - Prompt Type: define  
   - Enable output parser.  
   - Set the prompt text as follows (adjust if needed):  
     ```
     Your task is to extract the exact information specified by the user.

     User’s extraction request:
     "{{ $('Web Scraper form submission').item.json['Data to extract'] }}"

     Rules:
     1. Extract ONLY the requested information.
     2. If multiple matches exist, combine them into a single string separated by commas.
     3. Do NOT add explanations or extra text—output only the extracted data.
     4. Maintain the original values unless formatting is requested.
     5. If no matches are found, return: { "result": "No data found" }.
     6. Always return the response in this format:
     {
         "result": "extracted value(s)"
     }

     Here is the source data:
     {{ $json.body }}
     ```
   - Connect:  
     - Input from HTML Extractor node.  
     - AI language model: Google Gemini Chat Model node.  
     - AI output parser: Structured Output Parser node.

7. **Add Gmail - Send Result node:**  
   - Type: Gmail  
   - Credentials: Configure with Gmail OAuth2 credential.  
   - Recipient email: set to actual target email address (replace placeholder).  
   - Subject:  
     ```
     ✅ Web Scraping Result for {{ $('Web Scraper form submission').item.json['Source URL'] }}
     ```  
   - Message:  
     ```
     Your web scraping task has been completed.

     Source URL:
     {{ $('Web Scraper form submission').item.json['Source URL'] }}

     Data Requested:
     {{ $('Web Scraper form submission').item.json['Data to extract'] }}

     Extracted Result:
     {{ $json.output.result }}

     Thank you for using our web scraping automation.
     ```  
   - Disable append attribution.

8. **Connect workflow nodes in sequence:**  
   - Web Scraper form submission → Get HTML from source url  
   - Get HTML from source url → HTML Extractor  
   - HTML Extractor → Data Extractor LLM Chain  
   - Data Extractor LLM Chain → Gmail - Send Result  
   - Connect Data Extractor LLM Chain’s AI language model input to Google Gemini Chat Model.  
   - Connect Data Extractor LLM Chain’s AI output parser input to Structured Output Parser.

9. **Add sticky notes as needed for documentation and setup instructions:**
   - Include notes about updating recipient email, Google Gemini API credential, Gmail OAuth2 credential, and prompt customization.

10. **Test the workflow by submitting the web form with valid URL and data extraction request.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                  |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Hi, I’m Billy! I help businesses build n8n workflows & AI automation projects. Contact me at billychartanto@gmail.com         | Sticky Note4 contact info                        |
| n8n Creator profile: [https://n8n.io/creators/billy](https://n8n.io/creators/billy/)                                            | Sticky Note4 contact info                        |
| My n8n Projects portfolio: [https://www.billychristi.com/n8n](https://www.billychristi.com/n8n)                                | Sticky Note4 contact info                        |
| Workflow requires Google Gemini (PaLM API) key and Gmail OAuth2 credentials for email delivery                                 | Sticky Note setup instructions                   |
| Adjust the JSON schema in Structured Output Parser to match your desired output format                                         | Sticky Note setup instructions                   |
| Modify LLM prompt in Data Extractor LLM Chain node to fit specific extraction needs                                            | Sticky Note setup instructions                   |
| Update Gmail node recipient email and email content before deploying to production                                            | Sticky Note5 Gmail node configuration            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.