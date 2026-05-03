Personalized CrunchBase Lead Outreach with AI-Generated Summaries and Gmail

https://n8nworkflows.xyz/workflows/personalized-crunchbase-lead-outreach-with-ai-generated-summaries-and-gmail-4790


# Personalized CrunchBase Lead Outreach with AI-Generated Summaries and Gmail

### 1. Workflow Overview

This workflow automates personalized outreach by fetching updated founder profiles from CrunchBase, extracting key information, generating concise AI-based summaries, and sending outreach emails via Gmail. It targets users such as startup scouts, sales outreach professionals, VC analysts, and CRM data enrichers who want to streamline founder discovery and engagement without manual data handling. The logic is organized into three primary blocks:

- **1.1 Trigger & Company Fetching:** Manually trigger the workflow to fetch updated companies or startups from CrunchBase using their API.
- **1.2 Founder Profile Fetching & Data Extraction:** For each company, retrieve detailed founder or executive profiles, then extract essential fields for summary generation.
- **1.3 AI Summary Generation & Email Outreach:** Use OpenAI GPT-4 with structured output parsing to create professional outreach summaries, then send these summaries via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Company Fetching

- **Overview:**  
This block initiates the workflow manually and fetches a paginated list of recently updated companies’ people (founders/executives) from CrunchBase via API calls.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Updated profiles List (HTTP Request)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual start of workflow for on-demand data fetching or testing.  
    - Config: No parameters; triggers downstream nodes on manual activation.  
    - Inputs: None  
    - Outputs: Updated profiles List  
    - Edge Cases: Users must manually trigger; no automatic scheduling.  
    - Notes: Ideal for controlled or test runs.

  - **Updated profiles List**  
    - Type: HTTP Request  
    - Role: Fetches a list of people associated with a specific organization UUID from CrunchBase API.  
    - Config:  
      - URL: `https://api.crunchbase.com/api/v4/relationships/organizations/1234abcd-5678-efgh-ijkl-9012mnop3456/people`  
      - Query Parameter: `page=1` (customizable for pagination)  
      - Header: `X-Cb-User-Key` with the CrunchBase API key (must be replaced with a valid key)  
    - Inputs: Triggered by Manual Trigger node  
    - Outputs: Founder Profiles by UUID node  
    - Edge Cases:  
      - API rate-limits or invalid API key errors  
      - Pagination limits require manual adjustment  
      - Network timeouts or CrunchBase API downtime  
    - Customization: Adjust `page` parameter or include date filters (`updated_since`) to refine results.

#### 1.2 Founder Profile Fetching & Data Extraction

- **Overview:**  
For each person fetched in the prior block, this block calls CrunchBase API again to get detailed profile information, then extracts key fields relevant for outreach.

- **Nodes Involved:**  
  - Founder Profiles by UUID (HTTP Request)  
  - Extract Key Profile Fields (Set node)

- **Node Details:**

  - **Founder Profiles by UUID**  
    - Type: HTTP Request  
    - Role: Retrieves detailed person profile data from CrunchBase based on UUID from previous node’s output.  
    - Config:  
      - URL dynamically constructed using expression:  
        `https://api.crunchbase.com/api/v4/entities/people/{{ $json.data.items[0].uuid }}`  
      - Header: `X-Cb-User-Key` with CrunchBase API key  
    - Inputs: Updated profiles List  
    - Outputs: Extract Key Profile Fields  
    - Edge Cases:  
      - Missing UUID or malformed expression failures  
      - API errors or invalid keys  
      - Empty or incomplete profile data  
    - Customization: Can be adjusted to fetch different people by modifying array index in expression.

  - **Extract Key Profile Fields**  
    - Type: Set  
    - Role: Maps and extracts specific fields from raw profile JSON into simpler variables for AI processing.  
    - Config: Assigns following fields:  
      - Full name (`$json.data.properties.full_name`)  
      - Title (`$json.data.properties.title`)  
      - Biography (`$json.data.properties.biography`)  
      - Education (`$json.data.properties.education`)  
      - Social Links (`$json.data.properties.social_links`)  
      - Associated companies (`$json.data.properties.associated_companies`)  
    - Inputs: Founder Profiles by UUID  
    - Outputs: Summarizer Agent  
    - Edge Cases: Missing or null fields may cause empty summaries; ensure non-null checks if customizing.

#### 1.3 AI Summary Generation & Email Outreach

- **Overview:**  
This block creates a concise, personalized summary of each founder’s profile using OpenAI GPT-4 with structured output parsing, then sends the summary as an email using Gmail.

- **Nodes Involved:**  
  - OpenAI Chat Model (Language Model)  
  - Structured Output Parser  
  - Summarizer Agent (Langchain Agent)  
  - Send email for outreach (Gmail)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Processes input text using GPT-4o-mini model to generate AI output for summarization.  
    - Config:  
      - Model: GPT-4o-mini selected for balanced performance.  
      - No additional options set.  
      - Requires OpenAI API credentials.  
    - Inputs: Summarizer Agent (via AI languageModel input)  
    - Outputs: Structured Output Parser  
    - Edge Cases: API quota limits, network errors, or exceeding token limits.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI output into a defined JSON schema with `subject` and `body` fields for email.  
    - Config: Example JSON schema provided to guide output format, ensuring consistent structure for downstream use.  
    - Inputs: OpenAI Chat Model (AI output)  
    - Outputs: Summarizer Agent (ai_outputParser input)  
    - Edge Cases: AI output not matching schema causing parse failures.

  - **Summarizer Agent**  
    - Type: Langchain Agent  
    - Role: Orchestrates summarization using extracted profile fields and AI model; produces final formatted summary text.  
    - Config:  
      - Text input composed from extracted fields: Full name, Title, Biography, Education, Social Links, Associated companies.  
      - System message instructs the AI to generate concise, professional summaries focused on outreach relevance.  
      - Output parser enabled to enforce structured output.  
    - Inputs: Extract Key Profile Fields (main input), Structured Output Parser (ai_outputParser), OpenAI Chat Model (ai_languageModel)  
    - Outputs: Send email for outreach  
    - Edge Cases: Incomplete input fields may affect summary quality; parsing or AI model errors.

  - **Send email for outreach**  
    - Type: Gmail node  
    - Role: Sends the generated summary as an email to a specified recipient.  
    - Config:  
      - Recipient: `shahkar.genai@gmail.com` (customizable)  
      - Subject line: Fixed as "CrunchBase profile for outreach" (modifiable)  
      - Message body: Uses AI-generated summary from previous node.  
      - OAuth2 credentials for Gmail required.  
    - Inputs: Summarizer Agent  
    - Outputs: None (end node)  
    - Edge Cases: Authentication failures, email sending limits, invalid recipient address.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                            | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                                     |
|---------------------------|-----------------------------------------|--------------------------------------------|-------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                         | Manual initiation of workflow               | None                    | Updated profiles List     | 🔘 Manual Trigger allows on-demand execution.                                                                                   |
| Updated profiles List      | HTTP Request                            | Fetch updated people list from CrunchBase  | When clicking ‘Test workflow’ | Founder Profiles by UUID  | 🌐 Fetch updated companies; customizable page and date filters.                                                                |
| Founder Profiles by UUID   | HTTP Request                            | Fetch detailed founder profile by UUID     | Updated profiles List    | Extract Key Profile Fields | 🌐 Fetch detailed profile; person index customizable.                                                                           |
| Extract Key Profile Fields | Set                                    | Extract and map key profile fields          | Founder Profiles by UUID | Summarizer Agent         | ✏️ Extracts Full name, Title, Biography, Education, Social Links, Associated companies.                                         |
| OpenAI Chat Model          | Langchain OpenAI Chat Model             | AI model for generating summary             | Summarizer Agent        | Structured Output Parser | 🤖 Uses GPT-4o-mini for balanced performance.                                                                                   |
| Structured Output Parser   | Langchain Structured Output Parser      | Parses AI output into structured JSON       | OpenAI Chat Model       | Summarizer Agent         | Parses output into subject and body for email.                                                                                  |
| Summarizer Agent          | Langchain Agent                        | Combines profile data and AI model for summary | Extract Key Profile Fields, Structured Output Parser, OpenAI Chat Model | Send email for outreach | Generates professional outreach summary focused on relevant info.                                                              |
| Send email for outreach    | Gmail                                  | Sends the AI-generated summary email        | Summarizer Agent        | None                     | 📧 Sends email to configured recipient; OAuth2 Gmail credentials required.                                                     |
| Sticky Note                | Sticky Note                            | Provides section explanations and tips      | None                    | None                     | Multiple sticky notes cover sections: Trigger + Fetch, Profile Extraction, AI Summary + Email outreach, and support info.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start for running the workflow on-demand.

2. **Add HTTP Request Node to Fetch Updated Profiles:**  
   - Name: "Updated profiles List"  
   - Type: HTTP Request  
   - URL: `https://api.crunchbase.com/api/v4/relationships/organizations/1234abcd-5678-efgh-ijkl-9012mnop3456/people`  
   - Query Parameter: `page=1` (adjustable for pagination)  
   - Header: `X-Cb-User-Key` with your CrunchBase API key (create credential)  
   - Connect "When clicking ‘Test workflow’" → "Updated profiles List".

3. **Add HTTP Request Node to Fetch Founder Profiles by UUID:**  
   - Name: "Founder Profiles by UUID"  
   - Type: HTTP Request  
   - URL: Expression:  
     ```
     =https://api.crunchbase.com/api/v4/entities/people/{{ $json.data.items[0].uuid }}
     ```  
   - Header: `X-Cb-User-Key` with same CrunchBase API key credential  
   - Connect "Updated profiles List" → "Founder Profiles by UUID".

4. **Add Set Node to Extract Key Profile Fields:**  
   - Name: "Extract Key Profile Fields"  
   - Type: Set  
   - Assign the following fields from incoming JSON:  
     - Full name: `={{ $json.data.properties.full_name }}`  
     - Title: `={{ $json.data.properties.title }}`  
     - Biography: `={{ $json.data.properties.biography }}`  
     - Education: `={{ $json.data.properties.education }}`  
     - Social Links: `={{ $json.data.properties.social_links }}`  
     - Associated companies: `={{ $json.data.properties.associated_companies }}`  
   - Connect "Founder Profiles by UUID" → "Extract Key Profile Fields".

5. **Add Langchain Agent Node for Summary Generation:**  
   - Name: "Summarizer Agent"  
   - Type: Langchain Agent  
   - Text Input: Template using extracted fields:  
     ```
     Full name: {{ $json['Full name'] }}
     Title: {{ $json.Title }}
     biography: {{ $json.biography }}
     Education: {{ $json.Education }}
     Social Links: {{ $json['Social Links'] }}
     Associated companies: {{ $json['Associated companies'] }}
     ```  
   - System Message:  
     ```
     You are an expert at writing concise, professional summaries for business email outreach. Given detailed information about a person’s professional profile, your task is to generate a short summary focusing only on the most relevant information for outreach emails.

     Only include:
     - Full name
     - Current title/role
     - Brief, relevant part of the biography (max 1-2 sentences)
     - Key education highlights (school and degree)
     - Social links (only LinkedIn or Twitter if available)
     - Most relevant associated company (name and role)

     Do NOT include unnecessary details. Make the summary clear, engaging, and suitable for a personalized outreach email introduction.
     ```  
   - Enable output parser (structured)  
   - Connect "Extract Key Profile Fields" → "Summarizer Agent".

6. **Add OpenAI Chat Model Node:**  
   - Name: "OpenAI Chat Model"  
   - Type: Langchain OpenAI Chat Model  
   - Model: Select `gpt-4o-mini` or similar GPT-4 variant  
   - Credentials: Link your OpenAI API credentials  
   - Connect "Summarizer Agent" (ai_languageModel input) → "OpenAI Chat Model".

7. **Add Structured Output Parser Node:**  
   - Name: "Structured Output Parser"  
   - Type: Langchain Structured Output Parser  
   - JSON Schema Example: Use schema to extract "subject" and "body" fields for email content, e.g.:  
     ```json
     {
       "subject": "🚀 Startup Founder Highlights for Outreach - June 6, 2025",
       "body": "Summary text here..."
     }
     ```  
   - Connect "OpenAI Chat Model" (AI output) → "Structured Output Parser".

8. **Connect Structured Output Parser back to Summarizer Agent:**  
   - Connect "Structured Output Parser" (ai_outputParser output) → "Summarizer Agent".

9. **Add Gmail Node to Send Email:**  
   - Name: "Send email for outreach"  
   - Type: Gmail  
   - Credentials: Connect your Gmail OAuth2 credential  
   - Recipient (Send To): Set to your target outreach email, e.g., `shahkar.genai@gmail.com`  
   - Subject: Use expression from structured parser, e.g., `={{$json.subject}}` or fixed string  
   - Message: Use `={{ $json.output.body }}` from the summary agent output  
   - Connect "Summarizer Agent" → "Send email for outreach".

10. **Save and test the full workflow:**  
    - Trigger manually through "When clicking ‘Test workflow’"  
    - Verify each step completes without errors  
    - Adjust API keys, pagination, or email parameters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| For support or questions regarding this workflow, contact Yaron at Yaron@nofluff.online.                                                                     | Workflow assistance contact                                                                         |
| Explore more tips and video tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos                                                                   | YouTube channel for workflow guidance                                                              |
| LinkedIn profile for additional professional resources: https://www.linkedin.com/in/yaronbeen/                                                               | LinkedIn profile of workflow author                                                                 |
| This workflow optimizes CrunchBase data handling by eliminating manual copy-pasting, ideal for startup scouting, cold outreach, and CRM enrichment.          | Workflow purpose and use cases                                                                      |
| To customize outreach recipients, modify the Gmail node’s “Send To” parameter; to handle multiple recipients, provide comma-separated emails.               | Customization tip                                                                                   |
| Adjust CrunchBase API queries (page number, updated_since) in the HTTP Request nodes to refine which companies and profiles are fetched.                     | API query customization tips                                                                         |
| Ensure valid CrunchBase API keys and Gmail OAuth2 credentials are configured before running to avoid authentication errors.                                   | Credential setup requirement                                                                        |
| The AI summarization prompt is designed to produce concise, relevant founder summaries suitable for personalized outreach emails, avoiding unnecessary data. | AI prompt design and summary focus                                                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, respecting all applicable content policies. It contains no illegal, offensive, or protected content. All data handled is legal and publicly accessible.