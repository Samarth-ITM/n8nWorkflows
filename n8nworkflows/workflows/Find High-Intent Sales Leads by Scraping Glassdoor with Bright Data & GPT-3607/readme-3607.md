Find High-Intent Sales Leads by Scraping Glassdoor with Bright Data & GPT

https://n8nworkflows.xyz/workflows/find-high-intent-sales-leads-by-scraping-glassdoor-with-bright-data---gpt-3607


# Find High-Intent Sales Leads by Scraping Glassdoor with Bright Data & GPT

### 1. Workflow Overview

This workflow automates the process of discovering high-intent sales leads by scraping job listings from Glassdoor using Bright Data’s dataset API and generating targeted outreach pitches with OpenAI’s GPT models. It is designed primarily for sales teams, recruiters, and marketers who want to streamline job prospecting and lead generation.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Captures user input parameters (location, keyword, country) via a custom form trigger.
- **1.2 Bright Data Job Scraping:** Initiates a Bright Data dataset snapshot based on the input, polls for completion, and retrieves the job listings data.
- **1.3 Google Sheets Integration:** Stores the retrieved job listings into a structured Google Sheet template for organized data management.
- **1.4 Automated Pitch Generation (AI):** Processes job listings to generate personalized sales pitches using OpenAI GPT, then updates the Google Sheet with these AI-generated pitches.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects user-defined search parameters for job listings via a form trigger node. It enables dynamic and customizable queries for job location, keyword, and country code.

**Nodes Involved:**  
- On form submission - Discover Jobs  
- Sticky Note1 (Parameter Guide)

**Node Details:**

- **On form submission - Discover Jobs**  
  - Type: Form Trigger  
  - Role: Entry point; captures user input for job location, keyword, and country code.  
  - Configuration:  
    - Form fields:  
      - Job Location (required)  
      - Keyword (required)  
      - Country (2 letters, required)  
    - Webhook ID set for external form submissions.  
  - Inputs: None (trigger node)  
  - Outputs: Passes form data downstream as JSON.  
  - Edge Cases: Missing or invalid input fields may cause incomplete queries. Validation is enforced by required fields.  
  - Notes: Enables on-demand or scheduled runs via form submission.

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Documentation for users on how to format input parameters (location, keyword, country) with examples.  
  - No inputs or outputs.

---

#### 2.2 Bright Data Job Scraping

**Overview:**  
This block triggers a Bright Data dataset snapshot based on the input parameters, polls the snapshot progress until completion, and retrieves the full dataset of job listings.

**Nodes Involved:**  
- HTTP Request- Post API call to Bright Data  
- Wait - Polling Bright Data  
- Snapshot Progress  
- If - Checking status of Snapshot - if data is ready or not  
- HTTP Request - Getting data from Bright Data  
- Sticky Note2 (Bright Data Trigger explanation)  
- Sticky Note3 (Bright Data Getting Jobs)

**Node Details:**

- **HTTP Request- Post API call to Bright Data**  
  - Type: HTTP Request (POST)  
  - Role: Initiates a Bright Data dataset snapshot with user filters.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/trigger`  
    - Method: POST  
    - Headers: Authorization with Bearer token (Bright Data API key placeholder)  
    - Query Parameters: dataset_id, include_errors, type, discover_by, uncompressed_webhook  
    - Body: JSON array with location, keyword, country from form input  
  - Inputs: Receives form data from Input Reception block  
  - Outputs: Returns snapshot_id and initial response data  
  - Edge Cases: API key invalid or expired, network errors, invalid dataset ID, malformed input parameters.

- **Wait - Polling Bright Data**  
  - Type: Wait  
  - Role: Pauses workflow execution for a specified time (minutes) before polling snapshot progress again.  
  - Configuration: Wait unit set to minutes, no fixed duration (configured dynamically or default)  
  - Inputs: Triggered after snapshot initiation or after polling if snapshot not ready  
  - Outputs: Triggers Snapshot Progress node  
  - Edge Cases: Excessive wait time may delay workflow; too short may cause API rate limits.

- **Snapshot Progress**  
  - Type: HTTP Request (GET)  
  - Role: Checks the progress status of the Bright Data snapshot.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/progress/{{snapshot_id}}` (snapshot_id from previous node)  
    - Headers: Authorization with Bearer token  
  - Inputs: Triggered after Wait node  
  - Outputs: Returns snapshot status JSON  
  - Edge Cases: Snapshot ID invalid, API errors, network issues.

- **If - Checking status of Snapshot - if data is ready or not**  
  - Type: If  
  - Role: Evaluates if snapshot status is "running" to decide whether to wait more or proceed.  
  - Configuration: Condition checks if `status == "running"`  
  - Inputs: Snapshot Progress node output  
  - Outputs:  
    - True: Loop back to Wait node to poll again  
    - False: Proceed to retrieve data  
  - Edge Cases: Unexpected status values, missing status field.

- **HTTP Request - Getting data from Bright Data**  
  - Type: HTTP Request (GET)  
  - Role: Retrieves the full dataset snapshot once ready.  
  - Configuration:  
    - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{snapshot_id}}`  
    - Query Parameter: format=json  
    - Headers: Authorization with Bearer token  
  - Inputs: Triggered when snapshot is ready  
  - Outputs: Returns full job listings dataset JSON  
  - Edge Cases: Large dataset causing timeouts, API limits, invalid snapshot ID.

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Explains the Bright Data trigger node and customization options.  
  - No inputs or outputs.

- **Sticky Note3**  
  - Type: Sticky Note  
  - Role: Notes about Bright Data job retrieval process.  
  - No inputs or outputs.

---

#### 2.3 Google Sheets Integration

**Overview:**  
This block writes the retrieved job listings data into a Google Sheets document using a pre-built template, organizing job details for easy access and further processing.

**Nodes Involved:**  
- Google Sheets - Adding All Job Posts  
- Sticky Note10 (Google Sheets template instructions)

**Node Details:**

- **Google Sheets - Adding All Job Posts**  
  - Type: Google Sheets  
  - Role: Appends job listing data rows into a Google Sheet tab.  
  - Configuration:  
    - Operation: Append  
    - Document ID: Points to the provided Google Sheets template (user must copy and link their own)  
    - Sheet Name: Default tab (gid=0)  
    - Columns: Auto-mapped from input JSON fields (company_name, job_title, job_overview, company_rating, etc.)  
    - Handling Extra Data: Insert in new column if unexpected fields appear  
  - Inputs: Receives full job listings JSON from Bright Data snapshot retrieval  
  - Outputs: Passes data downstream for pitch generation  
  - Credentials: Google Sheets OAuth2 required  
  - Edge Cases: Sheet access permission errors, quota limits, malformed data rows.

- **Sticky Note10**  
  - Type: Sticky Note  
  - Role: Provides instructions and link to Google Sheets template for users to copy and use.  
  - No inputs or outputs.

---

#### 2.4 Automated Pitch Generation (AI)

**Overview:**  
This block splits job listing data into components, sends relevant fields to OpenAI GPT via LangChain to generate personalized sales pitches or icebreakers, and updates the Google Sheet with the generated content.

**Nodes Involved:**  
- Split Out  
- Basic LLM Chain  
- OpenAI Chat Model  
- Google Sheets - Update Pitches

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Separates the job listing data into individual parts for processing (company_name, job_title, description_text).  
  - Configuration: Field to split out includes `company_name, job_title, description_text` (description_text is likely mapped from job_overview or similar)  
  - Inputs: Receives job listings data from Google Sheets append node  
  - Outputs: Sends each split item downstream for AI processing  
  - Edge Cases: Missing fields may cause empty splits.

- **Basic LLM Chain**  
  - Type: LangChain LLM Chain  
  - Role: Defines the prompt and logic for generating pitches based on job data.  
  - Configuration:  
    - Prompt includes company name, job title, and job description overview.  
    - Conditional logic:  
      - If job relates to marketing, content creation, or audience engagement, generate 1–2 concise icebreaker sentences referencing company/job context and pitch content repurposing service.  
      - Otherwise, respond with "---JOB POST NOT RELEVANT---".  
    - Uses expressions to read fields from Google Sheets node data.  
  - Inputs: Receives split job data  
  - Outputs: Passes generated text downstream  
  - Edge Cases: Prompt failures, API errors, irrelevant job data.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Executes the GPT model call to generate text based on the prompt from Basic LLM Chain.  
  - Configuration:  
    - Model: "gpt-4o-mini" (a GPT-4 variant)  
    - Credentials: OpenAI API key required  
  - Inputs: Receives prompt from Basic LLM Chain  
  - Outputs: Returns generated pitch text  
  - Edge Cases: API rate limits, invalid API key, model unavailability.

- **Google Sheets - Update Pitches**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet with the AI-generated pitch text matched by company_name.  
  - Configuration:  
    - Operation: Update  
    - Matching Column: company_name  
    - Columns updated: Pitch (with generated text), company_name (for matching)  
    - Document and sheet same as previous Google Sheets node  
  - Inputs: Receives generated pitch text and company_name from Basic LLM Chain  
  - Outputs: Final step in workflow  
  - Credentials: Google Sheets OAuth2 required  
  - Edge Cases: Matching failures if company_name is inconsistent, sheet permission issues.

---

### 3. Summary Table

| Node Name                                | Node Type                      | Functional Role                          | Input Node(s)                                | Output Node(s)                               | Sticky Note                                                                                          |
|-----------------------------------------|--------------------------------|----------------------------------------|----------------------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------|
| Sticky Note9                            | Sticky Note                   | Workflow assistance and contact info   | None                                         | None                                         | Scrape Glassdoor Job Listings For Prospecting with Bright Data and LLMS; contact & docs links       |
| On form submission - Discover Jobs     | Form Trigger                  | Captures user input for job search     | None                                         | HTTP Request- Post API call to Bright Data   |                                                                                                    |
| Sticky Note1                           | Sticky Note                   | Parameter guide for form inputs         | None                                         | None                                         | Explains location, keyword, country input format with examples                                     |
| HTTP Request- Post API call to Bright Data | HTTP Request (POST)           | Initiates Bright Data dataset snapshot | On form submission - Discover Jobs           | Wait - Polling Bright Data                    |                                                                                                    |
| Sticky Note2                           | Sticky Note                   | Explains Bright Data trigger node       | None                                         | None                                         | Explains customization of Bright Data job query                                                   |
| Wait - Polling Bright Data             | Wait                         | Waits before polling snapshot progress | HTTP Request- Post API call to Bright Data   | Snapshot Progress                             |                                                                                                    |
| Snapshot Progress                      | HTTP Request (GET)            | Checks snapshot progress status         | Wait - Polling Bright Data                    | If - Checking status of Snapshot              |                                                                                                    |
| If - Checking status of Snapshot       | If                           | Decides to wait or fetch data           | Snapshot Progress                             | Wait - Polling Bright Data, HTTP Request - Getting data from Bright Data |                                                                                                    |
| HTTP Request - Getting data from Bright Data | HTTP Request (GET)            | Retrieves full job listings dataset     | If - Checking status of Snapshot              | Google Sheets - Adding All Job Posts          |                                                                                                    |
| Sticky Note3                           | Sticky Note                   | Notes about Bright Data job retrieval   | None                                         | None                                         | Bright Data Getting Jobs                                                                           |
| Google Sheets - Adding All Job Posts   | Google Sheets                 | Appends job listings to Google Sheet    | HTTP Request - Getting data from Bright Data | Split Out                                     | Instructions to use Google Sheets template (Sticky Note10)                                        |
| Sticky Note10                          | Sticky Note                   | Google Sheets template instructions     | None                                         | None                                         | Provides link and instructions for Google Sheets template                                        |
| Split Out                             | Split Out                    | Splits job data into components          | Google Sheets - Adding All Job Posts          | Basic LLM Chain                               |                                                                                                    |
| Basic LLM Chain                      | LangChain LLM Chain          | Defines AI prompt and logic for pitches | Split Out                                     | Google Sheets - Update Pitches                 |                                                                                                    |
| OpenAI Chat Model                    | LangChain OpenAI Chat Model | Executes GPT model call                   | Basic LLM Chain                               | Basic LLM Chain (ai_languageModel input)      |                                                                                                    |
| Google Sheets - Update Pitches         | Google Sheets                 | Updates Google Sheet with AI pitches     | Basic LLM Chain                               | None                                         |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form fields:  
     - Job Location (required)  
     - Keyword (required)  
     - Country (2 letters, required)  
   - Set form title and description accordingly.  
   - Save and note webhook URL for external triggers.

2. **Add HTTP Request Node to Trigger Bright Data Snapshot**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.brightdata.com/datasets/v3/trigger`  
   - Method: POST  
   - Headers: Add Authorization header with `Bearer <YOUR_BRIGHT_DATA_API_KEY>`  
   - Query Parameters:  
     - dataset_id: your Bright Data dataset ID (e.g., `gd_lpfbbndm1xnopbrcr0`)  
     - include_errors: true  
     - type: discover_new  
     - discover_by: keyword  
     - uncompressed_webhook: true  
   - Body (JSON):  
     ```json
     [
       {
         "location": "{{ $json['Job Location'] }}",
         "keyword": "{{ $json.Keyword }}",
         "country": "{{ $json['Country (2 letters)'] }}"
       }
     ]
     ```  
   - Connect Form Trigger output to this node.

3. **Add Wait Node for Polling**  
   - Type: Wait  
   - Unit: Minutes  
   - Connect HTTP Request node output to Wait node.

4. **Add HTTP Request Node to Check Snapshot Progress**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Headers: Authorization with Bearer token  
   - Connect Wait node output to this node.

5. **Add If Node to Check Snapshot Status**  
   - Type: If  
   - Condition: Check if `status` field equals `"running"`  
   - True branch: Connect back to Wait node (to poll again)  
   - False branch: Proceed to next step.

6. **Add HTTP Request Node to Retrieve Snapshot Data**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Query Parameter: format=json  
   - Headers: Authorization with Bearer token  
   - Connect False branch of If node to this node.

7. **Add Google Sheets Node to Append Job Posts**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Use your copied Google Sheets template ID  
   - Sheet Name: Default tab (gid=0)  
   - Map columns from Bright Data JSON fields (company_name, job_title, job_overview, etc.)  
   - Connect HTTP Request snapshot data node output to this node.  
   - Configure Google Sheets OAuth2 credentials.

8. **Add Split Out Node**  
   - Type: Split Out  
   - Field to split: `company_name, job_title, description_text` (map description_text from job_overview or equivalent)  
   - Connect Google Sheets append node output to this node.

9. **Add LangChain LLM Chain Node**  
   - Type: LangChain LLM Chain  
   - Define prompt with variables for company_name, job_title, and job_overview.  
   - Logic: If job relates to marketing/content/audience engagement, generate icebreaker sentences referencing company/job and pitch content repurposing service; else reply "---JOB POST NOT RELEVANT---".  
   - Connect Split Out node output to this node.

10. **Add OpenAI Chat Model Node**  
    - Type: LangChain OpenAI Chat Model  
    - Model: gpt-4o-mini (or preferred GPT model)  
    - Connect LangChain LLM Chain node to this node’s language model input.  
    - Configure OpenAI API credentials.

11. **Connect OpenAI Chat Model output back to LangChain LLM Chain node**  
    - This is required for LangChain node to receive model output.

12. **Add Google Sheets Node to Update Pitches**  
    - Type: Google Sheets  
    - Operation: Update  
    - Document ID and Sheet Name: same as append node  
    - Matching Column: company_name  
    - Columns to update: Pitch (with generated text), company_name (for matching)  
    - Connect LangChain LLM Chain output to this node.  
    - Configure Google Sheets OAuth2 credentials.

13. **Add Sticky Notes** (optional but recommended)  
    - Add notes for user guidance, parameter explanations, API key reminders, and template links.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow assistance and contact info: For questions or support, email Yaron@nofluff.online               | Email: mailto:Yaron@nofluff.online                                                              |
| YouTube tutorials and tips by Yaron Been                                                                 | https://www.youtube.com/@YaronBeen/videos                                                       |
| LinkedIn profile for professional networking and updates                                                  | https://www.linkedin.com/in/yaronbeen/                                                          |
| Bright Data API documentation for dataset snapshots and usage                                            | https://docs.brightdata.com/introduction                                                        |
| Google Sheets template for storing job listings and AI-generated pitches                                  | https://docs.google.com/spreadsheets/d/1ZYRk83hNIQCyQNaKpchdnbTiapVxE4aG6ZFIQlwEoWM/edit?usp=sharing |

---

This document provides a detailed and structured reference for understanding, reproducing, and modifying the "Find High-Intent Sales Leads by Scraping Glassdoor with Bright Data & GPT" workflow in n8n. It covers all nodes, their configurations, and integration points, enabling advanced users and AI agents to work effectively with the workflow.