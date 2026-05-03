Generate & Enrich LinkedIn Leads with Apollo.io, LinkedIn API, Mail.so & GPT-3.5

https://n8nworkflows.xyz/workflows/generate---enrich-linkedin-leads-with-apollo-io--linkedin-api--mail-so---gpt-3-5-3791


# Generate & Enrich LinkedIn Leads with Apollo.io, LinkedIn API, Mail.so & GPT-3.5

### 1. Workflow Overview

This workflow automates the process of generating, enriching, and analyzing LinkedIn leads by integrating Apollo.io, LinkedIn Data API (via RapidAPI), Mail.so, Google Sheets, and OpenAI's GPT-3.5. It is designed for sales teams, founders, and B2B marketers to build personalized lead lists with verified contact information and insightful profile summaries.

The workflow is logically divided into the following blocks:

- **1.1 Lead Discovery (Apollo.io Search & Google Sheets Update):** Initiates lead search based on user input, cleans data, and appends leads to a Google Sheet.
- **1.2 LinkedIn Username Extraction:** Extracts LinkedIn usernames from profile URLs using OpenAI for further API enrichment.
- **1.3 Email Lookup and Validation:** Retrieves verified emails from Apollo.io, validates them via Mail.so API, and updates lead status accordingly.
- **1.4 Profile Summary Enrichment:** Fetches LinkedIn profile summaries via RapidAPI, cleans data, summarizes with OpenAI, and updates Google Sheets.
- **1.5 Recent Activity Collection & Summarization:** Retrieves recent LinkedIn posts, cleans and summarizes them with OpenAI, and updates the sheet.
- **1.6 Final Leads Database Update:** Aggregates fully enriched profiles and appends or updates a separate Google Sheet database.
- **1.7 Smart Retry & Status Management:** Scheduled triggers reset failed or invalid statuses back to pending for retry, ensuring data completeness.

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Discovery (Apollo.io Search & Google Sheets Update)

**Overview:**  
This block starts the workflow by accepting user input via a form trigger, uses Apollo.io API to search leads by job title, location, and number of leads, cleans the data, and appends it to a Google Sheet.

**Nodes Involved:**  
- On form submission  
- Generate Leads with Apollo.io  
- Split Out  
- Clean Data  
- Add Leads to Google Sheet  
- Sticky Note1 (comment)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; collects user input for job title, location, and number of leads.  
  - Config: Form fields are Job Title (required), Location (required), Number of Leads (required, number).  
  - Outputs: Triggers "Generate Leads with Apollo.io".  
  - Edge cases: Missing required fields; form submission errors.

- **Generate Leads with Apollo.io**  
  - Type: HTTP Request  
  - Role: Calls Apollo.io API to search leads based on form input.  
  - Config: POST request to Apollo's mixed_people/search endpoint with JSON body including person_locations, person_titles, per_page (number of leads), and projection fields (id, name, linkedin_url, title).  
  - Headers include x-api-key (Apollo API key).  
  - Outputs: JSON response with leads array.  
  - Edge cases: API rate limits, invalid API key, network errors.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of leads into individual items for processing.  
  - Config: Splits on "people" field from Apollo response.  
  - Outputs: Individual lead objects.  
  - Edge cases: Empty or malformed "people" array.

- **Clean Data**  
  - Type: Set  
  - Role: Extracts and normalizes relevant lead fields: id, name, linkedin_url, title, organization (from employment_history).  
  - Config: Assigns these fields explicitly for consistency.  
  - Outputs: Cleaned lead JSON objects.  
  - Edge cases: Missing employment_history or organization_name.

- **Add Leads to Google Sheet**  
  - Type: Google Sheets  
  - Role: Appends cleaned lead data to a Google Sheet with initial status columns for scraping/enrichment.  
  - Config: Maps fields to columns including posts_scrape_status ("unscraped"), contacts_scrape_status ("pending"), profile_summary_scrape ("pending"), extract_username_status ("pending").  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Google Sheets API limits, permission errors.

- **Sticky Note1**  
  - Content: "## Scrape Leads from Apollo"  
  - Context: Describes this block's purpose.

---

#### 1.2 LinkedIn Username Extraction

**Overview:**  
Extracts the LinkedIn username from the full LinkedIn URL using OpenAI to prepare for further API calls that require usernames.

**Nodes Involved:**  
- Google Sheets Trigger (listens for new rows)  
- Get Pending Username Row (filters rows with extract_username_status = "pending")  
- OpenAI1 (extract username)  
- Add Linkedin Username (update sheet)  
- Sticky Note (comment)

**Node Details:**

- **Google Sheets Trigger**  
  - Type: Google Sheets Trigger  
  - Role: Watches the leads sheet for new rows added.  
  - Config: Polls every minute on the specified sheet and document.  
  - Outputs: Triggers "Get Pending Username Row".  
  - Edge cases: Trigger delays, API quota.

- **Get Pending Username Row**  
  - Type: Google Sheets (Read)  
  - Role: Retrieves the first row where extract_username_status is "pending".  
  - Config: Filter on "extract_username_status" = "pending", returns first match.  
  - Outputs: Single lead row for processing.  
  - Edge cases: No pending rows found.

- **OpenAI1**  
  - Type: OpenAI (Langchain)  
  - Role: Uses GPT-3.5 to remove URL parts and extract the LinkedIn username.  
  - Config: Message content expression: `remove the http or https://www.linkedin.com/in/ from this  {{ $json.linkedin_url }}`  
  - Credentials: OpenAI API key.  
  - Outputs: Extracted username in response message.  
  - Edge cases: API errors, malformed URLs, rate limits.

- **Add Linkedin Username**  
  - Type: Google Sheets  
  - Role: Updates the lead row with the extracted username and sets extract_username_status to "finished".  
  - Config: Matches on apollo_id, updates linkedin_username and status.  
  - Credentials: Google Sheets OAuth2.  
  - Edge cases: Update failures, concurrency issues.

- **Sticky Note**  
  - Content: "## Extract Linkedin Username"  
  - Context: Describes this block's purpose.

---

#### 1.3 Email Lookup and Validation

**Overview:**  
Retrieves verified work emails from Apollo.io using the Apollo User ID, validates email deliverability and MX records via Mail.so API, and updates lead status accordingly.

**Nodes Involved:**  
- Google Sheets Trigger2  
- Get Pending Email Statuses (filter contacts_scrape_status = "pending")  
- Get Email from Apollo (Apollo API call)  
- Confirm Email Validity (Mail.so API call)  
- If (checks deliverability and MX record)  
- Add Email Address (update sheet if valid)  
- Mark Invalid Email (update sheet if invalid)  
- Schedule Trigger (retry invalid emails every 4 weeks)  
- get invalid email rows (fetch invalid emails)  
- update_to_pending (reset invalid to pending)  
- Sticky Note2, Sticky Note3 (comments)

**Node Details:**

- **Google Sheets Trigger2**  
  - Watches for new rows added to the leads sheet.  
  - Triggers "Get Pending Email Statuses".

- **Get Pending Email Statuses**  
  - Reads first row with contacts_scrape_status = "pending".  
  - Outputs lead data for email lookup.

- **Get Email from Apollo**  
  - HTTP POST to Apollo's people/match endpoint with apollo_id.  
  - Query params reveal_personal_emails=true, reveal_phone_number=false.  
  - Headers include x-api-key.  
  - Outputs lead's email in response.  
  - Edge cases: API errors, missing email.

- **Confirm Email Validity**  
  - HTTP GET to Mail.so validate endpoint with email parameter.  
  - Headers include x-mails-api-key and x-rapidapi-key.  
  - Outputs validation result including mx_record and deliverability.  
  - Edge cases: API rate limits, invalid API keys.

- **If**  
  - Checks if mx_record is not "null" and result contains "deliverable".  
  - Routes valid emails to "Add Email Address", invalid to "Mark Invalid Email".

- **Add Email Address**  
  - Updates Google Sheet row with email_address and sets contacts_scrape_status to "finished".  
  - Matches on apollo_id.

- **Mark Invalid Email**  
  - Updates contacts_scrape_status to "invalid_email" for invalid emails.

- **Schedule Trigger**  
  - Runs every 4 weeks on Tuesday at 8 AM.  
  - Triggers "get invalid email rows".

- **get invalid email rows**  
  - Reads all rows with contacts_scrape_status = "invalid_email".

- **update_to_pending**  
  - Updates those rows back to contacts_scrape_status = "pending" for retry.

- **Sticky Note2**  
  - Content: "## Get Email Address, Validate Deliverability & Update Column Status"

- **Sticky Note3**  
  - Content: "## Update Contact Scrape Status from Invalid back to Pending"

---

#### 1.4 Profile Summary Enrichment

**Overview:**  
Fetches LinkedIn profile summaries via RapidAPI, cleans and structures the data, summarizes it with OpenAI, and updates the Google Sheet with enriched profile information.

**Nodes Involved:**  
- Google Sheets Trigger3  
- Get Pending About and Posts Rows (filter profile_summary_scrape = "pending")  
- Get About Profile (RapidAPI call)  
- Clean Profile Data (Code node)  
- Stringify Profile Data1 (Code node)  
- AI Profile Summarizer (OpenAI)  
- Update Profile Summary (Google Sheets update)  
- update status to failed (Google Sheets update on error)  
- get_failed_profile_summary_rows (fetch failed rows)  
- update_to_pending1 (reset failed to pending)  
- Schedule Trigger2 (retry every 4 weeks)  
- Sticky Note4, Sticky Note8 (comments)

**Node Details:**

- **Google Sheets Trigger3**  
  - Watches for new rows added to leads sheet.  
  - Triggers "Get Pending About and Posts Rows".

- **Get Pending About and Posts Rows**  
  - Reads first row with profile_summary_scrape = "pending".

- **Get About Profile**  
  - HTTP GET to RapidAPI LinkedIn Data API with linkedin_username.  
  - Headers include x-rapidapi-host and x-rapidapi-key.  
  - Outputs profile JSON including summary, headline, education, employment, etc.  
  - Retries on failure (max 2 tries).

- **Clean Profile Data**  
  - JavaScript code extracts relevant fields (summary, headline, nationality, language, education, employment details).  
  - Retries on failure.

- **Stringify Profile Data1**  
  - Converts cleaned profile JSON to pretty-printed string for AI input.

- **AI Profile Summarizer**  
  - OpenAI GPT-3.5 summarizes profile string for personalized cold emails.  
  - Outputs concise summary.

- **Update Profile Summary**  
  - Updates Google Sheet row with about_linkedin_profile (summary) and sets profile_summary_scrape to "completed".

- **update status to failed**  
  - If cleaning or summarizing fails, sets profile_summary_scrape to "failed".

- **get_failed_profile_summary_rows**  
  - Reads rows with profile_summary_scrape = "failed".

- **update_to_pending1**  
  - Resets failed rows back to "pending" for retry.

- **Schedule Trigger2**  
  - Runs every 4 weeks on Tuesday at 8 AM to retry failed profile summaries.

- **Sticky Notes**  
  - Sticky Note4: "## Get Profile Summary & Update Status"  
  - Sticky Note8: "## Update profile summary status from failed back to pending"

---

#### 1.5 Recent Activity Collection & Summarization

**Overview:**  
Retrieves recent LinkedIn posts and reposts via RapidAPI, cleans and extracts key post texts and dates, summarizes them with OpenAI, and updates the Google Sheet.

**Nodes Involved:**  
- Google Sheets Trigger4  
- Get Pending About and Posts Rows1 (filter posts_scrape_status = "unscraped")  
- Get Profile Posts (RapidAPI call)  
- Clean Posts Data (Code node)  
- Stringify Posts Data (Code node)  
- Posts AI Summarizer (OpenAI)  
- Update Posts Summary (Google Sheets update)  
- Google Sheets (update posts_scrape_status to "failed" on error)  
- get_failed_posts_summary_rows1 (fetch failed rows)  
- update_to_unscraped (reset failed to unscraped)  
- Schedule Trigger3 (retry every 4 weeks)  
- Sticky Note5, Sticky Note6, Sticky Note9 (comments)

**Node Details:**

- **Google Sheets Trigger4**  
  - Watches for new rows added to leads sheet.  
  - Triggers "Get Pending About and Posts Rows1".

- **Get Pending About and Posts Rows1**  
  - Reads first row with posts_scrape_status = "unscraped".

- **Get Profile Posts**  
  - HTTP GET to RapidAPI LinkedIn Data API get-profile-posts endpoint with linkedin_username.  
  - Headers include x-rapidapi-host and x-rapidapi-key.  
  - Retries on failure (max 2 tries).  
  - On error, continues output.

- **Clean Posts Data**  
  - JavaScript extracts text and dates from up to 5 posts/reposts, handling reshared posts.  
  - Retries on failure with 3-second wait.

- **Stringify Posts Data**  
  - Converts cleaned posts JSON to pretty-printed string for AI input.

- **Posts AI Summarizer**  
  - OpenAI GPT-3.5 summarizes recent posts into 2 concise paragraphs focusing on themes and tone.

- **Update Posts Summary**  
  - Updates Google Sheet row with recent_posts_summary and sets posts_scrape_status to "scraped".

- **Google Sheets (update posts_scrape_status to failed)**  
  - If summarization fails, sets posts_scrape_status to "failed".

- **get_failed_posts_summary_rows1**  
  - Reads rows with posts_scrape_status = "failed".

- **update_to_unscraped**  
  - Resets failed rows back to "unscraped" for retry.

- **Schedule Trigger3**  
  - Runs every 4 weeks on Tuesday at 8 AM to retry failed posts summaries.

- **Sticky Notes**  
  - Sticky Note5: "## Get Summary of Latest Linkedin Profile Posts"  
  - Sticky Note6: "## Update Completely Enriched Profile to Final Database"  
  - Sticky Note9: "## Update posts summary status from failed back to pending"

---

#### 1.6 Final Leads Database Update

**Overview:**  
Collects fully enriched leads (with finished email, completed profile summary, and scraped posts), and appends or updates them in a separate Google Sheet database for outreach or further use.

**Nodes Involved:**  
- Google Sheets Trigger5  
- Get Completely Enriched Profiles (filter rows with all enrichment statuses done)  
- Append to Enriched Leads Database (Google Sheets appendOrUpdate)  
- Sticky Note6 (comment)

**Node Details:**

- **Google Sheets Trigger5**  
  - Watches for new rows added to leads sheet.

- **Get Completely Enriched Profiles**  
  - Reads first row where contacts_scrape_status = "finished", profile_summary_scrape = "completed", posts_scrape_status = "scraped".

- **Append to Enriched Leads Database**  
  - Appends or updates lead info in a separate Google Sheet (Enriched Leads Database).  
  - Maps fields: Lead Name, Email Address, Company/Organization, Position, Linkedin Profile Summary, Recent Posts Summary.  
  - Matches on Email Address to avoid duplicates.

- **Sticky Note6**  
  - Content: "## Update Completely Enriched Profile to Final Database"

---

#### 1.7 Smart Retry & Status Management

**Overview:**  
Scheduled triggers periodically reset failed or invalid statuses back to pending or unscraped to retry enrichment steps, minimizing data loss and ensuring completeness.

**Nodes Involved:**  
- Schedule Trigger (invalid emails retry)  
- get invalid email rows  
- update_to_pending  
- Schedule Trigger2 (failed profile summaries retry)  
- get_failed_profile_summary_rows  
- update_to_pending1  
- Schedule Trigger3 (failed posts summaries retry)  
- get_failed_posts_summary_rows1  
- update_to_unscraped  
- Sticky Notes2,3,8,9 (comments)

**Node Details:**  
- Each schedule trigger runs every 4 weeks on Tuesday at 8 AM.  
- Reads rows with failed or invalid statuses.  
- Updates their status columns back to pending or unscraped.  
- This allows the main enrichment processes to pick them up again.

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                                  | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                         |
|-------------------------------|--------------------------------|-------------------------------------------------|----------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger                   | Entry point: collects lead search parameters    | -                                | Generate Leads with Apollo.io     | ## Scrape Leads from Apollo                                                                        |
| Generate Leads with Apollo.io  | HTTP Request                  | Searches leads via Apollo API                     | On form submission               | Split Out                       |                                                                                                   |
| Split Out                     | Split Out                     | Splits leads array into individual items         | Generate Leads with Apollo.io    | Clean Data                      |                                                                                                   |
| Clean Data                    | Set                           | Normalizes lead fields                            | Split Out                       | Add Leads to Google Sheet        |                                                                                                   |
| Add Leads to Google Sheet      | Google Sheets                 | Appends leads to Google Sheet with initial status| Clean Data                      | -                              |                                                                                                   |
| Google Sheets Trigger          | Google Sheets Trigger         | Watches for new rows added                        | -                                | Get Pending Username Row          | ## Extract Linkedin Username                                                                       |
| Get Pending Username Row       | Google Sheets                 | Gets first row with extract_username_status=pending| Google Sheets Trigger           | OpenAI1                        |                                                                                                   |
| OpenAI1                      | OpenAI (Langchain)            | Extracts LinkedIn username from URL              | Get Pending Username Row          | Add Linkedin Username            |                                                                                                   |
| Add Linkedin Username          | Google Sheets                 | Updates sheet with extracted username            | OpenAI1                        | -                              |                                                                                                   |
| Google Sheets Trigger2         | Google Sheets Trigger         | Watches for new rows added                        | -                                | Get Pending Email Statuses        | ## Get Email Address, Validate Deliverability & Update Column Status                               |
| Get Pending Email Statuses     | Google Sheets                 | Gets first row with contacts_scrape_status=pending| Google Sheets Trigger2          | Get Email from Apollo            |                                                                                                   |
| Get Email from Apollo          | HTTP Request                  | Retrieves verified email from Apollo              | Get Pending Email Statuses       | Confirm Email Validity           |                                                                                                   |
| Confirm Email Validity         | HTTP Request                  | Validates email deliverability via Mail.so       | Get Email from Apollo            | If                             |                                                                                                   |
| If                            | If                           | Routes based on email validity                     | Confirm Email Validity           | Add Email Address / Mark Invalid Email |                                                                                                   |
| Add Email Address              | Google Sheets                 | Updates sheet with valid email and status         | If (valid)                     | -                              |                                                                                                   |
| Mark Invalid Email             | Google Sheets                 | Marks email as invalid in sheet                    | If (invalid)                   | -                              |                                                                                                   |
| Schedule Trigger              | Schedule Trigger             | Scheduled retry for invalid emails                | -                                | get invalid email rows           | ## Update Contact Scrape Status from Invalid back to Pending                                      |
| get invalid email rows         | Google Sheets                 | Gets rows with invalid_email status               | Schedule Trigger               | update_to_pending               |                                                                                                   |
| update_to_pending             | Google Sheets                 | Resets invalid emails to pending                   | get invalid email rows          | -                              |                                                                                                   |
| Google Sheets Trigger3         | Google Sheets Trigger         | Watches for new rows added                        | -                                | Get Pending About and Posts Rows  | ## Get Profile Summary & Update Status                                                            |
| Get Pending About and Posts Rows| Google Sheets               | Gets first row with profile_summary_scrape=pending| Google Sheets Trigger3          | Get About Profile               |                                                                                                   |
| Get About Profile             | HTTP Request                  | Fetches LinkedIn profile summary via RapidAPI    | Get Pending About and Posts Rows | Clean Profile Data             |                                                                                                   |
| Clean Profile Data            | Code                         | Extracts relevant profile fields                   | Get About Profile               | Stringify Profile Data1          |                                                                                                   |
| Stringify Profile Data1       | Code                         | Converts profile JSON to string for AI input      | Clean Profile Data              | AI Profile Summarizer           |                                                                                                   |
| AI Profile Summarizer         | OpenAI (Langchain)            | Summarizes profile for personalized outreach      | Stringify Profile Data1         | Update Profile Summary          |                                                                                                   |
| Update Profile Summary        | Google Sheets                 | Updates sheet with profile summary and status     | AI Profile Summarizer           | -                              |                                                                                                   |
| update status to failed       | Google Sheets                 | Marks profile summary as failed on error          | Clean Profile Data              | -                              |                                                                                                   |
| get_failed_profile_summary_rows| Google Sheets               | Gets rows with failed profile summary status      | Schedule Trigger2              | update_to_pending1              |                                                                                                   |
| update_to_pending1            | Google Sheets                 | Resets failed profile summaries to pending        | get_failed_profile_summary_rows | -                              |                                                                                                   |
| Schedule Trigger2             | Schedule Trigger             | Scheduled retry for failed profile summaries      | -                                | get_failed_profile_summary_rows  |                                                                                                   |
| Google Sheets Trigger4         | Google Sheets Trigger         | Watches for new rows added                        | -                                | Get Pending About and Posts Rows1 | ## Get Summary of Latest Linkedin Profile Posts                                                   |
| Get Pending About and Posts Rows1| Google Sheets             | Gets first row with posts_scrape_status=unscraped | Google Sheets Trigger4          | Get Profile Posts              |                                                                                                   |
| Get Profile Posts            | HTTP Request                  | Fetches recent LinkedIn posts via RapidAPI        | Get Pending About and Posts Rows1| Clean Posts Data              |                                                                                                   |
| Clean Posts Data             | Code                         | Extracts text and dates from posts                  | Get Profile Posts              | Stringify Posts Data            |                                                                                                   |
| Stringify Posts Data         | Code                         | Converts posts JSON to string for AI input          | Clean Posts Data               | Posts AI Summarizer            |                                                                                                   |
| Posts AI Summarizer          | OpenAI (Langchain)            | Summarizes recent posts into concise paragraphs    | Stringify Posts Data           | Update Posts Summary           |                                                                                                   |
| Update Posts Summary         | Google Sheets                 | Updates sheet with posts summary and status         | Posts AI Summarizer            | -                              |                                                                                                   |
| Google Sheets (update posts_scrape_status to failed) | Google Sheets | Marks posts summary as failed on error             | Clean Posts Data               | -                              |                                                                                                   |
| get_failed_posts_summary_rows1| Google Sheets               | Gets rows with failed posts summary status          | Schedule Trigger3              | update_to_unscraped            |                                                                                                   |
| update_to_unscraped          | Google Sheets                 | Resets failed posts summaries to unscraped          | get_failed_posts_summary_rows1 | -                              |                                                                                                   |
| Schedule Trigger3            | Schedule Trigger             | Scheduled retry for failed posts summaries          | -                                | get_failed_posts_summary_rows1  |                                                                                                   |
| Google Sheets Trigger5         | Google Sheets Trigger         | Watches for new rows added                        | -                                | Get Completely Enriched Profiles | ## Update Completely Enriched Profile to Final Database                                           |
| Get Completely Enriched Profiles| Google Sheets              | Gets rows with all enrichment statuses completed   | Google Sheets Trigger5          | Append to Enriched Leads Database|                                                                                                   |
| Append to Enriched Leads Database| Google Sheets             | Appends or updates final enriched leads database   | Get Completely Enriched Profiles| -                              |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: Form Trigger  
   - Configure form fields: Job Title (required), Location (required), Number of Leads (required, number).  
   - This node will start the lead search.

2. **Add HTTP Request node "Generate Leads with Apollo.io"**  
   - Method: POST  
   - URL: `https://api.apollo.io/api/v1/mixed_people/search`  
   - Body (JSON): Use expressions to insert form data for person_locations, person_titles, per_page, and projection fields (id, name, linkedin_url, title).  
   - Headers: Include `x-api-key` with your Apollo API key.  
   - Connect from Form Trigger.

3. **Add Split Out node**  
   - Field to split: "people" (from Apollo response).  
   - Connect from Apollo HTTP Request.

4. **Add Set node "Clean Data"**  
   - Assign fields: id, name, linkedin_url, title, organization (from employment_history[0].organization_name).  
   - Connect from Split Out.

5. **Add Google Sheets node "Add Leads to Google Sheet"**  
   - Operation: Append  
   - Map cleaned fields to columns: name, title, apollo_id, linkedin_url, organization.  
   - Add status columns: posts_scrape_status="unscraped", contacts_scrape_status="pending", profile_summary_scrape="pending", extract_username_status="pending".  
   - Configure Google Sheets credentials and specify document and sheet.  
   - Connect from Clean Data.

6. **Add Google Sheets Trigger node**  
   - Watches the leads sheet for new rows added.  
   - Poll every minute.

7. **Add Google Sheets node "Get Pending Username Row"**  
   - Filter: extract_username_status = "pending"  
   - Return first match.  
   - Connect from Google Sheets Trigger.

8. **Add OpenAI node "OpenAI1"**  
   - Model: GPT-3.5 Turbo  
   - Message: Remove "http(s)://www.linkedin.com/in/" from linkedin_url to extract username.  
   - Connect from Get Pending Username Row.  
   - Configure OpenAI credentials.

9. **Add Google Sheets node "Add Linkedin Username"**  
   - Operation: Update  
   - Match on apollo_id  
   - Update linkedin_username with OpenAI output and set extract_username_status = "finished".  
   - Connect from OpenAI1.

10. **Add Google Sheets Trigger2**  
    - Watches leads sheet for new rows added.  
    - Poll every minute.

11. **Add Google Sheets node "Get Pending Email Statuses"**  
    - Filter: contacts_scrape_status = "pending"  
    - Return first match.  
    - Connect from Google Sheets Trigger2.

12. **Add HTTP Request node "Get Email from Apollo"**  
    - POST to `https://api.apollo.io/api/v1/people/match`  
    - Body param: id = apollo_id from input  
    - Query params: reveal_personal_emails=true, reveal_phone_number=false  
    - Headers: x-api-key with Apollo key  
    - Connect from Get Pending Email Statuses.

13. **Add HTTP Request node "Confirm Email Validity"**  
    - GET to `https://api.mails.so/v1/validate?email={{email}}`  
    - Headers: x-mails-api-key (Mail.so), x-rapidapi-key (RapidAPI)  
    - Connect from Get Email from Apollo.

14. **Add If node**  
    - Condition: mx_record not "null" AND result contains "deliverable"  
    - Connect from Confirm Email Validity.

15. **Add Google Sheets node "Add Email Address"**  
    - Operation: Update  
    - Match on apollo_id  
    - Update email_address and contacts_scrape_status = "finished"  
    - Connect from If (true branch).

16. **Add Google Sheets node "Mark Invalid Email"**  
    - Operation: Update  
    - Match on apollo_id  
    - Update contacts_scrape_status = "invalid_email"  
    - Connect from If (false branch).

17. **Add Schedule Trigger node**  
    - Runs every 4 weeks on Tuesday at 8 AM.

18. **Add Google Sheets node "get invalid email rows"**  
    - Filter: contacts_scrape_status = "invalid_email"  
    - Connect from Schedule Trigger.

19. **Add Google Sheets node "update_to_pending"**  
    - Operation: Update  
    - Match on apollo_id  
    - Set contacts_scrape_status = "pending"  
    - Connect from get invalid email rows.

20. **Add Google Sheets Trigger3**  
    - Watches leads sheet for new rows added.

21. **Add Google Sheets node "Get Pending About and Posts Rows"**  
    - Filter: profile_summary_scrape = "pending"  
    - Return first match.  
    - Connect from Google Sheets Trigger3.

22. **Add HTTP Request node "Get About Profile"**  
    - GET to RapidAPI LinkedIn Data API root with username param  
    - Headers: x-rapidapi-host, x-rapidapi-key  
    - Connect from Get Pending About and Posts Rows.

23. **Add Code node "Clean Profile Data"**  
    - Extract fields: summary, headline, nationality, language, education, employment details.  
    - Connect from Get About Profile.

24. **Add Code node "Stringify Profile Data1"**  
    - JSON.stringify cleaned profile data with indentation.  
    - Connect from Clean Profile Data.

25. **Add OpenAI node "AI Profile Summarizer"**  
    - Model: GPT-3.5 Turbo  
    - Message: Summarize profileString for cold email personalization.  
    - Connect from Stringify Profile Data1.

26. **Add Google Sheets node "Update Profile Summary"**  
    - Operation: Update  
    - Match on apollo_id  
    - Update about_linkedin_profile with AI summary and profile_summary_scrape = "completed".  
    - Connect from AI Profile Summarizer.

27. **Add Google Sheets node "update status to failed"**  
    - Operation: Update  
    - Match on apollo_id  
    - Set profile_summary_scrape = "failed"  
    - Connect from Clean Profile Data (error branch).

28. **Add Schedule Trigger2**  
    - Runs every 4 weeks on Tuesday at 8 AM.

29. **Add Google Sheets node "get_failed_profile_summary_rows"**  
    - Filter: profile_summary_scrape = "failed"  
    - Connect from Schedule Trigger2.

30. **Add Google Sheets node "update_to_pending1"**  
    - Operation: Update  
    - Match on apollo_id  
    - Set profile_summary_scrape = "pending"  
    - Connect from get_failed_profile_summary_rows.

31. **Add Google Sheets Trigger4**  
    - Watches leads sheet for new rows added.

32. **Add Google Sheets node "Get Pending About and Posts Rows1"**  
    - Filter: posts_scrape_status = "unscraped"  
    - Return first match.  
    - Connect from Google Sheets Trigger4.

33. **Add HTTP Request node "Get Profile Posts"**  
    - GET to RapidAPI LinkedIn Data API get-profile-posts endpoint with username param  
    - Headers: x-rapidapi-host, x-rapidapi-key  
    - Connect from Get Pending About and Posts Rows1.

34. **Add Code node "Clean Posts Data"**  
    - Extracts text and dates from posts and reshared posts.  
    - Connect from Get Profile Posts.

35. **Add Code node "Stringify Posts Data"**  
    - JSON.stringify cleaned posts data.  
    - Connect from Clean Posts Data.

36. **Add OpenAI node "Posts AI Summarizer"**  
    - Model: GPT-3.5 Turbo  
    - Message: Summarize recent posts into 2 concise paragraphs.  
    - Connect from Stringify Posts Data.

37. **Add Google Sheets node "Update Posts Summary"**  
    - Operation: Update  
    - Match on apollo_id  
    - Update recent_posts_summary and posts_scrape_status = "scraped".  
    - Connect from Posts AI Summarizer.

38. **Add Google Sheets node (update posts_scrape_status to failed)**  
    - Operation: Update  
    - Match on apollo_id  
    - Set posts_scrape_status = "failed"  
    - Connect from Clean Posts Data (error branch).

39. **Add Schedule Trigger3**  
    - Runs every 4 weeks on Tuesday at 8 AM.

40. **Add Google Sheets node "get_failed_posts_summary_rows1"**  
    - Filter: posts_scrape_status = "failed"  
    - Connect from Schedule Trigger3.

41. **Add Google Sheets node "update_to_unscraped"**  
    - Operation: Update  
    - Match on apollo_id  
    - Set posts_scrape_status = "unscraped"  
    - Connect from get_failed_posts_summary_rows1.

42. **Add Google Sheets Trigger5**  
    - Watches leads sheet for new rows added.

43. **Add Google Sheets node "Get Completely Enriched Profiles"**  
    - Filter: contacts_scrape_status = "finished", profile_summary_scrape = "completed", posts_scrape_status = "scraped"  
    - Return first match.  
    - Connect from Google Sheets Trigger5.

44. **Add Google Sheets node "Append to Enriched Leads Database"**  
    - Operation: Append or Update  
    - Match on Email Address  
    - Map fields: Lead Name, Email Address, Company/Organization, Position, Linkedin Profile Summary, Recent Posts Summary.  
    - Connect from Get Completely Enriched Profiles.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Google Sheets templates for setup:                                                                              | [Apollo Leads Scraping & Enrichment](https://docs.google.com/spreadsheets/d/1d99PlHkp9RPeSAtmATgQ4OC4Selcp8JSFLNuKx-n1EQ/edit?gid=0) and [Enriched Leads Database](https://docs.google.com/spreadsheets/d/1c5USULUPS-2_RdNf29cyDguuHH7A7JNwzFCjQQUJTvE/edit?gid=0) |
| Apollo.io API key setup requires "Master API Key" toggled ON for full endpoint access.                          | [Apollo.io API Keys](https://developer.apollo.io/keys/)                                                  |
| RapidAPI LinkedIn Data API subscription needed for profile and posts data.                                      | [LinkedIn Data API on RapidAPI](https://rapidapi.com/rockapis-rockapis-default/api/linkedin-data-api)    |
| Mail.so API key required for email validation.                                                                  | [Mail.so API](https://mails.so/dashboard/api)                                                            |
| Credentials should be configured in n8n as "Generic Credential" types for API keys to simplify header management.|                                                                                                          |
| Retry intervals and filters can be customized to fit specific business needs (e.g., weekly retries).            |                                                                                                          |
| AI summarization prompts can be customized for tone and detail level.                                          |                                                                                                          |

---

This structured documentation provides a full understanding of the workflow’s logic, node configurations, error handling, and integration points, enabling reproduction or modification by developers or AI agents.