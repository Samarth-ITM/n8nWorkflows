Scrape & Enrich Google Maps Leads with Decodo API and Gemini 2.5 Flash

https://n8nworkflows.xyz/workflows/scrape---enrich-google-maps-leads-with-decodo-api-and-gemini-2-5-flash-10760


# Scrape & Enrich Google Maps Leads with Decodo API and Gemini 2.5 Flash

### 1. Workflow Overview

This n8n workflow named **"Decodo Google Maps Lead Enrichment"** automates the process of scraping leads from Google Maps, enriching those leads with AI-driven business insights, scoring their quality, and preparing personalized outreach messages for high-potential prospects. It is designed for B2B lead generation and cold outreach campaigns focused on local businesses.

The workflow is logically divided into the following blocks:

- **1.1 Input Configuration & Data Collection:** Receives search parameters, scrapes Google Maps data via the Decodo Scraper API, and parses raw HTML responses into structured lead objects.
- **1.2 Lead Enrichment with AI:** Processes leads in batches to generate detailed business insights using Google Gemini 2.5 Flash AI, including value propositions, pain points, outreach hooks, and lead scores.
- **1.3 Data Management & Filtering:** Merges enriched data, saves all leads to Google Sheets, filters for high-quality leads (lead score ≥7 with contact info), then prepares personalized outreach messages.
- **1.4 Outreach Message Preparation & Saving:** Formats email subjects and bodies using AI-generated content, and saves these personalized messages back to Google Sheets.
- **1.5 Error Handling:** Captures and formats errors occurring anywhere in the workflow and sends Telegram notifications to the admin.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Configuration & Data Collection

**Overview:**  
This block sets the initial search parameters, calls the Decodo Google Maps scraping API, and parses the HTML response to extract structured business leads.

**Nodes Involved:**  
- Manual Trigger  
- Set Search Parameters  
- Decodo Maps Scraper  
- Parse & Normalize Data  
- Split Into Batches  

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node to manually start the workflow.  
  - Role: Entry point to initiate a new scraping/enrichment cycle.  
  - Connections: Outputs to "Set Search Parameters".  
  - Edge cases: Manual start only; no input validation here.

- **Set Search Parameters**  
  - Type: Set node to define static parameters.  
  - Role: Defines search query, target language, country, and results limit for the scraper.  
  - Key Variables:  
    - `searchQuery` (string): The Google Maps search string, e.g., "coffee shops in Austin".  
    - `targetLanguage` (string): Language code, e.g., "en".  
    - `country` (string): Name of target country.  
    - `resultsLimit` (number): Number of results to fetch.  
  - Connections: Outputs to "Decodo Maps Scraper".  
  - Edge cases: User must update these parameters for each campaign.

- **Decodo Maps Scraper**  
  - Type: HTTP Request node (POST).  
  - Role: Calls Decodo's Google Maps scraper API with configured parameters.  
  - Configuration:  
    - URL: `https://scraper-api.decodo.com/v2/scrape`  
    - Authentication: HTTP Header Auth with Decodo API key (Authorization: Basic).  
    - Body parameters include target "google_maps", query, page range, language, geo, and headless HTML output.  
  - Inputs: Receives search parameters from previous node.  
  - Outputs: JSON response containing raw HTML results.  
  - Edge cases: API rate limits, auth errors, timeouts (timeout set to 120s), malformed responses.

- **Parse & Normalize Data**  
  - Type: Code node (JavaScript).  
  - Role: Parses the raw Decodo response HTML, normalizes it into structured lead objects with fields like business name, address, phone, coordinates, website, opening hours, and URLs.  
  - Key logic:  
    - Uses regex and string extraction helpers to parse HTML chunks by business listing.  
    - Handles missing or malformed data gracefully, skipping invalid entries.  
    - Throws error if no valid leads extracted.  
  - Inputs: JSON from "Decodo Maps Scraper".  
  - Outputs: Array of normalized lead JSON items.  
  - Edge cases: Parsing failures, empty or missing results arrays, changed HTML structure breaking extraction.

- **Split Into Batches**  
  - Type: SplitInBatches node.  
  - Role: Divides lead list into smaller batches for subsequent AI enrichment to avoid timeouts and API limits.  
  - Configuration: Default batch size (not explicitly set here, so defaults apply).  
  - Inputs: List of normalized leads.  
  - Outputs: Batches of lead items.  
  - Edge cases: Batch size too large may cause timeout; too small may increase runtime.

---

#### 1.2 Lead Enrichment with AI

**Overview:**  
Processes batches of leads through an AI language model (Google Gemini 2.5 Flash) to generate detailed enrichment data including value propositions, pain points, outreach hooks, lead scores, and engagement strategies.

**Nodes Involved:**  
- Lead Enrichment (Chain LLM)  
- 2.5 Flash (Google Gemini AI Node)  
- Result Parser  

**Node Details:**

- **Lead Enrichment**  
  - Type: Chain LLM (Language Model Chain) node.  
  - Role: Sends structured business data as prompt to AI for lead enrichment analysis.  
  - Configuration:  
    - Prompt includes detailed instructions to generate JSON with value proposition, three pain points, outreach hook, lead score (1-10), engagement strategy, and reasoning.  
    - Uses dynamic expressions to insert business data from current lead item.  
  - Inputs: Single lead item per batch.  
  - Outputs: Raw AI JSON response.  
  - Edge cases: AI response format errors, missing data causing incomplete output, API call failures.

- **2.5 Flash**  
  - Type: AI language model node (Google Gemini 2.5 Flash).  
  - Role: Executes the AI prompt from "Lead Enrichment" node.  
  - Configuration: Requires Google Palm API credentials.  
  - Inputs: Receives prompt text from "Lead Enrichment".  
  - Outputs: AI-generated text output.  
  - Edge cases: API quota limits, authentication failures, latency.

- **Result Parser**  
  - Type: Langchain structured output parser node.  
  - Role: Parses AI output text into structured JSON based on expected schema.  
  - Inputs: AI text output from "2.5 Flash".  
  - Outputs: Parsed JSON with enrichment fields.  
  - Edge cases: Parsing errors if AI output is malformed; fallback not defined.

---

#### 1.3 Data Management & Filtering

**Overview:**  
Merges the original lead data with AI enrichment results, saves all leads to Google Sheets, filters for hot leads (score ≥7 with contact info), and routes hot leads for outreach preparation.

**Nodes Involved:**  
- Merge Enrichment Data  
- Save to Google Sheets  
- Filter Hot Leads  

**Node Details:**

- **Merge Enrichment Data**  
  - Type: Set node.  
  - Role: Combines original lead fields with AI enrichment outputs into a unified JSON object per lead.  
  - Configuration: Maps fields from batch item and AI output to new properties including id, businessName, category, valueProposition, painPoints (first item only), outreachHook, leadScore (number), engagementStrategy, reasoning, address, phone, website, rating, reviewCount, googleMapsUrl, scrapedAt, and sets statusOutreach as "not send".  
  - Inputs: AI enrichment output and original lead batch item.  
  - Outputs: Enriched lead objects ready to save.  
  - Edge cases: Missing AI output fields, data type mismatches.

- **Save to Google Sheets**  
  - Type: Google Sheets node.  
  - Role: Saves enriched leads to a Google Sheets document, appending or updating rows to avoid duplicates.  
  - Configuration:  
    - Document ID and Sheet ID configured to a specific spreadsheet (Google Maps Outreach).  
    - Uses `id` as matching column for update/append.  
    - Saves multiple fields including leadScore, outreachHook, engagementStrategy, timestamps, etc.  
    - Uses OAuth2 credentials for authentication.  
  - Inputs: Enriched lead objects.  
  - Outputs: Lead records saved confirmation.  
  - Edge cases: Auth failure, API rate limits, spreadsheet access issues.

- **Filter Hot Leads**  
  - Type: If node.  
  - Role: Filters leads with leadScore ≥ 7 and at least one contact method (phone or website) for immediate outreach.  
  - Configuration:  
    - Condition 1: leadScore (number) ≥ 7  
    - Condition 2: phone or website field not empty  
  - Inputs: Saved leads from Google Sheets node.  
  - Outputs:  
    - True branch: Leads that meet criteria.  
    - False branch: Leads that do not qualify as hot leads.  
  - Edge cases: Missing or malformed leadScore, empty contact fields.

---

#### 1.4 Outreach Message Preparation & Saving

**Overview:**  
Prepares personalized outreach messages using AI-generated hooks, then saves these messages to Google Sheets marking the leads as "HOT" for follow-up.

**Nodes Involved:**  
- Prepare Outreach Message  
- Save Outreach To Sheets  

**Node Details:**

- **Prepare Outreach Message**  
  - Type: Set node.  
  - Role: Constructs email subject and body using enriched lead data for personalized outreach.  
  - Configuration:  
    - Subject example: "Quick question about [BusinessName]"  
    - Body template includes outreach hook, value proposition, and a closing call to action.  
    - Sets recipient email to lead's email if available, else placeholder "MANUAL_LOOKUP_NEEDED".  
  - Inputs: Hot leads from Filter Hot Leads node.  
  - Outputs: Ready-to-send outreach message objects.  
  - Edge cases: Missing email addresses, incomplete hooks or propositions.

- **Save Outreach To Sheets**  
  - Type: Google Sheets node.  
  - Role: Saves the outreach messages and marks lead category as "HOT" in the same Google Sheets document.  
  - Configuration:  
    - Uses appendOrUpdate operation keyed on lead `id`.  
    - Saves enrichedAt timestamp, leadsCategory as "HOT", and full outreach message (subject + body).  
    - Uses OAuth2 credentials.  
  - Inputs: Outreach message objects.  
  - Outputs: Confirmation of saved outreach data.  
  - Edge cases: Authentication issues, spreadsheet write errors.

---

#### 1.5 Error Handling

**Overview:**  
Listens for any errors occurring in the workflow, formats a detailed error message, and sends a Telegram notification to alert administrators.

**Nodes Involved:**  
- Error Handler (Error Trigger)  
- Format Error Message (Code)  
- Send Error Notification (Telegram)  

**Node Details:**

- **Error Handler**  
  - Type: Error Trigger node.  
  - Role: Captures any workflow execution errors.  
  - Outputs: Sends error data to "Format Error Message".  
  - Edge cases: Only triggers on failures.

- **Format Error Message**  
  - Type: Code node (JavaScript).  
  - Role: Extracts and formats key error details from the error trigger input to a concise human-readable message.  
  - Details include workflow name, node name, error message, description, HTTP code, timestamp, and execution ID.  
  - Output: JSON object with formatted error message.  
  - Edge cases: Missing error fields, unexpected error structure.

- **Send Error Notification**  
  - Type: Telegram node.  
  - Role: Sends the formatted error message to a Telegram chat for immediate admin notification.  
  - Configuration: Requires Telegram bot token and chat ID.  
  - Edge cases: Telegram API issues, incorrect chat ID, message delivery failures.

---

### 3. Summary Table

| Node Name                | Node Type                         | Functional Role                                   | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                     |
|--------------------------|----------------------------------|-------------------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| Manual Trigger           | Manual Trigger                   | Starts workflow manually                         |                             | Set Search Parameters        | ## Input Configuration<br>Set search query, target country, language, and results limit.       |
| Set Search Parameters    | Set                             | Defines search query and parameters              | Manual Trigger              | Decodo Maps Scraper          | ## Input Configuration<br>Set search query, target country, language, and results limit.       |
| Decodo Maps Scraper      | HTTP Request                    | Scrapes Google Maps via Decodo API               | Set Search Parameters       | Parse & Normalize Data       | ## Create Decodo Credentials<br>Get API key at https://dashboard.decodo.com/web-scraping-api/  |
| Parse & Normalize Data   | Code                            | Parses raw HTML response into structured leads   | Decodo Maps Scraper         | Split Into Batches           | ## Data Collection<br>Scrapes Google Maps via Decodo API, then parses HTML into structured data|
| Split Into Batches       | SplitInBatches                  | Splits leads into batches for processing         | Parse & Normalize Data      | Lead Enrichment (batch 2) & Lead Enrichment (batch 1) | ## AI Enrichment Loop<br>Processes leads in batches. Gemini 2.5 Flash analyzes each business.     |
| Lead Enrichment          | Chain LLM                      | Generates AI enrichment data per lead            | Split Into Batches          | Merge Enrichment Data        | ## AI Enrichment Loop<br>Processes leads in batches. Gemini 2.5 Flash analyzes each business.     |
| 2.5 Flash                | AI Language Model (Google Gemini) | Executes AI prompt for enrichment                | Lead Enrichment             | Result Parser                | ## AI Enrichment Loop<br>Processes leads in batches. Gemini 2.5 Flash analyzes each business.     |
| Result Parser            | Output Parser (Langchain)       | Parses AI output into structured JSON             | 2.5 Flash                  | Lead Enrichment              | ## AI Enrichment Loop<br>Processes leads in batches. Gemini 2.5 Flash analyzes each business.     |
| Merge Enrichment Data    | Set                             | Merges AI data with original lead info            | Lead Enrichment             | Save to Google Sheets        | ## Hot Leads Processing<br>Saves all enriched leads to Sheets.                                  |
| Save to Google Sheets    | Google Sheets                   | Saves enriched leads, appending or updating rows | Merge Enrichment Data       | Filter Hot Leads             | ## Hot Leads Processing<br>Saves all enriched leads to Sheets.                                  |
| Filter Hot Leads         | If                              | Filters leads for high score and contact info     | Save to Google Sheets       | Prepare Outreach Message (true) / Split Into Batches (false) | ## Hot Leads Processing<br>Filters high-quality prospects (score ≥7 with contact info).           |
| Prepare Outreach Message | Set                             | Creates personalized outreach email message       | Filter Hot Leads (true)     | Save Outreach To Sheets      | ## Hot Leads Processing<br>Prepares personalized outreach messages.                            |
| Save Outreach To Sheets  | Google Sheets                   | Saves personalized outreach messages              | Prepare Outreach Message    | Split Into Batches           | ## Hot Leads Processing<br>Saves outreach messages and marks leads as HOT.                     |
| Error Handler            | Error Trigger                   | Triggers on workflow errors                        |                             | Format Error Message         | ## Error Handling<br>Catches workflow failures and sends Telegram alerts.                      |
| Format Error Message     | Code                            | Formats detailed error notification message        | Error Handler              | Send Error Notification      | ## Error Handling<br>Catches workflow failures and sends Telegram alerts.                      |
| Send Error Notification  | Telegram                       | Sends error message to Telegram admins             | Format Error Message        |                             | ## Error Handling<br>Catches workflow failures and sends Telegram alerts.                      |
| Sticky Note (various)    | Sticky Note                    | Provides documentation and instructions            |                             |                             | Contains setup instructions, explanations for blocks, and helpful links.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Purpose: Start the workflow manually.  
   - No parameters needed.

2. **Create Set node "Set Search Parameters"**  
   - Add variables:  
     - `searchQuery` (string): e.g. "YOUR-QUERY_HERE"  
     - `targetLanguage` (string): "en"  
     - `country` (string): "Country Name"  
     - `resultsLimit` (number): 5 (start small for testing)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request node "Decodo Maps Scraper"**  
   - Method: POST  
   - URL: https://scraper-api.decodo.com/v2/scrape  
   - Authentication: HTTP Header Auth  
     - Header Name: Authorization  
     - Header Value: Basic YOUR_DECODO_API_KEY (replace with your key)  
   - Headers: Accept: application/json, Content-Type: application/json  
   - Body parameters (JSON):  
     - target: "google_maps"  
     - query: `={{ $json.searchQuery }}`  
     - page_from: 1  
     - page_to: `={{ Math.ceil(parseInt($json.resultsLimit) / 10) }}`  
     - headless: "html"  
     - google_results_language: `={{ $json.targetLanguage }}`  
     - geo: `={{ $json.country }}`  
   - Timeout: 120000 ms  
   - Connect output of "Set Search Parameters" to this node.

4. **Create Code node "Parse & Normalize Data"**  
   - Paste JavaScript code to parse Decodo HTML response into structured leads.  
   - Input: JSON from "Decodo Maps Scraper".  
   - Output: Array of normalized lead objects with fields like businessName, address, phone, website, lat, lng, rating, reviewCount, googleMapsUrl, etc.  
   - Connect output of "Decodo Maps Scraper" to this node.

5. **Create SplitInBatches node "Split Into Batches"**  
   - Purpose: Split leads into manageable batches for AI processing.  
   - Default batch size or configure as needed (e.g., 5).  
   - Connect output of "Parse & Normalize Data" to this node.

6. **Create Chain LLM node "Lead Enrichment"**  
   - Set prompt text including business data placeholders (e.g., {{ $json.businessName }}) and detailed instructions for AI to generate value proposition, pain points, hooks, score, and strategy.  
   - Enable output parser with JSON schema matching AI output.  
   - Connect the **second** output of "Split Into Batches" (batch items) to this node.

7. **Create AI Language Model node "2.5 Flash"**  
   - Use Google Gemini 2.5 Flash model.  
   - Configure with Google Palm API credentials.  
   - Connect output of "Lead Enrichment" node (prompt) to this node.

8. **Create Output Parser node "Result Parser"**  
   - Set JSON schema example matching expected AI output.  
   - Connect output of "2.5 Flash" to this node.

9. **Connect output of "Result Parser" back to "Lead Enrichment" node's AI output parser input**  
   - Enables structured JSON output from AI.

10. **Create Set node "Merge Enrichment Data"**  
    - Map fields combining original lead data (from batch) and AI enrichment results into a single JSON.  
    - Include id, businessName, category, valueProposition, painPoints[0], outreachHook, leadScore (number), engagementStrategy, reasoning, address, phone, website, rating, reviewCount, googleMapsUrl, scrapedAt (formatted), and add statusOutreach = "not send".  
    - Connect output of "Lead Enrichment" (structured JSON) to this node.

11. **Create Google Sheets node "Save to Google Sheets"**  
    - Operation: appendOrUpdate  
    - Document ID: your Google Sheets document ID for storing leads  
    - Sheet Name: specify sheet by gid or name  
    - Matching column: "id"  
    - Map all relevant columns from merged data, including leadScore, outreachHook, engagementStrategy, timestamps, etc.  
    - Use OAuth2 credentials with appropriate scopes.  
    - Connect output of "Merge Enrichment Data" to this node.

12. **Create If node "Filter Hot Leads"**  
    - Condition 1: leadScore (number) >= 7  
    - Condition 2: phone or website is not empty  
    - Connect output of "Save to Google Sheets" to this node.

13. **Create Set node "Prepare Outreach Message"**  
    - Create personalized email subject and body using AI-generated hooks and propositions.  
    - Set recipient email (if available) or placeholder "MANUAL_LOOKUP_NEEDED".  
    - Include leadScore and id fields.  
    - Connect True output of "Filter Hot Leads" to this node.

14. **Create Google Sheets node "Save Outreach To Sheets"**  
    - Operation: appendOrUpdate  
    - Document ID and Sheet Name: same as "Save to Google Sheets"  
    - Matching column: "id"  
    - Map columns: enrichedAt timestamp, leadsCategory = "HOT", outreachMessage (subject + body), status.  
    - Use OAuth2 credentials.  
    - Connect output of "Prepare Outreach Message" to this node.

15. **Connect False output of "Filter Hot Leads" back to "Split Into Batches" node**  
    - Enables reprocessing or handling of leads that do not qualify as hot.

16. **Create Error Trigger node "Error Handler"**  
    - Connect as error trigger to entire workflow.

17. **Create Code node "Format Error Message"**  
    - JavaScript code to extract error details from error trigger input and format a concise message with workflow name, node, error message, details, timestamp, and execution ID.

18. **Create Telegram node "Send Error Notification"**  
    - Set chat ID and use Telegram bot credentials.  
    - Use HTML parse mode.  
    - Connect output of "Format Error Message" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Decodo API key required: create credentials with HTTP Header Auth using your API key as Authorization header (Basic YOUR_KEY).                         | https://dashboard.decodo.com/web-scraping-api/scraper?target=google_maps                          |
| Google Gemini 2.5 Flash API key must be configured in the "2.5 Flash" node for AI enrichment.                                                           | API from ai.dev or Google Cloud Palm API                                                        |
| For error notifications, configure a Telegram Bot and set your chat ID in "Send Error Notification" node.                                              | Telegram Bot API                                                                                 |
| Workflow includes detailed sticky notes explaining each block and setup steps for clarity.                                                             | Visible within n8n workflow canvas                                                               |
| Use small `resultsLimit` values (e.g., 5) during testing to avoid rate limits and timeouts.                                                            | Adjust in "Set Search Parameters" node                                                          |
| The AI enrichment prompt is carefully designed to produce actionable, specific, and personalized B2B sales insights rather than generic advice.         | See "Lead Enrichment" node prompt for details                                                   |
| The "Parse & Normalize Data" code node is critical and may require maintenance if Decodo changes HTML structure.                                        | Watch for extraction failures or empty lead lists                                               |
| Google Sheets nodes require OAuth2 credentials with write access to the target spreadsheet.                                                             | Ensure proper authentication and spreadsheet permissions                                        |
| Lead scoring is based on multiple weighted criteria including reputation, social proof, digital maturity, contact availability, and engagement.        | See prompt instructions in "Lead Enrichment" node                                              |
| Outreach message templates are customizable and designed for cold outreach campaigns with a conversational and non-pushy tone.                         | Modify "Prepare Outreach Message" node for personalization                                      |

---

**Disclaimer:** The provided text and workflow are solely from an automated n8n workflow created with n8n integration and automation tool. The process strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.