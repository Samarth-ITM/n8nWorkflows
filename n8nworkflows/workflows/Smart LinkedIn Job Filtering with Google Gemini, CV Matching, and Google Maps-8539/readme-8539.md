Smart LinkedIn Job Filtering with Google Gemini, CV Matching, and Google Maps

https://n8nworkflows.xyz/workflows/smart-linkedin-job-filtering-with-google-gemini--cv-matching--and-google-maps-8539


# Smart LinkedIn Job Filtering with Google Gemini, CV Matching, and Google Maps

### 1. Workflow Overview

This workflow automates the process of filtering LinkedIn job postings to identify those that best match a candidate’s CV and preferences. It leverages advanced AI models (Google Gemini) for semantic analysis, CV-to-job matching, and geolocation-based commute filtering using Google Maps. The workflow scrapes job listings, applies multi-layer AI-driven validation and scoring, and then saves qualified jobs to a database while sending alerts for the highest matches.

**Target use cases:**  
- Job seekers who want personalized job recommendations matched to their CV and preferences.  
- Automated job market monitoring with intelligent filtering on seniority, language, commute, and applicant volume.  
- Integration of AI analysis with data enrichment (commute time) and notification systems for real-time alerts.

**Logical blocks:**  
- **1.1 Configuration Setup**: Central parameter definition including CV text, job search keywords, location, and filtering criteria.  
- **1.2 Job Data Acquisition**: Scraping LinkedIn jobs via an API with configured filters.  
- **1.3 Initial AI Triage (Seniority & Language Validation)**: Fast AI parsing to confirm job seniority and language relevance.  
- **1.4 Geolocation Filtering**: Commute time calculation and filtering for onsite/hybrid jobs.  
- **1.5 Deep AI Analysis (CV Matching)**: Detailed AI comparison between candidate CV and job description to generate scores, summaries, and reasoning.  
- **1.6 Data Storage and Notification**: Save filtered results to Supabase DB and send Telegram alerts for top matches.  
- **1.7 Workflow Control & Looping**: Batch processing of multiple job items and trigger management.

---

### 2. Block-by-Block Analysis

#### 1.1 Configuration Setup

- **Overview:** Defines all user-specific parameters for the workflow, such as CV content, job search keywords, experience level, location, commute constraints, language preferences, and API actor ID for scraping.  
- **Nodes Involved:**  
  - `Config` (Set node)  
  - `Sticky Note` (Documentation)  

- **Node Details:**  
  - **Config**  
    - Type: Set  
    - Role: Central configuration storage for workflow parameters.  
    - Configurations:  
      - `MyCV`: Full text of user's CV for AI matching.  
      - `JobKeywords`: Keywords to search jobs (e.g., "AI Integration").  
      - `ExperienceLevel`: Target seniority (e.g., "mid_senior").  
      - `JobsToScrape`: Max job postings to fetch (e.g., 20).  
      - `HomeLocation`: Candidate’s home city for commute calculations.  
      - `MaxCommuteMinutes`: Max acceptable one-way commute.  
      - `TargetLanguage`: Preferred job posting language(s).  
      - `Under10Applicants`: Boolean to filter jobs with fewer applicants.  
      - `ActorId`: API identifier for LinkedIn scraping service.  
    - Input/Output: No input; output feeds into job scraping node.  
    - Edge Cases: Missing or malformed CV or keywords may reduce AI matching quality. Incorrect location format affects commute filtering.  
    - Notes: This node is the single source of truth for all runtime parameters.

  - **Sticky Note**  
    - Provides detailed explanations of configuration parameters for user clarity.

---

#### 1.2 Job Data Acquisition

- **Overview:** Fetches job postings from LinkedIn via the Apify API based on parameters from the `Config` node.  
- **Nodes Involved:**  
  - `Scrape LinkenIn Jobs` (HTTP Request)  
  - `Items Loop` (Split In Batches)  

- **Node Details:**  
  - **Scrape LinkenIn Jobs**  
    - Type: HTTP Request  
    - Role: Calls LinkedIn Jobs Scraper API using bearer authentication.  
    - Configuration:  
      - POST request to Apify API endpoint with JSON body containing filters: date_posted="day", experienceLevel (from Config), keywords, limit, under_10_applicants flag, sort by relevance, location fixed to "91000000" (likely a location code), page_number=1.  
      - Authentication via HTTP Bearer token (credential stored in n8n).  
    - Inputs: Receives Config data.  
    - Outputs: JSON array of job items.  
    - Edge Cases: API rate limits, invalid credentials, network timeouts, or empty results.  
  - **Items Loop**  
    - Type: Split In Batches  
    - Role: Processes each job posting individually for downstream logic.  
    - Configuration: Default batch size (assumed 1 by default).  
    - Inputs: Job postings array.  
    - Outputs: Single job item per iteration.  
    - Edge Cases: Empty input array results in no processing.

---

#### 1.3 Initial AI Triage (Seniority & Language Validation)

- **Overview:** Uses AI to validate the job’s seniority level and language to quickly filter out irrelevant jobs before deeper analysis.  
- **Nodes Involved:**  
  - `Deconstruct Job Data` (Langchain Agent)  
  - `Structured Output Parser` (Langchain Output Parser)  
  - `If Seniority Match` (If node)  
  - `If Language Match` (If node)  

- **Node Details:**  
  - **Deconstruct Job Data**  
    - Type: Langchain Agent  
    - Role: Analyzes job description text to confirm seniority and language.  
    - Configuration:  
      - Prompt instructs the AI to validate seniority against Config’s ExperienceLevel and extract primary language.  
      - Output structured as JSON with fields: jobLanguage, confirmed_seniority, is_match.  
      - Input data: job description from current batch item.  
    - Inputs: Job item from split batch.  
    - Outputs: AI JSON output.  
    - Edge Cases: Ambiguous job descriptions, inconsistent seniority tags, language detection errors.  
  - **Structured Output Parser**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses AI raw output into structured JSON.  
    - Configuration: Example JSON schema provided for validation.  
    - Inputs: Raw AI output from `Deconstruct Job Data`.  
    - Outputs: Parsed JSON object.  
    - Edge Cases: Parsing failures due to malformed AI output.  
  - **If Seniority Match**  
    - Type: If  
    - Role: Checks if AI confirmed seniority exactly matches Config ExperienceLevel.  
    - Condition: `$json.output.confirmed_seniority == Config.ExperienceLevel` (exact string match).  
    - Inputs: Parsed AI output.  
    - Outputs: True branch proceeds, false branch discards or skips item.  
  - **If Language Match**  
    - Type: If  
    - Role: Checks if the detected job language is included in Config TargetLanguage (case-insensitive, supports multiple comma-separated languages).  
    - Condition: `Config.TargetLanguage.toLowerCase().split(', ').includes(jobLanguage.toLowerCase())`.  
    - Inputs: Parsed AI output.  
    - Outputs: True branch continues, false branch skips.

---

#### 1.4 Geolocation Filtering

- **Overview:** For non-remote jobs, calculates commute time from user’s home location to job location using Google Maps Directions API and filters jobs exceeding max commute time.  
- **Nodes Involved:**  
  - `If is not Remote` (If node)  
  - `Get Commute Time` (HTTP Request)  
  - `Is Commute Acceptable` (If node)  

- **Node Details:**  
  - **If is not Remote**  
    - Type: If  
    - Role: Checks if job’s work_type field is not “remote” (case-insensitive).  
    - Inputs: Current job item.  
    - Outputs: True branch triggers commute time calculation; False branch skips to deep AI analysis (remote jobs do not require commute check).  
  - **Get Commute Time**  
    - Type: HTTP Request  
    - Role: Calls Google Maps Directions API with origin=HomeLocation and destination=job location.  
    - Configurations:  
      - Authenticated with Google Directions API key.  
      - Query parameters set dynamically from Config and job data.  
    - Inputs: Job item details.  
    - Outputs: JSON response with routes, legs, and duration.  
    - Edge Cases: Invalid location strings, API quota exceeded, no routes found.  
  - **Is Commute Acceptable**  
    - Type: If  
    - Role: Checks if commute duration (in seconds) is less than or equal to MaxCommuteMinutes * 60.  
    - Condition: `routes[0].legs[0].duration.value <= MaxCommuteMinutes * 60`  
    - Inputs: Google Maps response.  
    - Outputs: True branch proceeds to deep AI analysis; False branch skips job.

---

#### 1.5 Deep AI Analysis (CV Matching)

- **Overview:** Performs a detailed semantic analysis comparing candidate CV to the job description, generating a summary, key skills, match score, salary info, and reasoning to quantify fit.  
- **Nodes Involved:**  
  - `AI Deep Analysis` (Langchain Agent)  
  - `Google Gemini Deep Analysis` (Google Gemini LM Chat)  
  - `Structured Output Parser Deep Analysis` (Langchain Output Parser)  
  - `If Has High Match Score` (If node)  

- **Node Details:**  
  - **AI Deep Analysis**  
    - Type: Langchain Agent  
    - Role: Core AI intelligence performing CV-to-job detailed comparison.  
    - Configuration:  
      - System message defines role as career coach and HR specialist.  
      - Prompt includes candidate CV (from Config) and job description (current item).  
      - Output JSON fields: summary (2 bullet points), key_skills (array of 5), match_score (1-10 int), salaryInfo (salary or "Not specified"), reasoning (single sentence).  
      - Linked to `Google Gemini Deep Analysis` LM chat node for processing.  
    - Inputs: Job description and candidate CV.  
    - Outputs: Raw AI output.  
    - Edge Cases: Ambiguous resume or job description, AI hallucination, API errors.  
  - **Google Gemini Deep Analysis**  
    - Type: Google Gemini LM Chat  
    - Role: Language model engine backing `AI Deep Analysis`.  
    - Inputs: Text prompt from `AI Deep Analysis`.  
    - Outputs: AI raw response.  
  - **Structured Output Parser Deep Analysis**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses deep AI analysis output into structured JSON.  
    - Inputs: Raw AI output.  
    - Outputs: Parsed JSON with match score and other fields.  
    - Edge Cases: Parsing errors if AI output does not conform to schema.  
  - **If Has High Match Score**  
    - Type: If  
    - Role: Filters jobs with match_score >= 8 for notification.  
    - Condition: match_score >= 8  
    - Outputs: True branch triggers Telegram notification; False branch continues workflow.

---

#### 1.6 Data Storage and Notification

- **Overview:** Saves all qualified job matches to Supabase database and sends Telegram notifications for high-scoring jobs.  
- **Nodes Involved:**  
  - `Save Data` (Supabase node)  
  - `Send a Notification` (Telegram node)  
  - `To The Next` (NoOp node)  

- **Node Details:**  
  - **Save Data**  
    - Type: Supabase  
    - Role: Inserts job match data into Supabase table `job_tracker`.  
    - Configuration: Maps job title, company, AI match score, AI summary, job URLs, salary, reasoning, and apply URL from prior nodes.  
    - Inputs: Parsed AI analysis output and job item data.  
    - Outputs: Continues to `If Has High Match Score`.  
    - Edge Cases: Database connection issues, data validation errors.  
  - **Send a Notification**  
    - Type: Telegram  
    - Role: Sends formatted Telegram message for high match jobs.  
    - Configuration:  
      - Message includes job title, company, match score, AI reasoning, key skills, salary, apply link, and job URL.  
      - Uses HTML formatting and chat ID for target group.  
    - Inputs: Job data and AI analysis output.  
    - Outputs: Continues workflow to next item.  
    - Edge Cases: Telegram API errors, invalid chat ID.  
  - **To The Next**  
    - Type: NoOp  
    - Role: Acts as a pass-through node to trigger loop continuation.  
    - Inputs: From notification or directly from `If Has High Match Score` false branch.  
    - Outputs: Loops back to `Items Loop` for next job processing.

---

#### 1.7 Workflow Control & Looping

- **Overview:** Manages workflow triggers and batch iteration over job postings.  
- **Nodes Involved:**  
  - `Schedule Trigger` (Schedule Trigger)  
  - `Manual Executing` (Manual Trigger)  
  - `Done!` (NoOp)  

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at 9:00 AM.  
    - Inputs: None  
    - Outputs: Starts at `Config` node.  
  - **Manual Executing**  
    - Type: Manual Trigger  
    - Role: Allows manual start of the workflow for testing or on-demand runs.  
    - Inputs: None  
    - Outputs: Starts at `Config` node.  
  - **Done!**  
    - Type: NoOp  
    - Role: Marks end of a successful batch processing cycle.  
    - Inputs: From `Items Loop` when processing completes.  
    - Outputs: None (end of flow for that batch).  

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                                 | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                                                                                    |
|---------------------------|---------------------------------|------------------------------------------------|------------------------------|----------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Config                    | Set                             | Central config for all parameters               | Manual Executing, Schedule Trigger | Scrape LinkenIn Jobs         | ⚙️ Workflow Configuration: Central control panel for CV, keywords, location, commute, language, seniority, etc.                                                               |
| Scrape LinkenIn Jobs      | HTTP Request                    | Fetch LinkedIn jobs via API                      | Config                       | Items Loop                 |                                                                                                                                                                               |
| Items Loop                | Split In Batches                | Batch iteration over scraped job items          | Scrape LinkenIn Jobs, To The Next | Done!, Deconstruct Job Data |                                                                                                                                                                               |
| Deconstruct Job Data      | Langchain Agent                 | AI triage: validate seniority & language        | Items Loop                   | If Seniority Match          | 🤖 AI Triage: Fast AI filter extracting language, seniority, and match status from job description.                                                                           |
| Structured Output Parser  | Langchain Output Parser Structured | Parses AI triage output into structured JSON    | Deconstruct Job Data          | If Seniority Match          |                                                                                                                                                                               |
| If Seniority Match        | If                             | Filter jobs matching desired seniority           | Structured Output Parser      | If Language Match           |                                                                                                                                                                               |
| If Language Match         | If                             | Filter jobs matching target language             | If Seniority Match            | If is not Remote, Items Loop |                                                                                                                                                                               |
| If is not Remote          | If                             | Branch for non-remote jobs                        | If Language Match             | Get Commute Time, AI Deep Analysis | 📍 Geolocation Check: Calculate commute time only for onsite/hybrid jobs.                                                                                                   |
| Get Commute Time          | HTTP Request                   | Call Google Maps to get commute duration         | If is not Remote              | Is Commute Acceptable       |                                                                                                                                                                               |
| Is Commute Acceptable     | If                             | Filter jobs based on commute time                 | Get Commute Time              | AI Deep Analysis, Items Loop |                                                                                                                                                                               |
| AI Deep Analysis          | Langchain Agent                | Detailed AI CV-to-job matching                    | Is Commute Acceptable, If is not Remote | Save Data               | 🧠 AI Deep Analysis: Core CV matching and detailed scoring.                                                                                                                  |
| Google Gemini Deep Analysis | LM Chat Google Gemini          | AI model running CV matching prompt               | AI Deep Analysis             | Structured Output Parser Deep Analysis |                                                                                                                                                                               |
| Structured Output Parser Deep Analysis | Langchain Output Parser Structured | Parse detailed AI analysis output                 | Google Gemini Deep Analysis  | Save Data, If Has High Match Score |                                                                                                                                                                               |
| Save Data                 | Supabase                       | Store qualified job data in DB                     | AI Deep Analysis             | If Has High Match Score     | ✅ Action & Alerting: Save to Supabase and prepare for alerting.                                                                                                             |
| If Has High Match Score   | If                             | Filter high match jobs (score >= 8)               | Save Data                    | Send a Notification, To The Next |                                                                                                                                                                               |
| Send a Notification       | Telegram                       | Send Telegram alert for high match jobs            | If Has High Match Score      | To The Next                |                                                                                                                                                                               |
| To The Next               | NoOp                           | Loop control to process next job                   | Send a Notification, If Has High Match Score | Items Loop              |                                                                                                                                                                               |
| Schedule Trigger          | Schedule Trigger               | Daily automatic workflow start                     | None                        | Config                     |                                                                                                                                                                               |
| Manual Executing          | Manual Trigger                | Manual workflow start                              | None                        | Config                     |                                                                                                                                                                               |
| Done!                     | NoOp                           | Marks completion of processing batch               | Items Loop                   | None                       |                                                                                                                                                                               |
| Sticky Note               | Sticky Note                   | Documentation                                      | None                        | None                       | See detailed notes in relevant blocks.                                                                                                                                       |
| Sticky Note1              | Sticky Note                   | Documentation                                      | None                        | None                       | 🤖 AI Triage explanation                                                                                                                                                      |
| Sticky Note2              | Sticky Note                   | Documentation                                      | None                        | None                       | 📍 Geolocation Check explanation                                                                                                                                              |
| Sticky Note3              | Sticky Note                   | Documentation                                      | None                        | None                       | 🧠 AI Deep Analysis explanation                                                                                                                                               |
| Sticky Note4              | Sticky Note                   | Documentation                                      | None                        | None                       | ✅ Action & Alerting explanation                                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the `Config` (Set) node:**  
   - Add string fields:  
     - `MyCV`: Paste your full CV text.  
     - `JobKeywords`: e.g., "AI Integration".  
     - `ExperienceLevel`: e.g., "mid_senior".  
     - `Under10Applicants`: boolean false or true.  
     - `HomeLocation`: e.g., "Breda, Netherlands".  
     - `TargetLanguage`: e.g., "English".  
   - Add number fields:  
     - `JobsToScrape`: e.g., 20.  
     - `MaxCommuteMinutes`: e.g., 75.  
   - Add string field:  
     - `ActorId`: your Apify LinkedIn scraper actor ID (e.g., "apimaestro~linkedin-jobs-scraper-api").  

2. **Add `Scrape LinkenIn Jobs` (HTTP Request):**  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/{{ $json.ActorId }}/run-sync-get-dataset-items`  
   - Body (JSON):  
     ```json
     {
       "date_posted": "day",
       "experienceLevel": "{{ $json.ExperienceLevel }}",
       "keywords": "{{ $json.JobKeywords }}",
       "limit": {{ $json.JobsToScrape }},
       "under_10_applicants": {{ $json.Under10Applicants }},
       "sort": "relevant",
       "location": "91000000",
       "page_number": 1
     }
     ```  
   - Authentication: HTTP Bearer with Apify API token stored in n8n credentials.

3. **Add `Items Loop` (Split In Batches):**  
   - Use default batch size (1).  
   - Connect `Scrape LinkenIn Jobs` output to `Items Loop` input.  

4. **Add `Deconstruct Job Data` (Langchain Agent):**  
   - Prompt: Embed Config ExperienceLevel and current job description.  
   - Instructions to validate seniority and extract primary language, returning structured JSON with fields: jobLanguage, confirmed_seniority, is_match.  
   - Enable structured output parsing.  

5. **Add `Structured Output Parser` (Langchain Output Parser Structured):**  
   - Use JSON schema example with jobLanguage, confirmed_seniority, is_match fields.  
   - Connect AI output from `Deconstruct Job Data` to parser input.  

6. **Add `If Seniority Match` (If node):**  
   - Condition: Check if confirmed_seniority equals Config ExperienceLevel.  
   - True branch proceeds; false branch ends or loops next.  

7. **Add `If Language Match` (If node):**  
   - Condition: Check if Config TargetLanguage (comma-separated list) includes jobLanguage (case-insensitive).  
   - True branch proceeds; false branch ends or loops next.  

8. **Add `If is not Remote` (If node):**  
   - Condition: Check if job.work_type (lowercase) is NOT "remote".  
   - True branch leads to commute calculation; false branch skips to deep AI analysis.  

9. **Add `Get Commute Time` (HTTP Request):**  
   - URL: `https://maps.googleapis.com/maps/api/directions/json`  
   - Query parameters:  
     - origin = Config HomeLocation  
     - destination = job.location  
   - Authentication: HTTP Query Auth with Google Directions API key credential.  

10. **Add `Is Commute Acceptable` (If node):**  
    - Condition: commute duration (seconds) <= `MaxCommuteMinutes * 60`.  
    - True branch proceeds to AI Deep Analysis; false branch ends or loops next.  

11. **Add `AI Deep Analysis` (Langchain Agent):**  
    - System message: Expert career coach and HR specialist.  
    - Input prompt: Candidate CV (Config MyCV) and job description.  
    - Output fields: summary, key_skills, match_score (1-10), salaryInfo, reasoning.  
    - Enable structured output parsing.  

12. **Add `Google Gemini Deep Analysis` (LM Chat Google Gemini):**  
    - Used as AI model backend for `AI Deep Analysis`.  
    - Provide Google PaLM API credentials.  

13. **Add `Structured Output Parser Deep Analysis` (Langchain Output Parser Structured):**  
    - JSON schema with summary, key_skills, match_score, salaryInfo, reasoning.  

14. **Add `Save Data` (Supabase):**  
    - Table: `job_tracker`  
    - Map fields from job item and AI analysis output (jobTitle, companyName, matchScore, aiSummary, jobURL, salaryInfo, aiReasoning, applyURL).  
    - Provide Supabase credentials.  

15. **Add `If Has High Match Score` (If node):**  
    - Condition: match_score >= 8.  
    - True branch goes to notification, false branch loops to next.  

16. **Add `Send a Notification` (Telegram):**  
    - Message with job title, company, match score, reasoning, key skills, salary, and links to apply and job post.  
    - Chat ID: target Telegram group or user.  
    - Use HTML formatting.  
    - Provide Telegram API credentials.  

17. **Add `To The Next` (NoOp):**  
    - Connect from notification and false branch of high match score check.  
    - Loop back to `Items Loop` to continue processing remaining jobs.  

18. **Add `Done!` (NoOp):**  
    - Connect from `Items Loop` when all items processed. Marks end of workflow iteration.  

19. **Add Triggers:**  
    - `Schedule Trigger` set to fire daily at 9:00 AM connected to `Config`.  
    - `Manual Executing` trigger for manual runs connected to `Config`.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                             | Context or Link                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| Central configuration node `Config` controls all key parameters for CV, filtering, and API integration.                                                                                                                                                                  | See node `Config` and related sticky note.                |
| The initial AI triage is designed to be fast and cost-effective by only validating jobs on seniority and language before deeper costly AI analysis.                                                                                                                     | Sticky Note1: AI Triage explanation.                       |
| Geolocation filtering applies only to non-remote jobs using Google Maps Directions API to calculate realistic commute times. Jobs exceeding user-configured max commute are discarded.                                                                                   | Sticky Note2: Geolocation Check.                           |
| Deep AI analysis uses Google Gemini (PaLM) large language model, providing detailed semantic matching between the candidate CV and job description to generate actionable insights and match scoring.                                                                     | Sticky Note3: AI Deep Analysis.                            |
| All qualified jobs are saved into a Supabase database for tracking and future reference. High match jobs trigger immediate Telegram alerts with rich formatting and links.                                                                                               | Sticky Note4: Action & Alerting.                           |
| API credentials required: Apify LinkedIn Scraper (HTTP Bearer), Google Directions API key, Google Gemini (PaLM) API key, Supabase account, Telegram Bot API token.                                                                                                       | Credentials must be configured securely in n8n.            |
| Location code `"91000000"` in LinkedIn scraping API is fixed; verify if location filtering needs adjustment based on user preferences or expand to dynamic location parameters if needed.                                                                                  | API parameter detail to review.                            |
| The workflow is designed for extensibility; additional AI filters or integrations can be added after the deep analysis step if desired (e.g., salary negotiation bots, interview prep).                                                                                   | Design note for advanced customization.                    |
| Telegram notification uses HTML parse mode and disables attribution to keep messages clean and professional.                                                                                                                                                            | Telegram message formatting best practices.                |
| The workflow respects API rate limits and batch processing to avoid service overuse; monitor logs for throttling or errors.                                                                                                                                             | Operational note for monitoring in production.             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.