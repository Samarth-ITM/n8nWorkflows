Personalized B2B Email Outreach with Apollo, LinkedIn, GPT-4.1 and SendGrid

https://n8nworkflows.xyz/workflows/personalized-b2b-email-outreach-with-apollo--linkedin--gpt-4-1-and-sendgrid-6039


# Personalized B2B Email Outreach with Apollo, LinkedIn, GPT-4.1 and SendGrid

---

### 1. Workflow Overview

This workflow, titled **Personalized B2B Email Outreach with Apollo, LinkedIn, GPT-4.1 and SendGrid**, automates the process of generating hyper-personalized outreach emails for B2B sales or marketing campaigns. It integrates data scraping from Apollo.io and LinkedIn, enriches lead profiles with AI analysis via GPT-4.1, assesses lead relevance, and sends customized emails through SendGrid. The workflow is ideal for sales teams and marketing professionals targeting executives or decision-makers with personalized messages based on detailed profile and social media content.

The logic is organized into these main functional blocks:

- **1.1 Lead Collection & Data Ingestion:** Scrape leads from Apollo API and enrich profile data from LinkedIn APIs.
- **1.2 Data Processing & Validation:** Extract key fields, clean data, check for missing values, and aggregate information.
- **1.3 AI-Powered Personalization:** Use GPT-4.1 models with LangChain nodes to analyze pain points, user profiling, and generate personalized email content.
- **1.4 Lead Qualification:** Evaluate lead relevance based on AI outputs and update lead status in Supabase database.
- **1.5 Email Composition & Sending:** Format the personalized email HTML and send it via SendGrid.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Collection & Data Ingestion

**Overview:**  
This block collects raw lead data from Apollo.io and LinkedIn, parsing relevant URLs and profile information to prepare for personalization.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Apollo Scraper  
- Required Data  
- Supabase (for saving raw leads)  
- When chat message received (chat trigger webhook)  
- LinkedIn Data  
- Code (extract LinkedIn handle)  
- LinkedIn Posts  
- Summery+First Name  
- Last 5 Posts  
- Merge1  
- Aggregate1

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual test runs  
  - Inputs: None  
  - Outputs: Apollo Scraper  
  - Edge cases: None, manual start only

- **Apollo Scraper**  
  - Type: HTTP Request  
  - Role: Calls Apify actor for Apollo.io data scraping with parameters to get personal and work emails, limited to 500 records  
  - Config: POST request with JSON body specifying scraping options; requires valid APIFY token  
  - Inputs: Trigger  
  - Outputs: Required Data  
  - Failures: API key invalid, rate limits, network errors

- **Required Data**  
  - Type: Set  
  - Role: Extracts and sets `linkedin_url` and `email` from incoming JSON for downstream use  
  - Inputs: Apollo Scraper  
  - Outputs: Supabase  
  - Edge cases: Missing or malformed URLs/emails

- **Supabase**  
  - Type: Supabase database node  
  - Role: Inserts or updates lead data with LinkedIn URL and email into the "Data" table for persistence  
  - Inputs: Required Data  
  - Outputs: None (end of this ingestion chain)  
  - Edge cases: DB connection errors, schema mismatches

- **When chat message received**  
  - Type: LangChain Chat Trigger webhook  
  - Role: Listens for external chat messages (likely for interactive or triggered runs)  
  - Outputs: LinkedIn Data and Code nodes  
  - Edge cases: Webhook availability, auth

- **LinkedIn Data**  
  - Type: HTTP Request  
  - Role: Calls an API (RapidAPI li-data-scraper) to get detailed LinkedIn profile data by URL  
  - Config: Requires `x-rapidapi-host` and `x-rapidapi-key` headers  
  - Inputs: Chat trigger  
  - Outputs: Summery+First Name  
  - Failures: API limits, invalid keys, profile not found

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Parses LinkedIn profile URL to extract username/handle by stripping URL prefixes  
  - Inputs: Chat trigger  
  - Outputs: LinkedIn Posts  
  - Edge cases: Unexpected URL formats

- **LinkedIn Posts**  
  - Type: HTTP Request  
  - Role: Fetches latest LinkedIn posts for the extracted username via RapidAPI  
  - Inputs: Code node (handle)  
  - Outputs: Last 5 Posts  
  - Failures: API limits, invalid username

- **Summery+First Name**  
  - Type: Set  
  - Role: Extracts and sets `firstName`, `summary`, and first position's company name and title for use in personalization  
  - Inputs: LinkedIn Data  
  - Outputs: Merge1  
  - Edge cases: Missing profile fields

- **Last 5 Posts**  
  - Type: Set  
  - Role: Extracts text from the first five LinkedIn posts for use in AI personalization  
  - Inputs: LinkedIn Posts  
  - Outputs: Merge1

- **Merge1**  
  - Type: Merge  
  - Role: Combines Summery+First Name and Last 5 Posts data streams for aggregation  
  - Inputs: Summery+First Name, Last 5 Posts  
  - Outputs: Aggregate1

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates all merged items into a single data structure  
  - Inputs: Merge1  
  - Outputs: Check Null Values

---

#### 1.2 Data Processing & Validation

**Overview:**  
Validates collected data for completeness and initiates user profiling if data quality is sufficient.

**Nodes Involved:**  
- Check Null Values  
- User Profiling according to Product  
- OpenAI Chat Model  
- Structured Output Parser1  
- Name  
- History  
- Merge2  
- Aggregate2  
- Edit Fields1  
- Story  
- Structured Output Parser  
- OpenAI Chat Model2  
- Merge  
- Aggregate

**Node Details:**

- **Check Null Values**  
  - Type: If  
  - Role: Checks if key fields (firstName, summary, LinkedIn posts data) exist and are not null  
  - Inputs: Aggregate1  
  - Outputs:  
    - True: Proceeds to User Profiling and sets Name and History nodes  
    - False: Ends workflow with All Done node  
  - Edge cases: Missing data leads to termination

- **User Profiling according to Product**  
  - Type: LangChain Agent  
  - Role: Uses AI to analyze aggregated data and generate user profile insights relevant to the product  
  - Inputs: Check Null Values (true branch)  
  - Outputs: Relevance Check  
  - Config: Uses GPT-4.1, with output parser Structured Output Parser1  
  - Edge cases: AI timeout, token limits, parsing errors

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: AI language model node supporting User Profiling agent  
  - Inputs: User Profiling according to Product  
  - Outputs: User Profiling

- **Structured Output Parser1**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI output into structured JSON with fields "Yes", "No", and "Pain Point + Solution"  
  - Inputs: User Profiling according to Product

- **Name**  
  - Type: Set  
  - Role: Extracts firstName from data for further processing  
  - Inputs: Check Null Values (true branch)

- **History**  
  - Type: Set  
  - Role: Extracts full position history from LinkedIn data for context  
  - Inputs: Check Null Values (true branch)

- **Merge2**  
  - Type: Merge  
  - Role: Combines output from User Profiling and History for aggregation  
  - Inputs: User Profiling according to Product and History

- **Aggregate2**  
  - Type: Aggregate  
  - Role: Aggregates merged data into a single item for editing fields  
  - Inputs: Merge2

- **Edit Fields1**  
  - Type: Set  
  - Role: Extracts "Pain Point + Solution" and full position data for input into Story node  
  - Inputs: Aggregate2  
  - Outputs: Story

- **Story**  
  - Type: LangChain Agent  
  - Role: Generates personalized outreach content based on pain points and user history using GPT-4.1  
  - Inputs: Edit Fields1  
  - Outputs: Merge (for final aggregation)  
  - Config: Has structured output parser Structured Output Parser

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses Story output into structured JSON with "Subject" and "Body" fields

- **OpenAI Chat Model2**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Supports Story node for AI content generation

- **Merge**  
  - Type: Merge  
  - Role: Combines output from Name and Story nodes for final aggregation

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates merged items for HTML email formatting

---

#### 1.3 Lead Qualification

**Overview:**  
Determines whether the lead is relevant based on AI profiling and updates lead status accordingly in the Supabase database.

**Nodes Involved:**  
- Relevance Check  
- Irrelevant Leads  
- Successful Outreach  
- All Done

**Node Details:**

- **Relevance Check**  
  - Type: If  
  - Role: Checks AI output (boolean true/false) indicating if lead is relevant  
  - Inputs: User Profiling output  
  - Outputs:  
    - True: Leads to Merge2 for further processing and eventual sending  
    - False: Leads to Irrelevant Leads node updating lead status

- **Irrelevant Leads**  
  - Type: Supabase  
  - Role: Updates lead status to "False" in "C-Suite" table indicating irrelevance  
  - Inputs: Relevance Check (false branch)  
  - Outputs: All Done

- **Successful Outreach**  
  - Type: Supabase  
  - Role: Updates lead status to "True" after successful email sending  
  - Inputs: SendGrid  
  - Outputs: All Done

- **All Done**  
  - Type: NoOp  
  - Role: Terminal node indicating end of workflow branch

---

#### 1.4 Email Composition & Sending

**Overview:**  
Formats the personalized email content into HTML and sends it via SendGrid to the lead's email address.

**Nodes Involved:**  
- HTML Modifier  
- OpenAI Chat Model1  
- SendGrid

**Node Details:**

- **HTML Modifier**  
  - Type: LangChain Agent  
  - Role: Takes AI-generated email "Body" and lead's first name to create a polished HTML email  
  - Inputs: Aggregate  
  - Outputs: SendGrid  
  - Config: Uses GPT-4.1 for formatting

- **OpenAI Chat Model1**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Supports HTML Modifier node

- **SendGrid**  
  - Type: SendGrid node  
  - Role: Sends the composed email to the lead's email address with subject and HTML body  
  - Inputs: HTML Modifier  
  - Config: Uses SendGrid API credentials  
  - Edge cases: Email sending failures, invalid email addresses

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                        | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                                              |
|---------------------------|------------------------------------|-------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                     | Manual start for scraping            | None                         | Apollo Scraper              |                                                                                                                          |
| Apollo Scraper            | HTTP Request                       | Scrapes Apollo.io data               | When clicking ‘Test workflow’ | Required Data               | See sticky note: "Go to apollo.io, use filters and paste link in HTTP node URL. Pay for APify. Create Supabase table."    |
| Required Data             | Set                               | Extracts linkedin_url and email      | Apollo Scraper               | Supabase                   |                                                                                                                          |
| Supabase                  | Supabase                          | Save lead data                      | Required Data                | None                       |                                                                                                                          |
| When chat message received | LangChain Chat Trigger            | Webhook for chat-triggered runs     | None                         | LinkedIn Data, Code        | See sticky note about setup steps and API payment requirements.                                                          |
| LinkedIn Data             | HTTP Request                      | Fetch LinkedIn profile data          | When chat message received   | Summery+First Name          |                                                                                                                          |
| Code                      | Code                              | Extract LinkedIn username/handle    | When chat message received   | LinkedIn Posts              |                                                                                                                          |
| LinkedIn Posts            | HTTP Request                      | Fetch LinkedIn posts                 | Code                        | Last 5 Posts                |                                                                                                                          |
| Summery+First Name        | Set                               | Extract key profile fields          | LinkedIn Data               | Merge1                     |                                                                                                                          |
| Last 5 Posts              | Set                               | Extract texts from last 5 posts     | LinkedIn Posts              | Merge1                     |                                                                                                                          |
| Merge1                    | Merge                             | Combine profile and post data       | Summery+First Name, Last 5 Posts | Aggregate1               |                                                                                                                          |
| Aggregate1                | Aggregate                        | Aggregate merged data                | Merge1                      | Check Null Values           |                                                                                                                          |
| Check Null Values         | If                                | Validate key data presence          | Aggregate1                  | User Profiling..., Name, History (true); All Done (false) |                                                                                                                          |
| User Profiling according to Product | LangChain Agent               | AI analysis for user profiling      | Check Null Values (true)     | Relevance Check             |                                                                                                                          |
| OpenAI Chat Model         | LangChain LM Chat OpenAI           | AI model for user profiling          | User Profiling according to Product | User Profiling according to Product |                                                                                                                          |
| Structured Output Parser1 | LangChain Output Parser Structured | Parse AI output for profiling       | User Profiling according to Product | User Profiling according to Product |                                                                                                                          |
| Name                      | Set                               | Extract firstName                   | Check Null Values (true)     | Merge                      |                                                                                                                          |
| History                   | Set                               | Extract full position history       | Check Null Values (true)     | Merge2                     |                                                                                                                          |
| Merge2                    | Merge                             | Combine profiling and history       | User Profiling, History      | Aggregate2                  |                                                                                                                          |
| Aggregate2                | Aggregate                        | Aggregate merged data                | Merge2                      | Edit Fields1                |                                                                                                                          |
| Edit Fields1              | Set                               | Prepare fields for story generation | Aggregate2                  | Story                      |                                                                                                                          |
| Story                     | LangChain Agent                   | AI generates personalized content   | Edit Fields1                | Merge                      |                                                                                                                          |
| Structured Output Parser  | LangChain Output Parser Structured | Parse personalized content output   | Story                       | Story                      |                                                                                                                          |
| OpenAI Chat Model2        | LangChain LM Chat OpenAI           | AI model for story generation        | Story                       | Story                      |                                                                                                                          |
| Merge                     | Merge                             | Combine Name and Story outputs       | Name, Story                 | Aggregate                  |                                                                                                                          |
| Aggregate                 | Aggregate                        | Aggregate final data for email       | Merge                       | HTML Modifier              |                                                                                                                          |
| HTML Modifier             | LangChain Agent                   | Format email content to HTML         | Aggregate                   | SendGrid                   |                                                                                                                          |
| OpenAI Chat Model1        | LangChain LM Chat OpenAI           | AI model for HTML formatting         | HTML Modifier               | HTML Modifier              |                                                                                                                          |
| SendGrid                  | SendGrid                         | Send email                           | HTML Modifier               | Successful Outreach        |                                                                                                                          |
| Relevance Check           | If                                | Check lead relevance decision        | User Profiling              | Merge2 (true), Irrelevant Leads (false) |                                                                                                                          |
| Irrelevant Leads          | Supabase                          | Update lead status as irrelevant     | Relevance Check (false)     | All Done                   |                                                                                                                          |
| Successful Outreach       | Supabase                          | Update lead status as successful     | SendGrid                   | All Done                   |                                                                                                                          |
| All Done                  | NoOp                             | Terminal node                       | Irrelevant Leads, Successful Outreach, Check Null Values (false) | None                       |                                                                                                                          |

**Sticky Notes Summary:**

- For Apollo Scraper, Required Data, Supabase nodes:  
  "Go to apollo.io, use filters and after that copy the link from tab and paste that in the http node url section (paste inside the \"\"). Pay for APify to get more data. Create a supabase table beforehand according to your fields and make changes to both scrape and outreach workflow accordingly."

- For the overall workflow setup (applies to When chat message received and other scraping nodes):  
  "Hyper-Personalised Mail Generator  
  1. Add twilio for smtp (manages everything for you).  
  2. Pay for rapid api otherwise the flow will fail after 20-25 scrapes in free tier.  
  3. Pay for supabase if data is too much.  
  4. Ensure that you have enough tokens in your openAI for your outreach volume.  
  5. 7000-12000 total token consumption per run (avg).  
  6. Give the linkedin id link as input."

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Test workflow’  
   - Purpose: Start scraping on demand.

2. **Add HTTP Request Node for Apollo Scraper**  
   - Name: Apollo Scraper  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/code_crafter~apollo-io-scraper/run-sync-get-dataset-items?token=YOUR_APIFY_KEY` (replace with valid token)  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "getPersonalEmails": true,
       "getWorkEmails": true,
       "totalRecords": 500,
       "url": ""
     }
     ```  
   - Connect output of manual trigger to this node.

3. **Add Set Node to Extract Key Fields**  
   - Name: Required Data  
   - Assignments:  
     - `linkedin_url`: `{{$json.linkedin_url}}`  
     - `email`: `{{$json.email}}`  
   - Connect Apollo Scraper output here.

4. **Add Supabase Node to Store Leads**  
   - Name: Supabase  
   - Operation: Insert or update into `Data` table  
   - Fields: `linkedin_url` and `email` from Required Data  
   - Configure Supabase credentials.  
   - Connect Required Data to Supabase.

5. **Add LangChain Chat Trigger Node for Chat-Triggered Runs**  
   - Name: When chat message received  
   - Set webhook ID (auto-generated)  
   - Connect outputs to LinkedIn Data and Code node.

6. **Add HTTP Request Node for LinkedIn Data API**  
   - Name: LinkedIn Data  
   - URL: `https://li-data-scraper.p.rapidapi.com/get-profile-data-by-url?url={{$json.linkedin_url}}`  
   - Add headers:  
     - `x-rapidapi-host`: `li-data-scraper.p.rapidapi.com`  
     - `x-rapidapi-key`: your RapidAPI key  
   - Connect When chat message received to this node.

7. **Add Code Node to Extract LinkedIn Username**  
   - Name: Code  
   - JavaScript:  
     ```js
     const url    = $input.first().json.linkedin_url; 
     const p1     = "http://www.linkedin.com/in/";
     const p2     = "linkedin.com/in/";

     let handle = url;
     if (handle.startsWith(p1)) {
       handle = handle.slice(p1.length);
     } else if (handle.startsWith(p2)) {
       handle = handle.slice(p2.length);
     }
     return [{ json: { handle } }];
     ```  
   - Connect When chat message received to this node.

8. **Add HTTP Request Node for LinkedIn Posts**  
   - Name: LinkedIn Posts  
   - URL: `https://li-data-scraper.p.rapidapi.com/get-profile-posts`  
   - Query Param: `username` = `{{$json.handle}}`  
   - Headers: same as LinkedIn Data  
   - Connect Code output here.

9. **Add Set Node for Summery+First Name**  
   - Name: Summery+First Name  
   - Assign:  
     - `firstName` = `{{$json.firstName}}`  
     - `summary` = `{{$json.summary}}`  
     - `position[0].companyName` = `{{$json.position[0].companyName}}`  
     - `position[0].title` = `{{$json.position[0].title}}`  
   - Connect LinkedIn Data output here.

10. **Add Set Node for Last 5 Posts**  
    - Name: Last 5 Posts  
    - Assign first five post texts from LinkedIn Posts output to `data[0].text` through `data[4].text`  
    - Connect LinkedIn Posts output here.

11. **Add Merge Node to Combine Profile and Posts**  
    - Name: Merge1  
    - Mode: Merge by index (default)  
    - Inputs: Summery+First Name, Last 5 Posts

12. **Add Aggregate Node to Aggregate Merged Data**  
    - Name: Aggregate1  
    - Operation: Aggregate all items  
    - Input: Merge1 output

13. **Add If Node to Check Null Values**  
    - Name: Check Null Values  
    - Conditions: All must exist and not be null:  
      - `data[0].firstName`  
      - `data[0].summary`  
      - `data[1].data`  
    - True branch: proceed; False branch: end with NoOp node

14. **Add LangChain Agent Node for User Profiling**  
    - Name: User Profiling according to Product  
    - Text: `={{ $json.data }}` (aggregated data)  
    - Model: GPT-4.1 with output parser Structured Output Parser1 (defined below)  
    - Connect Check Null Values (true) here.

15. **Add LangChain Output Parser Structured Node**  
    - Name: Structured Output Parser1  
    - JSON Schema Example:  
      ```json
      {
        "Yes": "True",
        "No": "False",
        "Pain Point + Solution": ""
      }
      ```  
    - Connect User Profiling node here.

16. **Add Set Nodes for Name and History**  
    - Name: Name (set firstName)  
    - Name: History (set full position from LinkedIn Data)  
    - Connect Check Null Values (true) to both.

17. **Add Merge Node to Combine User Profiling and History**  
    - Name: Merge2  
    - Inputs: User Profiling output and History

18. **Add Aggregate Node to Aggregate Merge2 Output**  
    - Name: Aggregate2

19. **Add Set Node to Extract Fields for Story Generation**  
    - Name: Edit Fields1  
    - Assign:  
      - "Pain Point + Solution" from User Profiling output  
      - "Fullposition" from History output

20. **Add LangChain Agent Node for Story Generation**  
    - Name: Story  
    - Text: Template combining Pain Point + Solution and Fullposition  
    - Model: GPT-4.1 with Structured Output Parser (defined below)  
    - Connect Edit Fields1 output here.

21. **Add LangChain Output Parser Structured Node**  
    - Name: Structured Output Parser (for Story output)  
    - JSON Schema Example:  
      ```json
      {
        "Subject": "",
        "Body": ""
      }
      ```  
    - Connect to Story node.

22. **Add Merge Node to Combine Name and Story Outputs**  
    - Name: Merge

23. **Add Aggregate Node for Final Merge Output**  
    - Name: Aggregate

24. **Add LangChain Agent Node to Format HTML Email**  
    - Name: HTML Modifier  
    - Text: Use Subject and Body from aggregate data and first name to produce HTML content  
    - Model: GPT-4.1  
    - Connect Aggregate output here.

25. **Add SendGrid Node**  
    - Name: SendGrid  
    - Subject: From aggregated Story output Subject  
    - To Email: From lead email field  
    - Content Type: text/html  
    - Content Value: From HTML Modifier output  
    - Configure SendGrid API credentials  
    - Connect HTML Modifier to SendGrid.

26. **Add Supabase Node to Update Successful Outreach**  
    - Name: Successful Outreach  
    - Table: "C-Suite"  
    - Update status to "True" where email matches  
    - Connect SendGrid output here.

27. **Add If Node for Relevance Check**  
    - Name: Relevance Check  
    - Condition: AI output "Yes" == true  
    - True branch: Connect to Merge2 for Story generation flow  
    - False branch: Connect to Irrelevant Leads

28. **Add Supabase Node for Irrelevant Leads**  
    - Name: Irrelevant Leads  
    - Table: "C-Suite"  
    - Update status to "False"  
    - Connect Relevance Check false branch here.

29. **Add NoOp Node**  
    - Name: All Done  
    - Connect from Irrelevant Leads, Successful Outreach, and Check Null Values (false branch) to end workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                       | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Go to apollo.io, use filters and after that copy the link from tab and paste that in the HTTP node URL section. Pay for APify to get more data. Create a Supabase table beforehand according to your fields and make changes to both scrape and outreach workflow accordingly. | Sticky Note on Apollo Scraper and related nodes                                                     |
| Hyper-Personalised Mail Generator: 1. Add Twilio for SMTP (manages everything for you). 2. Pay for RapidAPI to avoid free tier limits after ~20-25 scrapes. 3. Pay for Supabase if data volume is large. 4. Ensure enough OpenAI tokens for outreach volume (~7000-12000 tokens per run). 5. Provide LinkedIn ID link as input. | Sticky Note on overall workflow setup and prerequisites                                              |
| Apollo.io API key, RapidAPI keys, Supabase credentials, OpenAI API keys, and SendGrid API keys are all required for this workflow to function correctly. | Credential dependencies                                                                              |
| The workflow uses GPT-4.1 model via LangChain nodes for AI-powered personalization and content generation.         | AI integration details                                                                             |
| The workflow is designed to be modular; the chat-triggered run and manual trigger allow flexible operation modes.  | Workflow design note                                                                               |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---