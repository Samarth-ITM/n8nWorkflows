Automated LinkedIn Lead Generation with AI Email Personalization (NOT FINISH)

https://n8nworkflows.xyz/workflows/automated-linkedin-lead-generation-with-ai-email-personalization--not-finish--3926


# Automated LinkedIn Lead Generation with AI Email Personalization (NOT FINISH)

### 1. Workflow Overview

This workflow automates LinkedIn lead generation and AI-based personalized cold email creation for sales, business development, and marketing teams targeting B2B companies. It fetches qualified companies from a Google Sheet CRM, identifies decision-makers via LinkedIn data using the Ghost Genius API, generates and verifies potential business emails, crafts personalized email sequences using AI based on prospect LinkedIn content, and updates the CRM with enriched lead data.

Logical blocks:

- **1.1 Input & Variable Initialization**  
  Loads target companies from Google Sheets and sets user-defined variables controlling targeting, scoring, and messaging.

- **1.2 Company Filtering**  
  Filters companies from the CRM based on score and qualification status to prioritize promising leads.

- **1.3 Employee Discovery & Selection**  
  Extracts company domains, finds employees matching target roles via Ghost Genius API, retrieves detailed profiles, and uses AI to select the top 3 decision-makers.

- **1.4 Email Generation & Verification**  
  Generates multiple email formats per contact, verifies them via the MillionVerifier API, and filters valid emails.

- **1.5 Personalized Email Content Creation**  
  Retrieves LinkedIn posts for each selected profile, uses AI to create personalization setups, generates cold email sequences and subject lines.

- **1.6 Lead Data Storage & Workflow Looping**  
  Updates Google Sheets CRM with lead status and detailed lead info, loops back for batch processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Variable Initialization

- **Overview:**  
  Starts the workflow manually, sets key variables controlling API credentials, targeting criteria, product details, and messaging preferences, then loads the list of companies from the Google Sheet CRM.

- **Nodes Involved:**  
  - Start  
  - Set Variables  
  - Companies Recovery

- **Node Details:**

  - **Start**  
    - Type: Manual Trigger  
    - Role: Initiates the workflow manually.  
    - Connections: Output → Set Variables  
    - Edge Cases: None.

  - **Set Variables**  
    - Type: Set  
    - Role: Defines static variables like API keys, target roles, product info, email language, and messaging tone.  
    - Config: Assigns multiple string variables, e.g., "Target Employees" = `"CTO" OR "CEO" OR "Marketing"`, "Your product" set to a consulting service.  
    - Connections: Input ← Start, Output → Companies Recovery  
    - Edge Cases: Missing API keys or misconfigured variables could cause downstream failures.

  - **Companies Recovery**  
    - Type: Google Sheets  
    - Role: Loads companies from a specific Google Sheet tab ("Companies") for processing.  
    - Config: Reads all rows, using OAuth2 credentials.  
    - Connections: Input ← Set Variables, Output → Filter Score and State  
    - Edge Cases: Google API quota limits, missing or invalid document ID, incorrect sheet name.

---

#### 2.2 Company Filtering

- **Overview:**  
  Filters companies to continue processing only those with a score >= 7 and state "Qualified," optimizing API usage for relevant leads.

- **Nodes Involved:**  
  - Filter Score and State

- **Node Details:**

  - **Filter Score and State**  
    - Type: Filter  
    - Role: Applies conditions to JSON data items filtering by numeric score ≥ 7 and string state equals "Qualified."  
    - Connections: Input ← Companies Recovery, Output → Loop Over Items  
    - Edge Cases: Items missing score or state fields may be excluded unintentionally.

---

#### 2.3 Employee Discovery & Selection

- **Overview:**  
  For each qualified company, extracts the company domain, finds employees matching target roles, verifies minimum employee count, fetches detailed profiles, and uses AI to select the top 3 decision-makers.

- **Nodes Involved:**  
  - Loop Over Items  
  - Extract Domain  
  - Find Employees  
  - Check profiles Found  
  - Not enough employees (Google Sheets update)  
  - Split Profiles  
  - Get Profile Details  
  - Keep relevant information  
  - Combine Profiles  
  - Select Top 3  
  - Split Out

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over filtered companies one by one to process sequentially or in manageable batches.  
    - Connections: Input ← Filter Score and State, Outputs → Extract Domain (on "continue" path), empty on "end" path.  
    - Edge Cases: Large batch sizes can overload APIs.

  - **Extract Domain**  
    - Type: LangChain OpenAI (GPT-4.1-mini)  
    - Role: Extracts main domain from company website URL using an AI prompt.  
    - Config: System prompt instructs to parse URLs to domain only.  
    - Connections: Input ← Loop Over Items, Output → Find Employees  
    - Credentials: OpenAI API  
    - Edge Cases: Malformed URLs, API errors, rate limits.

  - **Find Employees**  
    - Type: HTTP Request  
    - Role: Calls Ghost Genius API's Sales Navigator endpoint to find employees for the current company and target roles.  
    - Config: Query parameters include current company ID, account ID (from variables), page=1, keywords (target employees roles).  
    - Authentication: HTTP Header Auth with Ghost Genius API key.  
    - Connections: Input ← Extract Domain, Output → Check profiles Found  
    - Edge Cases: API downtime, authentication failure, empty response.

  - **Check profiles Found**  
    - Type: IF  
    - Role: Checks if total profiles found ≥ 3 to proceed or exit.  
    - Connections: Input ← Find Employees, Outputs → Split Profiles (if true), Not enough employees (if false)  
    - Edge Cases: Unexpected response structure, zero or null total.

  - **Not enough employees**  
    - Type: Google Sheets  
    - Role: Updates CRM marking companies with insufficient employees as "Not enough employees" in state.  
    - Connections: Input ← Check profiles Found (false), Output → Loop Over Items (continues next company)  
    - Edge Cases: Google Sheets update failure.

  - **Split Profiles**  
    - Type: SplitOut  
    - Role: Splits array of employee profiles into individual items for detailed processing.  
    - Connections: Input ← Check profiles Found (true), Output → Get Profile Details

  - **Get Profile Details**  
    - Type: HTTP Request  
    - Role: Fetches detailed profile information from Ghost Genius API using profile URL.  
    - Config: Uses batching (1 request per 2 seconds) to avoid rate limits.  
    - Authentication: HTTP Header Auth  
    - Connections: Input ← Split Profiles, Output → Keep relevant information  
    - Edge Cases: API limits, missing profile URLs.

  - **Keep relevant information**  
    - Type: Code (JavaScript)  
    - Role: Simplifies profile data to essential fields: id, url, first/last name, headline, position, company domain, skills.  
    - Connections: Input ← Get Profile Details, Output → Combine Profiles  
    - Edge Cases: Missing fields, null values.

  - **Combine Profiles**  
    - Type: Aggregate  
    - Role: Combines all simplified profile items back into one array for AI processing.  
    - Connections: Input ← Keep relevant information, Output → Select Top 3

  - **Select Top 3**  
    - Type: LangChain OpenAI (GPT-3.5-turbo-0125)  
    - Role: Uses AI to select exactly 3 decision-makers most relevant to the product/interest criteria from combined profiles.  
    - Config: Detailed prompt with instructions on ideal candidate profiles and strict requirement for exactly 3 outputs in JSON format.  
    - Connections: Input ← Combine Profiles, Output → Split Out  
    - Credentials: OpenAI API  
    - Edge Cases: AI response errors, fewer than 3 profiles available.

  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits the AI output JSON containing the 3 selected profiles into individual items for further processing.  
    - Connections: Input ← Select Top 3, Output → Generate Emails

---

#### 2.4 Email Generation & Verification

- **Overview:**  
  Generates multiple potential email formats per contact using name and company domain, verifies each email using MillionVerifier API, aggregates results, and filters valid emails via AI.

- **Nodes Involved:**  
  - Generate Emails  
  - Split Out Mails  
  - Verify Emails  
  - Combine Results  
  - Filter Valid Emails  
  - Check Email Valid  
  - No mail founded (Google Sheets update)

- **Node Details:**

  - **Generate Emails**  
    - Type: Code (JavaScript)  
    - Role: Creates 5 email variants per contact (e.g., firstname.lastname@domain, flastname@domain).  
    - Uses cleaned first and last names, domain from Extract Domain node.  
    - Connections: Input ← Split Out, Output → Split Out Mails  
    - Edge Cases: Missing names or domain cause malformed emails.

  - **Split Out Mails**  
    - Type: SplitOut  
    - Role: Splits the array of potential emails into individual items for verification.  
    - Connections: Input ← Generate Emails, Output → Verify Emails

  - **Verify Emails**  
    - Type: HTTP Request  
    - Role: Calls MillionVerifier API to validate each potential email's quality.  
    - Config: API key from Set Variables, email from current item.  
    - Connections: Input ← Split Out Mails, Output → Combine Results  
    - Edge Cases: API limits, invalid API key, network errors.

  - **Combine Results**  
    - Type: Aggregate  
    - Role: Aggregates all verified email results into a single array.  
    - Connections: Input ← Verify Emails, Output → Filter Valid Emails

  - **Filter Valid Emails**  
    - Type: LangChain OpenAI (O3-mini model)  
    - Role: Filters for emails with quality = "good," removes "risky," returns one good email per person with LinkedIn profile data, or "No email found."  
    - Connections: Input ← Combine Results, Output → Check Email Valid  
    - Credentials: OpenAI API  
    - Edge Cases: AI parsing errors, no good emails.

  - **Check Email Valid**  
    - Type: IF  
    - Role: Checks if AI output is not "No email found" to continue or mark failure.  
    - Connections: Input ← Filter Valid Emails, Output → Split Out (if valid), No mail founded (if not)  
    - Edge Cases: Unexpected AI output.

  - **No mail founded**  
    - Type: Google Sheets  
    - Role: Updates CRM marking companies with no valid emails found.  
    - Connections: Input ← Check Email Valid (false), Output → Loop Over Items (process next company)  
    - Edge Cases: Google Sheets API errors.

---

#### 2.5 Personalized Email Content Creation

- **Overview:**  
  Retrieves recent LinkedIn posts of the selected profiles, uses AI to create personalized email setups, generates three cold email templates and matching subject lines, then prepares data for CRM storage.

- **Nodes Involved:**  
  - Split Out (from Check Email Valid)  
  - Get Profile Post  
  - Create Personalization  
  - Generate Emails Messages  
  - Generate Emails Subjects  
  - Add lead(s)  
  - Lead(s) found (Google Sheets update)

- **Node Details:**

  - **Split Out**  
    - Type: SplitOut  
    - Role: Splits valid emails with profile info into single items for processing.  
    - Connections: Input ← Check Email Valid (true), Output → Get Profile Post

  - **Get Profile Post**  
    - Type: HTTP Request  
    - Role: Fetches recent LinkedIn posts (up to 3) for each profile using Ghost Genius API.  
    - Batching: 1 request per 2 seconds to avoid quota issues.  
    - Connections: Input ← Split Out, Output → Create Personalization  
    - Edge Cases: API errors, private profiles.

  - **Create Personalization**  
    - Type: LangChain OpenAI (GPT-4.1)  
    - Role: Generates a structured personalization setup for each prospect's cold email based on profile info and recent posts.  
    - Instructions emphasize focusing on product benefits relevant to prospect challenges, excluding irrelevant use cases.  
    - Connections: Input ← Get Profile Post, Output → Generate Emails Messages  
    - Credentials: OpenAI API  
    - Edge Cases: AI model errors, incomplete data.

  - **Generate Emails Messages**  
    - Type: LangChain OpenAI (GPT-4.1)  
    - Role: Writes 3 short, informal cold email templates (initial + 2 follow-ups) using personalization setup and product info.  
    - Tone: Direct, upbeat, non-salesy, first-person from sender role.  
    - Connections: Input ← Create Personalization, Output → Generate Emails Subjects  
    - Credentials: OpenAI API

  - **Generate Emails Subjects**  
    - Type: LangChain OpenAI (GPT-4.1)  
    - Role: Creates catchy, informal subject lines for each of the 3 emails tailored to the prospect.  
    - Connections: Input ← Generate Emails Messages, Output → Add lead(s)  
    - Credentials: OpenAI API

  - **Add lead(s)**  
    - Type: Google Sheets  
    - Role: Appends detailed lead data including contact info, emails, email sequences, LinkedIn URLs, and company info into the "Leads" tab of the CRM.  
    - Connections: Input ← Generate Emails Subjects, Output → Lead(s) found

  - **Lead(s) found**  
    - Type: Google Sheets  
    - Role: Updates the original company row in CRM marking state as "Lead(s) found."  
    - Connections: Input ← Add lead(s), Output → Loop Over Items (process next company)

---

#### 2.6 Workflow Looping & Exits

- **Overview:**  
  Manages looping over companies and exiting the workflow at points where companies do not meet criteria.

- **Nodes Involved:**  
  - Loop Over Items (loops back)  
  - Not enough employees (loops back)  
  - No mail founded (loops back)  
  - Sticky Note nodes labeled "Exit" at key decision points (visual indicators)

- **Node Details:**

  - Loop Over Items  
    - Loops to next company after processing completes or upon no leads.

  - Not enough employees, No mail founded, Lead(s) found nodes update CRM and loop back.

  - Sticky Notes provide visual documentation of exits.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                                       | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                                        |
|------------------------|----------------------------------|------------------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------|
| Start                  | Manual Trigger                   | Initiates workflow manually                           |                             | Set Variables               |                                                                                                                    |
| Set Variables          | Set                              | Defines static configuration variables               | Start                       | Companies Recovery          |                                                                                                                    |
| Companies Recovery     | Google Sheets                    | Loads companies from CRM                              | Set Variables               | Filter Score and State      |                                                                                                                    |
| Filter Score and State | Filter                           | Filters companies by score and qualification          | Companies Recovery          | Loop Over Items             |                                                                                                                    |
| Loop Over Items        | SplitInBatches                  | Iterates over companies one by one                    | Filter Score and State      | Extract Domain              |                                                                                                                    |
| Extract Domain         | LangChain OpenAI (GPT-4.1-mini) | Extracts company domain from URL                      | Loop Over Items             | Find Employees             |                                                                                                                    |
| Find Employees         | HTTP Request                    | Searches for employees matching target roles         | Extract Domain              | Check profiles Found        | Sticky Note2: "Find and select the right employees"                                                                |
| Check profiles Found   | IF                              | Checks if enough employees found (≥3)                 | Find Employees              | Split Profiles / Not enough employees |                                                                                                                    |
| Not enough employees   | Google Sheets                   | Updates CRM marking insufficient employees            | Check profiles Found        | Loop Over Items             |                                                                                                                    |
| Split Profiles         | SplitOut                        | Splits employee profiles array                        | Check profiles Found        | Get Profile Details         |                                                                                                                    |
| Get Profile Details    | HTTP Request                   | Retrieves detailed LinkedIn profile info              | Split Profiles             | Keep relevant information   |                                                                                                                    |
| Keep relevant information | Code (JS)                    | Simplifies profile data to essential fields           | Get Profile Details         | Combine Profiles            |                                                                                                                    |
| Combine Profiles       | Aggregate                      | Combines simplified profiles into one array           | Keep relevant information   | Select Top 3                | Sticky Note1: Cost overview                                                                                        |
| Select Top 3           | LangChain OpenAI (GPT-3.5)      | AI selects top 3 decision-makers                      | Combine Profiles            | Split Out                  | Sticky Note2: "Find and select the right employees"                                                                |
| Split Out              | SplitOut                        | Splits selected profiles into individual items        | Select Top 3                | Generate Emails             |                                                                                                                    |
| Generate Emails        | Code (JS)                      | Generates multiple email format candidates             | Split Out                  | Split Out Mails             | Sticky Note5: Email enrichment and verification                                                                    |
| Split Out Mails        | SplitOut                        | Splits potential emails array                          | Generate Emails             | Verify Emails              |                                                                                                                    |
| Verify Emails          | HTTP Request                   | Calls MillionVerifier API to validate emails           | Split Out Mails             | Combine Results             |                                                                                                                    |
| Combine Results        | Aggregate                      | Aggregates email verification responses                | Verify Emails              | Filter Valid Emails         |                                                                                                                    |
| Filter Valid Emails    | LangChain OpenAI (O3-mini)      | Filters only good quality emails, removes risky ones  | Combine Results            | Check Email Valid           | Sticky Note5: Email enrichment and verification                                                                    |
| Check Email Valid      | IF                              | Proceeds if good emails found, else marks "No mail"   | Filter Valid Emails         | Split Out / No mail founded |                                                                                                                    |
| No mail founded        | Google Sheets                   | Marks company state as "No mail founded"               | Check Email Valid (false)   | Loop Over Items             |                                                                                                                    |
| Split Out              | SplitOut                        | Prepares valid emails for personalization              | Check Email Valid (true)    | Get Profile Post            |                                                                                                                    |
| Get Profile Post       | HTTP Request                   | Retrieves recent LinkedIn posts for personalization    | Split Out                  | Create Personalization      | Sticky Note6: Email generation and storage                                                                         |
| Create Personalization | LangChain OpenAI (GPT-4.1)      | Creates personalization setup for AI email generation | Get Profile Post            | Generate Emails Messages    | Sticky Note6: Email generation and storage                                                                         |
| Generate Emails Messages | LangChain OpenAI (GPT-4.1)    | Generates 3 cold email templates                        | Create Personalization      | Generate Emails Subjects    | Sticky Note6: Email generation and storage                                                                         |
| Generate Emails Subjects | LangChain OpenAI (GPT-4.1)    | Generates catchy subject lines for emails              | Generate Emails Messages    | Add lead(s)                 | Sticky Note6: Email generation and storage                                                                         |
| Add lead(s)            | Google Sheets                   | Stores enriched lead and email data in CRM             | Generate Emails Subjects    | Lead(s) found              |                                                                                                                    |
| Lead(s) found          | Google Sheets                   | Marks company state as "Lead(s) found"                  | Add lead(s)                 | Loop Over Items             |                                                                                                                    |
| Not enough employees   | Google Sheets                   | Updates CRM marking insufficient employees             | Check profiles Found (false) | Loop Over Items            | Sticky Note3: Exit                                                                                                  |
| No mail founded        | Google Sheets                   | Updates CRM marking no valid emails                     | Check Email Valid          | Loop Over Items             | Sticky Note4: Exit                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a manual trigger node ("Start")** to initiate the workflow manually.

2. **Add a "Set" node ("Set Variables")**  
   - Define variables:  
     - `account_id (Ghost Genius API)`: your Ghost Genius account id  
     - `api_key MillionVerifier (millionverifier.com)`: your MillionVerifier API key  
     - `Your product`: description string of your product/service  
     - `Interest`: keywords related to your product's relevant decision areas  
     - `Target Audience`: description of your target audience  
     - `Target Employees`: string with role keywords, e.g. `"CTO" OR "CEO" OR "Marketing"`  
     - `Irrelevant Use Case`: string describing use cases to exclude  
     - `Email Language`: e.g., "English"  
     - `Sender Role (your role)`: role you represent, e.g., "Manager of Global Partnerships"  
     - `Product Benefits`: key benefits of your product

3. **Add Google Sheets node ("Companies Recovery")**  
   - Operation: Read rows from your CRM Google Sheet, sheet "Companies"  
   - Connect credentials with OAuth2  
   - Connect output from "Set Variables"

4. **Add Filter node ("Filter Score and State")**  
   - Conditions:  
     - Score ≥ 7  
     - State equals "Qualified"  
   - Connect input from "Companies Recovery"

5. **Add SplitInBatches node ("Loop Over Items")**  
   - Default batch size (can be 1 or small number)  
   - Input from "Filter Score and State"

6. **Add LangChain OpenAI node ("Extract Domain")**  
   - Model: GPT-4.1-mini  
   - Prompt: Extract main domain from company website URL  
   - Use expression to pass `{{ $json.Website }}`  
   - Connect input from "Loop Over Items" continuation path  
   - Configure OpenAI credentials

7. **Add HTTP Request node ("Find Employees")**  
   - URL: Ghost Genius API Sales Navigator endpoint  
   - Query parameters: current_company = company ID, account_id, page=1, keywords from variables  
   - Authentication: HTTP Header Auth with Ghost Genius API key  
   - Connect input from "Extract Domain"

8. **Add IF node ("Check profiles Found")**  
   - Condition: total profiles found ≥ 3  
   - Connect input from "Find Employees"  
   - True path → "Split Profiles"  
   - False path → "Not enough employees" (Google Sheets update)

9. **Add Google Sheets node ("Not enough employees")**  
   - Update row in CRM with State = "Not enough employees"  
   - Connect input from "Check profiles Found" false output  
   - Output loops back to "Loop Over Items"

10. **Add SplitOut node ("Split Profiles")**  
    - Field to split out: employee profiles array from API response  
    - Connect input from "Check profiles Found" true output

11. **Add HTTP Request node ("Get Profile Details")**  
    - URL: Ghost Genius API profile endpoint  
    - Query param: profile URL from split profile item  
    - Enable batching (1 request per 2 seconds)  
    - Authentication: HTTP Header Auth  
    - Connect input from "Split Profiles"

12. **Add Code node ("Keep relevant information")**  
    - Simplify profile JSON to id, url, first_name, last_name, headline, position, position_description, company_website (from Extract Domain), skills  
    - Connect input from "Get Profile Details"

13. **Add Aggregate node ("Combine Profiles")**  
    - Aggregate all simplified profiles into one list  
    - Connect input from "Keep relevant information"

14. **Add LangChain OpenAI node ("Select Top 3")**  
    - Model: GPT-3.5-turbo-0125  
    - Prompt: Select exactly 3 best decision-makers based on product and interest variables  
    - Output as JSON with keys first_profile, second_profile, third_profile  
    - Connect input from "Combine Profiles"  
    - Configure OpenAI credentials

15. **Add SplitOut node ("Split Out")**  
    - Split the AI JSON output of the 3 profiles into 3 items  
    - Connect input from "Select Top 3"

16. **Add Code node ("Generate Emails")**  
    - Generate 5 common business email formats using cleaned first/last names and extracted domain  
    - Connect input from "Split Out"

17. **Add SplitOut node ("Split Out Mails")**  
    - Split generated emails array into individual items  
    - Connect input from "Generate Emails"

18. **Add HTTP Request node ("Verify Emails")**  
    - Call MillionVerifier API with email and API key from variables  
    - Connect input from "Split Out Mails"

19. **Add Aggregate node ("Combine Results")**  
    - Aggregate all email verification results into one array  
    - Connect input from "Verify Emails"

20. **Add LangChain OpenAI node ("Filter Valid Emails")**  
    - Model: O3-mini  
    - Prompt: Filter emails with quality=good, exclude risky, return one email per person or "No email found"  
    - Connect input from "Combine Results"  
    - Configure OpenAI credentials

21. **Add IF node ("Check Email Valid")**  
    - Condition: AI output ≠ "No email found"  
    - True → "Split Out"  
    - False → "No mail founded" (Google Sheets update)

22. **Add Google Sheets node ("No mail founded")**  
    - Update CRM company state to "No mail founded"  
    - Connect input from "Check Email Valid" false output  
    - Output loops back to "Loop Over Items"

23. **Add SplitOut node ("Split Out")**  
    - Split filtered valid emails with profile data  
    - Connect input from "Check Email Valid" true output

24. **Add HTTP Request node ("Get Profile Post")**  
    - Call Ghost Genius API profile posts endpoint to fetch recent posts for personalization  
    - Batching enabled (1 request per 2 seconds)  
    - Connect input from "Split Out"

25. **Add LangChain OpenAI node ("Create Personalization")**  
    - Model: GPT-4.1  
    - Prompt: Analyze posts and profile info, create structured personalization setup for email AI (not full email)  
    - Connect input from "Get Profile Post"  
    - Configure OpenAI credentials

26. **Add LangChain OpenAI node ("Generate Emails Messages")**  
    - Model: GPT-4.1  
    - Prompt: Write 3 short informal cold emails (initial + follow-ups) using personalization setup and variables  
    - Connect input from "Create Personalization"  
    - Configure OpenAI credentials

27. **Add LangChain OpenAI node ("Generate Emails Subjects")**  
    - Model: GPT-4.1  
    - Prompt: Generate catchy personalized subject lines for each email in the sequence  
    - Connect input from "Generate Emails Messages"  
    - Configure OpenAI credentials

28. **Add Google Sheets node ("Add lead(s)")**  
    - Append detailed lead info including names, LinkedIn URLs, emails, email sequences, company info to "Leads" tab  
    - Connect input from "Generate Emails Subjects"

29. **Add Google Sheets node ("Lead(s) found")**  
    - Update original company row in CRM with State = "Lead(s) found"  
    - Connect input from "Add lead(s)"  
    - Output loops back to "Loop Over Items"

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                       | Context or Link                                                                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow relies heavily on the Ghost Genius API for LinkedIn data (employees, profiles, posts). You must have a valid Ghost Genius account and API key.                                                                                                                                       | Ghost Genius API documentation: https://ghostgenius.fr/api-docs                                        |
| MillionVerifier API is used for email validation; requires API key.                                                                                                                                                                                                                              | MillionVerifier: https://millionverifier.com/api                                                       |
| OpenAI API keys are required for multiple AI nodes (profile selection, domain extraction, email filtering, personalization, email generation, subject lines). Be mindful of usage and costs.                                                                                                    | OpenAI API docs: https://platform.openai.com/docs                                                      |
| Google Sheets is used as a CRM for input data and storing enriched leads. Ensure proper OAuth2 credentials and sheet/tab IDs configured.                                                                                                                                                         | Google Sheets API: https://developers.google.com/sheets/api                                            |
| Costs for 100 companies estimated around 2800 Ghost Genius API credits + 1500 MillionVerifier credits (~$2). Batch sizes and rate limits are carefully managed to avoid quota exhaustion, e.g., 1 request per 2 seconds for profile/post calls.                                                       | Sticky Note1 and Sticky Note7 in workflow provide cost and scaling notes.                               |
| The workflow is designed to loop continuously over batches of companies, updating the CRM states as it processes, enabling incremental progress and error recovery.                                                                                                                               |                                                                                                          |
| The email personalization goes beyond simple token replacements, leveraging recent LinkedIn post content and AI analysis to boost response rates.                                                                                                                                               | Sticky Note6 describes the email generation and storage process.                                        |
| Customize the "Set Variables" node to tailor targeting, product description, and messaging style for different campaigns or products.                                                                                                                                                            |                                                                                                          |
| The system enforces selecting exactly 3 decision-makers via AI, even if fewer perfect matches exist, ensuring a consistent email outreach set.                                                                                                                                                   |                                                                                                          |
| Sticky Notes in the workflow provide detailed explanations of each major logical section and operational costs.                                                                                                                                            |                                                                                                          |

---

This structured documentation enables understanding, reproduction, and troubleshooting of the workflow, ensuring smooth integration with APIs and effective lead generation and email personalization.