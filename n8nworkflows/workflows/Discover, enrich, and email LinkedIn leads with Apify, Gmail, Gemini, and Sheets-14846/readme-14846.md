Discover, enrich, and email LinkedIn leads with Apify, Gmail, Gemini, and Sheets

https://n8nworkflows.xyz/workflows/discover--enrich--and-email-linkedin-leads-with-apify--gmail--gemini--and-sheets-14846


# Discover, enrich, and email LinkedIn leads with Apify, Gmail, Gemini, and Sheets

Now I have a complete understanding of the workflow. Let me write the comprehensive document. 1. Workflow Overview

This workflow automates the full lifecycle of LinkedIn-based B2B lead generation: **discovery → enrichment → AI-powered message generation → multi-step email outreach with response tracking**. It orchestrates four external services—Apify (for LinkedIn search and profile scraping), Google Sheets (as the persistent data store), Google Gemini (for cold email generation), and Gmail (for sending emails with embedded response forms).

The workflow is organized into three major phases, each of which can be triggered independently:

| Phase | Purpose | Entry Point |
|-------|---------|-------------|
| **Phase 1 – Lead Discovery & Collection** | Reads search keywords from a Google Sheet, runs Apify Google Search SERP scraper to find LinkedIn profile URLs, parses and stores them in a "Raw Profiles" sheet, then marks the input row as done. | Read Input Leads (no manual trigger; starts from the sheet read) |
| **Phase 2 – Profile Enrichment & AI Message Generation** | Reads un-enriched profiles, scrapes each via two Apify actors (LinkedIn Profile Enrichment + LinkedIn to Email), saves enriched data (name, email, company, etc.) into a "Leads" sheet, then uses Google Gemini to generate three cold-email variants (initial + 2 follow-ups) and writes them back. | Read Unenriched Profiles (no manual trigger; starts from the sheet read) |
| **Phase 3 – Email Outreach & Response Tracking** | A manual trigger reads leads that have not yet been emailed, sends the first cold email via Gmail's sendAndWait (with an embedded form), waits for a response or a 1-day delay, then escalates through up to two follow-up emails, recording every response back into the "Leads" sheet. | Manual Outreach Starting Point |

### Logical Block Map

| Block # | Block Name | Functional Role |
|---------|-----------|-----------------|
| 1 | Fetch Initial Lead Data | Read search keywords from Input_List sheet and gate on unprocessed rows |
| 2 | Apify Scraping Process | Batch-execute Apify Google Search, parse results, append LinkedIn URLs to Raw Profiles, mark input as done |
| 3 | Load Profiles for Enrichment | Read un-enriched rows from Raw Profiles and gate on `Enriched = "No"` |
| 4 | LinkedIn Data Scraping | Call Apify LinkedIn Profile Enrichment then LinkedIn-to-Email actors for each profile; skip if enrichment data already exists |
| 5 | Save Enriched Data | Append enriched lead data to Leads sheet and mark Raw Profiles row as `Enriched = "Yes"` |
| 6 | Prepare for Follow-Up Messaging | Read leads from Leads sheet, gate on unprocessed enrichment status, batch them |
| 7 | Generate Follow-Up Message | Use Google Gemini to produce 3 cold-email texts, write Email_1/2/3 columns back to Leads, mark profile as enriched |
| 8 | Start Email Outreach | Manual trigger → read leads where `Email Sent = "No"` → batch them |
| 9 | Email 1 – Send & Check Response | Send initial email via Gmail sendAndWait; if response received, record it; otherwise proceed to follow-up |
| 10 | Follow-Up Email 2 | Wait 1 day, send second email, check response, record or escalate |
| 11 | Final Follow-Up Email 3 | Wait 1 day, send third email, check response, record it |

---

## 2. Block-by-Block Analysis

---

### Block 1 – Fetch Initial Lead Data

**Overview:** Reads the **Input_List** sheet containing search keywords and locations. An IF node gates execution so that only rows whose status has not yet been marked as "Done" proceed into the scraping pipeline.

**Nodes Involved:**
- Read Input Leads
- Check Input Sheet is Empty

#### Read Input Leads
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.7) |
| Operation | Read |
| Sheet | **Input_List** (gid=0) |
| Document | Linkedin_Leads_Generation and Follow Up (placeholder `YOUR_SHEET_ID`) |
| Credential | Google Sheets OAuth2 |
| Output | All rows from the Input_List sheet |
| Edge cases | If the sheet is empty or the credential is revoked, the node returns zero items |

#### Check Input Sheet is Empty
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **is empty** on a field derived from the input sheet (template placeholder `YOUR_SHEET_ID` – in practice, the **Status** column) |
| True output → | Batch LinkedIn Searches (proceed when status is blank → unprocessed) |
| False output → | No connection (workflow stops if all rows are already "Done") |
| Edge cases | If the leftValue expression references the wrong column, the condition may always evaluate true or false; strict type validation is off, so `"Done"` vs `""` must match exactly |

---

### Block 2 – Apify Scraping Process

**Overview:** Batches the unprocessed input rows, calls the Apify **Google Search Results SERP Scraper** for each keyword to find LinkedIn profile URLs, parses the JSON response to extract only `linkedin.com/in` links, appends them to the **Raw Profiles** sheet, and then updates the Input_List row status to "Done" before looping back to process the next batch.

**Nodes Involved:**
- Batch LinkedIn Searches
- Post Apify Search Request
- Parse Apify Response
- Append Profiles to Raw Sheet
- Update Input Sheet Status

#### Batch LinkedIn Searches
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.splitInBatches` (v3) |
| Role | Iterates over each input row one at a time; output 0 (done) is unused; output 1 (each batch item) → Post Apify Search Request |
| Loop-back | Receives from Update Input Sheet Status after each row is processed |

#### Post Apify Search Request
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.httpRequest` (v4.2) |
| Method | POST |
| URL | `https://api.apify.com/v2/acts/scraperlink~google-search-results-serp-scraper/run-sync-get-dataset-items?token=YOUR_TOKEN_HERE` |
| Body (JSON) | `{"keyword": "Product Manager New York site:linkedin.com/in", "limit": "20", "page": 1, "start": 1}` — **the keyword should be dynamically mapped from the current batch item** (template has a hardcoded placeholder) |
| Operation | Synchronous run – waits for the actor to complete and returns dataset items |
| Edge cases | Apify token invalid or expired → 401; actor timeout → HTTP 504; rate limiting → 429; `limit` must be a string per the actor's schema |

#### Parse Apify Response
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.code` (v2) |
| Language | JavaScript |
| Logic | Iterates over each page in `items`, then each result in `page.json.results`. Filters for URLs containing `linkedin.com/in`. Outputs objects with: `search_term`, `linkedin_url`, `title`, `description`, `source` ("Apify Google Search"), `collected_at` (ISO timestamp) |
| Edge cases | If Apify returns an empty `results` array, the code returns zero items (no error); if `results` is undefined, the `|| []` fallback prevents a crash |

#### Append Profiles to Raw Sheet
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | Append |
| Sheet | **Raw Profiles** |
| Columns mapped | Collected At, Search Term, Linkedin URL, Title, Description, Enriched (hardcoded `"No"`) |
| Document | Linkedin_Leads_Generation and Follow Up (placeholder `YOUR_SHEET_ID`) |
| Edge cases | Duplicate LinkedIn URLs can be appended if not de-duplicated upstream; column schema mismatch will cause a write error |

#### Update Input Sheet Status
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.7) |
| Operation | Update |
| Sheet | **Input_List** (gid=0) |
| Matching column | `row_number` |
| Columns updated | Status → `"Done"`, Last_Run_At → current timestamp, Total_Profiles_Found → count of parsed profiles |
| Loop-back | Output → Batch LinkedIn Searches (to process the next input row) |
| Edge cases | If `row_number` is not resolved correctly, the update may target the wrong row or fail silently |

---

### Block 3 – Load Profiles for Enrichment

**Overview:** Reads all rows from the **Raw Profiles** sheet and filters for those where `Enriched` contains `"No"`, meaning the profile has not yet been enriched. A SplitInBatches node then processes them one by one.

**Nodes Involved:**
- Read Unenriched Profiles
- Check Profile Unenriched
- Batch Process Profiles

#### Read Unenriched Profiles
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | Read |
| Sheet | **Raw Profiles** |
| Document | Linkedin_Leads_Generation and Follow Up (placeholder `YOUR_SHEET_ID`) |

#### Check Profile Unenriched
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **contains** `"No"` on the `Enriched` column |
| Loose type validation | Enabled |
| True output → | Batch Process Profiles |
| False output → | No connection (already enriched profiles are skipped) |

#### Batch Process Profiles
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.splitInBatches` (v3) |
| Output 0 (done) | Unused |
| Output 1 (each item) → | Post LinkedIn Profile Enrichment |
| Loop-back | Receives from Update Raw Profile Enrichment after each profile is processed |

---

### Block 4 – LinkedIn Data Scraping

**Overview:** For each un-enriched profile, two sequential Apify actors are called: first the **LinkedIn Profile Enrichment** actor (to get name, headline, company, designation), then the **LinkedIn to Email** actor (to retrieve the professional email address). An IF node checks whether the enrichment API actually returned data; if not, the profile is skipped.

**Nodes Involved:**
- Post LinkedIn Profile Enrichment
- Post LinkedIn to Email API
- Check Enrichment Exists

#### Post LinkedIn Profile Enrichment
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.httpRequest` (v4.2) |
| Method | POST |
| URL | `https://api.apify.com/v2/acts/anchor~linkedin-profile-enrichment/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN` |
| Body | `{"startUrls": [{"url": "<current item's LinkedIn URL>"}]}` — the URL expression is a template placeholder that must be mapped to the current batch item's `linkedin_url` field |
| Output → | Post LinkedIn to Email API |
| Edge cases | Actor may return an empty dataset if the profile is private or restricted; API token must have access to the `anchor~linkedin-profile-enrichment` actor |

#### Post LinkedIn to Email API
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.httpRequest` (v4.2) |
| Error handling | `continueRegularOutput` — if the API call fails (e.g., no email found), execution continues with empty output |
| `alwaysOutputData` | true — ensures at least one empty item is emitted on failure |
| Method | POST |
| URL | `https://api.apify.com/v2/acts/anchor~linkedin-to-email/run-sync-get-dataset-items?token=YOUR_APIFY_TOKEN` |
| Body | `{"startUrls": [{"url": "<current item's LinkedIn URL>", "id": "1"}]}` |
| Output → | Check Enrichment Exists |
| Edge cases | If the actor cannot find an email, the response dataset may be empty; the `continueRegularOutput` setting prevents the workflow from stopping |

#### Check Enrichment Exists
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **exists** on a field derived from the enrichment response (template placeholder `YOUR_SHEET_ID` – in practice, the email or name field) |
| True output (0) | No connection — enrichment data already exists in the sheet (duplicate check) |
| False output (1) → | Append Enriched Data to Leads — new enrichment data is saved |
| Edge cases | If the leftValue expression is misconfigured, every item may route to the false branch (always appending, potentially creating duplicates) |

---

### Block 5 – Save Enriched Data

**Overview:** Appends the enriched lead data (Name, Email, Company, Headline, Designation, LinkedIn_URL) to the **Leads** sheet with `Email Sent = "No"`. Then updates the corresponding row in the **Raw Profiles** sheet, setting `Enriched = "Yes"` to prevent re-processing.

**Nodes Involved:**
- Append Enriched Data to Leads
- Update Raw Profile Enrichment

#### Append Enriched Data to Leads
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | Append |
| Sheet | **Leads** |
| Columns mapped | Name, LinkedIn_URL, Designation, Company, Headline, Email, Email Sent (hardcoded `"No"`) |
| Hidden/removed columns | Phone, Email_1, Email_2, Email_3, Response (present in schema but removed from mapping) |
| Output → | Update Raw Profile Enrichment |

#### Update Raw Profile Enrichment
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.7) |
| Operation | Update |
| Sheet | **Raw Profiles** |
| Matching column | `row_number` |
| Columns updated | Enriched → `"Yes"` |
| Loop-back | Output → Batch Process Profiles (process next un-enriched profile) |
| Edge cases | If `row_number` cannot be resolved, the update will fail |

---

### Block 6 – Prepare for Follow-Up Messaging

**Overview:** Reads all rows from the **Leads** sheet and filters for profiles where the enrichment status indicates they still need AI-generated emails (the IF checks if a status column contains `"No"`). Profiles that pass the filter are batched for Gemini processing.

**Nodes Involved:**
- Read Enriched Profiles
- Check Profile Enriched
- Batch Process Enriched Profiles

#### Read Enriched Profiles
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | Read |
| Sheet | **Leads** |
| Document | Linkedin_Leads_Generation and Follow Up (placeholder `YOUR_SHEET_ID`) |

#### Check Profile Enriched
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **contains** `"No"` on a status field (template placeholder `YOUR_SHEET_ID` – in practice, a column indicating whether AI emails have been generated) |
| True output → | Batch Process Enriched Profiles |
| False output | No connection (profiles already have emails) |

#### Batch Process Enriched Profiles
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.splitInBatches` (v3) |
| Output 0 (done) | Unused |
| Output 1 (each item) → | AI Follow-Up Message Generator |
| Loop-back | Receives from Update Profile Enriched Status after each profile is processed |

---

### Block 7 – Generate Follow-Up Message

**Overview:** Calls Google Gemini (`gemini-2.0-flash-lite`) with a system prompt defining a B2B cold email assistant and three user messages requesting (1) the initial cold email, (2) a first follow-up, and (3) a final follow-up. The three generated email texts are then written into the **Leads** sheet columns `Email_1`, `Email_2`, `Email_3`, and the profile's enrichment status is updated.

**Nodes Involved:**
- AI Follow-Up Message Generator
- Append Follow-Up to Leads
- Update Profile Enriched Status

#### AI Follow-Up Message Generator
| Attribute | Detail |
|-----------|--------|
| Type | `@n8n/n8n-nodes-langchain.googleGemini` (v1) |
| Model | `models/gemini-2.0-flash-lite` |
| JSON output | Enabled |
| System message | "You are a professional B2B cold email assistant. Write short, polite, human emails using only the provided prospect details. Rules: professional tone, no emojis, no salesy language, no buzzwords, no links, do not mention AI, do not invent facts, use only given data, keep concise, output plain text only." |
| User message 1 | "Prospect details: Name: _, Role: _, Company: _, Headline: _. Write the first cold email. Include: a short subject line, email body, signature with name 'Dinakar'" |
| User message 2 | "The prospect did not reply to the first email. Write a short, polite follow-up email. Include subject, body, and signature." |
| User message 3 | "The prospect did not reply to previous emails. Write a final short follow-up email. Be polite and respectful. Include subject, body, and signature." |
| Note | The Name, Role, Company, Headline placeholders in the messages must be dynamically mapped to the current batch item's fields (template uses placeholder expressions) |
| Credential | Google Gemini API key (configured at the credential level) |
| Edge cases | If Gemini returns malformed JSON (when `jsonOutput` is true but the model produces invalid JSON), the node will error; if the prospect fields are empty, the model may generate generic content |

#### Append Follow-Up to Leads
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | appendOrUpdate |
| Sheet | **Leads** |
| Matching column | LinkedIn_URL |
| Columns mapped | Email_1, Email_2, Email_3 (from Gemini output), LinkedIn_URL (from current item) |
| Output → | Update Profile Enriched Status |

#### Update Profile Enriched Status
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | Update |
| Sheet | **Profile from APify** (a separate sheet within the same document) |
| Matching column | LinkedIn_URL |
| Columns updated | Enriched → `"Yes"` |
| Loop-back | Output → Batch Process Enriched Profiles (process next profile) |

---

### Block 8 – Start Email Outreach

**Overview:** A **manual trigger** initiates the outreach phase. The workflow reads all leads from the **Leads** sheet, filters for those where `Email Sent` contains `"No"` (i.e., no email sent yet), and batches them for sequential Gmail sending.

**Nodes Involved:**
- Manual Outreach Starting Point
- Read Leads for Outreach
- Check Email Sent Status
- Batch Process Email Leads

#### Manual Outreach Starting Point
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.manualTrigger` (v1) |
| Role | User clicks "Execute Workflow" in n8n to start outreach |
| Output → | Read Leads for Outreach |

#### Read Leads for Outreach
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | Read |
| Sheet | **Leads** |
| Credential | Google Sheets OAuth2 |
| Output → | Check Email Sent Status |

#### Check Email Sent Status
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **contains** `"No"` on the `Email Sent` column |
| True output → | Batch Process Email Leads |
| False output | No connection (leads already emailed are skipped) |

#### Batch Process Email Leads
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.splitInBatches` (v3) |
| Output 0 (done) | Unused |
| Output 1 (each item) → | Send Initial Email via Gmail |
| Loop-back | Receives from Append Email 1 Response, Append Email 2 Response, and Append Email 3 Response (after each lead's email cycle completes) |

---

### Block 9 – Email 1: Send & Check Response

**Overview:** Sends the first cold email via Gmail's `sendAndWait` operation with an embedded response form. The workflow pauses until the recipient responds or the wait time expires. An IF node checks whether a response was received; if yes, the response is recorded and the lead's cycle ends. If no response, the workflow proceeds to a 1-day wait before the second email.

**Nodes Involved:**
- Send Initial Email via Gmail
- Check Response to Email 1
- Append Email 1 Response

#### Send Initial Email via Gmail
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.2) |
| Operation | sendAndWait |
| Subject | "First Mail" (hardcoded; should ideally reference `Email_1` subject from Gemini output) |
| Message body | Dynamically mapped from the `Email_1` column (template has placeholder expression) |
| Response type | customForm |
| Form fields | Radio: "Interested?" (Yes/No, default "Yes", required); Text: "Notice Period" (placeholder: "Enter your Notice period") |
| Wait limit | 0.1 minutes (6 seconds) — this is the `limitWaitTime` for the sendAndWait operation, not the follow-up delay |
| Attribution | Disabled (`appendAttribution: false`) |
| Button label | "Respond" |
| Credential | Gmail OAuth2 |
| Webhook ID | `9f7a0518-b399-4d36-9833-70409355af8f` |
| Output → | Check Response to Email 1 |
| Edge cases | If the lead's email address is invalid, Gmail will return a delivery error; if the recipient never opens the form, the sendAndWait will time out after 0.1 minutes |

#### Check Response to Email 1
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | Object **is not empty** on a field derived from the Gmail response (template placeholder – in practice, the form response data) |
| True output (0) → | Append Email 1 Response |
| False output (1) → | Wait 1 Day for Follow-Up 2 |

#### Append Email 1 Response
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | appendOrUpdate |
| Sheet | **Leads** |
| Matching column | LinkedIn_URL |
| Columns updated | Response → `"Interested : <form field value>\n\nNotice period : <form field value>"`, Email Sent → `"Yes"` |
| Credential | Google Sheets OAuth2 |
| Loop-back | Output → Batch Process Email Leads (next lead) |

---

### Block 10 – Follow-Up Email 2

**Overview:** After 1 day with no response to the first email, the workflow sends a second follow-up email with the same embedded form. Response handling mirrors Block 9: if the recipient responds, the answer is recorded; if not, the workflow waits another day for the final follow-up.

**Nodes Involved:**
- Wait 1 Day for Follow-Up 2
- Send Follow-Up Email 2
- Check Response to Email 2
- Append Email 2 Response

#### Wait 1 Day for Follow-Up 2
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.wait` (v1.1) |
| Duration | 1 day |
| Webhook ID | `d7bca42f-de7e-410c-b4d9-33f879c84d6c` |
| Output → | Send Follow-Up Email 2 |

#### Send Follow-Up Email 2
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.2) |
| Operation | sendAndWait |
| Subject | "Second Mail" |
| Message body | Dynamically mapped from `Email_2` column |
| Response type | customForm (same fields as Email 1) |
| Wait limit | 0.2 minutes (12 seconds) |
| Credential | Gmail OAuth2 |
| Webhook ID | `23989cf2-ad15-4c1e-9480-51b0119efdba` |
| Output → | Check Response to Email 2 |

#### Check Response to Email 2
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **is not empty** on the Gmail form response field |
| True output (0) → | Append Email 2 Response |
| False output (1) → | Wait 1 Day for Final Follow-Up |

#### Append Email 2 Response
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | appendOrUpdate |
| Sheet | **Leads** |
| Matching column | LinkedIn_URL |
| Columns updated | Response → `"Interested : <value>\n\nNotice period : <value>"`, Email 2 Sent → `"Yes"` |
| Credential | Google Sheets OAuth2 |
| Loop-back | Output → Batch Process Email Leads |

---

### Block 11 – Final Follow-Up Email 3

**Overview:** After another 1-day wait with no response to the second email, the workflow sends the third and final email. Regardless of whether a response is received, the result is recorded and the lead's email cycle ends.

**Nodes Involved:**
- Wait 1 Day for Final Follow-Up
- Send Final Follow-Up Email 3
- Check Response to Email 3
- Append Email 3 Response

#### Wait 1 Day for Final Follow-Up
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.wait` (v1.1) |
| Duration | 1 day |
| Webhook ID | `d7bca42f-de7e-410c-b4d9-33f879c84d6c` (same as Follow-Up 2) |
| Output → | Send Final Follow-Up Email 3 |

#### Send Final Follow-Up Email 3
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.gmail` (v2.2) |
| Operation | sendAndWait |
| Subject | "Third Mail" |
| Message body | Dynamically mapped from `Email_3` column |
| Response type | customForm (same fields as previous emails) |
| Wait limit | 0.3 minutes (18 seconds) |
| Credential | Gmail OAuth2 |
| Webhook ID | `23989cf2-ad15-4c1e-9480-51b0119efdba` |
| Output → | Check Response to Email 3 |

#### Check Response to Email 3
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.if` (v2.3) |
| Condition | String **is not empty** on the Gmail form response field |
| True output (0) → | Append Email 3 Response |
| False output (1) | No connection (no response → lead cycle ends without recorded response) |

#### Append Email 3 Response
| Attribute | Detail |
|-----------|--------|
| Type | `n8n-nodes-base.googleSheets` (v4.5) |
| Operation | appendOrUpdate |
| Sheet | **Leads** |
| Matching column | LinkedIn_URL |
| Columns updated | Response → `"Interested : <value>\n\nNotice period : <value>"`, Email 3 Sent → `"Yes"` |
| Credential | Google Sheets OAuth2 |
| Loop-back | Output → Batch Process Email Leads (next lead) |

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Sticky Note | stickyNote (v1) | Overview documentation | — | — | ## Automate LinkedIn lead discovery, enrichment, and email follow-ups using Apify and Google Sheets. How it works: 1. Loads LinkedIn profiles that need enrichment and processes them in batches. 2. Scrapes additional data for each LinkedIn profile using Apify. 3. Saves the enriched data back to Google Sheets and updates their status. 4. Starts a manual trigger for a series of email follow-ups based on data from Google Sheets. 5. Sends follow-up emails at scheduled intervals and saves responses. 6. Checks for responses to emails and proceeds according to the specified conditions. Setup steps: Ensure Apify API key is configured for LinkedIn scraping. Setup Google Sheets API credentials. Configure Gmail API for sending follow-up emails. Customization: Adjust the scraping frequency and email content as per business needs. GSheets Link: https://docs.google.com/spreadsheets/d/1JlFjEo7yhuHprDwa41kxogXsLTe02K3uUjFgjtumrzQ/edit?usp=sharing |
| Sticky Note1 | stickyNote (v1) | Block annotation | — | — | Load profiles for enrichment. Initial loading of unenriched LinkedIn profiles to be processed in batches. |
| Sticky Note2 | stickyNote (v1) | Block annotation | — | — | LinkedIn data scraping. Scraping LinkedIn profile data using Apify for each profile in the batch. |
| Sticky Note3 | stickyNote (v1) | Block annotation | — | — | Save enriched data. Save the enriched data back into Google Sheets and mark them as enriched. |
| Sticky Note4 | stickyNote (v1) | Block annotation | — | — | Prepare for follow-up messaging. Load previously enriched profiles, check conditions, and prepare for message generation. |
| Sticky Note5 | stickyNote (v1) | Block annotation | — | — | Generate follow-up message. Generate follow-up messages using Google Gemini and update leads as enriched. |
| Sticky Note6 | stickyNote (v1) | Block annotation | — | — | Fetch initial lead data. Fetch lead records from Google Sheets to determine processing needs. |
| Sticky Note7 | stickyNote (v1) | Block annotation | — | — | Apify scraping process. Handles the LinkedIn search and scraping logic through Apify. |
| Sticky Note8 | stickyNote (v1) | Block annotation | — | — | Start email outreach. Initiates the email sending sequence for outreach processes. |
| Sticky Note9 | stickyNote (v1) | Block annotation | — | — | Email sending and response check. Gmail nodes send emails and check responses for different stages of follow-ups. |
| Sticky Note10 | stickyNote (v1) | Block annotation | — | — | Response handling for follow-up 1. Processes the response for the first follow-up email in the series. |
| Sticky Note11 | stickyNote (v1) | Block annotation | — | — | Second follow-up email process. Handles sending and checking responses for the second email follow-up. |
| Sticky Note12 | stickyNote (v1) | Block annotation | — | — | Final follow-up email process. Completes the follow-up sequence with the third and final email and response handling. |
| Read Input Leads | googleSheets (v4.7) | Read search keywords from Input_List sheet | — (entry) | Check Input Sheet is Empty | Fetch initial lead data. Fetch lead records from Google Sheets to determine processing needs. |
| Check Input Sheet is Empty | if (v2.3) | Gate: proceed only if input rows are unprocessed | Read Input Leads | Batch LinkedIn Searches (true) | Fetch initial lead data. Fetch lead records from Google Sheets to determine processing needs. |
| Batch LinkedIn Searches | splitInBatches (v3) | Iterate over each input keyword row | Check Input Sheet is Empty (true), Update Input Sheet Status (loop) | Post Apify Search Request (output 1) | Apify scraping process. Handles the LinkedIn search and scraping logic through Apify. |
| Post Apify Search Request | httpRequest (v4.2) | POST to Apify Google Search SERP Scraper | Batch LinkedIn Searches | Parse Apify Response | Apify scraping process. Handles the LinkedIn search and scraping logic through Apify. |
| Parse Apify Response | code (v2) | Extract LinkedIn URLs from SERP results | Post Apify Search Request | Append Profiles to Raw Sheet | Apify scraping process. Handles the LinkedIn search and scraping logic through Apify. |
| Append Profiles to Raw Sheet | googleSheets (v4.5) | Append discovered LinkedIn URLs to Raw Profiles sheet | Parse Apify Response | Update Input Sheet Status | |
| Update Input Sheet Status | googleSheets (v4.7) | Mark input row as Done with timestamp and count | Append Profiles to Raw Sheet | Batch LinkedIn Searches (loop) | |
| Read Unenriched Profiles | googleSheets (v4.5) | Read all rows from Raw Profiles sheet | — (entry) | Check Profile Unenriched | Load profiles for enrichment. Initial loading of unenriched LinkedIn profiles to be processed in batches. |
| Check Profile Unenriched | if (v2.3) | Gate: proceed only if Enriched = "No" | Read Unenriched Profiles | Batch Process Profiles (true) | Load profiles for enrichment. Initial loading of unenriched LinkedIn profiles to be processed in batches. |
| Batch Process Profiles | splitInBatches (v3) | Iterate over each un-enriched profile | Check Profile Unenriched (true), Update Raw Profile Enrichment (loop) | Post LinkedIn Profile Enrichment (output 1) | Load profiles for enrichment. Initial loading of unenriched LinkedIn profiles to be processed in batches. |
| Post LinkedIn Profile Enrichment | httpRequest (v4.2) | POST to Apify LinkedIn Profile Enrichment actor | Batch Process Profiles | Post LinkedIn to Email API | LinkedIn data scraping. Scraping LinkedIn profile data using Apify for each profile in the batch. |
| Post LinkedIn to Email API | httpRequest (v4.2) | POST to Apify LinkedIn-to-Email actor | Post LinkedIn Profile Enrichment | Check Enrichment Exists | LinkedIn data scraping. Scraping LinkedIn profile data using Apify for each profile in the batch. |
| Check Enrichment Exists | if (v2.3) | Gate: skip if enrichment data already present | Post LinkedIn to Email API | Append Enriched Data to Leads (false) | LinkedIn data scraping. Scraping LinkedIn profile data using Apify for each profile in the batch. |
| Append Enriched Data to Leads | googleSheets (v4.5) | Append enriched lead data to Leads sheet | Check Enrichment Exists (false) | Update Raw Profile Enrichment | Save enriched data. Save the enriched data back into Google Sheets and mark them as enriched. |
| Update Raw Profile Enrichment | googleSheets (v4.7) | Set Enriched = "Yes" on the Raw Profiles row | Append Enriched Data to Leads | Batch Process Profiles (loop) | Save enriched data. Save the enriched data back into Google Sheets and mark them as enriched. |
| Read Enriched Profiles | googleSheets (v4.5) | Read leads from Leads sheet | — (entry) | Check Profile Enriched | Prepare for follow-up messaging. Load previously enriched profiles, check conditions, and prepare for message generation. |
| Check Profile Enriched | if (v2.3) | Gate: proceed only if emails not yet generated | Read Enriched Profiles | Batch Process Enriched Profiles (true) | Prepare for follow-up messaging. Load previously enriched profiles, check conditions, and prepare for message generation. |
| Batch Process Enriched Profiles | splitInBatches (v3) | Iterate over each lead needing email generation | Check Profile Enriched (true), Update Profile Enriched Status (loop) | AI Follow-Up Message Generator (output 1) | Prepare for follow-up messaging. Load previously enriched profiles, check conditions, and prepare for message generation. |
| AI Follow-Up Message Generator | googleGemini (v1) | Generate 3 cold-email variants via Gemini | Batch Process Enriched Profiles | Append Follow-Up to Leads | Generate follow-up message. Generate follow-up messages using Google Gemini and update leads as enriched. |
| Append Follow-Up to Leads | googleSheets (v4.5) | Write Email_1, Email_2, Email_3 columns to Leads | AI Follow-Up Message Generator | Update Profile Enriched Status | Generate follow-up message. Generate follow-up messages using Google Gemini and update leads as enriched. |
| Update Profile Enriched Status | googleSheets (v4.5) | Mark Profile from APify sheet as Enriched = "Yes" | Append Follow-Up to Leads | Batch Process Enriched Profiles (loop) | Generate follow-up message. Generate follow-up messages using Google Gemini and update leads as enriched. |
| Manual Outreach Starting Point | manualTrigger (v1) | Manual trigger for email outreach phase | — (manual) | Read Leads for Outreach | Start email outreach. Initiates the email sending sequence for outreach processes. |
| Read Leads for Outreach | googleSheets (v4.5) | Read leads from Leads sheet for emailing | Manual Outreach Starting Point | Check Email Sent Status | Start email outreach. Initiates the email sending sequence for outreach processes. |
| Check Email Sent Status | if (v2.3) | Gate: proceed only if Email Sent = "No" | Read Leads for Outreach | Batch Process Email Leads (true) | Start email outreach. Initiates the email sending sequence for outreach processes. |
| Batch Process Email Leads | splitInBatches (v3) | Iterate over each lead to email | Check Email Sent Status (true), Append Email 1/2/3 Response (loop) | Send Initial Email via Gmail (output 1) | Start email outreach. Initiates the email sending sequence for outreach processes. |
| Send Initial Email via Gmail | gmail (v2.2) | Send first cold email with response form | Batch Process Email Leads | Check Response to Email 1 | Email sending and response check. Gmail nodes send emails and check responses for different stages of follow-ups. |
| Check Response to Email 1 | if (v2.3) | Check if recipient responded to Email 1 | Send Initial Email via Gmail | Append Email 1 Response (true), Wait 1 Day for Follow-Up 2 (false) | Email sending and response check. Gmail nodes send emails and check responses for different stages of follow-ups. |
| Append Email 1 Response | googleSheets (v4.5) | Record Email 1 response and mark Email Sent = "Yes" | Check Response to Email 1 (true) | Batch Process Email Leads (loop) | Response handling for follow-up 1. Processes the response for the first follow-up email in the series. |
| Wait 1 Day for Follow-Up 2 | wait (v1.1) | Wait 1 day before second email | Check Response to Email 1 (false) | Send Follow-Up Email 2 | Response handling for follow-up 1. Processes the response for the first follow-up email in the series. |
| Send Follow-Up Email 2 | gmail (v2.2) | Send second follow-up email with response form | Wait 1 Day for Follow-Up 2 | Check Response to Email 2 | Second follow-up email process. Handles sending and checking responses for the second email follow-up. |
| Check Response to Email 2 | if (v2.3) | Check if recipient responded to Email 2 | Send Follow-Up Email 2 | Append Email 2 Response (true), Wait 1 Day for Final Follow-Up (false) | Second follow-up email process. Handles sending and checking responses for the second email follow-up. |
| Append Email 2 Response | googleSheets (v4.5) | Record Email 2 response and mark Email 2 Sent = "Yes" | Check Response to Email 2 (true) | Batch Process Email Leads (loop) | Second follow-up email process. Handles sending and checking responses for the second email follow-up. |
| Wait 1 Day for Final Follow-Up | wait (v1.1) | Wait 1 day before third email | Check Response to Email 2 (false) | Send Final Follow-Up Email 3 | Second follow-up email process. Handles sending and checking responses for the second email follow-up. |
| Send Final Follow-Up Email 3 | gmail (v2.2) | Send third and final email with response form | Wait 1 Day for Final Follow-Up | Check Response to Email 3 | Final follow-up email process. Completes the follow-up sequence with the third and final email and response handling. |
| Check Response to Email 3 | if (v2.3) | Check if recipient responded to Email 3 | Send Final Follow-Up Email 3 | Append Email 3 Response (true) | Final follow-up email process. Completes the follow-up sequence with the third and final email and response handling. |
| Append Email 3 Response | googleSheets (v4.5) | Record Email 3 response and mark Email 3 Sent = "Yes" | Check Response to Email 3 (true) | Batch Process Email Leads (loop) | Final follow-up email process. Completes the follow-up sequence with the third and final email and response handling. |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites

1. **Apify account** with API token and access to these actors:
   - `scraperlink~google-search-results-serp-scraper`
   - `anchor~linkedin-profile-enrichment`
   - `anchor~linkedin-to-email`
2. **Google account** with:
   - Google Sheets API enabled
   - Gmail API enabled
3. **Google Gemini API key** with access to `models/gemini-2.0-flash-lite`
4. **Google Sheets document** created with the following sheets/tabs:
   - **Input_List**: Columns — Keyword, Location, Status, Last_Run_At, Total_Profiles_Found, Notes
   - **Raw Profiles**: Columns — Collected At, Search Term, Linkedin URL, Title, Description, Enriched
   - **Leads**: Columns — Name, LinkedIn_URL, Designation, Company, Headline, Email, Phone, Email_1, Email_2, Email_3, Email Sent, Email 2 Sent, Email 3 Sent, Response
   - **Profile from APify**: Columns — (same as Raw Profiles or a subset including LinkedIn_URL and Enriched)

### Step-by-Step Build

#### Phase 1: Lead Discovery & Collection

1. **Create** a **Google Sheets (Read)** node named `Read Input Leads`.
   - Operation: Read
   - Document: Select your Google Sheets document
   - Sheet: Input_List
   - Credential: Google Sheets OAuth2

2. **Create** an **IF** node named `Check Input Sheet is Empty`.
   - Condition: String → is empty → on the `Status` column from the input
   - Connect: Read Input Leads → Check Input Sheet is Empty

3. **Create** a **Split In Batches** node named `Batch LinkedIn Searches`.
   - Connect: Check Input Sheet is Empty (true output) → Batch LinkedIn Searches

4. **Create** an **HTTP Request** node named `Post Apify Search Request`.
   - Method: POST
   - URL: `https://api.apify.com/v2/acts/scraperlink~google-search-results-serp-scraper/run-sync-get-dataset-items?token=<YOUR_APIFY_TOKEN>`
   - Send Body: Yes, specify body as JSON
   - JSON Body:
     ```
     {
       "keyword": "{{ $json.Keyword }} site:linkedin.com/in",
       "limit": "20",
       "page": 1,
       "start": 1
     }
     ```
   - Connect: Batch LinkedIn Searches (output 1) → Post Apify Search Request

5. **Create** a **Code** node named `Parse Apify Response`.
   - Language: JavaScript
   - Code:
     ```javascript
     const output = [];
     for (const page of items) {
       const searchTerm = page.json.search_term || null;
       const results = page.json.results || [];
       for (const r of results) {
         if (!r.url || !r.url.includes('linkedin.com/in')) continue;
         output.push({
           json: {
             search_term: searchTerm,
             linkedin_url: r.url,
             title: r.title || null,
             description: r.description || null,
             source: "Apify Google Search",
             collected_at: new Date().toISOString()
           }
         });
       }
     }
     return output;
     ```
   - Connect: Post Apify Search Request → Parse Apify Response

6. **Create** a **Google Sheets** node named `Append Profiles to Raw Sheet`.
   - Operation: Append
   - Document: Your Google Sheets document
   - Sheet: Raw Profiles
   - Columns mapped: Collected At (from parsed item), Search Term, Linkedin URL, Title, Description, Enriched (static `"No"`)
   - Connect: Parse Apify Response → Append Profiles to Raw Sheet

7. **Create** a **Google Sheets** node named `Update Input Sheet Status`.
   - Operation: Update
   - Document: Your Google Sheets document
   - Sheet: Input_List
   - Matching column: row_number
   - Columns updated: Status → `"Done"`, Last_Run_At → current timestamp, Total_Profiles_Found → count
   - Connect: Append Profiles to Raw Sheet → Update Input Sheet Status
   - Connect: Update Input Sheet Status → Batch LinkedIn Searches (loop back to process next input row)

#### Phase 2: Profile Enrichment

8. **Create** a **Google Sheets (Read)** node named `Read Unenriched Profiles`.
   - Operation: Read
   - Document: Your Google Sheets document
   - Sheet: Raw Profiles

9. **Create** an **IF** node named `Check Profile Unenriched`.
   - Condition: String → contains → `"No"` on the `Enriched` column
   - Connect: Read Unenriched Profiles → Check Profile Unenriched

10. **Create** a **Split In Batches** node named `Batch Process Profiles`.
    - Connect: Check Profile Unenriched (true) → Batch Process Profiles

11. **Create** an **HTTP Request** node named `Post LinkedIn Profile Enrichment`.
    - Method: POST
    - URL: `https://api.apify.com/v2/acts/anchor~linkedin-profile-enrichment/run-sync-get-dataset-items?token=<YOUR_APIFY_TOKEN>`
    - JSON Body:
      ```
      {
        "startUrls": [
          { "url": "{{ $json.linkedin_url }}" }
        ]
      }
      ```
    - Connect: Batch Process Profiles (output 1) → Post LinkedIn Profile Enrichment

12. **Create** an **HTTP Request** node named `Post LinkedIn to Email API`.
    - Method: POST
    - URL: `https://api.apify.com/v2/acts/anchor~linkedin-to-email/run-sync-get-dataset-items?token=<YOUR_APIFY_TOKEN>`
    - JSON Body:
      ```
      {
        "startUrls": [
          { "url": "{{ $json.linkedin_url }}", "id": "1" }
        ]
      }
      ```
    - Error handling: Continue (so the workflow proceeds even if the email actor returns nothing)
    - Always output data: Enabled
    - Connect: Post LinkedIn Profile Enrichment → Post LinkedIn to Email API

13. **Create** an **IF** node named `Check Enrichment Exists`.
    - Condition: String → exists on the email/name field from the enrichment response
    - True output: No connection (data already exists → skip)
    - False output → Append Enriched Data to Leads
    - Connect: Post LinkedIn to Email API → Check Enrichment Exists

14. **Create** a **Google Sheets** node named `Append Enriched Data to Leads`.
    - Operation: Append
    - Document: Your Google Sheets document
    - Sheet: Leads
    - Columns mapped: Name, LinkedIn_URL, Designation, Company, Headline, Email (from enrichment responses), Email Sent (static `"No"`)
    - Connect: Check Enrichment Exists (false) → Append Enriched Data to Leads

15. **Create** a **Google Sheets** node named `Update Raw Profile Enrichment`.
    - Operation: Update
    - Document: Your Google Sheets document
    - Sheet: Raw Profiles
    - Matching column: row_number
    - Columns updated: Enriched → `"Yes"`
    - Connect: Append Enriched Data to Leads → Update Raw Profile Enrichment
    - Connect: Update Raw Profile Enrichment → Batch Process Profiles (loop back)

#### Phase 3: AI Message Generation

16. **Create** a **Google Sheets (Read)** node named `Read Enriched Profiles`.
    - Operation: Read
    - Document: Your Google Sheets document
    - Sheet: Leads

17. **Create** an **IF** node named `Check Profile Enriched`.
    - Condition: String → contains → `"No"` on a column indicating emails have not been generated (e.g., a custom status column or check if Email_1 is empty)
    - Connect: Read Enriched Profiles → Check Profile Enriched

18. **Create** a **Split In Batches** node named `Batch Process Enriched Profiles`.
    - Connect: Check Profile Enriched (true) → Batch Process Enriched Profiles

19. **Create** a **Google Gemini (LangChain)** node named `AI Follow-Up Message Generator`.
    - Model: `models/gemini-2.0-flash-lite`
    - JSON Output: Enabled
    - System Message:
      ```
      You are a professional B2B cold email assistant.
      Write short, polite, human emails using only the provided prospect details.
      Rules:
      - Professional and simple tone
      - No emojis
      - No salesy language
      - No buzzwords
      - No links
      - Do not mention AI or automation
      - Do not invent any facts
      - Use only the given data
      - Keep emails concise and natural
      - Output plain text only
      ```
    - Messages (3 values):
      - Message 1: `Prospect details: Name: {{ $json.Name }}, Role: {{ $json.Designation }}, Company: {{ $json.Company }}, Headline: {{ $json.Headline }}. Write the first cold email. Include: a short subject line, email body, signature with name "Dinakar"`
      - Message 2: `Prospect details: Name: {{ $json.Name }}, Role: {{ $json.Designation }}, Company: {{ $json.Company }}, Headline: {{ $json.Headline }}. The prospect did not reply to the first email. Write a short, polite follow-up email. Include subject, body, and signature.`
      - Message 3: `Prospect details: Name: {{ $json.Name }}, Role: {{ $json.Designation }}, Company: {{ $json.Company }}, Headline: {{ $json.Headline }}. The prospect did not reply to previous emails. Write a final short follow-up email. Be polite and respectful. Include subject, body, and signature.`
    - Credential: Google Gemini API key
    - Connect: Batch Process Enriched Profiles (output 1) → AI Follow-Up Message Generator

20. **Create** a **Google Sheets** node named `Append Follow-Up to Leads`.
    - Operation: appendOrUpdate
    - Document: Your Google Sheets document
    - Sheet: Leads
    - Matching column: LinkedIn_URL
    - Columns mapped: Email_1, Email_2, Email_3 (from Gemini output), LinkedIn_URL (from current item)
    - Connect: AI Follow-Up Message Generator → Append Follow-Up to Leads

21. **Create** a **Google Sheets** node named `Update Profile Enriched Status`.
    - Operation: Update
    - Document: Your Google Sheets document
    - Sheet: Profile from APify
    - Matching column: LinkedIn_URL
    - Columns updated: Enriched → `"Yes"`
    - Connect: Append Follow-Up to Leads → Update Profile Enriched Status
    - Connect: Update Profile Enriched Status → Batch Process Enriched Profiles (loop back)

#### Phase 4: Email Outreach

22. **Create** a **Manual Trigger** node named `Manual Outreach Starting Point`.

23. **Create** a **Google Sheets (Read)** node named `Read Leads for Outreach`.
    - Operation: Read
    - Document: Your Google Sheets document
    - Sheet: Leads
    - Credential: Google Sheets OAuth2
    - Connect: Manual Outreach Starting Point → Read Leads for Outreach

24. **Create** an **IF** node named `Check Email Sent Status`.
    - Condition: String → contains → `"No"` on the `Email Sent` column
    - Connect: Read Leads for Outreach → Check Email Sent Status

25. **Create** a **Split In Batches** node named `Batch Process Email Leads`.
    - Connect: Check Email Sent Status (true) → Batch Process Email Leads

26. **Create** a **Gmail** node named `Send Initial Email via Gmail`.
    - Operation: sendAndWait
    - Subject: Expression referencing the subject from `Email_1` (e.g., extract the subject line, or use a static "First Mail")
    - Message: Expression referencing the body from `Email_1`
    - Response type: Custom Form
    - Form fields:
      - Radio: "Interested?" with options Yes/No (default Yes, required)
      - Text: "Notice Period" with placeholder "Enter your Notice period"
    - Limit Wait Time: 0.1 minutes
    - Append Attribution: Disabled
    - Button Label: "Respond"
    - Credential: Gmail OAuth2
    - Connect: Batch Process Email Leads (output 1) → Send Initial Email via Gmail

27. **Create** an **IF** node named `Check Response to Email 1`.
    - Condition: Object → is not empty on the form response data field
    - True → Append Email 1 Response
    - False → Wait 1 Day for Follow-Up 2
    - Connect: Send Initial Email via Gmail → Check Response to Email 1

28. **Create** a **Google Sheets** node named `Append Email 1 Response`.
    - Operation: appendOrUpdate
    - Sheet: Leads
    - Matching column: LinkedIn_URL
    - Columns updated: Response (formatted string with Interested + Notice Period values), Email Sent → `"Yes"`
    - Connect: Check Response to Email 1 (true) → Append Email 1 Response
    - Connect: Append Email 1 Response → Batch Process Email Leads (loop back)

29. **Create** a **Wait** node named `Wait 1 Day for Follow-Up 2`.
    - Duration: 1 day
    - Connect: Check Response to Email 1 (false) → Wait 1 Day for Follow-Up 2

30. **Create** a **Gmail** node named `Send Follow-Up Email 2`.
    - Operation: sendAndWait
    - Subject: From `Email_2` subject line (or static "Second Mail")
    - Message: From `Email_2` body
    - Response type: Custom Form (same fields as Email 1)
    - Limit Wait Time: 0.2 minutes
    - Credential: Gmail OAuth2
    - Connect: Wait 1 Day for Follow-Up 2 → Send Follow-Up Email 2

31. **Create** an **IF** node named `Check Response to Email 2`.
    - Condition: String → is not empty on form response
    - True → Append Email 2 Response
    - False → Wait 1 Day for Final Follow-Up
    - Connect: Send Follow-Up Email 2 → Check Response to Email 2

32. **Create** a **Google Sheets** node named `Append Email 2 Response`.
    - Operation: appendOrUpdate
    - Sheet: Leads
    - Matching column: LinkedIn_URL
    - Columns updated: Response, Email 2 Sent → `"Yes"`
    - Connect: Check Response to Email 2 (true) → Append Email 2 Response
    - Connect: Append Email 2 Response → Batch Process Email Leads (loop back)

33. **Create** a **Wait** node named `Wait 1 Day for Final Follow-Up`.
    - Duration: 1 day
    - Connect: Check Response to Email 2 (false) → Wait 1 Day for Final Follow-Up

34. **Create** a **Gmail** node named `Send Final Follow-Up Email 3`.
    - Operation: sendAndWait
    - Subject: From `Email_3` subject line (or static "Third Mail")
    - Message: From `Email_3` body
    - Response type: Custom Form (same fields)
    - Limit Wait Time: 0.3 minutes
    - Credential: Gmail OAuth2
    - Connect: Wait 1 Day for Final Follow-Up → Send Final Follow-Up Email 3

35. **Create** an **IF** node named `Check Response to Email 3`.
    - Condition: String → is not empty on form response
    - True → Append Email 3 Response
    - False → No connection (end of cycle for this lead)
    - Connect: Send Final Follow-Up Email 3 → Check Response to Email 3

36. **Create** a **Google Sheets** node named `Append Email 3 Response`.
    - Operation: appendOrUpdate
    - Sheet: Leads
    - Matching column: LinkedIn_URL
    - Columns updated: Response, Email 3 Sent → `"Yes"`
    - Connect: Check Response to Email 3 (true) → Append Email 3 Response
    - Connect: Append Email 3 Response → Batch Process Email Leads (loop back)

### Credential Configuration Summary

| Credential | Type | Required For | Setup Notes |
|-----------|------|-------------|-------------|
| Google Sheets OAuth2 | OAuth2 | All Google Sheets nodes | Must have edit access to the target spreadsheet |
| Gmail OAuth2 | OAuth2 | All Gmail sendAndWait nodes | Must have send permissions; the Gmail account is the sender |
| Apify API Token | Bearer Token (in URL) | All HTTP Request nodes calling Apify | Replace `YOUR_TOKEN_HERE` / `YOUR_APIFY_TOKEN` in URLs |
| Google Gemini API Key | API Key | AI Follow-Up Message Generator | Configured in the LangChain credential for the Gemini node |

### Important Configuration Notes

- **All `YOUR_SHEET_ID` placeholders** in the template must be replaced with the actual Google Sheets document ID and specific sheet GIDs.
- **All `YOUR_APIFY_TOKEN` / `YOUR_TOKEN_HERE` placeholders** must be replaced with a valid Apify API token.
- **The `sendAndWait` limit wait times** (0.1, 0.2, 0.3 minutes) are extremely short and intended for testing. In production, increase these to allow recipients adequate time to respond via the embedded form before the execution resumes on the "no response" path.
- **The Gmail `sendAndWait` form response** requires the n8n instance to be publicly accessible (or have a tunnel configured) so that the webhook embedded in the email can receive the recipient's form submission.
- **The Gemini node's three user messages** are sent in a single API call. The model generates all three email variants at once. Ensure the output parsing correctly maps `Email_1`, `Email_2`, and `Email_3` from the JSON response.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|-------------|-----------------|
| The workflow uses three distinct Apify actors. Ensure your Apify subscription covers the compute units required for the Google Search SERP Scraper, LinkedIn Profile Enrichment, and LinkedIn-to-Email actors. | https://apify.com/store |
| The shared Google Sheets template is available for reference and copying. | https://docs.google.com/spreadsheets/d/1JlFjEo7yhuHprDwa41kxogXsLTe02K3uUjFgjtumrzQ/edit?usp=sharing |
| The Gmail `sendAndWait` operation embeds a response form in the email. This requires n8n's webhook functionality to be reachable from the internet. If n8n runs locally, use ngrok or n8n's built-in tunnel. | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.gmail/ |
| The signature name "Dinakar" in the Gemini prompt is hardcoded. Update it to match your sender identity. | AI Follow-Up Message Generator system message |
| The `limitWaitTime` values (0.1, 0.2, 0.3 min) are likely placeholder/test values. Real outreach campaigns should use longer timeouts (e.g., 3–7 days) to give recipients time to respond before the workflow continues to the next follow-up. | Gmail sendAndWait configuration |
| The Apify `run-sync-get-dataset-items` endpoint is synchronous and blocks until the actor finishes. For large batches, consider using the asynchronous run API with a polling loop to avoid HTTP timeouts. | https://docs.apify.com/api/v2 |
| The Check Input Sheet is Empty IF node's condition references a template placeholder (`YOUR_SHEET_ID`). You must replace this with an expression that reads the `Status` column of the current row (e.g., `{{ $json.Status }}`). | Check Input Sheet is Empty node |
| The Google Sheets document contains four tabs: Input_List, Raw Profiles, Leads, and Profile from APify. Make sure all four exist before running the workflow. | Google Sheets template link above |
| The `continueRegularOutput` error handling on the Post LinkedIn to Email API node means that if no email is found for a profile, the workflow will still continue. The subsequent Check Enrichment Exists node will route such items to the false branch only if enrichment data is present. Profiles without an email will be skipped. | Post LinkedIn to Email API node |
| The workflow's execution order is set to `v1` (the legacy mode). If migrating to a newer n8n version, verify that all node versions remain compatible. | Workflow settings |
| The Gemini model `gemini-2.0-flash-lite` is a lightweight model. For more nuanced or higher-quality email copy, consider switching to `gemini-1.5-pro` or `gemini-2.0-flash`. | AI Follow-Up Message Generator node |