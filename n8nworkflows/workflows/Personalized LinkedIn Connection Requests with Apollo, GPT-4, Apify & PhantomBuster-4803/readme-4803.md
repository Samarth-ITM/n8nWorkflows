Personalized LinkedIn Connection Requests with Apollo, GPT-4, Apify & PhantomBuster

https://n8nworkflows.xyz/workflows/personalized-linkedin-connection-requests-with-apollo--gpt-4--apify---phantombuster-4803


# Personalized LinkedIn Connection Requests with Apollo, GPT-4, Apify & PhantomBuster

### 1. Workflow Overview

This n8n workflow automates personalized LinkedIn connection requests using a multi-step process integrating Apollo for lead discovery, OpenAI’s GPT-4 for AI-powered personalization, Apify for web scraping, Google Sheets for data storage, and PhantomBuster for executing connection campaigns. It targets sales and outreach professionals who want to efficiently identify, personalize messages for, and engage with ideal LinkedIn prospects.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception and Audience Description:** Collects user input describing the target audience in plain English via a secure web form.
- **1.2 Lead Discovery and Data Extraction:** Uses AI to convert the description into an Apollo search URL and runs an Apify actor to scrape detailed lead data.
- **1.3 AI-Powered Personalization:** Generates concise, authentic icebreaker messages personalized to each lead’s LinkedIn profile data.
- **1.4 Outreach Execution and Tracking:** Stores lead data with personalized messages in Google Sheets, aggregates leads, and triggers PhantomBuster to send LinkedIn connection requests in batches.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Audience Description

**Overview:**  
This block collects a natural language description of the ideal LinkedIn prospect audience from the user via a secure form, preparing for AI processing.

**Nodes Involved:**  
- On form submission (Form Trigger)  
- Sticky Note 1 (Step 1 explanation)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; receives audience description from the user through a form with basic authentication.  
  - Configuration: Single required textarea field for free-text description of target audience; basic HTTP auth protects form access.  
  - Inputs: External user input via webhook  
  - Outputs: JSON containing user’s audience description  
  - Edge cases: Form authentication failure, invalid or empty input  
  - Version: 2.2  

- **Sticky Note 1**  
  - Provides context and instructions for this step: converting plain English audience description into Apollo search parameters.

---

#### 2.2 Lead Discovery and Data Extraction

**Overview:**  
This block converts the user’s natural language description into a precise Apollo search URL using GPT-4, then runs an Apify actor to scrape up to 500 leads matching the criteria. It also limits the batch size for testing.

**Nodes Involved:**  
- Generate Search URL (OpenAI GPT-4)  
- Run Apify Actor & Get Results (HTTP Request)  
- Limit (Limit node)  
- Sticky Note 2 (Step 2 explanation)

**Node Details:**

- **Generate Search URL**  
  - Type: OpenAI API node (Langchain OpenAI integration)  
  - Role: Transforms plain English input into a valid Apollo search URL using GPT-4.5-preview model.  
  - Configuration: System prompt defines Apollo URL structure; user input injected dynamically; outputs JSON with key `searchUrl`.  
  - Input: Audience description from form submission  
  - Output: JSON with `searchUrl` property  
  - Edge cases: GPT API timeouts, malformed outputs, invalid URL formats  
  - Credential: OpenAI API key required  

- **Run Apify Actor & Get Results**  
  - Type: HTTP Request  
  - Role: Calls Apify’s actor API to scrape Apollo for lead data based on the generated search URL.  
  - Configuration: POST request with JSON body including `getPersonalEmails`, `getWorkEmails`, `totalRecords` (max 500), and dynamically inserted Apollo URL.  
  - Input: Search URL from previous node  
  - Output: JSON array of lead profiles including LinkedIn URLs, emails, job titles, and employment history  
  - Edge cases: HTTP errors, API key issues, rate limits, empty or incomplete scrape results  
  - Credential: Apify API key required  

- **Limit**  
  - Type: Limit  
  - Role: Restricts the number of leads processed downstream (set to 3 for testing; can be increased for production).  
  - Input: Lead data array from Apify  
  - Output: Limited subset of leads  
  - Edge cases: Misconfiguration leading to zero or too large batch sizes  

- **Sticky Note 2**  
  - Explains the purpose of this block and the three main steps: URL generation, scraping, and batch size limiting. Emphasizes output fields like LinkedIn profiles and emails.

---

#### 2.3 AI-Powered Personalization

**Overview:**  
Generates custom, human-like icebreaker messages for each prospect by analyzing their LinkedIn profile data, paraphrasing details to maintain authenticity and avoid robotic phrasing.

**Nodes Involved:**  
- Personalize Outreach (OpenAI GPT-4)  
- Sticky Note 3 (Step 3 explanation)

**Node Details:**

- **Personalize Outreach**  
  - Type: OpenAI API node (Langchain OpenAI integration)  
  - Role: Creates short, punchy icebreaker messages using GPT-4o model, tailored per lead based on their LinkedIn profile.  
  - Configuration: System prompt instructs to paraphrase LinkedIn data fields and follow a strict message template. Input is a formatted string of profile attributes (name, location, job titles, employment history). Output is JSON with an `icebreaker` property.  
  - Input: Limited lead records from previous block  
  - Output: JSON containing personalized icebreaker message  
  - Edge cases: API rate limits, incomplete profile data causing poor personalization, malformed output JSON  
  - Credential: OpenAI API key required  

- **Sticky Note 3**  
  - Details the personalization logic, emphasizing paraphrasing and brevity to ensure messages feel human-crafted and authentic.

---

#### 2.4 Outreach Execution and Tracking

**Overview:**  
Stores the enriched lead data and personalized icebreakers into a Google Sheet for tracking, aggregates leads into batch format, and triggers PhantomBuster to launch the LinkedIn connection request campaign.

**Nodes Involved:**  
- Add to Google Sheet (Google Sheets node)  
- Aggregate (Aggregate node)  
- Trigger PhantomBuster Agent (HTTP Request)  
- Sticky Note 4 (Step 4 explanation)

**Node Details:**

- **Add to Google Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates lead data with icebreakers into a specified Google Sheet for record-keeping and monitoring outreach progress.  
  - Configuration: Maps fields such as id, name, title, email status, LinkedIn URL, photo URL, and icebreaker message. Uses appendOrUpdate operation keyed by `id`.  
  - Input: Personalized lead data with icebreakers  
  - Output: Confirmation of sheet update  
  - Edge cases: Credential expiration, API quota limits, mapping errors, sheet access issues  
  - Credential: OAuth2 for Google Sheets required  

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all input lead items into a single aggregated item to prepare batch data for PhantomBuster.  
  - Input: Multiple lead data items from Google Sheets node  
  - Output: Single aggregated data object  
  - Edge cases: Empty input array, aggregation failures  

- **Trigger PhantomBuster Agent**  
  - Type: HTTP Request  
  - Role: Sends a POST request to PhantomBuster API to launch a LinkedIn connection campaign agent using the aggregated leads and personalized messages.  
  - Configuration: Includes PhantomBuster agent ID and API key in headers and body.  
  - Input: Aggregated batch data  
  - Output: API response confirming agent launch  
  - Edge cases: API key invalid, rate limits, incorrect agent ID, network errors  
  - Credential: PhantomBuster API key required  

- **Sticky Note 4**  
  - Describes the final outreach steps, including data storage, batching, and campaign triggering. Advises starting with low daily connection volumes to avoid LinkedIn restrictions (~100/week limit).

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                           | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                               |
|-----------------------------|-----------------------------|------------------------------------------|-------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------|
| On form submission          | Form Trigger                | Collect user input audience description  | —                       | Generate Search URL          | Step 1: Natural Language Lead Targeting — form for plain English audience description                    |
| Generate Search URL         | OpenAI API (Langchain)      | Convert description to Apollo search URL | On form submission      | Run Apify Actor & Get Results| Step 2: Apollo Lead Extraction — AI generates Apollo search URL                                          |
| Run Apify Actor & Get Results| HTTP Request               | Scrape Apollo for leads                   | Generate Search URL      | Limit                       | Step 2: Apollo Lead Extraction — scrape leads from Apollo                                                |
| Limit                      | Limit                       | Restrict lead batch size for testing     | Run Apify Actor & Get Results | Personalize Outreach     | Step 2: Apollo Lead Extraction — batch size control                                                      |
| Personalize Outreach        | OpenAI API (Langchain)      | Generate personalized icebreakers        | Limit                   | Add to Google Sheet          | Step 3: AI Personalization Engine — create human-like icebreakers                                        |
| Add to Google Sheet         | Google Sheets               | Store lead data and icebreakers           | Personalize Outreach     | Aggregate                   | Step 4: Outreach Execution — leads and messages stored for tracking                                      |
| Aggregate                  | Aggregate                   | Combine leads into batch                   | Add to Google Sheet      | Trigger PhantomBuster Agent  | Step 4: Outreach Execution — prepare batch for campaign                                                  |
| Trigger PhantomBuster Agent | HTTP Request                | Launch PhantomBuster LinkedIn campaign   | Aggregate                | —                           | Step 4: Outreach Execution — trigger LinkedIn connection requests                                        |
| sticky-note-1               | Sticky Note                 | Explanation of Step 1                      | —                       | —                           | Step 1: Natural Language Lead Targeting                                                                 |
| sticky-note-2               | Sticky Note                 | Explanation of Step 2                      | —                       | —                           | Step 2: Apollo Lead Extraction                                                                           |
| sticky-note-3               | Sticky Note                 | Explanation of Step 3                      | —                       | —                           | Step 3: AI Personalization Engine                                                                        |
| sticky-note-4               | Sticky Note                 | Explanation of Step 4                      | —                       | —                           | Step 4: Outreach Execution                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new n8n workflow and name it**: "Personalized LinkedIn Connection Requests with Apollo, GPT-4, Apify & PhantomBuster".

2. **Add a Form Trigger node** named `On form submission`:  
   - Set `webhookId` to a unique ID or leave auto-generated.  
   - Configure a form with one required textarea field labeled "Describe your audience in plain English."  
   - Enable Basic Authentication and configure credentials (username/password).  
   - This node receives the user’s plain English description of the target audience.

3. **Add an OpenAI node** named `Generate Search URL`:  
   - Select the GPT-4.5-preview model.  
   - Set system prompt to instruct conversion of plain text to Apollo search URL with a strict JSON response containing `searchUrl`.  
   - Use an expression to pass the form field input as user content.  
   - Enable JSON output parsing.  
   - Connect input from `On form submission`.

4. **Add an HTTP Request node** named `Run Apify Actor & Get Results`:  
   - Set method to POST.  
   - URL: `https://api.apify.com/v2/acts/jljBwyyQakqrL1wae/run-sync-get-dataset-items`  
   - Body (JSON):  
     ```json
     {
       "getPersonalEmails": true,
       "getWorkEmails": true,
       "totalRecords": 500,
       "url": "{{ $json.message.content.searchUrl }}"
     }
     ```  
   - Add headers:  
     - `Accept: application/json`  
     - `Authorization: Bearer <your-apify-api-key-here>`  
   - Connect input from `Generate Search URL`.  
   - Configure Apify API key credentials.

5. **Add a Limit node** named `Limit`:  
   - Set max items to 3 for testing (increase in production).  
   - Connect input from `Run Apify Actor & Get Results`.

6. **Add an OpenAI node** named `Personalize Outreach`:  
   - Select GPT-4o model.  
   - System prompt: Instruct to generate a short, punchy icebreaker in JSON format using paraphrased LinkedIn profile data.  
   - Format input as a string concatenating lead’s first name, last name, city, current title, and last three employment records. Use expressions to map data from the `Limit` node.  
   - Enable JSON output parsing.  
   - Connect input from `Limit`.  
   - Configure OpenAI API credentials.

7. **Add a Google Sheets node** named `Add to Google Sheet`:  
   - Operation: Append or update.  
   - Document ID: Your Google Sheet ID (e.g., `12VzSF9drN_kWYsGhNjir9hAGCE0k56N7HkfAV8EuI7Q`).  
   - Sheet Name: `gid=0`.  
   - Map columns: id, first_name, last_name, name, linkedin_url, title, email_status, photo_url, icebreaker (using expressions to extract from input JSON and AI output).  
   - Connect input from `Personalize Outreach`.  
   - Configure Google Sheets OAuth2 credentials.

8. **Add an Aggregate node** named `Aggregate`:  
   - Aggregate all item data into a single object.  
   - Connect input from `Add to Google Sheet`.

9. **Add an HTTP Request node** named `Trigger PhantomBuster Agent`:  
   - Method: POST  
   - URL: `https://api.phantombuster.com/api/v2/agents/launch`  
   - Body parameters: `{ "id": "4108795702201397" }` (PhantomBuster agent ID for LinkedIn connection requests)  
   - Header: `X-Phantombuster-Key: <your-phantombuster-api-key-here>`  
   - Connect input from `Aggregate`.  
   - Configure PhantomBuster API key credentials.

10. **Add sticky notes** at appropriate positions to document each step with provided contents (Step 1 to Step 4).

11. **Connect nodes in the following order**:  
    - `On form submission` → `Generate Search URL` → `Run Apify Actor & Get Results` → `Limit` → `Personalize Outreach` → `Add to Google Sheet` → `Aggregate` → `Trigger PhantomBuster Agent`.

12. **Test the workflow** with a sample input such as:  
    `"Creative agencies in the US with 1-100 employees, targeting CEOs and founders"`.

13. **Adjust Limit node max items for production** to handle batches safely (e.g., 5-10 connections/day).

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow is part of the "N8N Course" tag, useful for learning advanced workflow automation with n8n.      | Tag metadata in workflow JSON.                                                                    |
| Guidance to start outreach campaigns with 5-10 LinkedIn connections per day to avoid LinkedIn restrictions. | Sticky Note 4. LinkedIn limits ~100/week to prevent account restrictions.                          |
| Example Apollo search URL template and explanation for AI prompt provided.                                 | Sticky Note 1 and 2. Useful for modifying AI prompt or expanding lead criteria.                    |
| OpenAI GPT-4 models used: GPT-4.5-preview for URL generation; GPT-4o for message personalization.         | Node configurations; ensure API keys have access to these models.                                 |
| PhantomBuster agent ID `4108795702201397` is specific to LinkedIn connection requests with personalized messages. | HTTP Request node `Trigger PhantomBuster Agent`.                                                  |
| Google Sheets document used for tracking leads and messages, ensure correct sharing and OAuth2 setup.     | Node `Add to Google Sheet`; check permissions for appendOrUpdate operations.                      |

---

*Disclaimer: The text provided originates exclusively from an automated workflow built with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*