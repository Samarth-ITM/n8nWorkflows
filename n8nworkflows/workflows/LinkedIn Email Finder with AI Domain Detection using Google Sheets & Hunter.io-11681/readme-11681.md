LinkedIn Email Finder with AI Domain Detection using Google Sheets & Hunter.io

https://n8nworkflows.xyz/workflows/linkedin-email-finder-with-ai-domain-detection-using-google-sheets---hunter-io-11681


# LinkedIn Email Finder with AI Domain Detection using Google Sheets & Hunter.io

### 1. Workflow Overview

This workflow automates the process of finding professional email addresses associated with LinkedIn profile data stored in a Google Sheet. Its core purpose is to enrich contact data by identifying company domains using AI and then retrieving corresponding email addresses via Hunter.io. The workflow is targeted at sales, marketing, or recruitment professionals who want to automate domain detection and email discovery at scale.

The workflow consists of three main logical blocks:

- **1.1 Input Reception and Domain Identification:**  
  Reads contact information (name, position, description) from Google Sheets, then uses an AI agent powered by Google Gemini (PaLM) to detect the company domain or generate a targeted search term if the domain is missing.

- **1.2 Domain-Based Email Lookup:**  
  If a domain is identified, directly queries Hunter.io’s Email Finder API to retrieve email addresses and updates Google Sheets accordingly.

- **1.3 Domain Search and Email Lookup Fallback:**  
  If the domain is not found initially, performs a Google Custom Search with the AI-generated search term, extracts candidate domains using JavaScript, then applies another AI agent to select the correct domain. Finally, it queries Hunter.io with this domain and updates the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Domain Identification

- **Overview:**  
  This block triggers the workflow manually, fetches rows from a Google Sheet containing LinkedIn profile data, and uses AI to identify the company domain or generate a search term.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get row(s) in sheet (Google Sheets)  
  - Google Gemini Chat Model (Google Gemini AI)  
  - AI Agent (Langchain Agent)  
  - Code in JavaScript (JSON Cleaner)  
  - Switch (Conditional Routing)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow on demand  
    - Configuration: Default, no parameters  
    - Inputs: None  
    - Outputs: To "Get row(s) in sheet"  
    - Failures: None expected; user-triggered

  - **Get row(s) in sheet**  
    - Type: Google Sheets node  
    - Role: Reads contact data (name, position, description) from a specific Google Sheet and worksheet  
    - Configuration:  
      - Document ID: specified Google Sheet storing LinkedIn profiles  
      - Sheet name: "Sheet1" (gid=0)  
      - Credentials: OAuth2 Google Sheets account  
    - Inputs: From manual trigger  
    - Outputs: To "AI Agent"  
    - Failures: Auth errors, sheet access issues, empty rows

  - **Google Gemini Chat Model**  
    - Type: Google Gemini (PaLM) Language Model  
    - Role: Provides AI language model capabilities for the Langchain agent  
    - Configuration: Connected to Google Gemini API credentials  
    - Inputs: From AI Agent (ai_languageModel input)  
    - Outputs: To AI Agent  
    - Failures: API key invalid, request timeout, quota exceeded

  - **AI Agent**  
    - Type: Langchain Agent Node  
    - Role: Processes profile data to detect company domain or formulate a search term  
    - Configuration:  
      - Prompt instructs strict JSON output of either domain or search term  
      - Inputs variables: name, position, description from sheet row  
    - Inputs: From "Get row(s) in sheet"  
    - Outputs: To "Code in JavaScript"  
    - Failures: AI hallucination, malformed JSON output

  - **Code in JavaScript**  
    - Type: Code Node  
    - Role: Cleans AI raw text output by removing markdown and parsing JSON  
    - Configuration: Custom JS script to parse AI output reliably  
    - Inputs: From AI Agent  
    - Outputs: To Switch  
    - Failures: JSON parse errors if AI output is invalid

  - **Switch**  
    - Type: Switch Node  
    - Role: Routes data depending on whether a domain or search term was returned by AI  
    - Configuration:  
      - Condition 1: domain is not "null" → output “domain”  
      - Condition 2: search term is not "null" → output "no domain"  
    - Inputs: From Code in JavaScript  
    - Outputs: To either "HTTP Request" or "HTTP Request1" nodes  
    - Failures: Condition mismatches, unexpected values

#### 2.2 Domain-Based Email Lookup

- **Overview:**  
  If the domain is known, calls Hunter.io Email Finder API with the domain and contact name, then updates the Google Sheet with the found email and domain.

- **Nodes Involved:**  
  - HTTP Request (Hunter.io email finder)  
  - Append or update row in sheet (Google Sheets)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Queries Hunter.io Email Finder API with domain, first name, last name to get email  
    - Configuration:  
      - URL: Hunter.io email-finder endpoint  
      - Query Parameters: domain, first_name, last_name (from sheet data and AI domain)  
      - Authentication: HTTP Bearer with Hunter.io API token  
    - Inputs: From Switch output “domain”  
    - Outputs: To Append or update row in sheet  
    - Failures: Auth errors, API rate limits, no email found

  - **Append or update row in sheet**  
    - Type: Google Sheets  
    - Role: Updates the original Google Sheet row with retrieved email and domain  
    - Configuration:  
      - Document and sheet same as input  
      - Matching column: name  
      - Fields updated: name, email, domain  
      - Credentials: same Google Sheets OAuth2  
    - Inputs: From HTTP Request  
    - Outputs: None (end of this path)  
    - Failures: Write permission errors, data mismatches

#### 2.3 Domain Search and Email Lookup Fallback

- **Overview:**  
  If no domain was initially found, the workflow performs a Google Custom Search using the AI-generated search term, extracts candidate domains via JavaScript, uses a second AI agent to select the correct domain, then calls Hunter.io and updates the sheet.

- **Nodes Involved:**  
  - HTTP Request1 (Google Custom Search API)  
  - Code in JavaScript1 (Domain extraction from search results)  
  - AI Agent1 (Domain selection)  
  - Google Gemini Chat Model1  
  - Code in JavaScript2 (Clean AI output)  
  - HTTP Request2 (Hunter.io email finder)  
  - Append or update row in sheet1 (Google Sheets update)

- **Node Details:**

  - **HTTP Request1**  
    - Type: HTTP Request  
    - Role: Performs Google Custom Search API call with the AI-generated search term  
    - Configuration:  
      - URL: Google Custom Search endpoint  
      - Query Params: cx (Custom Search Engine ID), q (search term)  
      - Authentication: HTTP Basic Auth (Google API key)  
    - Inputs: From Switch output “no domain”  
    - Outputs: To Code in JavaScript1  
    - Failures: Auth errors, quota limits, malformed search term

  - **Code in JavaScript1**  
    - Type: Code Node  
    - Role: Parses Google search results JSON to extract all domain candidates from URLs  
    - Configuration: Custom JS script that extracts hostnames from multiple URL fields in the search results  
    - Inputs: From HTTP Request1  
    - Outputs: To AI Agent1  
    - Failures: Parsing errors, unexpected result formats

  - **AI Agent1**  
    - Type: Langchain Agent Node  
    - Role: Matches search term with extracted domain list to pick the correct domain  
    - Configuration:  
      - Prompt requests strict JSON output containing the selected domain  
      - Inputs: search term (from Switch node), domains array (from previous code node)  
    - Inputs: From Code in JavaScript1  
    - Outputs: To Code in JavaScript2  
    - Failures: Incorrect matching, AI hallucination

  - **Google Gemini Chat Model1**  
    - Type: Google Gemini (PaLM) Language Model  
    - Role: Provides AI language model support for AI Agent1  
    - Configuration: Uses same Google Gemini API credentials  
    - Inputs: From AI Agent1 (ai_languageModel)  
    - Outputs: To AI Agent1  
    - Failures: API limits, invalid key

  - **Code in JavaScript2**  
    - Type: Code Node  
    - Role: Cleans AI output JSON from AI Agent1 by stripping markdown and parsing JSON  
    - Configuration: Similar script to first cleaning node  
    - Inputs: From AI Agent1  
    - Outputs: To HTTP Request2  
    - Failures: JSON parsing errors

  - **HTTP Request2**  
    - Type: HTTP Request  
    - Role: Calls Hunter.io Email Finder API with the newly identified domain and contact names  
    - Configuration: Same as HTTP Request node but uses domain from fallback AI output  
    - Inputs: From Code in JavaScript2  
    - Outputs: To Append or update row in sheet1  
    - Failures: Same as HTTP Request

  - **Append or update row in sheet1**  
    - Type: Google Sheets  
    - Role: Updates the sheet row with found email and domain from fallback lookup  
    - Configuration: Same as first Append or update node  
    - Inputs: From HTTP Request2  
    - Outputs: None (end of workflow)  
    - Failures: Write permission, data integrity issues

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                               | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                                                   |
|----------------------------|--------------------------------|-----------------------------------------------|------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Entry point for manual execution               | None                         | Get row(s) in sheet              |                                                                                                                               |
| Get row(s) in sheet        | Google Sheets                 | Reads LinkedIn profile rows from Google Sheet | When clicking ‘Execute workflow’ | AI Agent                        | Covered in "1. get sheet data and check for domain is provided" sticky note                                                   |
| AI Agent                   | Langchain Agent (AI)          | Detects company domain or generates search term | Get row(s) in sheet           | Code in JavaScript               |                                                                                                                               |
| Google Gemini Chat Model   | AI Language Model (Google Gemini) | Provides AI model support for AI Agent         | AI Agent (ai_languageModel)   | AI Agent                        |                                                                                                                               |
| Code in JavaScript         | Code Node                    | Cleans and parses AI JSON output                | AI Agent                     | Switch                          |                                                                                                                               |
| Switch                    | Switch                       | Routes flow based on domain presence            | Code in JavaScript           | HTTP Request / HTTP Request1    |                                                                                                                               |
| HTTP Request              | HTTP Request (Hunter.io)      | Hunter.io API call when domain found            | Switch (domain)              | Append or update row in sheet    | Covered in "if domain exists find mail from hunter.io and update google sheets" sticky note                                    |
| Append or update row in sheet | Google Sheets                 | Updates Google Sheet with email and domain      | HTTP Request                 | None                           |                                                                                                                               |
| HTTP Request1             | HTTP Request (Google Custom Search) | Google Search API call for domain fallback      | Switch (no domain)           | Code in JavaScript1             | Covered in "search for domain and find the mail to update sheets" sticky note                                                 |
| Code in JavaScript1       | Code Node                    | Extracts domain candidates from Google search results | HTTP Request1                | AI Agent1                      |                                                                                                                               |
| AI Agent1                 | Langchain Agent (AI)          | Selects correct domain from candidate list      | Code in JavaScript1          | Code in JavaScript2             |                                                                                                                               |
| Google Gemini Chat Model1 | AI Language Model (Google Gemini) | Provides AI model support for AI Agent1         | AI Agent1 (ai_languageModel) | AI Agent1                      |                                                                                                                               |
| Code in JavaScript2       | Code Node                    | Cleans and parses AI Agent1 JSON output         | AI Agent1                    | HTTP Request2                  |                                                                                                                               |
| HTTP Request2             | HTTP Request (Hunter.io)      | Hunter.io API call with fallback domain         | Code in JavaScript2          | Append or update row in sheet1  |                                                                                                                               |
| Append or update row in sheet1 | Google Sheets                 | Updates Google Sheet with email and domain from fallback | HTTP Request2                | None                           |                                                                                                                               |
| Sticky Note               | Sticky Note                  | Workflow overview and setup instructions        | None                         | None                           | "Automated Email Finder from linkedin profile" detailed explanation and setup checklist                                         |
| Sticky Note1              | Sticky Note                  | Marks block 1: sheet data reading and domain check | None                         | None                           | "1. get sheet data and check for domain is provided"                                                                          |
| Sticky Note2              | Sticky Note                  | Marks block 2: domain exists - find mail & update | None                         | None                           | "2. if domain exists find mail from hunter.io and update google sheets"                                                        |
| Sticky Note3              | Sticky Note                  | Marks block 3: search for domain fallback and mail find | None                         | None                           | "3. search for domain and find the mail to update sheets"                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "When clicking ‘Execute workflow’"  
   - No special configuration.

2. **Add Google Sheets Node to Get Rows:**  
   - Name: "Get row(s) in sheet"  
   - Operation: Read rows  
   - Document ID: Your Google Sheet ID containing LinkedIn profile data  
   - Sheet Name: "Sheet1" or your target sheet  
   - Credentials: Connect Google Sheets OAuth2 account

3. **Add Google Gemini Chat Model Node:**  
   - Name: "Google Gemini Chat Model"  
   - Credentials: Add Google Gemini (PaLM) API credentials

4. **Add Langchain Agent Node:**  
   - Name: "AI Agent"  
   - Prompt:  
     ```
     you have to identify the domain of the company from provided data.
     if domain is not provided then make a search term that help me find the domain which should end with "official site"
     make value "null" if not available
     don't hallucinate or make assumptions
     strictly output in this json format
     {
       "domain": "company-domain/null",
       "search term": "search-term/null"
     }
     data:
      name:{{ $json.name }}
      position:{{ $json.position }}
      description:{{ $json.description }}
     ```  
   - Set "promptType" to "define"  
   - Connect Google Gemini Chat Model node as the language model input

5. **Add Code Node to Clean AI Output:**  
   - Name: "Code in JavaScript"  
   - Paste the script to remove markdown and parse JSON output from AI Agent.

6. **Add Switch Node to Route Based on Output:**  
   - Name: "Switch"  
   - Condition 1: If `domain` is not "null" → Output "domain"  
   - Condition 2: If `search term` is not "null" → Output "no domain"

7. **Add HTTP Request Node for Hunter.io Email Lookup (Domain Found Path):**  
   - Name: "HTTP Request"  
   - Method: GET  
   - URL: `https://api.hunter.io/v2/email-finder?`  
   - Query Parameters: domain, first_name, last_name (from sheet data and AI output)  
   - Authentication: HTTP Bearer with Hunter.io API Key

8. **Add Google Sheets Node to Append or Update Row (Domain Found):**  
   - Name: "Append or update row in sheet"  
   - Operation: Append or Update  
   - Matching Column: name  
   - Columns to update: name, email (from Hunter.io), domain  
   - Credentials: Google Sheets OAuth2  
   - Document ID and Sheet as in step 2

9. **Connect "Switch" "domain" output to HTTP Request, then to Append or update row in sheet**

10. **Add HTTP Request Node for Google Custom Search (Domain Not Found Path):**  
    - Name: "HTTP Request1"  
    - Method: GET  
    - URL: `https://www.googleapis.com/customsearch/v1`  
    - Query Parameters: cx (Custom Search Engine ID), q (search term from AI output)  
    - Authentication: HTTP Basic Auth (Google API key)

11. **Add Code Node to Extract Domains from Search Results:**  
    - Name: "Code in JavaScript1"  
    - Paste the provided JavaScript code that extracts domains from various URL fields

12. **Add Langchain Agent Node to Select Domain:**  
    - Name: "AI Agent1"  
    - Prompt:  
      ```
      you have to simple matching and finding task where i will give a search term and list of domain you have to find the real domain from that 
      strictly output in json
      {
        "domain": "domain"
      }
      search term:  {{ $('Switch').item.json["search term"] }}
      list of domains: {{ $json.domains }}
      ```  
    - Connect a Google Gemini Chat Model node (same as before) as language model

13. **Add Code Node to Clean AI Agent1 Output:**  
    - Name: "Code in JavaScript2"  
    - Use same cleaning script as before

14. **Add HTTP Request Node for Hunter.io Email Lookup (Fallback Domain):**  
    - Name: "HTTP Request2"  
    - Same as step 7 but domain from AI Agent1 output

15. **Add Google Sheets Node to Append or Update Row (Fallback Path):**  
    - Name: "Append or update row in sheet1"  
    - Same configuration as step 8

16. **Connect "Switch" "no domain" output to HTTP Request1, then Code in JavaScript1, AI Agent1, Code in JavaScript2, HTTP Request2, and finally Append or update row in sheet1**

17. **Validate all credentials:**  
    - Google Sheets OAuth2 API  
    - Google Gemini (PaLM) API  
    - Hunter.io API (Bearer token)  
    - Google Custom Search API (Basic Auth or API key as required)

18. **Test end-to-end by triggering manually and ensuring updates in Google Sheet**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Automated Email Finder from LinkedIn profile using AI to detect domains and Hunter.io to find emails. Setup requires keys for Google Sheets, Google Gemini (PaLM), Hunter.io, and Google Custom Search API.                                                                                   | Sticky Note on workflow overview                                                                                                                                    |
| Google Gemini (PaLM) nodes require API access and proper credential setup in n8n.                                                                                                                                                                                                              | Google Gemini Chat Model nodes                                                                                                                                    |
| Hunter.io API bearer token must be kept secure and is used in HTTP Request nodes for email lookup.                                                                                                                                                                                            | HTTP Request nodes calling Hunter.io API                                                                                                                          |
| Google Custom Search API requires a valid Custom Search Engine ID (CX) and API key.                                                                                                                                                                                                            | HTTP Request1 node for domain fallback search                                                                                                                     |
| The AI agents use strict JSON output prompts to avoid hallucinations and ensure parsable results.                                                                                                                                                                                              | AI Agent and AI Agent1 nodes prompts                                                                                                                             |
| The workflow handles two paths: direct domain email lookup and fallback domain search with AI verification, ensuring higher accuracy and fallback resilience.                                                                                                                                   | General workflow logic and Switch node                                                                                                                           |

---

This documentation fully describes the workflow "LinkedIn Email Finder with AI Domain Detection using Google Sheets & Hunter.io" to enable understanding, modification, and reliable reproduction.