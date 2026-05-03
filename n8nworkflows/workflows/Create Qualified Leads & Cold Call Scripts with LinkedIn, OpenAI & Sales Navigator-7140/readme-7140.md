Create Qualified Leads & Cold Call Scripts with LinkedIn, OpenAI & Sales Navigator

https://n8nworkflows.xyz/workflows/create-qualified-leads---cold-call-scripts-with-linkedin--openai---sales-navigator-7140


# Create Qualified Leads & Cold Call Scripts with LinkedIn, OpenAI & Sales Navigator

### 1. Workflow Overview

This n8n workflow automates the process of generating qualified sales leads and personalized cold call scripts by integrating LinkedIn company data, AI-driven scoring, and Sales Navigator employee insights. It targets sales and marketing professionals aiming to efficiently identify promising companies, verify their details, score their likelihood of interest, find decision-makers, and prepare customized call scripts for outreach.

The workflow is logically divided into three main blocks:

- **1.1 Input & Company Search**  
  Receives input parameters and settings, extracts clean keywords using AI, and queries the Ghost Genius LinkedIn API to search for companies matching the criteria.

- **1.2 Company Validation & Scoring**  
  Processes retrieved companies individually, enriches their data via Ghost Genius API, filters based on follower count and website presence, checks for duplicates in the CRM (Google Sheets), scores companies using OpenAI, and stores qualified companies.

- **1.3 Lead Enrichment & Call Script Generation**  
  Retrieves qualified companies, filters by score and status, fetches potential decision makers from Sales Navigator, enriches profiles with phone numbers, and generates personalized cold call scripts using OpenAI before saving leads to a Google Sheet.


---

### 2. Block-by-Block Analysis

#### 2.1 Input & Company Search

**Overview:**  
This block initiates the workflow. It gathers user-defined settings, processes the input to extract relevant company keywords, and performs paginated LinkedIn company searches via the Ghost Genius API.

**Nodes Involved:**  
- Start  
- Get Settings  
- Aggregate1  
- If (API keys presence check)  
- Make the perfect request (OpenAI keyword extraction)  
- Search Companies (Ghost Genius API)  
- Extract Company Data (splitOut)  
- Sticky Note4 (explanatory note)

**Node Details:**

- **Start**  
  - Manual trigger to start the workflow.  
  - No inputs; outputs to Get Settings.  
  - Potential failure: None (manual trigger).

- **Get Settings**  
  - Google Sheets node fetching user settings from a "Settings" sheet.  
  - Credentials: Google Sheets OAuth2 required.  
  - Pulls parameters such as API keys, account IDs, search filters (location, company size, keywords).  
  - Outputs aggregated settings data.  
  - Failure points: Auth errors, sheet access issues.

- **Aggregate1**  
  - Aggregates all settings data into a single JSON object for easy access downstream.  
  - Outputs to If node.

- **If (API Key Check)**  
  - Verifies presence of necessary Ghost Genius API key and account ID in settings.  
  - Routes flow to keyword extraction or error node.  
  - Failure: Missing API keys result in workflow termination.

- **Make the perfect request (OpenAI)**  
  - Uses OpenAI model "o3-mini" to extract clean, concise LinkedIn company keywords from user input.  
  - System prompt filters out irrelevant info (location, size, intent).  
  - Outputs keywords for company search.

- **Search Companies (Ghost Genius API)**  
  - HTTP Request node querying the Ghost Genius LinkedIn company search endpoint.  
  - Uses extracted keywords, location, company size from settings.  
  - Implements pagination with max pages and intervals to respect API limits.  
  - Outputs raw company data JSON.

- **Extract Company Data**  
  - SplitOut node extracts the "data" array of companies from the search response into individual items.  
  - Outputs each company for further processing.

- **Sticky Note4**  
  - Informational note describing LinkedIn company search logic, API limits, and usage tips.

---

#### 2.2 Company Validation & Scoring

**Overview:**  
Processes each company individually: enriches with detailed info, filters based on follower count and website presence, checks for duplicates in Google Sheets CRM, scores companies for likelihood of interest using OpenAI, and appends qualified companies to the CRM.

**Nodes Involved:**  
- Process Each Company (splitInBatches)  
- Get Company Info (Ghost Genius API)  
- Filter Valid Companies (If)  
- Check If Company Exists (Google Sheets lookup)  
- Is New Company? (If)  
- AI Company Scoring (OpenAI)  
- Add Company to CRM (Google Sheets append)  
- Sticky Note1 (explanatory note)  
- Sticky Note5 (explanatory note)

**Node Details:**

- **Process Each Company**  
  - Splits company list into batches for sequential processing.  
  - Controls flow to Get Company Info or loops back.

- **Get Company Info**  
  - HTTP Request node calling Ghost Genius "Get Company" API by LinkedIn URL to fetch detailed info.  
  - Uses Bearer token from settings.  
  - Retry enabled for robustness.  
  - Outputs enriched company data.

- **Filter Valid Companies**  
  - If node filtering companies to ensure:  
    - Website field is not empty  
    - Followers count > 200 (threshold for credibility)  
  - Filters out unqualified or incomplete companies.

- **Check If Company Exists**  
  - Google Sheets lookup by LinkedIn ID to prevent duplicates in CRM.  
  - Requires Google Sheets OAuth2 credentials.  
  - Outputs existing company data or empty.

- **Is New Company?**  
  - If node checks if lookup returned empty (company not in CRM).  
  - Routes new companies to AI scoring or skips.

- **AI Company Scoring**  
  - OpenAI node using "o3-mini" to score company relevance on a 0-10 scale.  
  - Provides explanation based on industry fit, profile, and pain points.  
  - Inputs include company name, tagline, description, employee count, industry, specialties, location, foundation year.  
  - Outputs JSON with score and explanation.

- **Add Company to CRM**  
  - Google Sheets append operation adding scored and qualified companies to "Companies" sheet.  
  - Fields: ID, Name, Score, State (set to "Qualified"), Summary, Website, LinkedIn, Explanation.  
  - Ensures structured CRM update.

- **Sticky Note1 & Sticky Note5**  
  - Notes explaining company data processing logic, filtering criteria, scoring importance, and Google Sheets storage.

---

#### 2.3 Lead Enrichment & Call Script Generation

**Overview:**  
Retrieves qualified companies from CRM, filters by score and status, limits processing to 100 companies, fetches potential decision makers via Sales Navigator, enriches profiles with phone numbers, uses OpenAI to generate personalized cold call scripts, and saves enriched leads to Google Sheets.

**Nodes Involved:**  
- Schedule Trigger  
- Get Settings1 (Google Sheets)  
- Aggregate  
- If1 (API keys check)  
- Companies Recovery (Google Sheets)  
- Filter Score and State (Filter)  
- Limit (Limit)  
- Loop Over Items (splitInBatches)  
- Find Employees (Ghost Genius Sales Navigator API)  
- Check profiles Found (If)  
- Split Profiles (splitOut)  
- Get Profile details (Ghost Genius API)  
- Get Mobile (Ghost Genius API)  
- Filter (Filter)  
- If2 (If)  
- Keep relevant information (Code)  
- OpenAI (cold call script generation)  
- Google Sheets (Leads append)  
- Lead(s) found (Google Sheets update)  
- No decision maker found / No decision maker found1 (Google Sheets update)  
- Sticky Notes 2, 3, 6, 12, 13 (explanatory and exit notes)

**Node Details:**

- **Schedule Trigger**  
  - Scheduled start node triggering the lead enrichment process periodically.  
  - Runs automatically on defined intervals.

- **Get Settings1**  
  - Reads settings again (e.g., API keys, account IDs) for Sales Navigator queries.

- **Aggregate**  
  - Aggregates settings data for easy use downstream.

- **If1 (API keys check)**  
  - Validates presence of necessary API keys and account IDs before proceeding.

- **Companies Recovery**  
  - Reads companies from "Companies" Google Sheet.  
  - Inputs all stored companies for filtering.

- **Filter Score and State**  
  - Filters companies with Score ≥ 7 and State == "Qualified" to select best prospects.

- **Limit**  
  - Limits the number of companies processed to 100 to avoid exceeding LinkedIn Sales Navigator rate limits.

- **Loop Over Items**  
  - Processes each qualified company individually.

- **Find Employees**  
  - Calls Ghost Genius Sales Navigator API to find employees with specified titles at each company.  
  - Uses account ID and API key from settings.  
  - Supports retries and waits between tries to handle API limits.

- **Check profiles Found**  
  - Checks if any employee profiles were found (total ≥ 1).  
  - Routes to profile processing or "no decision maker" update.

- **Split Profiles**  
  - Splits employee profile list into individual items.

- **Get Profile details**  
  - Fetches detailed profile info for each employee by URL.  
  - Includes batching with delay to respect API limits.

- **Get Mobile**  
  - Fetches phone number for each profile via API call.  
  - Batching and retry enabled.

- **Filter**  
  - Filters profiles to keep only those with successful phone retrieval.

- **If2**  
  - Checks success flag for final filtering.

- **Keep relevant information (Code node)**  
  - Custom JavaScript code extracts and simplifies profile and company info into a structured object for script generation.

- **OpenAI (cold call script generation)**  
  - Uses GPT-4.1 to create a personalized, 20-second cold call script per profile.  
  - Input includes prospect’s name, position, summary, company info, and user’s own info and product details.  
  - Output is a JSON with the generated script.

- **Google Sheets (Leads append)**  
  - Saves enriched lead data including phone, script, name, LinkedIn URL, company, and position into a "Leads" sheet.

- **Lead(s) found**  
  - Updates the company's state to "Enriched" in the "Companies" sheet.

- **No decision maker found / No decision maker found1**  
  - Updates company state to "No decision maker found" when no suitable profiles or phone numbers are retrieved.

- **Sticky Notes 2, 3, 6, 12, 13**  
  - Provide explanations for decision maker search, exit points, and process summaries.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                              | Input Node(s)                  | Output Node(s)                      | Sticky Note                                                                                                          |
|---------------------------|---------------------------------|----------------------------------------------|-------------------------------|------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Start                     | Manual Trigger                  | Workflow entry point                         | -                             | Get Settings                       |                                                                                                                      |
| Get Settings              | Google Sheets                  | Load user settings                           | Start                         | Aggregate1                        |                                                                                                                      |
| Aggregate1                | Aggregate                      | Aggregate settings into one JSON             | Get Settings                  | If                              |                                                                                                                      |
| If                        | If                             | Check API key & account ID presence          | Aggregate1                    | Make the perfect request, StopAndError |                                                                                                                      |
| Make the perfect request   | OpenAI                         | Extract keywords from user input             | If                           | Search Companies                 |                                                                                                                      |
| Search Companies          | HTTP Request                   | Search LinkedIn companies via API            | Make the perfect request       | Extract Company Data             |                                                                                                                      |
| Extract Company Data      | SplitOut                      | Split company list array into individual items | Search Companies             | Process Each Company             |                                                                                                                      |
| Process Each Company      | SplitInBatches                | Process companies one by one                  | Extract Company Data           | Get Company Info, Process Each Company | Sticky Note1                                                                                                         |
| Get Company Info          | HTTP Request                   | Enrich company data                           | Process Each Company           | Filter Valid Companies           |                                                                                                                      |
| Filter Valid Companies    | If                             | Filter by website presence & followers count | Get Company Info              | Check If Company Exists, Process Each Company |                                                                                                                      |
| Check If Company Exists   | Google Sheets                  | Check for duplicates in CRM                   | Filter Valid Companies         | Is New Company                  |                                                                                                                      |
| Is New Company?           | If                             | Determine if company is new                   | Check If Company Exists        | AI Company Scoring, Process Each Company |                                                                                                                      |
| AI Company Scoring         | OpenAI                         | Score company relevance                       | Is New Company?                | Add Company to CRM              | Sticky Note5                                                                                                         |
| Add Company to CRM        | Google Sheets                  | Append qualified company to CRM               | AI Company Scoring             | Process Each Company             |                                                                                                                      |
| Schedule Trigger          | Schedule Trigger               | Scheduled start for lead enrichment           | -                             | Get Settings1                  |                                                                                                                      |
| Get Settings1             | Google Sheets                  | Load settings for lead enrichment             | Schedule Trigger              | Aggregate                       |                                                                                                                      |
| Aggregate                 | Aggregate                      | Aggregate settings into one JSON              | Get Settings1                 | If1                            |                                                                                                                      |
| If1                       | If                             | Check API keys for lead enrichment            | Aggregate                     | Companies Recovery, StopAndError |                                                                                                                      |
| Companies Recovery        | Google Sheets                  | Retrieve companies from CRM                    | If1                          | Filter Score and State          | Sticky Note (Data recovery)                                                                                          |
| Filter Score and State    | Filter                         | Filter companies by score ≥7 and status       | Companies Recovery            | Limit                          |                                                                                                                      |
| Limit                     | Limit                          | Limit max companies processed to 100          | Filter Score and State        | Loop Over Items                |                                                                                                                      |
| Loop Over Items           | SplitInBatches                | Process each company individually              | Limit                        | Find Employees                 |                                                                                                                      |
| Find Employees            | HTTP Request                   | Search Sales Navigator for decision makers    | Loop Over Items              | Check profiles Found           | Sticky Note2                                                                                                         |
| Check profiles Found      | If                             | Check if decision makers found                 | Find Employees               | Split Profiles, No decision maker found |                                                                                                                      |
| Split Profiles            | SplitOut                      | Split employee profiles into individual items | Check profiles Found         | Get Profile details           |                                                                                                                      |
| Get Profile details       | HTTP Request                   | Retrieve detailed profile info                  | Split Profiles               | Get Mobile                    |                                                                                                                      |
| Get Mobile                | HTTP Request                   | Retrieve phone number for profile               | Get Profile details          | Filter                        |                                                                                                                      |
| Filter                    | Filter                         | Keep only profiles with phone number           | Get Mobile                   | If2                          |                                                                                                                      |
| If2                       | If                             | Confirm success of phone retrieval              | Filter                       | Keep relevant information, No decision maker found1 |                                                                                                                      |
| Keep relevant information | Code                          | Simplify profile and company data               | If2                         | OpenAI                       |                                                                                                                      |
| OpenAI                   | OpenAI                         | Generate personalized cold call script          | Keep relevant information    | Google Sheets                 |                                                                                                                      |
| Google Sheets             | Google Sheets                  | Save enriched leads with scripts                 | OpenAI                      | Lead(s) found                |                                                                                                                      |
| Lead(s) found             | Google Sheets                  | Update company state to "Enriched"               | Google Sheets               | Loop Over Items               |                                                                                                                      |
| No decision maker found   | Google Sheets                  | Update company state if no decision maker found | Check profiles Found         | Loop Over Items               |                                                                                                                      |
| No decision maker found1  | Google Sheets                  | Update company state if no phone found           | If2                         | Loop Over Items               |                                                                                                                      |
| Missing API Key or Account ID | Stop and Error            | Stop workflow on missing API key                 | If                          | -                           |                                                                                                                      |
| Missing API Key or Account ID1 | Stop and Error          | Stop workflow on missing API key for lead enrichment | If1                      | -                           |                                                                                                                      |
| Sticky Note4             | Sticky Note                   | Explanation of LinkedIn company search           | -                           | -                           | LinkedIn company search: explains API limits and search tips                                                        |
| Sticky Note1             | Sticky Note                   | Explanation of company processing                 | -                           | -                           | Company data processing, filtering, and duplicate checking                                                          |
| Sticky Note5             | Sticky Note                   | Explanation of AI scoring and CRM storage         | -                           | -                           | AI scoring importance, CRM integration                                                                             |
| Sticky Note              | Sticky Note                   | Data recovery explanation                           | -                           | -                           | Explains data recovery and API limits                                                                               |
| Sticky Note2             | Sticky Note                   | Explanation of decision maker search                | -                           | -                           |                                                                                                                      |
| Sticky Note3             | Sticky Note                   | Exit point                                           | -                           | -                           |                                                                                                                      |
| Sticky Note6             | Sticky Note                   | Exit point                                           | -                           | -                           |                                                                                                                      |
| Sticky Note12            | Sticky Note                   | Exit point                                           | -                           | -                           |                                                                                                                      |
| Sticky Note13            | Sticky Note                   | Exit point                                           | -                           | -                           |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "Start" to initiate the workflow manually.

2. **Add a Google Sheets node** named "Get Settings" configured to read the "Settings" sheet from your Google Sheets document:  
   - Document ID: Your Google Sheet ID  
   - Sheet Name or GID: The sheet containing API keys and parameters  
   - Authentication: Google Sheets OAuth2 credentials  
   - Set to execute once (fetch all settings at start).

3. **Add an Aggregate node** named "Aggregate1" to combine all settings rows into a single JSON object.

4. **Add an If node** named "If" to check if the API Key and Account ID fields from the aggregated settings are non-empty:  
   - Condition 1: settings[6]['Value (edit with your use case)'] != "" (API Key)  
   - Condition 2: settings[2]['Value (edit with your use case)'] != "" (Account ID)  
   - If true: proceed, else stop workflow with an error node "Missing API Key or Account ID".

5. **Add an OpenAI node** named "Make the perfect request" with model "o3-mini":  
   - System prompt instructs AI to extract keywords related to company sector and activity only.  
   - Input prompt uses user raw input from settings.  
   - Output is keywords string for searching companies.

6. **Add an HTTP Request node** named "Search Companies":  
   - URL: Ghost Genius LinkedIn Search API endpoint  
   - Query parameters: keywords from OpenAI output, location and company size from settings  
   - Headers: Authorization Bearer token from settings  
   - Enable pagination to fetch multiple pages with max 100 pages and 2s delay  
   - Output: JSON list of companies.

7. **Add a SplitOut node** named "Extract Company Data" to split the companies array from the search response.

8. **Add a SplitInBatches node** named "Process Each Company" to process companies one at a time.

9. **Add an HTTP Request node** named "Get Company Info" to call Ghost Genius "Get Company" API with each company's LinkedIn URL:  
   - Use Bearer token for authorization  
   - Enable retry on failure.

10. **Add an If node** named "Filter Valid Companies" to filter companies:  
    - Website field is not empty  
    - Followers count > 200.

11. **Add a Google Sheets node** named "Check If Company Exists" to lookup companies in the CRM by LinkedIn ID:  
    - Document and sheet same as settings document but different sheet for "Companies"  
    - Lookup on "ID" column.

12. **Add an If node** named "Is New Company?" to check if lookup returned empty (company not present).

13. **Add an OpenAI node** named "AI Company Scoring" with model "o3-mini":  
    - System prompt instructs scoring companies 0-10 based on industry fit, size, pain points  
    - Input includes company data from previous nodes.

14. **Add a Google Sheets node** named "Add Company to CRM" to append scored companies:  
    - Columns include ID, Name, Score, State="Qualified", Summary, Website, LinkedIn, Explanation.

15. **Connect "Add Company to CRM" back to "Process Each Company"** for batch iteration.

16. **Create a Schedule Trigger node** named "Schedule Trigger" to run lead enrichment periodically.

17. **Add a Google Sheets node** named "Get Settings1" to fetch settings again for lead enrichment.

18. **Add an Aggregate node** named "Aggregate" to combine lead enrichment settings.

19. **Add an If node** named "If1" to verify API key and account ID presence for lead enrichment.

20. **Add a Google Sheets node** named "Companies Recovery" to read from the "Companies" sheet.

21. **Add a Filter node** named "Filter Score and State" to keep companies with Score ≥ 7 and State = "Qualified".

22. **Add a Limit node** named "Limit" to restrict processing to 100 companies to avoid Sales Navigator limits.

23. **Add a SplitInBatches node** named "Loop Over Items" to process companies individually.

24. **Add an HTTP Request node** named "Find Employees" to query Sales Navigator API for decision makers:  
    - Parameters: current_company ID, account_id, current_title from settings  
    - Use Bearer token authorization  
    - Enable retry and wait between tries for API limits.

25. **Add an If node** named "Check profiles Found" to check if any employees found (total ≥ 1).

26. **Add a SplitOut node** named "Split Profiles" to split employee profiles list.

27. **Add an HTTP Request node** named "Get Profile details" for detailed profile data:  
    - Batched with delay  
    - Uses Bearer token.

28. **Add an HTTP Request node** named "Get Mobile" to retrieve phone numbers for profiles:  
    - Batched with delay  
    - Uses Bearer token.

29. **Add a Filter node** named "Filter" to keep only profiles with successful phone retrieval.

30. **Add an If node** named "If2" to verify phone retrieval success.

31. **Add a Code node** named "Keep relevant information" to extract and simplify profile and company data for script generation.

32. **Add an OpenAI node** named "OpenAI" with model "gpt-4.1" to generate personalized 20-second cold call scripts:  
    - Uses prospect and company info, plus user’s product details.

33. **Add a Google Sheets node** named "Google Sheets" to append enriched leads including phone, script, LinkedIn, etc., to "Leads" sheet.

34. **Add a Google Sheets node** named "Lead(s) found" to update company state to "Enriched" in "Companies" sheet.

35. **Add Google Sheets nodes** named "No decision maker found" and "No decision maker found1" to update company state when no decision makers or phones found.

36. **Add Stop and Error nodes** to handle missing API keys or account IDs at respective checks.

37. **Add Sticky Note nodes** with descriptive comments at relevant points for documentation and user guidance.

38. **Wire nodes according to the connections described in Section 3, ensuring error handling and looping as specified.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Use Ghost Genius API for LinkedIn and Sales Navigator data to avoid scraping risks and enjoy cookieless access with API limits management.                                    | https://ghostgenius.fr                                                                                                           |
| Recommended Google Sheet template for settings, companies, and leads available to be copied and customized.                                                                   | https://docs.google.com/spreadsheets/d/1o1AHiPiHEXVOkUhO2ms-lw1Ygu1eWIWW-8Qe1OoHpCo/edit?usp=sharing                             |
| AI scoring system is highly customizable; test with multiple companies to adjust system prompt and scoring thresholds for best results.                                       |                                                                                                                                |
| Sales Navigator daily limit is 2,500 search results; processing capped at 100 companies per day to stay within limits.                                                        |                                                                                                                                |
| Cold call scripts generated by GPT-4.1 are personalized, casual, and include a meeting proposal with two options, increasing appointment booking rates.                         |                                                                                                                                |
| Workflow uses batching, retries, and wait intervals extensively to respect API rate limits and ensure reliability.                                                           |                                                                                                                                |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.