Automated Company Research & Lead Enrichment with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/automated-company-research---lead-enrichment-with-gpt-4o-and-google-sheets-6776


# Automated Company Research & Lead Enrichment with GPT-4o and Google Sheets

---

## 1. Workflow Overview

This workflow automates the process of researching and enriching company data using AI (GPT-4o) and Google Sheets. It targets users who maintain lead or company lists in Google Sheets and want to augment these leads with detailed business intelligence automatically.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Triggering the workflow manually or on a schedule and reading company names from a Google Sheet.
- **1.2 Batch Processing**: Splitting the list of companies into individual items for sequential processing.
- **1.3 AI-driven Company Research**: Using GPT-4o with integrated search tools (SerpAPI or ScrapingBee) and a website content extraction sub-workflow to find detailed company information.
- **1.4 Data Structuring and Parsing**: Parsing the AI's structured JSON output to extract specific data fields.
- **1.5 Data Merging and Update**: Merging parsed data and updating the original Google Sheet row with enrichment results.
- **1.6 Workflow Management and Documentation**: Notes and instructions embedded as sticky notes for user guidance.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block handles the initiation of the workflow via manual trigger or schedule, and fetches rows from a Google Sheet where enrichment is pending.

**Nodes Involved:**  
- When clicking "Test workflow" (Manual Trigger)  
- Schedule Trigger  
- Get rows to enrich (Google Sheets)  
- Input (Set node)

**Node Details:**

- **When clicking "Test workflow"**  
  - Type: Manual Trigger  
  - Role: Entry point for manual execution.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Triggers "Get rows to enrich" node.  
  - Edge Cases: None specific; manual trigger can be missed.  
  - Version: v1

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically triggers workflow every 2 hours.  
  - Configuration: Interval set to 2 hours.  
  - Inputs: None  
  - Outputs: Triggers "Get rows to enrich" node.  
  - Edge Cases: Misconfigured schedule or disabled node disables automatic runs.  
  - Version: v1.2  

- **Get rows to enrich**  
  - Type: Google Sheets (Read)  
  - Role: Reads rows from Google Sheet where `enrichment_status` column is empty or indicates pending.  
  - Configuration:  
    - Document ID and Sheet Name linked to a specific Google Sheet template.  
    - Filters rows by `enrichment_status` column (returns all matches).  
  - Inputs: Trigger from manual or schedule node.  
  - Outputs: Emits JSON items for each row to enrich.  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Authentication errors, empty result sets, or API limits.  
  - Version: v4.3  

- **Input**  
  - Type: Set  
  - Role: Extracts and assigns `company_input` and `row_number` from incoming JSON.  
  - Configuration: Assigns two variables:  
    - `company_input` as string input from `$json.input`  
    - `row_number` as number from `$json.row_number`  
  - Inputs: From "Get rows to enrich"  
  - Outputs: Forwards data to "Loop Over Items"  
  - Edge Cases: Missing input values cause failures downstream.  
  - Version: v3.3  

---

### 2.2 Batch Processing

**Overview:**  
This block ensures companies are processed one at a time for stability and ease of debugging.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Sticky Note1 (Explanation)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes each company row individually.  
  - Configuration: Default batch size (1, implicit).  
  - Inputs: From "Input" node.  
  - Outputs: Triggers AI research block nodes sequentially.  
  - Edge Cases: Large datasets may slow processing; batch size can be adjusted.  
  - Version: v3  

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Documentation; explains batch processing purpose.  
  - Content: "This Split in Batches node ensures companies are researched individually. This makes the workflow more stable and easier to debug."  
  - Inputs/Outputs: None  

---

### 2.3 AI-driven Company Research

**Overview:**  
The core AI agent uses GPT-4o to research company info based on the company name. It integrates external tools for search and website content extraction to enrich the AI prompt context.

**Nodes Involved:**  
- AI company researcher (LangChain Agent)  
- OpenAI Chat Model (GPT-4o)  
- SerpAPI - Search Google (optional external search)  
- Search Google with ScrapingBee (alternative search tool)  
- Get website content (sub-workflow tool)  
- Structured Output Parser  
- Sticky Note2, Sticky Note3, Sticky Note4, Sticky Note6 (documentation and API keys)

**Node Details:**

- **AI company researcher**  
  - Type: LangChain Agent  
  - Role: Central AI node that crafts and sends the research prompt.  
  - Configuration:  
    - Prompt instructs AI to find LinkedIn URL, domain, market (B2B/B2C), pricing plans, case study URL, API availability, enterprise plan presence, free trial availability, and integrations.  
    - Max iterations set to 10 to allow multi-step reasoning.  
    - Uses integrated AI tools (SerpAPI, website content extraction).  
    - Has output parser enabled to enforce structured output.  
  - Inputs: Receives company name from "Loop Over Items" node.  
  - Outputs: Produces AI response JSON to "AI Researcher Output Data" node.  
  - Edge Cases:  
    - API limits or auth errors on OpenAI or search tools.  
    - AI prompt misunderstanding or unexpected output format.  
  - Version: v1.6  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides the GPT-4o model interface.  
  - Configuration:  
    - Model set to "gpt-4o" with temperature 0.3 for controlled responses.  
  - Inputs: Connected as AI language model to "AI company researcher".  
  - Outputs: Feeds AI researcher with language model inference results.  
  - Credentials: OpenAI API Key  
  - Edge Cases: API key invalid, rate limits, network issues.  
  - Version: v1  

- **SerpAPI - Search Google**  
  - Type: LangChain Tool (SerpAPI)  
  - Role: Provides Google search results to enrich AI context.  
  - Configuration: Default parameters; uses SerpAPI key.  
  - Inputs: Connected as AI tool to "AI company researcher".  
  - Outputs: Search results JSON to AI researcher.  
  - Credentials: SerpAPI API Key  
  - Edge Cases: API limits, quota exceeded, malformed queries.  
  - Version: v1  

- **Search Google with ScrapingBee**  
  - Type: LangChain Tool (HttpRequest sub-workflow)  
  - Role: Alternative cheaper search API for Google queries.  
  - Configuration: Calls ScrapingBee API with Google search query parameters.  
  - Inputs: Connected as AI tool alternative to "AI company researcher" (optional).  
  - Outputs: Search results for AI.  
  - Credentials: ScrapingBee API Key  
  - Edge Cases: Authentication failures, API quota limits.  
  - Version: v1.1  

- **Get website content**  
  - Type: LangChain Tool (Sub-workflow)  
  - Role: Extracts textual content from a given URL's HTML.  
  - Configuration: Sub-workflow visits URL, extracts text from `<html>`, skipping `<head>`.  
  - Inputs: Connected as AI tool to "AI company researcher".  
  - Outputs: Website textual content for AI prompt enrichment.  
  - Edge Cases: Website inaccessible, HTTP errors, parsing errors.  
  - Version: v1.1  

- **Structured Output Parser**  
  - Type: LangChain Output Parser  
  - Role: Defines expected AI output JSON schema for validation and parsing.  
  - Configuration: Explicit JSON schema with keys such as `case_study_link`, `domain`, `linkedinUrl`, `market`, pricing and plan booleans, and `integrations` array.  
  - Inputs: AI raw output from "AI company researcher".  
  - Outputs: Validated and parsed JSON to "AI Researcher Output Data".  
  - Edge Cases: AI output not matching schema causes parsing errors.  
  - Version: v1.2  

- **Sticky Notes (2,3,4,6)**  
  - Provide documentation on:  
    - Defining AI output schema  
    - The main AI prompt role  
    - SerpAPI API key info  
    - ScrapingBee as cost-effective alternative with key link  

---

### 2.4 Data Structuring and Parsing

**Overview:**  
This block reformats the AI output into discrete fields matching Google Sheets columns for update.

**Nodes Involved:**  
- AI Researcher Output Data (Set)  
- Merge data (Merge)

**Node Details:**

- **AI Researcher Output Data**  
  - Type: Set  
  - Role: Extracts and maps each AI output field into named variables for later use.  
  - Configuration: Assigns values from `$json.output` fields for domain, linkedinUrl, market, cheapest_plan, has_enterprise_plan, has_API, has_free_trial, integrations, case_study_link.  
  - Inputs: From "AI company researcher" node.  
  - Outputs: Passes structured data to "Merge data".  
  - Edge Cases: Missing or null values from AI output propagate downstream.  
  - Version: v3.3  

- **Merge data**  
  - Type: Merge  
  - Role: Combines original input data with AI-enriched data into one JSON object.  
  - Configuration: Mode "combine" with "mergeByPosition".  
  - Inputs: Receives AI Researcher Output Data and original batch item data.  
  - Outputs: Feeds combined data to Google Sheets update node.  
  - Edge Cases: Position mismatch may cause data misalignment.  
  - Version: v2.1  

---

### 2.5 Data Merging and Update

**Overview:**  
Updates the original Google Sheet row with enriched company data and marks enrichment status as done.

**Nodes Involved:**  
- Google Sheets - Update Row with data

**Node Details:**

- **Google Sheets - Update Row with data**  
  - Type: Google Sheets (Update)  
  - Role: Writes enriched data back into the Google Sheet row identified by `row_number`.  
  - Configuration:  
    - Mapping of all fields to corresponding sheet columns.  
    - Uses `row_number` to identify correct row.  
    - Sets `enrichment_status` as "done" to avoid reprocessing.  
  - Inputs: From "Merge data".  
  - Outputs: Triggers next batch item processing (loops back to "Loop Over Items").  
  - Credentials: Google Sheets OAuth2  
  - Edge Cases: Row mismatch, API errors, permission issues.  
  - Version: v4.3  

---

### 2.6 Workflow Management and Documentation

**Overview:**  
Sticky notes provide instructions, usage guides, and references to templates and API keys for user convenience.

**Nodes Involved:**  
- Read Me (Sticky Note)  
- Sticky Note5 (Activate instruction)  
- Sticky Note7 (Google Sheet template link)

**Node Details:**

- **Read Me**  
  - Content: Detailed instructions on workflow features, how to use, API key setup, and customization tips.  
  - Context: Includes Google Sheet template link, OpenAI and Search API setup instructions.

- **Sticky Note5**  
  - Content: "Activate - Run workflow manually or activate schedule trigger."

- **Sticky Note7**  
  - Content: Link to Google Sheet template with instructions to copy it.

---

## 3. Summary Table

| Node Name                         | Node Type                           | Functional Role                        | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                                          |
|----------------------------------|-----------------------------------|-------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"    | Manual Trigger                    | Manual workflow start                | None                       | Get rows to enrich             | ▶️ **Activate** Run the workflow manually or activate the schedule trigger to run it automatically.                                  |
| Schedule Trigger                 | Schedule Trigger                  | Scheduled workflow start             | None                       | Get rows to enrich             | ▶️ **Activate** Run the workflow manually or activate the schedule trigger to run it automatically.                                  |
| Get rows to enrich              | Google Sheets (Read)              | Fetch rows pending enrichment        | Manual Trigger, Schedule   | Input                         | 📄 **Get Your Sheet!** [Google Sheet template](https://docs.google.com/spreadsheets/d/1_F_An3yLSCsjgLN5w9RcJN-lJj_6mUT1nngMJPYws2Y) |
| Input                          | Set                              | Extract company name and row number | Get rows to enrich         | Loop Over Items                |                                                                                                                                      |
| Loop Over Items                | SplitInBatches                   | Process companies one by one         | Input                      | AI company researcher, Merge data | ### Process One by One This Split in Batches node ensures companies are researched individually. This makes the workflow more stable and easier to debug. |
| AI company researcher          | LangChain Agent                  | AI research using GPT-4o + tools    | Loop Over Items            | AI Researcher Output Data      | 🧠 **The AI's Brain** This is the main prompt! Edit the text here to tell the AI exactly what information to find.                    |
| OpenAI Chat Model              | LangChain OpenAI Chat Model      | GPT-4o language model interface      | AI company researcher (ai_languageModel) | AI company researcher (ai_languageModel) |                                                                                                                                      |
| SerpAPI - Search Google        | LangChain Tool (SerpAPI)         | Google search for AI context         | AI company researcher (ai_tool) | AI company researcher (ai_tool) | Get your free API key here https://serpapi.com/                                                                                    |
| Search Google with ScrapingBee | LangChain Tool (HTTP Request)    | Alternative Google search via ScrapingBee | AI company researcher (ai_tool) (optional) | AI company researcher (ai_tool) | ### Cost-Effective Search 💰 ScrapingBee can be a cheaper alternative to SerpAPI for Google searches. Get your free API key at https://www.scrapingbee.com/. Remember to connect this tool to the AI Agent if you use it! |
| Get website content            | LangChain Tool (Sub-workflow)    | Extract website textual content      | AI company researcher (ai_tool) | AI company researcher (ai_tool) |                                                                                                                                      |
| Structured Output Parser       | LangChain Output Parser          | Validate and parse AI JSON output    | AI company researcher (ai_outputParser) | AI company researcher          | 🎯 **Define Your Output** Specify the exact JSON schema the AI should follow for its response. This must match the data you request in the main AI prompt. |
| AI Researcher Output Data      | Set                             | Map parsed AI data to variables      | AI company researcher       | Merge data                    |                                                                                                                                      |
| Merge data                    | Merge                           | Combine original and AI data         | AI Researcher Output Data, Loop Over Items | Google Sheets - Update Row with data |                                                                                                                                      |
| Google Sheets - Update Row with data | Google Sheets (Update)          | Write enriched data back to Sheet    | Merge data                 | Loop Over Items               |                                                                                                                                      |
| Read Me                      | Sticky Note                     | Workflow documentation and instructions | None                      | None                         | ## 🤖 AI Agent: Automated Company Research ... (comprehensive instructions, features, and usage)                                     |
| Sticky Note1                 | Sticky Note                     | Explanation of batch processing      | None                      | None                         | ### Process One by One This Split in Batches node ensures companies are researched individually. This makes the workflow more stable and easier to debug. |
| Sticky Note2                 | Sticky Note                     | Defining AI output schema             | None                      | None                         | 🎯 **Define Your Output** Specify the exact JSON schema the AI should follow for its response. This must match the data you request in the main AI prompt. |
| Sticky Note3                 | Sticky Note                     | AI prompt explanation                 | None                      | None                         | 🧠 **The AI's Brain** This is the main prompt! Edit the text here to tell the AI exactly what information to find.                    |
| Sticky Note4                 | Sticky Note                     | SerpAPI key instructions              | None                      | None                         | Get your free API key here https://serpapi.com/                                                                                    |
| Sticky Note5                 | Sticky Note                     | Activation instruction                | None                      | None                         | ▶️ **Activate** Run the workflow manually or activate the schedule trigger to run it automatically.                                  |
| Sticky Note6                 | Sticky Note                     | ScrapingBee usage instructions        | None                      | None                         | ### Cost-Effective Search 💰 ScrapingBee can be a cheaper alternative to SerpAPI for Google searches. Get your free API key at https://www.scrapingbee.com/. Remember to connect this tool to the AI Agent if you use it! |
| Sticky Note7                 | Sticky Note                     | Google Sheet template link             | None                      | None                         | 📄 **Get Your Sheet!** [Grab the Google Sheet template here](https://docs.google.com/spreadsheets/d/1_F_An3yLSCsjgLN5w9RcJN-lJj_6mUT1nngMJPYws2Y/edit?usp=sharing) and follow the main "Read Me" instructions to set it up. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** node named `When clicking "Test workflow"` (default settings).  
   - Add a **Schedule Trigger** node named `Schedule Trigger`: configure to run every 2 hours.

2. **Configure Google Sheets Read Node**  
   - Add a **Google Sheets** node named `Get rows to enrich`.  
   - Set operation to "Read Rows".  
   - Connect Google Sheets OAuth2 credentials.  
   - Set Document ID to your Google Sheet (copy of the template).  
   - Set Sheet Name to the first sheet (`gid=0`).  
   - Add a filter on column `enrichment_status` to select rows pending enrichment.  
   - Connect outputs of both triggers to this node.

3. **Set Input Variables**  
   - Add a **Set** node named `Input`.  
   - Add two fields:  
     - `company_input` (string) with value `={{ $json.input }}`  
     - `row_number` (number) with value `={{ $json.row_number }}`  
   - Connect `Get rows to enrich` to `Input`.

4. **Batch Processing**  
   - Add **SplitInBatches** node named `Loop Over Items`.  
   - Default batch size of 1 (process one company at a time).  
   - Connect `Input` to `Loop Over Items`.

5. **Add AI Research Nodes**  
   - Add **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.  
     - Model: `gpt-4o`, Temperature: 0.3.  
     - Connect your OpenAI API credentials.  
   - Add **LangChain Agent** node named `AI company researcher`.  
     - Configure prompt with detailed instructions to find LinkedIn URL, domain, market type, pricing, case studies, API presence, enterprise plan, free trial, integrations.  
     - Max iterations: 10.  
     - Enable output parser.  
     - Connect `OpenAI Chat Model` as AI language model to this node.  
   - Add **SerpAPI - Search Google** node (optional) or **Search Google with ScrapingBee** node (alternative cheaper search).  
     - Connect respective API credentials.  
     - Connect as AI tool inputs to `AI company researcher`.  
   - Add **Get website content** node (LangChain toolWorkflow sub-workflow).  
     - Import or recreate the sub-workflow that fetches and extracts HTML content from URLs.  
     - Connect as AI tool input to `AI company researcher`.  
   - Add **Structured Output Parser** node.  
     - Define JSON schema for expected AI output (fields like domain, linkedinUrl, market, pricing, plans, integrations).  
     - Connect as AI output parser to `AI company researcher`.

6. **Map AI Output Data**  
   - Add a **Set** node named `AI Researcher Output Data`.  
   - Assign variables from `$json.output` fields to named variables for domain, linkedinUrl, market, cheapest_plan, has_enterprise_plan, has_API, has_free_trial, integrations, case_study_link.  
   - Connect `AI company researcher` output to this node.

7. **Merge Input and AI Data**  
   - Add a **Merge** node named `Merge data`.  
   - Mode: "combine", combination mode: "mergeByPosition".  
   - Connect `AI Researcher Output Data` and `Loop Over Items` (original item) to this node.

8. **Update Google Sheet Row**  
   - Add a **Google Sheets** node named `Google Sheets - Update Row with data`.  
   - Operation: "Update".  
   - Connect Google Sheets OAuth2 credentials.  
   - Use the same Document ID and Sheet Name as before.  
   - Configure mapping of columns: domain, linkedinUrl, market, cheapest_plan, has_free_trial, has_enterprise_plan, last_case_study_link, integrations, enrichment_status ("done"), row_number (used as matching column).  
   - Connect output of `Merge data` to this node.  
   - Connect output of this node back to `Loop Over Items` to continue batch processing.

9. **Add Sticky Notes for Documentation**  
   - Add sticky notes similar to the existing ones, containing:  
     - Workflow overview and usage instructions.  
     - Explanation of batch processing.  
     - AI prompt customization guidance.  
     - API key instructions for SerpAPI and ScrapingBee.  
     - Google Sheet template link.  
     - Activation instructions.

10. **Test and Activate**  
    - Run the workflow manually via the manual trigger.  
    - Confirm that rows in the Google Sheet with empty `enrichment_status` get enriched and updated.  
    - Activate the schedule trigger to run every 2 hours automatically.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses an AI agent to automatically research companies from a Google Sheet and enrich it with valuable data including domain, LinkedIn URL, market focus, pricing information, trial availability, API availability, software integrations, and case study links.                                      | Workflow purpose summary                                                                                                                |
| Google Sheet template for input and output columns: [https://docs.google.com/spreadsheets/d/1_F_An3yLSCsjgLN5w9RcJN-lJj_6mUT1nngMJPYws2Y/edit?usp=sharing](https://docs.google.com/spreadsheets/d/1_F_An3yLSCsjgLN5w9RcJN-lJj_6mUT1nngMJPYws2Y/edit?usp=sharing)                                                                 | Input/output data format and starting point                                                                                            |
| OpenAI API key is required with GPT-4o access for best results.                                                                                                                                                                                                                                                  | Credential setup                                                                                                                        |
| SerpAPI is recommended for easy Google search integration: [https://serpapi.com/](https://serpapi.com/)                                                                                                                                                                                                          | Search tool option                                                                                                                      |
| ScrapingBee offers a cost-effective alternative to SerpAPI for Google search functionality: [https://www.scrapingbee.com/](https://www.scrapingbee.com/)                                                                                                                                                         | Alternative search tool option                                                                                                         |
| The AI prompt and output schema can be customized to tailor the research to different or additional company data points.                                                                                                                                                                                        | Workflow customization tip                                                                                                             |
| Batch processing one company at a time improves stability and simplifies error tracing.                                                                                                                                                                                                                          | Operational insight                                                                                                                    |
| The included website content extraction sub-workflow fetches and cleans text from company websites to aid AI research.                                                                                                                                                                                          | AI tool workflow detail                                                                                                                |
| Possible failure points include API rate limits, authentication errors, network issues, and unexpected AI output formats requiring error handling and monitoring.                                                                                                                                               | Error and edge case awareness                                                                                                          |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---