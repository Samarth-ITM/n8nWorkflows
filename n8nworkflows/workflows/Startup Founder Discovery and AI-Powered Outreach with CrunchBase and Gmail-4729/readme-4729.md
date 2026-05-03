Startup Founder Discovery and AI-Powered Outreach with CrunchBase and Gmail

https://n8nworkflows.xyz/workflows/startup-founder-discovery-and-ai-powered-outreach-with-crunchbase-and-gmail-4729


# Startup Founder Discovery and AI-Powered Outreach with CrunchBase and Gmail

### 1. Workflow Overview

This n8n workflow automates the process of discovering startup founders via Crunchbase API and generating AI-powered personalized outreach email summaries sent through Gmail. It is designed for startup scouts, venture capital analysts, and sales or marketing teams who want to enrich founder data and streamline outreach efforts without manual data compilation.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Company Fetching:** Manually triggered retrieval of updated startup companies and their associated people from Crunchbase.
- **1.2 Founder Profile Fetching & Data Extraction:** For each identified person (founder/executive), fetch detailed profile data and extract key relevant fields for outreach.
- **1.3 AI-Powered Summary Generation & Email Outreach:** Use OpenAI GPT-4 to generate concise professional summaries of founders and send these summaries directly via Gmail.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Block: Trigger & Company Fetching

**Overview:**  
This block initiates the workflow manually and fetches a paginated list of people associated with a specific company (or companies) from Crunchbase. It is the starting point for data acquisition.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Updated profiles List (HTTP Request to Crunchbase API)  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Enables manual start of the workflow for testing or on-demand execution.  
  - Configuration: No parameters; triggers the next node when activated.  
  - Inputs: None  
  - Outputs: Connects to "Updated profiles List"  
  - Edge cases: None expected; manual trigger is user-initiated.

- **Updated profiles List**  
  - Type: HTTP Request  
  - Role: Queries Crunchbase API to fetch people related to a target organization (via UUID).  
  - Configuration:  
    - URL: `https://api.crunchbase.com/api/v4/relationships/organizations/{org_uuid}/people`  
    - Query param: `page=1` (paginated list)  
    - Header: `X-Cb-User-Key` with API key for authentication  
  - Key expressions: Static URL with organization UUID; API key must be replaced.  
  - Inputs: From Manual Trigger  
  - Outputs: JSON array of people with UUIDs and basic info, passed to next node.  
  - Edge cases:  
    - API key invalid or expired → auth error  
    - Network timeouts  
    - Pagination beyond available pages → empty results  
    - Rate limits by Crunchbase API

---

#### 2.2 Block: Founder Profile Fetching & Data Extraction

**Overview:**  
For each person fetched previously, this block retrieves detailed profile data and extracts the fields relevant for outreach emails.

**Nodes Involved:**  
- Founder Profiles by UUID (HTTP Request)  
- Extract Key Profile Fields (Set node)  

**Node Details:**

- **Founder Profiles by UUID**  
  - Type: HTTP Request  
  - Role: Fetches detailed Crunchbase profile of a single person using their UUID.  
  - Configuration:  
    - URL dynamically constructed: `https://api.crunchbase.com/api/v4/entities/people/{{ $json.data.items[0].uuid }}`  
    - Header: `X-Cb-User-Key` with API key  
  - Key expressions: Uses first person UUID from previous node's JSON array `[0]`; can be customized for other indices.  
  - Inputs: From "Updated profiles List"  
  - Outputs: Detailed person profile JSON  
  - Edge cases:  
    - Invalid UUID → 404 or empty response  
    - API auth errors  
    - Rate limiting  
    - Empty or missing fields in profile data

- **Extract Key Profile Fields**  
  - Type: Set  
  - Role: Maps raw JSON profile data to a simplified format with only key outreach fields.  
  - Configuration: Extracts fields:  
    - Full name: `data.properties.full_name`  
    - Title: `data.properties.title`  
    - Biography: `data.properties.biography`  
    - Education: `data.properties.education`  
    - Social Links: `data.properties.social_links`  
    - Associated companies: `data.properties.associated_companies`  
  - Inputs: Detailed profile JSON  
  - Outputs: Simplified JSON object with selected fields  
  - Edge cases: Missing or malformed fields could yield empty values

---

#### 2.3 Block: AI-Powered Summary Generation & Email Outreach

**Overview:**  
This block uses OpenAI's GPT model with a structured output parser to generate a concise, professional founder summary, then sends this summary via Gmail.

**Nodes Involved:**  
- OpenAI Chat Model (GPT-4 Mini)  
- Structured Output Parser (LangChain output parser)  
- Summarizer Agent (LangChain agent)  
- Send email for outreach (Gmail node)  

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Runs GPT-4o-mini model to generate text from prompt.  
  - Configuration: Model set to `"gpt-4o-mini"` with default options.  
  - Credentials: OpenAI API key configured.  
  - Inputs: Profile summary prompt from "Extract Key Profile Fields" (via Summarizer Agent)  
  - Outputs: Raw GPT completion text  
  - Edge cases:  
    - API key invalid or usage limits exceeded  
    - Network timeouts  
    - Unexpected GPT output formats or errors

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses GPT output into structured JSON according to example schema with subject and body fields.  
  - Configuration: Schema example provided to shape output.  
  - Inputs: Raw GPT text from OpenAI Chat Model  
  - Outputs: Structured JSON with email subject and body  
  - Edge cases: Parsing failures if GPT output format deviates

- **Summarizer Agent**  
  - Type: LangChain Agent  
  - Role: Orchestrates prompt construction and processing of extracted profile fields to produce concise summaries.  
  - Configuration:  
    - System message guides output format: concise, professional, including only full name, title, brief bio, education, social links (LinkedIn/Twitter), and key company role.  
    - Input text is a formatted string with extracted profile fields.  
    - Uses structured output parser for response.  
  - Inputs: Extracted profile data from "Extract Key Profile Fields" node  
  - Outputs: Parsed summary JSON  
  - Edge cases: Failure in parsing or model response; incomplete profile data affects summary quality

- **Send email for outreach**  
  - Type: Gmail  
  - Role: Sends email containing the founder summary to designated recipient(s).  
  - Configuration:  
    - Recipient: `shahkar.genai@gmail.com` (can be customized)  
    - Subject: Static or dynamic (e.g., "CrunchBase profile for outreach")  
    - Message body: Derived from AI-generated summary body field  
    - Gmail OAuth2 credentials configured  
  - Inputs: Summary JSON from "Summarizer Agent"  
  - Outputs: Email sent confirmation  
  - Edge cases:  
    - Gmail auth failure  
    - Invalid email addresses  
    - Rate limits on sending emails

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                          | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
|----------------------------|----------------------------------|----------------------------------------|-------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Manual start of workflow                | None                    | Updated profiles List     | 🔁 **SECTION 1: Trigger + Company Fetching**<br>Manual trigger for on-demand runs, ideal for testing or manual data fetch.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Updated profiles List       | HTTP Request                     | Fetch people list from Crunchbase API  | When clicking ‘Test workflow’ | Founder Profiles by UUID | 🔁 **SECTION 1: Trigger + Company Fetching**<br>Fetches updated companies and associated people from Crunchbase. Page and date filters customizable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Founder Profiles by UUID    | HTTP Request                     | Fetch detailed founder profile         | Updated profiles List    | Extract Key Profile Fields | 👤 **SECTION 2: Founder Profile Fetching + Field Mapping**<br>Fetches detailed profile by UUID; can index different founders by changing array index.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Extract Key Profile Fields  | Set                             | Extract relevant fields for outreach   | Founder Profiles by UUID | Summarizer Agent          | 👤 **SECTION 2: Founder Profile Fetching + Field Mapping**<br>Extracts full name, title, bio, education, social links, companies for clean input to AI.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| OpenAI Chat Model           | LangChain LM Chat OpenAI         | Runs GPT model to generate text        | Summarizer Agent (indirect via agent) | Structured Output Parser | 🤖📨 **SECTION 3: AI-Powered Summary + Email Outreach**<br>Generates professional concise summaries for outreach using GPT-4o-mini.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Structured Output Parser    | LangChain Output Parser Structured | Parses GPT output into structured JSON | OpenAI Chat Model       | Summarizer Agent          | 🤖📨 **SECTION 3: AI-Powered Summary + Email Outreach**<br>Parses AI response to structured email subject and body JSON.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Summarizer Agent            | LangChain Agent                 | Orchestrates prompt and summary output| Extract Key Profile Fields, Structured Output Parser | Send email for outreach | 🤖📨 **SECTION 3: AI-Powered Summary + Email Outreach**<br>Combines profile data into prompt, manages AI model call and parsing to produce email-ready summary.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Send email for outreach     | Gmail                           | Sends outreach email with summary      | Summarizer Agent        | None                      | 🤖📨 **SECTION 3: AI-Powered Summary + Email Outreach**<br>Sends email to configured recipient(s). Subject and recipient customizable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| Sticky Note                 | Sticky Note                     | Documentation and notes                 | None                    | None                      | See sticky note content below                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Sticky Note1                | Sticky Note                     | Documentation and notes                 | None                    | None                      | See sticky note content below                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Sticky Note2                | Sticky Note                     | Documentation and notes                 | None                    | None                      | See sticky note content below                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Sticky Note4                | Sticky Note                     | Documentation overview                  | None                    | None                      | See sticky note content below                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| Sticky Note9                | Sticky Note                     | Workflow assistance contact info       | None                    | None                      | For questions/support contact: Yaron@nofluff.online<br>YouTube: https://www.youtube.com/@YaronBeen/videos<br>LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "When clicking ‘Test workflow’" node**  
   - Type: Manual Trigger  
   - No special config needed.  

2. **Create "Updated profiles List" node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.crunchbase.com/api/v4/relationships/organizations/{org_uuid}/people` (replace `{org_uuid}` with target company UUID)  
   - Query parameters: `page=1` (adjustable)  
   - Header parameters: `X-Cb-User-Key` set to your Crunchbase API key credential  
   - Connect input from Manual Trigger  

3. **Create "Founder Profiles by UUID" node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.crunchbase.com/api/v4/entities/people/{{ $json.data.items[0].uuid }}` (dynamic using first UUID from previous node)  
   - Header: `X-Cb-User-Key` with same API key  
   - Connect input from "Updated profiles List"  

4. **Create "Extract Key Profile Fields" node**  
   - Type: Set node  
   - Add fields with expressions:  
     - Full name = `{{$json.data.properties.full_name}}`  
     - Title = `{{$json.data.properties.title}}`  
     - Biography = `{{$json.data.properties.biography}}`  
     - Education = `{{$json.data.properties.education}}`  
     - Social Links = `{{$json.data.properties.social_links}}`  
     - Associated companies = `{{$json.data.properties.associated_companies}}`  
   - Connect input from "Founder Profiles by UUID"  

5. **Create "Summarizer Agent" node**  
   - Type: LangChain Agent  
   - Configure system message:  
     "You are an expert at writing concise, professional summaries for business email outreach. Given detailed information about a person’s profile, generate a short summary focusing only on relevant outreach info: full name, current title, brief bio (1-2 sentences), education highlights, LinkedIn or Twitter social links, and most relevant company role. Do NOT add unnecessary details."  
   - Input prompt:  
     ```
     Full name: {{ $json['Full name'] }}
     Title: {{ $json.Title }}
     biography: {{ $json.biography }}
     Education: {{ $json.Education }}
     Social Links: {{ $json['Social Links'] }}
     Associated companies: {{ $json['Associated companies'] }}
     ```  
   - Enable structured output parser with JSON schema example specifying fields `subject` and `body`.  
   - Connect input from "Extract Key Profile Fields"  

6. **Create "OpenAI Chat Model" node**  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API key  
   - Connect input from "Summarizer Agent" (agent passes prompt here)  

7. **Create "Structured Output Parser" node**  
   - Type: LangChain Output Parser Structured  
   - Provide JSON schema example for parsing GPT output into `subject` and `body` fields  
   - Connect input from "OpenAI Chat Model"  

8. **Link "Structured Output Parser" output back to "Summarizer Agent"**  
   - This completes the agent's input-output loop for parsing  

9. **Create "Send email for outreach" node**  
   - Type: Gmail  
   - Credentials: Gmail OAuth2 account  
   - To: Set recipient email (e.g., `shahkar.genai@gmail.com`)  
   - Subject: Use expression `={{ $json.subject }}` or static string  
   - Message: Use expression `={{ $json.body }}` to include AI-generated summary  
   - Connect input from "Summarizer Agent"  

10. **Connect workflow nodes in order:**  
    Manual Trigger → Updated profiles List → Founder Profiles by UUID → Extract Key Profile Fields → Summarizer Agent → OpenAI Chat Model → Structured Output Parser → Summarizer Agent → Send email for outreach  

11. **Credential Setup:**  
    - Crunchbase API key saved as credential or used directly in headers  
    - OpenAI API key linked to LangChain OpenAI node  
    - Gmail OAuth2 credentials configured for sending emails  

12. **Customization:**  
    - Adjust `page` query parameter in "Updated profiles List" to fetch different data pages  
    - Change UUID or array index in "Founder Profiles by UUID" to fetch different founders  
    - Modify recipient email or subject in Gmail node as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| For any questions or support, contact Yaron at Yaron@nofluff.online                                             | Workflow assistance and support                                                                                      |
| Explore more tips and tutorials by Yaron: YouTube channel https://www.youtube.com/@YaronBeen/videos             | Video tutorials and workflow tips                                                                                     |
| Connect with Yaron on LinkedIn: https://www.linkedin.com/in/yaronbeen/                                           | Professional network and updates                                                                                      |
| This workflow is ideal for startup scouts, cold outreach campaigns, VC analysts, and CRM enrichment             | Use case guidance                                                                                                     |
| Customize pagination and founder indexing to tailor data fetching                                              | Practical customization notes                                                                                         |
| Ensure API keys for Crunchbase and OpenAI are valid and have sufficient quota                                   | Credential and quota management                                                                                        |
| Gmail node requires OAuth2 credentials with mail-sending permissions                                            | Credential setup reminder                                                                                            |
| Structured Output Parser expects GPT output to adhere strictly to JSON schema; inconsistent output may cause failures | Parsing robustness note                                                                                                |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. All data processed is legal and public. No illegal, offensive, or protected content is included.