HubSpot Contact AI Enrichment

https://n8nworkflows.xyz/workflows/hubspot-contact-ai-enrichment-8969


# HubSpot Contact AI Enrichment

### 1. Workflow Overview

This workflow automates the enrichment of newly created or updated HubSpot contacts by gathering detailed company information through AI-driven web research. It targets recently added contacts (excluding Gmail addresses) and enhances their HubSpot records with structured company data such as industry, location, size, and online presence.

**Primary Use Case:**  
Businesses using HubSpot CRM who want to automatically enrich new contacts with accurate company profiles to improve sales and marketing insights without manual research.

**Logical Blocks:**

- **1.1 Contact Retrieval and Filtering:** Fetch recently created or updated HubSpot contacts, then filter to retain only those created within the last 24 hours and exclude Gmail addresses.
- **1.2 Company Research via AI:** Extract the email domain, then perform web research using SerpAPI combined with Google Gemini Chat Model to generate a structured company profile.
- **1.3 HubSpot Contact Update:** Update the original HubSpot contact records with the enriched company information obtained from AI.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Contact Retrieval and Filtering

- **Overview:**  
  This block fetches the latest HubSpot contacts and filters them to focus on those created in the past 24 hours and excluding contacts with Gmail addresses, which are less relevant for company enrichment.

- **Nodes Involved:**  
  - Run daily  
  - Get recently created/updated contacts  
  - Filter contacts created in the last 24h

- **Node Details:**

  1. **Run daily**  
     - *Type & Role:* Schedule Trigger; initiates the workflow automatically once per day.  
     - *Configuration:* Default daily interval without additional parameters.  
     - *Connections:* Outputs to "Get recently created/updated contacts".  
     - *Edge Cases:* Scheduling misconfiguration could cause missed runs; ensure timezone settings align with expectations.

  2. **Get recently created/updated contacts**  
     - *Type & Role:* HubSpot node; retrieves contacts recently created or updated.  
     - *Configuration:*  
       - Operation: `getRecentlyCreatedUpdated`  
       - Limit: 1 (fetches one contact per run, can be increased for batch processing)  
       - Authentication: OAuth2 via configured HubSpot credentials.  
     - *Key Expressions:* None beyond static configuration.  
     - *Connections:* Outputs to "Filter contacts created in the last 24h".  
     - *Edge Cases:* API rate limits or auth token expiry; requires valid OAuth2 credentials.  
     - *Potential Issues:* Limiting to one contact may delay enrichment if multiple new contacts exist.

  3. **Filter contacts created in the last 24h**  
     - *Type & Role:* Filter node; removes contacts not created within last 24 hours or with Gmail addresses.  
     - *Configuration:*  
       - Conditions:  
         - Createdate >= current date minus 1 day, using dateTime comparison.  
         - Email domain does not contain "gmail.com".  
       - Expressions:  
         - `={{ $json.properties.createdate.value.toDateTime('ms') }}` for date comparison.  
         - `={{ $json['identity-profiles'][0].identities[0].value }}` to access email for domain check.  
     - *Connections:* Outputs filtered contacts to "Company Research Agent".  
     - *Edge Cases:* Missing or malformed email fields; date parsing errors.  
     - *Failure Modes:* If contact data structure changes, expressions may fail.

---

#### Block 1.2: Company Research via AI

- **Overview:**  
  This block uses the contact's email domain to search the web for company information with SerpAPI, then processes the results through Google Gemini Chat Model to produce a structured JSON company profile.

- **Nodes Involved:**  
  - Company Research Agent  
  - Search the web with SerpAPI  
  - Google Gemini Chat Model  
  - Structured Output Parser

- **Node Details:**

  1. **Company Research Agent**  
     - *Type & Role:* AI agent node orchestrating multiple AI tools and models.  
     - *Configuration:*  
       - Prompt: Analyst instruction to research the company domain extracted from the contact’s email.  
       - Output: Structured JSON with fields like company_name, industry, location, employee_count, website, linkedin, description.  
       - Tools used: SerpAPI for web search, Google Gemini for language model processing.  
       - Output parser enabled to enforce structured JSON output.  
     - *Expressions:*  
       - Domain extracted via: `{{ $json['identity-profiles'][0].identities[0].value }}` (email)  
     - *Connections:*  
       - Uses "Search the web with SerpAPI" as AI tool input  
       - Uses "Google Gemini Chat Model" as AI language model input  
       - Uses "Structured Output Parser" as AI output parser  
       - Outputs parsed data to "Add company info" node  
     - *Edge Cases:*  
       - Domain extraction failure if email format unexpected.  
       - API quota or network errors from SerpAPI and Google Gemini.  
       - AI output parsing errors if response not JSON or malformed.  
     - *Version Requirements:* Uses LangChain nodes, version 2 or above recommended for compatibility.

  2. **Search the web with SerpAPI**  
     - *Type & Role:* AI tool node to perform web searches.  
     - *Configuration:* Uses connected SerpAPI credentials; no additional parameters set, defaults active.  
     - *Connections:* Connected as AI tool in "Company Research Agent".  
     - *Edge Cases:* API key limits, query failure, or empty search results.

  3. **Google Gemini Chat Model**  
     - *Type & Role:* AI language model providing natural language processing and reasoning.  
     - *Configuration:* Uses Google Gemini (PaLM) API credentials.  
     - *Connections:* Connected as AI language model in "Company Research Agent".  
     - *Edge Cases:* Auth errors, rate limits, or model unavailability.

  4. **Structured Output Parser**  
     - *Type & Role:* Parses AI output enforcing a JSON schema.  
     - *Configuration:* Defines example JSON schema matching expected company profile fields.  
     - *Connections:* Parses output of "Company Research Agent" before forwarding data.  
     - *Edge Cases:* Parsing errors if AI output is malformed or incomplete.

---

#### Block 1.3: HubSpot Contact Update

- **Overview:**  
  Enriches the original HubSpot contact record with the structured company information generated by AI, updating fields like company name, size, website, city, country, and LinkedIn URL.

- **Nodes Involved:**  
  - Add company info

- **Node Details:**

  1. **Add company info**  
     - *Type & Role:* HubSpot node updating contact properties.  
     - *Configuration:*  
       - Operation: Update contact by email.  
       - Email: Dynamically set from the original contact’s email.  
       - Additional Fields:  
         - city, country, websiteUrl, companyName, companySize, linkedinUrl  
         - Values extracted from AI output JSON fields (`$json.output`)  
       - Authentication: OAuth2 via HubSpot credentials.  
     - *Expressions:*  
       - Email: `={{ $('Get recently created/updated contacts').item.json['identity-profiles'][0].identities[0].value }}`  
       - Other fields: `={{ $json.output.headquarters_city }}`, etc.  
     - *Connections:* Receives input from "Company Research Agent".  
     - *Edge Cases:*  
       - Missing or partial AI output fields.  
       - Update failures if contact email is incorrect or contact deleted.  
       - HubSpot API errors or auth token expiration.  
     - *Best Practices:* Validate existence of AI output data before updating to avoid overwriting with nulls.

---

### 3. Summary Table

| Node Name                         | Node Type                               | Functional Role                         | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                      |
|----------------------------------|---------------------------------------|---------------------------------------|---------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| Run daily                        | Schedule Trigger                      | Workflow start trigger                 |                                 | Get recently created/updated contacts |                                                                                                |
| Get recently created/updated contacts | HubSpot                              | Retrieve recent HubSpot contacts       | Run daily                      | Filter contacts created in the last 24h |                                                                                                |
| Filter contacts created in the last 24h | Filter                               | Filter contacts by creation date & domain | Get recently created/updated contacts | Company Research Agent           |                                                                                                |
| Company Research Agent            | LangChain Agent                      | AI research and company profile creation | Filter contacts created in the last 24h | Add company info                |                                                                                                |
| Search the web with SerpAPI       | LangChain Tool (SerpAPI)             | Web search tool for company data      |                                 | Company Research Agent (tool input) |                                                                                                |
| Google Gemini Chat Model          | LangChain Language Model (Google Gemini) | AI language model processing          |                                 | Company Research Agent (LM input) |                                                                                                |
| Structured Output Parser          | LangChain Output Parser              | Parse AI output to structured JSON    |                                 | Company Research Agent (parser output) |                                                                                                |
| Add company info                 | HubSpot                              | Update HubSpot contact with company data | Company Research Agent          |                                |                                                                                                |
| Sticky Note                     | Sticky Note                         | Explanation and setup instructions    |                                 |                                | AI company enrichment for new HubSpot contacts. Runs on a schedule, finds contacts from last 24h (skips Gmail), looks up company with SerpAPI + Gemini, updates HubSpot. Setup steps and optional tweaks included. |
| Sticky Note1                    | Sticky Note                         | Section title for contact retrieval   |                                 |                                | ## 1. Get contact(s)                                                                           |
| Sticky Note2                    | Sticky Note                         | Section title for company research    |                                 |                                | ## 2. Research company                                                                         |
| Sticky Note3                    | Sticky Note                         | Section title for contact update      |                                 |                                | ## 3. Update contact                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Run daily`  
   - Type: `Schedule Trigger`  
   - Configuration: Set to run once every day (default interval)  
   - Connect output to `Get recently created/updated contacts`.

2. **Create a HubSpot node for retrieving contacts**  
   - Name: `Get recently created/updated contacts`  
   - Type: `HubSpot`  
   - Operation: `getRecentlyCreatedUpdated`  
   - Limit: `1` (adjust as needed)  
   - Authentication: Connect your HubSpot OAuth2 credentials.  
   - Connect output to `Filter contacts created in the last 24h`.

3. **Create a Filter node**  
   - Name: `Filter contacts created in the last 24h`  
   - Type: `Filter`  
   - Conditions:  
     - Createdate >= (current date - 1 day), use expression: `={{ $json.properties.createdate.value.toDateTime('ms') }}` and compare with `={{ $today.minus(1, 'days') }}`  
     - Email domain must not contain "gmail.com", expression: `={{ $json['identity-profiles'][0].identities[0].value }}` for email extraction  
   - Connect output to `Company Research Agent`.

4. **Create a LangChain Agent node for company research**  
   - Name: `Company Research Agent`  
   - Type: `LangChain Agent`  
   - Prompt:  
     ```
     You’re an analyst specializing in company profiling. You’ll receive a domain and must return structured data only.

     Domain to research (extract it from the email and then search the web from the SerpAPI tool): {{ $json['identity-profiles'][0].identities[0].value }}

     Return only a JSON object with exactly these fields (keep the keys as written):

     company_name: Full legal or commonly used company name
     industry: Primary industry category
     headquarters_city: City
     headquarters_country: Country
     employee_count: Approximate employee count (number or range)
     website: Official site URL
     linkedin: Company LinkedIn URL (if available)
     description: One or two sentences summarizing what the company does

     Use reliable, verifiable sources. No commentary—output JSON only.
     ```
   - Enable Output Parser with a JSON schema example matching the above fields.  
   - Add AI tools:  
     - `Search the web with SerpAPI` (create next)  
     - `Google Gemini Chat Model` (create next)  
   - Connect outputs to `Add company info`.

5. **Create a LangChain Tool node**  
   - Name: `Search the web with SerpAPI`  
   - Type: `LangChain Tool (SerpAPI)`  
   - Authentication: Connect your SerpAPI account credentials.  
   - Leave options as default.  
   - Connect as AI tool input within `Company Research Agent`.

6. **Create a LangChain Language Model node**  
   - Name: `Google Gemini Chat Model`  
   - Type: `LangChain Language Model (Google Gemini)`  
   - Authentication: Connect Google Gemini (PaLM) API credentials.  
   - Leave options as default.  
   - Connect as AI language model input within `Company Research Agent`.

7. **Create a LangChain Output Parser node**  
   - Name: `Structured Output Parser`  
   - Type: `LangChain Output Parser (Structured)`  
   - Configure with JSON schema example matching the company profile fields (same as prompt).  
   - Connect as AI output parser input within `Company Research Agent`.

8. **Create a HubSpot node to update contact**  
   - Name: `Add company info`  
   - Type: `HubSpot`  
   - Operation: Update contact by Email  
   - Email parameter: `={{ $('Get recently created/updated contacts').item.json['identity-profiles'][0].identities[0].value }}`  
   - Additional Fields to update (mapped from AI output):  
     - city: `={{ $json.output.headquarters_city }}`  
     - country: `={{ $json.output.headquarters_country }}`  
     - websiteUrl: `={{ $json.output.website }}`  
     - companyName: `={{ $json.output.company_name }}`  
     - companySize: `={{ $json.output.employee_count }}`  
     - linkedinUrl: `={{ $json.output.linkedin }}`  
   - Authentication: Connect your HubSpot OAuth2 credentials.  
   - Connect input from `Company Research Agent`.

9. **Validate all nodes and credentials**  
   - Ensure OAuth2 credentials for HubSpot are connected on both HubSpot nodes.  
   - Ensure SerpAPI key is valid and connected.  
   - Ensure Google Gemini (PaLM) API key is valid and connected.

10. **Activate the workflow**  
    - Test with recent contacts to verify enrichment runs as expected.  
    - Adjust contact retrieval limit or schedule frequency as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                         | Context or Link                                                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| AI company enrichment for new HubSpot contacts. Runs on a schedule, finds contacts from the last 24 hours (skips Gmail), looks up the company with SerpAPI + Gemini, and updates the HubSpot contact with company details. Setup instructions included. | Explanation note in Sticky Note node at workflow start.          |
| Setup SerpAPI account and get API key at [https://serpapi.com/manage-api-key](https://serpapi.com/manage-api-key)                                                                                                                    | Credential setup guidance                                        |
| Create Google Gemini API key in [Google AI Studio](https://aistudio.google.com/)                                                                                                                                                      | Credential setup guidance                                        |
| The workflow uses LangChain nodes for AI orchestration, including agent, tool, model, and output parser. Compatibility with n8n versions supporting these nodes (v0.210+) recommended.                                                  | Technical note on node compatibility                            |

---

**Disclaimer:**  
The provided content is generated exclusively from an n8n automated workflow and follows all current content policies. No illegal, offensive, or protected data is included. All data processed is legal and public.