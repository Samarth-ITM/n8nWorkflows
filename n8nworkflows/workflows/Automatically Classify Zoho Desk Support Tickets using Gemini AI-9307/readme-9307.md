Automatically Classify Zoho Desk Support Tickets using Gemini AI

https://n8nworkflows.xyz/workflows/automatically-classify-zoho-desk-support-tickets-using-gemini-ai-9307


# Automatically Classify Zoho Desk Support Tickets using Gemini AI

### 1. Workflow Overview

This workflow automates the classification of support tickets from Zoho Desk using Gemini AI via OpenRouter. It targets customer support teams who want to categorize incoming tickets automatically for improved routing, prioritization, and analytics. The workflow processes all unclassified tickets in batches, analyzing their title and first request message to assign a relevant category.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Pagination:** Trigger and fetch all Zoho Desk tickets with pagination.
- **1.2 Filtering Unclassified Tickets:** Isolate tickets without existing classification.
- **1.3 Retrieve Ticket Content:** Fetch ticket conversation threads and extract the first message.
- **1.4 AI-based Classification:** Use Gemini AI to classify the ticket into predefined categories.
- **1.5 Update Zoho Desk Tickets:** Persist the AI-generated classification back into Zoho Desk.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Pagination

**Overview:**  
This block initiates the workflow manually and fetches all Zoho Desk tickets in pages of 100, sorted by creation date. It ensures the entire ticket dataset is retrieved through pagination until no more tickets remain.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Fetch All Tickets  
- Split Tickets  

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually for testing or on-demand execution.  
  - *Config:* No parameters; simply triggers the flow.  
  - *Input/Output:* No input; outputs initial trigger signal to "Fetch All Tickets".  
  - *Failures:* None expected.

- **Fetch All Tickets**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves Zoho Desk tickets with API pagination.  
  - *Config:*  
    - URL: `https://desk.zoho.eu/api/v1/tickets/search`  
    - Pagination using 'from' parameter (`$pageCount * 100`)  
    - Limit: 100 per page  
    - Sorted by `createdTime`  
    - OAuth2 authentication with Zoho credentials  
    - Pagination ends when response data length is 0 or less than 100  
  - *Input:* Trigger from manual node  
  - *Output:* JSON list of tickets to "Split Tickets"  
  - *Error Handling:* Continues on error to avoid stopping entire flow  
  - *Potential Failures:* OAuth2 token expiry, API rate limits, network errors.

- **Split Tickets**  
  - *Type:* Split Out  
  - *Role:* Transforms the bulk tickets array into individual ticket items for downstream processing.  
  - *Config:* Splits on the `data` field of the fetched tickets.  
  - *Input:* Ticket list array from "Fetch All Tickets"  
  - *Output:* Individual ticket JSON objects to filtering node  
  - *Failures:* None expected.

---

#### 1.2 Filtering Unclassified Tickets

**Overview:**  
Filters out tickets that already have a classification to avoid redundant processing.

**Nodes Involved:**  
- Filter classification = null  

**Node Details:**  

- **Filter classification = null**  
  - *Type:* Code (JavaScript)  
  - *Role:* Selects tickets with `classification` field equal to null.  
  - *Config:*  
    - Filters input items using: `item.json.classification === null`  
  - *Input:* Individual tickets from "Split Tickets"  
  - *Output:* Filtered tickets without classification to "Get threads"  
  - *Failures:* Expression errors if input lacks expected fields; no fallback implemented.

---

#### 1.3 Retrieve Ticket Content

**Overview:**  
Fetches conversation threads for each unclassified ticket and extracts the first message to provide sufficient context for classification.

**Nodes Involved:**  
- Get threads  
- Get first thread  

**Node Details:**  

- **Get threads**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves all conversation threads for a given ticket.  
  - *Config:*  
    - URL constructed with ticket ID from filtered ticket  
    - Query parameter: sortBy = sendDateTime (ascending)  
    - OAuth2 authentication  
  - *Input:* Tickets filtered to null classification  
  - *Output:* Threads array to "Get first thread"  
  - *Error Handling:* Continues on error to avoid halting  
  - *Failures:* OAuth2 issues, empty thread arrays, API errors.

- **Get first thread**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves the content of the first thread message only.  
  - *Config:*  
    - URL built with ticket ID and first thread ID (`data[0].id`)  
    - Query parameter: include = plainText (to get message body)  
    - OAuth2 authentication  
  - *Input:* Thread list from "Get threads"  
  - *Output:* Plain text first message to the AI classification node  
  - *Error Handling:* Continues on error  
  - *Failures:* Missing threads, indexing errors if no first thread, API errors.

---

#### 1.4 AI-based Classification

**Overview:**  
Uses Gemini AI via OpenRouter to classify the ticket based on its title and first message content into one of eight predefined categories.

**Nodes Involved:**  
- OpenRouter Chat Model  
- Classify  

**Node Details:**  

- **OpenRouter Chat Model**  
  - *Type:* Langchain OpenRouter Chat Model  
  - *Role:* Provides the Gemini AI model interface for language processing.  
  - *Config:*  
    - Model: `google/gemini-2.5-flash-lite-preview-09-2025`  
    - Credentials: OpenRouter API key  
  - *Input:* Prompt from "Classify" node (via langchain chaining)  
  - *Output:* AI response text to "Classify" node  
  - *Failures:* API errors, auth failures, model unavailability.

- **Classify**  
  - *Type:* Langchain LLM Chain  
  - *Role:* Constructs prompt and invokes AI to classify tickets.  
  - *Config:*  
    - Prompt defines role as expert classification system  
    - Task: classify ticket into categories: Content, Contract, Invoice, Featured Products, Affiliate-Partner, Bug, Feature, Other  
    - Uses ticket subject and first thread plainText in prompt via expressions  
  - *Input:* First thread message content  
  - *Output:* Classification text to "Update Ticket" node  
  - *Failures:* Prompt or expression errors, AI response unexpected formats.

---

#### 1.5 Update Zoho Desk Tickets

**Overview:**  
Saves the AI-generated classification back to the corresponding Zoho Desk ticket, enabling downstream use of this metadata.

**Nodes Involved:**  
- Update Ticket  

**Node Details:**  

- **Update Ticket**  
  - *Type:* HTTP Request  
  - *Role:* Updates the ticket's classification field in Zoho Desk.  
  - *Config:*  
    - URL built with ticket ID from filtered tickets  
    - Method: PUT  
    - Body parameter: classification = AI output text  
    - OAuth2 authentication  
  - *Input:* AI classification output  
  - *Output:* None (ends flow)  
  - *Error Handling:* Continues on error to allow other tickets to process  
  - *Failures:* OAuth2 token expiry, malformed classification data, API errors.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                     | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                                         |
|----------------------------|---------------------------------|-----------------------------------|-------------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Workflow start trigger             | None                          | Fetch All Tickets            | ## 🎯 WORKFLOW PURPOSE  Automatically classifies Zoho Desk tickets using AI based on their title and content. Processes all unclassified tickets in batches. |
| Fetch All Tickets          | HTTP Request                    | Fetch tickets with pagination      | When clicking ‘Execute workflow’ | Split Tickets               | ## 📥 FETCH TICKETS WITH PAGINATION  Retrieves 100 tickets/page, paginates all results, sorted by creation time  See OAuth2 setup guide: https://gist.github.com/Julian194/7c0ef5abaa5e3850f2bcc0a51bcd4633 |
| Split Tickets              | Split Out                      | Split bulk tickets into items      | Fetch All Tickets             | Filter classification = null |                                                                                                                                     |
| Filter classification = null | Code                           | Filter tickets missing classification | Split Tickets                | Get threads                 | ## 🔍 FILTER LOGIC  Only processes tickets with null classification to prevent reprocessing. Alternative: filter in API query params for efficiency.      |
| Get threads                | HTTP Request                   | Retrieve ticket conversation threads | Filter classification = null | Get first thread            |                                                                                                                                     |
| Get first thread           | HTTP Request                   | Retrieve first message content     | Get threads                  | Classify                    |                                                                                                                                     |
| OpenRouter Chat Model      | Langchain OpenRouter Chat Model | AI language model interface        | Classify                    | Classify (AI response)      |                                                                                                                                     |
| Classify                  | Langchain LLM Chain             | Prepare prompt and classify ticket | Get first thread             | Update Ticket               | ## 🤖 AI CLASSIFICATION CATEGORIES  Categories: Content, Contract, Invoice, Featured Products, Affiliate-Partner, Bug, Feature, Other. Customize as needed.   |
| Update Ticket             | HTTP Request                   | Update Zoho ticket with classification | Classify                    | None                       | ## 💾 SAVE CLASSIFICATION  Updates ticket with AI classification. Continues on errors to avoid stopping batch.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking ‘Execute workflow’" to start the workflow on demand.

2. **Add an HTTP Request node "Fetch All Tickets":**  
   - URL: `https://desk.zoho.eu/api/v1/tickets/search`  
   - Method: GET  
   - Query Parameters:  
     - limit = 100  
     - sortBy = createdTime  
   - Enable Pagination:  
     - Pagination parameter: `from = {{$pageCount * 100}}`  
     - Stop when response data length is 0 or less than 100  
   - Authentication: OAuth2 with Zoho Desk credentials configured  
   - Connect "When clicking ‘Execute workflow’" → "Fetch All Tickets"

3. **Add a Split Out node "Split Tickets":**  
   - Field to split out: `data`  
   - Connect "Fetch All Tickets" → "Split Tickets"

4. **Add a Code node "Filter classification = null":**  
   - JavaScript code to filter tickets where `classification === null`:  
     ```js
     const filteredItems = $input.all().filter(item => item.json.classification === null);
     return filteredItems;
     ```  
   - Connect "Split Tickets" → "Filter classification = null"

5. **Add HTTP Request node "Get threads":**  
   - URL: `https://desk.zoho.eu/api/v1/tickets/{{$json.id}}/threads`  
   - Method: GET  
   - Query Parameter: `sortBy=sendDateTime`  
   - Authentication: OAuth2 with Zoho Desk credentials  
   - On error: Continue  
   - Connect "Filter classification = null" → "Get threads"

6. **Add HTTP Request node "Get first thread":**  
   - URL: `https://desk.zoho.eu/api/v1/tickets/{{$("Filter classification = null").item.json.id}}/threads/{{$json.data[0].id}}`  
   - Query Parameter: `include=plainText`  
   - Method: GET  
   - Authentication: OAuth2 with Zoho Desk credentials  
   - On error: Continue  
   - Connect "Get threads" → "Get first thread"

7. **Add Langchain OpenRouter Chat Model node "OpenRouter Chat Model":**  
   - Model: `google/gemini-2.5-flash-lite-preview-09-2025`  
   - Credentials: OpenRouter API key configured  
   - No direct input connections; will be used as AI model for chain node.

8. **Add Langchain LLM Chain node "Classify":**  
   - Prompt type: Define prompt  
   - Text:  
     ```
     **Role:** You are an expert support ticket classification system.

     **Task:** Read the provided ticket title and request body. Based on the content, classify the ticket into one of the following categories. Respond with only the single, most appropriate category name.

     **Categories:**
     • Content
     • Contract
     • Invoice
     • Featured Products
     • Affiliate-Partner
     • Bug
     • Feature
     • Other

     ---

     **Ticket Title:**
     {{$("Filter classification = null").item.json.subject}}

     **Ticket Request:**
     {{$json.plainText}}

     **Category:**
     ```
   - Connect "OpenRouter Chat Model" → "Classify" (as the language model input)  
   - Connect "Get first thread" → "Classify" (as prompt input)

9. **Add HTTP Request node "Update Ticket":**  
   - URL: `https://desk.zoho.eu/api/v1/tickets/{{$("Filter classification = null").item.json.id}}`  
   - Method: PUT  
   - Body Parameter: JSON with `"classification": "={{$json.text}}"` (AI output)  
   - Authentication: OAuth2 with Zoho Desk credentials  
   - On error: Continue  
   - Connect "Classify" → "Update Ticket"

10. **Test the workflow:**  
    - Ensure OAuth2 credentials for Zoho Desk are valid and refreshed.  
    - Ensure OpenRouter API key is configured.  
    - Run the manual trigger node to execute the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| OAuth2 setup for Zoho Desk can be tricky. Follow detailed instructions at: https://gist.github.com/Julian194/7c0ef5abaa5e3850f2bcc0a51bcd4633                                                                                                                                                                                                                                | OAuth2 Setup Guide                                                                                   |
| Pagination pattern used (`from` parameter with `pageCount * 100`) is a best practice that applies to all Zoho Desk paginated endpoints. Stop condition is when returned data length < 100 or 0.                                                                                                                                                                               | Pagination Explanation                                                                               |
| AI classification categories are fully customizable. You may adapt categories and prompt to classify other attributes such as urgency, department, or product type.                                                                                                                                                                                                          | AI Classification Categories and Customization                                                     |
| This workflow includes error handling to continue processing other tickets if one ticket retrieval or update fails. This ensures robustness in batch processing of many tickets.                                                                                                                                                                                            | Error Handling Strategy                                                                              |
| The workflow uses Gemini AI via OpenRouter. Ensure your OpenRouter account has access to the specified model and API quota.                                                                                                                                                                                                                                                  | OpenRouter Gemini AI Model Access                                                                   |
| The workflow handles only the first thread message per ticket for classification to optimize API calls and processing time. You may extend to analyze more messages if needed.                                                                                                                                                                                               | Ticket Content Extraction Note                                                                      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.