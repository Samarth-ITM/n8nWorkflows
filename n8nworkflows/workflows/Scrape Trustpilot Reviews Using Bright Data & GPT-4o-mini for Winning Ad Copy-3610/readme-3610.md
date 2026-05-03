Scrape Trustpilot Reviews Using Bright Data & GPT-4o-mini for Winning Ad Copy

https://n8nworkflows.xyz/workflows/scrape-trustpilot-reviews-using-bright-data---gpt-4o-mini-for-winning-ad-copy-3610


# Scrape Trustpilot Reviews Using Bright Data & GPT-4o-mini for Winning Ad Copy

### 1. Workflow Overview

This workflow automates the process of scraping competitor reviews from Trustpilot using Bright Data’s dataset API, analyzing negative feedback with OpenAI’s GPT-4o-mini, and generating targeted Facebook ad copy based on competitor pain points. It is designed for marketers, business owners, and agencies who want to automate competitor analysis and ad copy generation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures competitor Trustpilot URL and desired review timeframe via a manual form trigger.
- **1.2 Data Fetching & Polling:** Initiates a Bright Data dataset snapshot request and polls until the data is ready.
- **1.3 Data Retrieval & Storage:** Retrieves the completed snapshot data and appends all reviews into a Google Sheet.
- **1.4 Filtering & Aggregation:** Filters reviews to only 1-2 star ratings and aggregates the negative reviews into a single summarized text.
- **1.5 AI Processing & Ad Copy Generation:** Uses OpenAI GPT-4o-mini to analyze aggregated negative reviews and generate three variations of Facebook ad copy.
- **1.6 Distribution:** Sends the summarized insights and generated ad copy via email to the marketing team.
- **1.7 Workflow Assistance & Documentation:** Sticky notes provide setup instructions, tips, and support contact information.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block collects user input via a form trigger node, requesting the competitor’s Trustpilot URL and the timeframe for reviews to scrape.

**Nodes Involved:**  
- On form submission - Discover Jobs  
- Sticky Note1  

**Node Details:**  

- **On form submission - Discover Jobs**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input  
  - Configuration:  
    - Webhook ID configured for external form submission  
    - Form fields:  
      - Competitor TRUSTPILOT URL (required, placeholder example provided)  
      - Review timeframe dropdown (options: Last 30 days, 3 months, 6 months, 12 months)  
  - Inputs: None (trigger node)  
  - Outputs: Passes form data to next node  
  - Edge cases: Missing or malformed URL input; invalid timeframe selection  
  - Version: 2.2  

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Instructional note prompting user to add competitor URL  
  - No inputs or outputs  

---

#### 1.2 Data Fetching & Polling

**Overview:**  
This block triggers Bright Data’s dataset API to start scraping reviews based on user input, then polls periodically until the snapshot is ready.

**Nodes Involved:**  
- HTTP Request- Post API call to Bright Data  
- Wait - Polling Bright Data  
- Snapshot Progress  
- If - Checking status of Snapshot - if data is ready or not  
- Sticky Note3  

**Node Details:**  

- **HTTP Request- Post API call to Bright Data**  
  - Type: HTTP Request  
  - Role: Initiates Bright Data dataset snapshot creation  
  - Configuration:  
    - POST to Bright Data dataset trigger endpoint  
    - JSON body includes user-provided URL and timeframe  
    - Query parameters specify dataset ID and error inclusion  
    - Authorization header with Bearer token (Bright Data API key)  
  - Inputs: Form data from trigger node  
  - Outputs: Snapshot ID for polling  
  - Edge cases: API auth failure, invalid URL, Bright Data service downtime  
  - Version: 4.2  

- **Wait - Polling Bright Data**  
  - Type: Wait  
  - Role: Pauses workflow for 2 minutes between polling attempts  
  - Configuration: Wait 2 minutes, execute repeatedly until data ready  
  - Inputs: From HTTP Request node  
  - Outputs: Triggers Snapshot Progress node  
  - Edge cases: Excessive wait time if snapshot delayed  

- **Snapshot Progress**  
  - Type: HTTP Request  
  - Role: Checks snapshot status using snapshot ID  
  - Configuration:  
    - GET request to Bright Data progress endpoint with snapshot ID  
    - Authorization header with Bearer token  
  - Inputs: From Wait node  
  - Outputs: Snapshot status JSON  
  - Edge cases: API errors, invalid snapshot ID  

- **If - Checking status of Snapshot - if data is ready or not**  
  - Type: If  
  - Role: Branches workflow based on snapshot status  
  - Configuration:  
    - Condition: If status equals "running" → continue waiting  
    - Else → proceed to data retrieval  
  - Inputs: Snapshot Progress node output  
  - Outputs:  
    - True branch: loops back to Wait node  
    - False branch: proceeds to data retrieval  
  - Edge cases: Unexpected status values  

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Notes that this block handles Bright Data review retrieval  

---

#### 1.3 Data Retrieval & Storage

**Overview:**  
Once the snapshot is ready, this block retrieves all review data from Bright Data and appends it into a structured Google Sheet for storage and further processing.

**Nodes Involved:**  
- HTTP Request - Getting data from Bright Data  
- Google Sheets - Adding All Reviews  
- Sticky Note10  

**Node Details:**  

- **HTTP Request - Getting data from Bright Data**  
  - Type: HTTP Request  
  - Role: Fetches snapshot data in JSON format  
  - Configuration:  
    - GET request to Bright Data snapshot endpoint with snapshot ID  
    - Query parameter: format=json  
    - Authorization header with Bearer token  
  - Inputs: From If node (snapshot ready branch)  
  - Outputs: JSON array of reviews  
  - Edge cases: API errors, large payloads causing timeouts  

- **Google Sheets - Adding All Reviews**  
  - Type: Google Sheets  
  - Role: Appends all fetched reviews into a Google Sheet  
  - Configuration:  
    - Document ID set to a shared template sheet (user must copy)  
    - Sheet name: gid=0 (default sheet)  
    - Auto-mapping input data fields to sheet columns (e.g., review_id, review_rating, review_content, etc.)  
    - OAuth2 credentials required for Google Sheets access  
  - Inputs: JSON reviews from Bright Data  
  - Outputs: Passes data to filtering node  
  - Edge cases: OAuth token expiration, Google Sheets API quota limits, malformed data  

- **Sticky Note10**  
  - Type: Sticky Note  
  - Role: Provides link and instructions to copy Google Sheets template for use  

---

#### 1.4 Filtering & Aggregation

**Overview:**  
Filters the stored reviews to isolate only negative feedback (1 or 2-star ratings) and aggregates their content into a single summarized text string.

**Nodes Involved:**  
- Filtering only bad reviews  
- Aggregating all filtered reviews  

**Node Details:**  

- **Filtering only bad reviews**  
  - Type: Filter  
  - Role: Filters reviews to only those with rating 1 or 2 stars  
  - Configuration:  
    - Conditions: review_rating equals "1" OR review_rating equals "2"  
    - Loose type validation enabled for flexibility  
  - Inputs: From Google Sheets node  
  - Outputs: Filtered reviews only  
  - Edge cases: Missing or malformed rating fields  

- **Aggregating all filtered reviews**  
  - Type: Aggregate  
  - Role: Concatenates all filtered review contents into one field named "Aggregated_reviews"  
  - Configuration:  
    - Aggregates field "review_content" into one string  
  - Inputs: Filtered reviews  
  - Outputs: Single aggregated text for AI processing  
  - Edge cases: Very large text aggregation may hit size limits  

---

#### 1.5 AI Processing & Ad Copy Generation

**Overview:**  
Uses OpenAI GPT-4o-mini to analyze aggregated negative reviews, summarize competitor weaknesses, and generate three Facebook ad copy variations targeting those pain points.

**Nodes Involved:**  
- Basic LLM Chain  
- OpenAI Chat Model  
- Sticky Note4  

**Node Details:**  

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain  
  - Role: Defines prompt template and sends aggregated reviews for summarization and ad copy generation  
  - Configuration:  
    - Prompt instructs to read aggregated bad reviews, summarize weaknesses without naming competitor, and write 3 different Facebook ad copies addressing concerns  
    - Uses expression to insert aggregated reviews dynamically  
  - Inputs: Aggregated reviews from previous node  
  - Outputs: Prompt text for OpenAI Chat Model  
  - Edge cases: Prompt formatting errors, empty input text  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Executes the prompt using GPT-4o-mini model  
  - Configuration:  
    - Model: gpt-4o-mini  
    - Credentials: OpenAI API key configured via OAuth2 credential  
  - Inputs: Prompt from Basic LLM Chain  
  - Outputs: Generated ad copy and summary text  
  - Edge cases: API rate limits, invalid API key, model unavailability  

- **Sticky Note4**  
  - Type: Sticky Note  
  - Role: Instructions to customize the prompt for company info, tone, or marketing material type  

---

#### 1.6 Distribution

**Overview:**  
Sends the generated summary and ad copy via email to the marketing team for review and action.

**Nodes Involved:**  
- Send Summary To Marketers  

**Node Details:**  

- **Send Summary To Marketers**  
  - Type: Gmail Node  
  - Role: Sends an email with the summary, ad copy, and aggregated reviews  
  - Configuration:  
    - Recipient email address configured (placeholder "youremail@gmail.com")  
    - Subject includes competitor URL for context  
    - Email body includes:  
      - Trustpilot page URL  
      - Summary and ad copy text from AI output  
      - Aggregated complaints attached in text form  
    - Uses OAuth2 credentials for Gmail access  
  - Inputs: AI-generated text from Basic LLM Chain  
  - Outputs: None (end node)  
  - Edge cases: OAuth token expiration, email sending failures, invalid recipient address  

---

#### 1.7 Workflow Assistance & Documentation

**Overview:**  
Sticky notes provide user guidance, support contacts, and links to resources.

**Nodes Involved:**  
- Sticky Note9  
- Sticky Note  

**Node Details:**  

- **Sticky Note9**  
  - Type: Sticky Note  
  - Role: Provides support contact email, YouTube and LinkedIn links, and Bright Data documentation link  

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: High-level workflow description and summary for user understanding  

---

### 3. Summary Table

| Node Name                                  | Node Type                      | Functional Role                            | Input Node(s)                                   | Output Node(s)                                    | Sticky Note                                                                                      |
|--------------------------------------------|--------------------------------|--------------------------------------------|------------------------------------------------|--------------------------------------------------|-------------------------------------------------------------------------------------------------|
| On form submission - Discover Jobs          | Form Trigger                   | Captures user input (URL and timeframe)    | None                                           | HTTP Request- Post API call to Bright Data        | Sticky Note1 prompts user to add competitor URL                                                 |
| HTTP Request- Post API call to Bright Data  | HTTP Request                  | Triggers Bright Data dataset snapshot       | On form submission - Discover Jobs             | Wait - Polling Bright Data                         |                                                                                                 |
| Wait - Polling Bright Data                   | Wait                          | Waits 2 minutes between polling attempts    | HTTP Request- Post API call to Bright Data     | Snapshot Progress                                  |                                                                                                 |
| Snapshot Progress                            | HTTP Request                  | Checks snapshot status                       | Wait - Polling Bright Data                      | If - Checking status of Snapshot                   | Sticky Note3 notes Bright Data review retrieval                                                 |
| If - Checking status of Snapshot             | If                            | Branches on snapshot status                  | Snapshot Progress                               | Wait - Polling Bright Data (if running) / HTTP Request - Getting data from Bright Data (if ready) |                                                                                                 |
| HTTP Request - Getting data from Bright Data| HTTP Request                  | Retrieves snapshot data                       | If - Checking status of Snapshot                | Google Sheets - Adding All Reviews                 |                                                                                                 |
| Google Sheets - Adding All Reviews           | Google Sheets                 | Appends reviews to Google Sheet              | HTTP Request - Getting data from Bright Data   | Filtering only bad reviews                         | Sticky Note10 provides Google Sheets template link and instructions                             |
| Filtering only bad reviews                    | Filter                        | Filters reviews to 1-2 star ratings          | Google Sheets - Adding All Reviews              | Aggregating all filtered reviews                   |                                                                                                 |
| Aggregating all filtered reviews              | Aggregate                     | Aggregates filtered review content           | Filtering only bad reviews                       | Basic LLM Chain                                   |                                                                                                 |
| Basic LLM Chain                              | LangChain LLM Chain           | Prepares prompt for AI summarization & ad copy | Aggregating all filtered reviews                | Send Summary To Marketers                          | Sticky Note4 suggests prompt customization options                                             |
| OpenAI Chat Model                            | LangChain OpenAI Chat Model   | Executes GPT-4o-mini prompt                   | Basic LLM Chain                                 | Basic LLM Chain (feedback loop)                    |                                                                                                 |
| Send Summary To Marketers                     | Gmail                        | Sends email with summary and ad copy         | Basic LLM Chain                                 | None                                             |                                                                                                 |
| Sticky Note1                                 | Sticky Note                  | Instruction to add competitor Trustpilot URL | None                                           | None                                             |                                                                                                 |
| Sticky Note3                                 | Sticky Note                  | Notes Bright Data review retrieval            | None                                           | None                                             |                                                                                                 |
| Sticky Note4                                 | Sticky Note                  | Instructions to customize AI prompt           | None                                           | None                                             |                                                                                                 |
| Sticky Note9                                 | Sticky Note                  | Support contacts and resources                 | None                                           | None                                             |                                                                                                 |
| Sticky Note10                                | Sticky Note                  | Google Sheets template and setup instructions | None                                           | None                                             |                                                                                                 |
| Sticky Note                                  | Sticky Note                  | Workflow overview and summary                  | None                                           | None                                             |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure webhook ID (auto-generated or custom)  
   - Add form fields:  
     - Text field: "Competitor TRUSTPILOT URL (include https://www.trustpilot.com/review/)", required  
     - Dropdown field: "Please select the time frame of reviews you'd like." Options: Last 30 days, Last 3 months, Last 6 months, Last 12 months  

2. **Create HTTP Request Node to Trigger Bright Data Snapshot**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://api.brightdata.com/datasets/v3/trigger  
   - Query parameters:  
     - dataset_id = gd_lm5zmhwd2sni130p  
     - include_errors = true  
   - Headers: Authorization: Bearer <YOUR_BRIGHT_DATA_API_KEY>  
   - Body (JSON):  
     ```json
     [
       {
         "url": "{{ $json['Competitor TRUSTPILOT URL (include https://www.trustpilot.com/review/'] }}",
         "date_posted": "{{ $json['Please select the time frame of reviews you\'d like. If it\'s a big brand go with 30 days'] }}"
       }
     ]
     ```  
   - Connect output of Form Trigger node to this node  

3. **Create Wait Node for Polling**  
   - Type: Wait  
   - Duration: 2 minutes  
   - Set to execute repeatedly until snapshot ready  
   - Connect output of HTTP Request node to this node  

4. **Create HTTP Request Node to Check Snapshot Progress**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://api.brightdata.com/datasets/v3/progress/{{ $('HTTP Request- Post API call to Bright Data').item.json.snapshot_id }}  
   - Headers: Authorization: Bearer <YOUR_BRIGHT_DATA_API_KEY>  
   - Connect output of Wait node to this node  

5. **Create If Node to Check Snapshot Status**  
   - Type: If  
   - Condition: Check if JSON field "status" equals "running"  
   - True branch: Connect back to Wait node (to continue polling)  
   - False branch: Connect to next node (data retrieval)  

6. **Create HTTP Request Node to Get Snapshot Data**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://api.brightdata.com/datasets/v3/snapshot/{{ $('HTTP Request- Post API call to Bright Data').item.json.snapshot_id }}  
   - Query parameter: format=json  
   - Headers: Authorization: Bearer <YOUR_BRIGHT_DATA_API_KEY>  
   - Connect False branch output of If node to this node  

7. **Create Google Sheets Node to Append Reviews**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use copied Google Sheets template ID  
   - Sheet Name: gid=0  
   - Mapping mode: Auto map input data fields to sheet columns (ensure all review fields are mapped)  
   - Credentials: Configure OAuth2 credentials for Google Sheets access  
   - Connect output of HTTP Request (snapshot data) node to this node  

8. **Create Filter Node to Select Only 1 and 2 Star Reviews**  
   - Type: Filter  
   - Condition: review_rating equals "1" OR review_rating equals "2"  
   - Connect output of Google Sheets node to this node  

9. **Create Aggregate Node to Concatenate Filtered Reviews**  
   - Type: Aggregate  
   - Aggregate field: review_content  
   - Output field name: Aggregated_reviews  
   - Connect output of Filter node to this node  

10. **Create LangChain LLM Chain Node to Prepare Prompt**  
    - Type: LangChain LLM Chain  
    - Prompt:  
      ```
      Read the following bad reviews, these are reviews of our competitors:
      {{ $json.Aggregated_reviews }}

      ---
      After reading them, summarize their weakest points.
      Don't mention the competitor name.

      Write 3 different ads copy for our Facebook ads campaign, addressing these concerns
      ```  
    - Connect output of Aggregate node to this node  

11. **Create LangChain OpenAI Chat Model Node**  
    - Type: LangChain OpenAI Chat Model  
    - Model: gpt-4o-mini  
    - Credentials: Configure OpenAI API key  
    - Connect output of LangChain LLM Chain node to this node  

12. **Create Gmail Node to Send Email**  
    - Type: Gmail  
    - Recipient: your marketing team email (e.g., youremail@gmail.com)  
    - Subject: Summary of Complaints of competitor: {{ competitor URL from form }}  
    - Message body: Include competitor URL, AI-generated summary and ad copy, and aggregated complaints  
    - Credentials: Configure Gmail OAuth2 credentials  
    - Connect output of LangChain LLM Chain node (or OpenAI Chat Model node if output chaining required) to this node  

13. **Add Sticky Notes**  
    - Add sticky notes at appropriate points for user guidance, including:  
      - Workflow overview and instructions  
      - Google Sheets template link and setup instructions  
      - Bright Data docs and support contacts  
      - Prompt customization tips  

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron@nofluff.online                                                                           | Support email                                                                                     |
| YouTube tutorials and tips available at https://www.youtube.com/@YaronBeen/videos                                                    | YouTube channel for workflow assistance                                                         |
| LinkedIn profile: https://www.linkedin.com/in/yaronbeen/                                                                             | Professional contact                                                                            |
| Bright Data official documentation: https://docs.brightdata.com/introduction                                                         | API and dataset usage documentation                                                             |
| Google Sheets template for storing reviews: https://docs.google.com/spreadsheets/d/1Zi758ds2_aWzvbDYqwuGiQNaurLgs-leS9wjLWWlbUU/edit | Copy this sheet before use; do not change column headers                                         |
| Tips: Bright Data snapshots may take time; polling is automated. Focus on 1-2 star reviews for actionable insights. Customize GPT prompts for tone or vertical. | Best practices                                                                                   |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the "Scrape Trustpilot Reviews Using Bright Data & GPT-4o-mini for Winning Ad Copy" workflow. It anticipates potential failure points such as API authentication errors, data retrieval delays, and input validation issues, enabling robust error handling and customization.