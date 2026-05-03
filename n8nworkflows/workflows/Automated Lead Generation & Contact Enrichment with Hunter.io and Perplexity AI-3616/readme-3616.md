Automated Lead Generation & Contact Enrichment with Hunter.io and Perplexity AI

https://n8nworkflows.xyz/workflows/automated-lead-generation---contact-enrichment-with-hunter-io-and-perplexity-ai-3616


# Automated Lead Generation & Contact Enrichment with Hunter.io and Perplexity AI

### 1. Workflow Overview

This workflow, titled **Automated Lead Generation & Contact Enrichment with Hunter.io and Perplexity AI**, is designed to automate the process of sourcing, enriching, validating, and logging B2B leads. It targets founders, marketers, and sales development representatives who want to scale lead generation using no-code/low-code tools. The workflow integrates AI-driven company discovery, email enrichment via Hunter.io, and data storage in Google Sheets, optionally triggered via chat or Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts user prompts via manual trigger, chat message, or Telegram.
- **1.2 Company Discovery via AI:** Uses OpenRouter (Perplexity) to generate a list of companies/domains based on the input prompt.
- **1.3 Data Cleaning & Deduplication:** Parses AI output, removes duplicates by comparing with existing Google Sheets data.
- **1.4 Email Enrichment:** Uses Hunter.io to find personal emails for each domain; falls back to AI if no emails found.
- **1.5 Data Validation & Formatting:** Cleans and validates JSON data, slices lead lists into manageable batches.
- **1.6 Data Logging:** Updates Google Sheets with enriched lead data, marking statuses accordingly.
- **1.7 Optional Triggers & Extensions:** Includes optional Telegram trigger and placeholders for Airtable or CRM integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow either manually or via chat/Telegram triggers, capturing the industry/topic prompt for lead generation.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- When chat message received  
- Telegram Trigger (disabled)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts workflow manually for testing  
  - Config: No parameters; triggers on manual click  
  - Outputs to: "Read all 'new' Leads" node  
  - Edge cases: None significant; manual trigger

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Listens for chat input to start lead generation  
  - Config: Webhook enabled with session management  
  - Outputs to: "Company Finder1" node  
  - Edge cases: Webhook failures, invalid input format

- **Telegram Trigger**  
  - Type: Telegram Trigger (disabled)  
  - Role: Optional trigger for Telegram chat input  
  - Config: Requires Telegram Bot credentials  
  - Disabled by default  
  - Edge cases: Telegram API limits, auth errors

---

#### 2.2 Company Discovery via AI

**Overview:**  
Generates a list of relevant companies and domains based on the input prompt using OpenRouter’s Perplexity model.

**Nodes Involved:**  
- OpenRouter Chat Model  
- Company Finder  
- OpenRouter Chat Model1  
- Company Finder1

**Node Details:**

- **OpenRouter Chat Model**  
  - Type: Langchain OpenRouter LLM Chat  
  - Role: Sends prompt to AI model to generate company data  
  - Config: Uses Perplexity Sonar Large model (default)  
  - Inputs: From "If" node or chat trigger  
  - Outputs: To "Company Finder" node  
  - Edge cases: API rate limits, model errors, invalid prompt

- **Company Finder**  
  - Type: Langchain Chain LLM  
  - Role: Processes AI response, extracts company/domain info  
  - Config: Chain of prompts to parse JSON from AI text  
  - Inputs: From "OpenRouter Chat Model"  
  - Outputs: To "Code2" node  
  - Edge cases: Parsing errors, malformed JSON

- **OpenRouter Chat Model1**  
  - Same as "OpenRouter Chat Model" but triggered from chat input  
  - Outputs to "Company Finder1"

- **Company Finder1**  
  - Same as "Company Finder" but for chat-triggered flow  
  - Outputs to "JSON cleaner code"

---

#### 2.3 Data Cleaning & Deduplication

**Overview:**  
Cleans AI output JSON, chunks data into individual items, and removes duplicates by comparing with existing leads in Google Sheets.

**Nodes Involved:**  
- JSON cleaner code  
- Chunk into separate items code  
- Read all from your Google Sheet "Leads List" sheet  
- List existing domains code  
- Merge  
- Keep only non-duplicates code

**Node Details:**

- **JSON cleaner code**  
  - Type: Code  
  - Role: Parses and cleans raw JSON from AI output  
  - Inputs: From "Company Finder1"  
  - Outputs: To "Chunk into separate items code"  
  - Edge cases: JSON parse errors

- **Chunk into separate items code**  
  - Type: Code  
  - Role: Splits array of companies into separate n8n items  
  - Inputs: From "JSON cleaner code"  
  - Outputs: To "Read all from your Google Sheet 'Leads List' sheet" and "Merge"  
  - Edge cases: Empty arrays, malformed data

- **Read all from your Google Sheet "Leads List" sheet**  
  - Type: Google Sheets  
  - Role: Reads existing leads to check for duplicates  
  - Config: Reads entire sheet with header row  
  - Outputs: To "List existing domains code"  
  - Edge cases: Google API quota, sheet access errors

- **List existing domains code**  
  - Type: Code  
  - Role: Extracts list of existing domains from Google Sheets data  
  - Inputs: From "Read all from your Google Sheet"  
  - Outputs: To "Merge"  
  - Edge cases: Empty sheet, missing domain column

- **Merge**  
  - Type: Merge  
  - Role: Combines new AI leads with existing domains list  
  - Inputs: From "Chunk into separate items code" and "List existing domains code"  
  - Outputs: To "Keep only non-duplicates code"  
  - Edge cases: Mismatched input counts

- **Keep only non-duplicates code**  
  - Type: Code  
  - Role: Filters out leads whose domains already exist in Google Sheets  
  - Inputs: From "Merge"  
  - Outputs: To "If2" node  
  - Edge cases: Logic errors, empty input

---

#### 2.4 Email Enrichment

**Overview:**  
Enriches each domain with up to 3 personal emails using Hunter.io; if no emails found, fallback AI attempts to locate general email info.

**Nodes Involved:**  
- Slice down to 10  
- Find real user data with Hunter  
- Clean Hunter output  
- If  
- Code2  
- If1  
- Leads List2  
- Leads List3  
- Leads List4

**Node Details:**

- **Slice down to 10**  
  - Type: Code  
  - Role: Limits lead batch size to 10 to protect API limits  
  - Inputs: From "Read all 'new' Leads" or chat flow  
  - Outputs: To "Find real user data with Hunter"  
  - Edge cases: Empty input, slicing errors

- **Find real user data with Hunter**  
  - Type: Hunter.io node  
  - Role: Queries Hunter.io API for personal emails per domain  
  - Config: Requires Hunter API key credential  
  - Inputs: From "Slice down to 10"  
  - Outputs: To "Clean Hunter output"  
  - Edge cases: API rate limits, invalid API key, no emails found

- **Clean Hunter output**  
  - Type: Code  
  - Role: Normalizes Hunter.io response, extracts relevant email data  
  - Inputs: From "Find real user data with Hunter"  
  - Outputs: To "If" node  
  - Edge cases: Unexpected API response format

- **If**  
  - Type: If  
  - Role: Checks if Hunter found emails or not  
  - Inputs: From "Clean Hunter output"  
  - Outputs:  
    - True: To "Company Finder" (fallback AI for no emails)  
    - False: To "Leads List" (Google Sheets update)  
  - Edge cases: Logic errors, empty input

- **Code2**  
  - Type: Code  
  - Role: Processes fallback AI output or formats data for Google Sheets  
  - Inputs: From "Company Finder"  
  - Outputs: To "If1"  
  - Edge cases: Parsing errors

- **If1**  
  - Type: If  
  - Role: Checks if fallback AI found any emails or info  
  - Inputs: From "Code2"  
  - Outputs:  
    - True: To "Leads List2" (Google Sheets update for fallback data)  
    - False: To "Leads List3" (Google Sheets update for no data)  
  - Edge cases: Logic errors

- **Leads List2, Leads List3, Leads List4**  
  - Type: Google Sheets  
  - Role: Update Google Sheets with enriched lead data, fallback data, or no data respectively  
  - Config: Writes to specific sheets/tabs with appropriate columns  
  - Inputs: From "If1" or "If2"  
  - Edge cases: Google API errors, write conflicts

---

#### 2.5 Data Validation & Formatting

**Overview:**  
Validates JSON data, slices lead lists, and prepares data for batch processing and logging.

**Nodes Involved:**  
- JSON cleaner code  
- Chunk into separate items code  
- Slice down to 10  
- List existing domains code

**Node Details:**  
(These nodes are shared with previous blocks; see 2.3 and 2.4 for details.)

---

#### 2.6 Data Logging

**Overview:**  
Writes enriched lead data back to Google Sheets, marking statuses as “Enriched” or “New” accordingly.

**Nodes Involved:**  
- Leads List  
- Leads List2  
- Leads List3  
- Leads List4

**Node Details:**  
(See 2.4 for details.)

---

#### 2.7 Optional Triggers & Extensions

**Overview:**  
Includes optional Telegram trigger and placeholders for Airtable integration or CRM push.

**Nodes Involved:**  
- Telegram Trigger (disabled)  
- Airtable (disabled)  
- Schedule Trigger (disabled)

**Node Details:**

- **Telegram Trigger**  
  - Disabled by default; requires Telegram Bot credentials  
  - Can be enabled to trigger lead generation via Telegram chat

- **Airtable**  
  - Disabled; placeholder for replacing Google Sheets with Airtable

- **Schedule Trigger**  
  - Disabled; can be used to schedule lead generation runs automatically

---

### 3. Summary Table

| Node Name                          | Node Type                        | Functional Role                                | Input Node(s)                              | Output Node(s)                         | Sticky Note                                                                                  |
|-----------------------------------|---------------------------------|-----------------------------------------------|-------------------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’      | Manual Trigger                  | Manual start trigger                           |                                           | Read all "new" Leads                  |                                                                                              |
| When chat message received         | Langchain Chat Trigger          | Chat input trigger                             |                                           | Company Finder1                      |                                                                                              |
| Telegram Trigger                  | Telegram Trigger (disabled)      | Optional Telegram chat trigger                 |                                           |                                       |                                                                                              |
| OpenRouter Chat Model             | Langchain OpenRouter LLM Chat    | AI model for company discovery                 | If                                      | Company Finder                      |                                                                                              |
| Company Finder                   | Langchain Chain LLM              | Parses AI output to extract company data       | OpenRouter Chat Model                    | Code2                               |                                                                                              |
| OpenRouter Chat Model1            | Langchain OpenRouter LLM Chat    | AI model for chat-triggered company discovery | When chat message received               | Company Finder1                    |                                                                                              |
| Company Finder1                  | Langchain Chain LLM              | Parses AI output from chat trigger              | OpenRouter Chat Model1                   | JSON cleaner code                  |                                                                                              |
| JSON cleaner code                | Code                            | Cleans and parses AI JSON output                | Company Finder1                         | Chunk into separate items code     |                                                                                              |
| Chunk into separate items code   | Code                            | Splits array into individual items              | JSON cleaner code                      | Read all from Google Sheet, Merge  |                                                                                              |
| Read all from your Google Sheet "Leads List" sheet | Google Sheets                  | Reads existing leads for deduplication          |                                       | List existing domains code          |                                                                                              |
| List existing domains code       | Code                            | Extracts existing domains from Google Sheets    | Read all from Google Sheet              | Merge                              |                                                                                              |
| Merge                          | Merge                           | Combines new leads and existing domains         | Chunk into separate items code, List existing domains code | Keep only non-duplicates code       |                                                                                              |
| Keep only non-duplicates code    | Code                            | Filters out duplicates                           | Merge                                | If2                                |                                                                                              |
| Slice down to 10                | Code                            | Limits batch size to 10 leads                    | Read all "new" Leads or chat flow       | Find real user data with Hunter     |                                                                                              |
| Find real user data with Hunter  | Hunter.io                      | Enriches domains with personal emails           | Slice down to 10                       | Clean Hunter output                |                                                                                              |
| Clean Hunter output             | Code                            | Normalizes Hunter.io response                     | Find real user data with Hunter        | If                                 |                                                                                              |
| If                            | If                              | Checks if Hunter found emails                     | Clean Hunter output                   | Company Finder (fallback AI), Leads List |                                                                                              |
| Code2                         | Code                            | Processes fallback AI output                      | Company Finder                       | If1                                |                                                                                              |
| If1                           | If                              | Checks if fallback AI found emails                | Code2                               | Leads List2, Leads List3            |                                                                                              |
| Leads List                    | Google Sheets                  | Updates Google Sheet with enriched leads          | If (Hunter emails found)              |                                   |                                                                                              |
| Leads List2                   | Google Sheets                  | Updates Google Sheet with fallback AI data        | If1 (fallback AI found)               |                                   |                                                                                              |
| Leads List3                   | Google Sheets                  | Updates Google Sheet for no emails found          | If1 (no fallback data)                |                                   |                                                                                              |
| Leads List4                   | Google Sheets                  | Updates Google Sheet with new leads from chat flow | If2                                |                                   |                                                                                              |
| Read all "new" Leads           | Google Sheets                  | Reads new leads for batch processing              | When clicking ‘Test workflow’          | Slice down to 10                   |                                                                                              |
| Read all from your Google Sheet "Leads List" sheet | Google Sheets                  | Reads existing leads for deduplication          | Chunk into separate items code        | List existing domains code          |                                                                                              |
| List existing domains code       | Code                            | Extracts existing domains from Google Sheets    | Read all from Google Sheet              | Merge                              |                                                                                              |
| Merge                          | Merge                           | Combines new leads and existing domains         | Chunk into separate items code, List existing domains code | Keep only non-duplicates code       |                                                                                              |
| Keep only non-duplicates code    | Code                            | Filters out duplicates                           | Merge                                | If2                                |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing.

2. **Create Chat Trigger Node**  
   - Type: Langchain Chat Trigger  
   - Configure webhook and session management for chat input.

3. **(Optional) Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure Telegram Bot credentials and enable if desired.

4. **Create OpenRouter Chat Model Node (for manual and conditional flow)**  
   - Type: Langchain OpenRouter LLM Chat  
   - Set model to Perplexity Sonar Large (or preferred)  
   - Connect input from manual trigger or conditional node.

5. **Create Company Finder Node**  
   - Type: Langchain Chain LLM  
   - Configure chain to parse AI response JSON containing companies and domains.

6. **Create JSON Cleaner Code Node**  
   - Type: Code  
   - Write JavaScript to parse and clean AI JSON output.

7. **Create Chunk Into Separate Items Code Node**  
   - Type: Code  
   - Split array of companies into individual items for processing.

8. **Create Google Sheets Node to Read Existing Leads**  
   - Type: Google Sheets (Read)  
   - Connect to your "Leads List" sheet with header row.

9. **Create Code Node to Extract Existing Domains**  
   - Type: Code  
   - Extract domain list from Google Sheets data.

10. **Create Merge Node**  
    - Type: Merge  
    - Merge new leads and existing domains for deduplication.

11. **Create Code Node to Filter Non-Duplicates**  
    - Type: Code  
    - Filter out leads whose domains already exist in Google Sheets.

12. **Create Slice Down to 10 Code Node**  
    - Type: Code  
    - Limit lead batch size to 10 to protect API limits.

13. **Create Hunter.io Node**  
    - Type: Hunter.io  
    - Configure with your Hunter API key credential.  
    - Input domains from previous node.

14. **Create Code Node to Clean Hunter Output**  
    - Type: Code  
    - Normalize Hunter.io response, extract emails and metadata.

15. **Create If Node to Check Hunter Results**  
    - Type: If  
    - Condition: Check if emails found.  
    - True branch: Proceed to Google Sheets update.  
    - False branch: Trigger fallback AI.

16. **Create Fallback AI Chain Node (Company Finder)**  
    - Type: Langchain Chain LLM  
    - Use AI to find general email/contact info if Hunter fails.

17. **Create Code Node to Process Fallback AI Output**  
    - Type: Code  
    - Format fallback data for Google Sheets.

18. **Create If Node to Check Fallback AI Results**  
    - Type: If  
    - Condition: Check if fallback AI found data.  
    - True branch: Update Google Sheets with fallback data.  
    - False branch: Update Google Sheets with "no data" status.

19. **Create Google Sheets Nodes to Update Leads**  
    - Leads List: For Hunter-enriched leads.  
    - Leads List2: For fallback AI leads.  
    - Leads List3: For leads with no emails found.  
    - Leads List4: For new leads from chat flow.

20. **Connect all nodes according to logical flow:**  
    - Manual trigger → Read new leads → Slice → Hunter → Clean → If → Leads List  
    - If no Hunter emails → Fallback AI → Process → If → Leads List2 or Leads List3  
    - Chat trigger → OpenRouter → Company Finder → Clean → Chunk → Read existing → Merge → Filter → Leads List4

21. **Configure credentials:**  
    - Google Sheets: OAuth2 or API key with access to your spreadsheet.  
    - Hunter.io: API key from Hunter.io account.  
    - OpenRouter: API key with access to Perplexity or other models.  
    - Telegram (optional): Bot token and webhook URL.

22. **Set default values and constraints:**  
    - Google Sheets must have header row with columns like Domain, Status, Email, Company, etc.  
    - Batch size limited to 10 to avoid API rate limits.  
    - AI model set to Perplexity Sonar Large by default but can be changed.

23. **Test workflow manually and via chat trigger to validate end-to-end operation.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow was designed by **Velebit from Innovatio**.                                                     |                                                                                                    |
| For customization, swap Google Sheets with Airtable for relational views and tagging.                         |                                                                                                    |
| Replace Perplexity with GPT-4, Claude, or Mixtral inside OpenRouter for different AI models.                  |                                                                                                    |
| Auto-send enriched leads to Gmail, Slack, or your CRM for outreach automation.                                |                                                                                                    |
| Connect to Telegram for on-the-go company generation.                                                        |                                                                                                    |
| For advanced lead scoring or auto-categorization, contact velebit@innovatio.design.                           |                                                                                                    |
| Official documentation and support links lead only to trusted sources with no affiliate tracking.             |                                                                                                    |
| Paid version on Gumroad includes commercial use rights and extended fallback logic.                           | https://gumroad.com/innovatio                                                                        |
| Innovatio’s signature visual sticky note system is used for color-coded workflow notes inside n8n canvas.     | Green = Main Steps, Blue = Personalization Tips, Yellow = Optional/Advanced, Gray = Guidance & CTAs |

---

This detailed reference document enables advanced users and AI agents to fully understand, reproduce, and customize the **Automated Lead Generation & Contact Enrichment** workflow in n8n, anticipating integration points and potential failure modes.