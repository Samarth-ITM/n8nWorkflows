Smart CSM Assignment & AI Welcome Emails for HubSpot Deal Wins with Gmail

https://n8nworkflows.xyz/workflows/smart-csm-assignment---ai-welcome-emails-for-hubspot-deal-wins-with-gmail-10398


# Smart CSM Assignment & AI Welcome Emails for HubSpot Deal Wins with Gmail

### 1. Workflow Overview

This workflow automates the assignment of Customer Success Managers (CSMs) and the sending of AI-generated personalized welcome emails when a HubSpot deal is marked as "Closed Won." It is designed for customer success and sales teams aiming to streamline onboarding communication and workload distribution.

The workflow comprises these logical blocks:

- **1.1 Trigger & Initial Setup:** Listens for "deal closed won" events in HubSpot and prepares template variables.
- **1.2 CSM Assignment:** Retrieves CSM workload data from an n8n Data Table, selects the least busy CSM, assigns them as contact owner in HubSpot, and updates their workload count.
- **1.3 Contact Processing & Role Filtering:** Extracts associated contacts from the won deal and filters them for those with the buying role "Champion."
- **1.4 AI-Powered Welcome Email Creation:** Uses OpenAI GPT-4 to generate a warm, personalized welcome email based on the contact and sender information.
- **1.5 Email Formatting & Sending:** Converts the AI-generated Markdown email to HTML and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initial Setup

**Overview:**  
Starts the workflow when a HubSpot deal's "Is closed won" property changes. It sets sender and company variables used in email generation.

**Nodes Involved:**  
- Trigger: Deal Is 'Closed Won'  
- Configure Template Variables  
- If Deal Is Won  
- Sticky Notes (contextual)

**Node Details:**

- **Trigger: Deal Is 'Closed Won'**  
  - Type: HubSpot Trigger  
  - Role: Watches for changes to the `hs_is_closed_won` property on deals. Fires on both True and False changes.  
  - Config: Watches `deal.propertyChange` event filtered for `hs_is_closed_won`. Requires HubSpot Developer API credentials.  
  - Input: External HubSpot event  
  - Output: JSON with deal info including deal ID  
  - Failure Modes: Authentication errors, webhook setup issues, or rate limits  
  - Sticky Note: Explains trigger behavior and credential requirement

- **Configure Template Variables**  
  - Type: Set node  
  - Role: Assigns static variables for email sender name, email, and company name  
  - Config: Three string variables: `company_name`, `sender_name`, `sender_email`  
  - Input: Trigger output  
  - Output: JSON with these variables for downstream use  
  - Failure Modes: None critical, but empty values will affect email personalization

- **If Deal Is Won**  
  - Type: If node  
  - Role: Filters runs to only continue if the deal is actually marked as won (true)  
  - Config: Checks if `hs_is_closed_won` property equals boolean true  
  - Input: From Configure Template Variables  
  - Output: True branch continues workflow, false branch stops  
  - Failure Modes: Expression errors if property missing or malformed

- **Sticky Notes**  
  - Provide setup instructions, context for trigger, and general workflow overview

---

#### 1.2 CSM Assignment

**Overview:**  
Retrieves the list of Customer Success Managers from an n8n Data Table, selects the least busy based on current deal counts, assigns that CSM as the contact owner in HubSpot, and increments their workload count in the table.

**Nodes Involved:**  
- Get CSM List  
- Find Least Busy CSM  
- HubSpot: Get Deal Details  
- HubSpot: Assign Contact Owner  
- Increment CSM Deal Count  
- Sticky Notes (contextual)

**Node Details:**

- **Get CSM List**  
  - Type: Data Table  
  - Role: Fetches all CSM records from the `csm_assignments` table  
  - Config: Operation `get`, returns all rows from the specified Data Table (user must select the correct table)  
  - Input: True branch from "If Deal Is Won"  
  - Output: JSON array with CSM IDs and current deal counts  
  - Failure Modes: Empty table (no CSMs), table ID missing or incorrect

- **Find Least Busy CSM**  
  - Type: Code (JavaScript)  
  - Role: Sorts CSMs by their deal count and selects the one with the fewest assigned deals  
  - Config: Custom JS code that throws error if no CSMs found; returns CSM id, row ID, and current deal count  
  - Input: Output from "Get CSM List"  
  - Output: Object with least busy CSM info for assignment  
  - Failure Modes: Empty input array, JS errors

- **HubSpot: Get Deal Details**  
  - Type: HubSpot node  
  - Role: Retrieves full deal details including associated contact IDs (`associatedVids`)  
  - Config: Operation `get` on `deal` resource, authenticated with OAuth2  
  - Input: From "Find Least Busy CSM"  
  - Output: Deal JSON including associated contacts  
  - Failure Modes: Authentication errors, invalid deal ID, API limits

- **HubSpot: Assign Contact Owner**  
  - Type: HubSpot node  
  - Role: Assigns the least busy CSM as the owner of the contact in HubSpot  
  - Config: Updates contact owner field with CSM ID from prior node  
  - Input: After sending email via Gmail  
  - Output: HubSpot update response  
  - Failure Modes: Invalid CSM ID, permission issues, API errors

- **Increment CSM Deal Count**  
  - Type: Data Table  
  - Role: Updates the deal count for the assigned CSM by incrementing it by 1  
  - Config: Operation `update` on the same Data Table, filters by row ID, increments `deal_count` field using expression  
  - Input: From "HubSpot: Assign Contact Owner"  
  - Output: Updated Data Table row  
  - Failure Modes: Table or row missing, expression errors

- **Sticky Notes**  
  - Explain setting up the `csm_assignments` Data Table, how to populate it, and node purposes.

---

#### 1.3 Contact Processing & Role Filtering

**Overview:**  
Processes associated contact IDs from the deal, fetches detailed contact info, and filters contacts to those whose `hs_buying_role` is "CHAMPION" before generating emails.

**Nodes Involved:**  
- Split Contact IDs  
- HubSpot: Get Contact Details  
- If Role is 'Champion'  

**Node Details:**

- **Split Contact IDs**  
  - Type: Code (JavaScript)  
  - Role: Takes the array of associated contact IDs from the deal and creates a separate workflow item for each contact  
  - Config: Custom JS extracting `associatedVids` from deal details and mapping each to an item  
  - Input: From "HubSpot: Get Deal Details"  
  - Output: Multiple items, each with one contact ID  
  - Failure Modes: Missing or empty `associatedVids`, JS errors

- **HubSpot: Get Contact Details**  
  - Type: HubSpot node  
  - Role: Retrieves contact details including the `hs_buying_role` property  
  - Config: `get` operation on contact resource by contactId, fetches property `hs_buying_role`  
  - Input: From "Split Contact IDs"  
  - Output: Contact JSON with properties  
  - Failure Modes: Invalid contact ID, auth issues, API errors

- **If Role is 'Champion'**  
  - Type: If node  
  - Role: Filters contacts to only those with buying role exactly "CHAMPION"  
  - Config: Checks if `hs_buying_role.value` equals "CHAMPION"  
  - Input: From "HubSpot: Get Contact Details"  
  - Output: True branch continues to AI email generation  
  - Failure Modes: Missing property, expression errors

---

#### 1.4 AI-Powered Welcome Email Creation

**Overview:**  
Generates a warm, personalized welcome email using OpenAI GPT-4 based on sender and recipient details.

**Nodes Involved:**  
- AI Model (Chat OpenAI)  
- Structured Output Parser  
- AI: Write Welcome Email  
- Sticky Note (contextual)

**Node Details:**

- **AI Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides GPT-4o-mini model to generate AI completions  
  - Config: Uses cached model "gpt-4o-mini", requires OpenAI credentials  
  - Input: Text prompt from "AI: Write Welcome Email"  
  - Output: Raw AI response  
  - Failure Modes: API limits, authentication, latency

- **Structured Output Parser**  
  - Type: LangChain Output Parser Structured  
  - Role: Parses AI output to a structured JSON schema with `subject` and `body` fields  
  - Config: Manual JSON schema requiring `subject` (string) and `body` (string in Markdown)  
  - Input: AI Model output  
  - Output: Parsed email content  
  - Failure Modes: Malformed AI output, parsing errors

- **AI: Write Welcome Email**  
  - Type: LangChain Agent  
  - Role: Constructs the prompt and sends it to the AI model; includes system message defining role as Customer Success Manager  
  - Config: Prompt includes instructions, rules, and variables from sender and contact details; output is parsed with the Structured Output Parser  
  - Input: From "If Role is 'Champion'" (contact details) and sender variables  
  - Output: Structured email content (subject and body)  
  - Failure Modes: Expression errors in variables, AI failures

- **Sticky Notes**  
  - Describe AI email writing purpose, credential requirements, and prompt customization

---

#### 1.5 Email Formatting & Sending

**Overview:**  
Converts the AI-generated Markdown email body to HTML and sends the email to the contact’s email address via Gmail.

**Nodes Involved:**  
- Util: Markdown to HTML  
- Gmail: Send Welcome Email  

**Node Details:**

- **Util: Markdown to HTML**  
  - Type: Markdown node  
  - Role: Converts Markdown-formatted email body from AI into HTML for rich email formatting  
  - Config: Mode set to `markdownToHtml`, input is email body from AI output  
  - Input: From "AI: Write Welcome Email" output parser  
  - Output: HTML string for email body  
  - Failure Modes: Empty or malformed Markdown input

- **Gmail: Send Welcome Email**  
  - Type: Gmail node  
  - Role: Sends the email to the contact using Gmail SMTP  
  - Config: Sends to contact email, uses subject from AI output, message is HTML body from previous node, requires Gmail OAuth2 credentials  
  - Input: HTML email and subject from Markdown node, recipient email from contact details  
  - Output: Send confirmation  
  - Failure Modes: Authentication errors, quota limits, recipient email invalid

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                           | Input Node(s)                          | Output Node(s)                      | Sticky Note                                                                                                                 |
|---------------------------|----------------------------------|-----------------------------------------|--------------------------------------|-----------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Trigger: Deal Is 'Closed Won' | HubSpot Trigger                  | Workflow trigger on deal closed won     | -                                    | Configure Template Variables       | This node is the **live** trigger for the workflow. Requires HubSpot Developer API credential.                                |
| Configure Template Variables | Set                              | Defines sender and company variables    | Trigger: Deal Is 'Closed Won'         | If Deal Is Won                    |                                                                                                                             |
| If Deal Is Won             | If                               | Filters for deals marked as won only    | Configure Template Variables          | Get CSM List                     |                                                                                                                             |
| Get CSM List               | Data Table                       | Retrieves CSM workload data              | If Deal Is Won                       | Find Least Busy CSM              | See sticky note about setting up the `csm_assignments` Data Table.                                                           |
| Find Least Busy CSM        | Code                             | Selects least busy CSM based on deal count | Get CSM List                        | HubSpot: Get Deal Details        | See sticky note about least-busy CSM assignment logic.                                                                       |
| HubSpot: Get Deal Details  | HubSpot                          | Gets full deal details including contacts | Find Least Busy CSM                 | Split Contact IDs                |                                                                                                                             |
| Split Contact IDs          | Code                             | Splits multiple contact IDs to individual items | HubSpot: Get Deal Details          | HubSpot: Get Contact Details     |                                                                                                                             |
| HubSpot: Get Contact Details | HubSpot                        | Fetches contact info including buying role | Split Contact IDs                   | If Role is 'Champion'            |                                                                                                                             |
| If Role is 'Champion'      | If                               | Filters contacts with buying role "CHAMPION" | HubSpot: Get Contact Details       | AI: Write Welcome Email          |                                                                                                                             |
| AI: Write Welcome Email    | LangChain Agent                  | Generates personalized welcome email via AI | If Role is 'Champion'               | Util: Markdown to HTML           | Uses OpenAI credentials; customizable prompt for company tone.                                                                |
| AI Model                  | LangChain LM Chat OpenAI         | GPT-4o-mini model for AI completions    | AI: Write Welcome Email              | Structured Output Parser          |                                                                                                                             |
| Structured Output Parser   | LangChain Output Parser Structured | Parses AI response to structured email content | AI Model                          | AI: Write Welcome Email          |                                                                                                                             |
| Util: Markdown to HTML     | Markdown                        | Converts Markdown email body to HTML    | AI: Write Welcome Email              | Gmail: Send Welcome Email         |                                                                                                                             |
| Gmail: Send Welcome Email  | Gmail                           | Sends the formatted welcome email       | Util: Markdown to HTML               | HubSpot: Assign Contact Owner     |                                                                                                                             |
| HubSpot: Assign Contact Owner | HubSpot                        | Assigns least busy CSM as contact owner | Gmail: Send Welcome Email            | Increment CSM Deal Count          |                                                                                                                             |
| Increment CSM Deal Count   | Data Table                      | Updates deal count in CSM Data Table    | HubSpot: Assign Contact Owner        | -                               | Instructions on how to increment deal_count field using expression.                                                          |
| Sticky Note7              | Sticky Note                     | Contact info for workflow author        | -                                    | -                               | Contact me: thomas@pollup.net; link to other workflows: https://n8n.io/creators/zeerobug/                                      |
| Sticky Note3              | Sticky Note                     | Workflow overview and setup instructions | -                                    | -                               | Detailed workflow explanation including how to create the CSM Data Table and configure nodes.                                 |
| Sticky Note10             | Sticky Note                     | Explanation of the trigger node          | -                                    | -                               | Explains trigger behavior and credential needs.                                                                               |
| Sticky Note9              | Sticky Note                     | Notes on AI Email Writer node            | -                                    | -                               | Notes about AI usage, OpenAI credentials, and prompt customization.                                                           |
| Sticky Note               | Sticky Note                     | Least-Busy CSM Assignment block summary | -                                    | -                               | Explains logic for CSM selection and deal count updates.                                                                      |
| Sticky Note1              | Sticky Note                     | Instructions for Increment CSM Deal Count | -                                  | -                               | Step-by-step on how to configure the update operation for deal_count increment.                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add a **HubSpot Trigger** node named "Trigger: Deal Is 'Closed Won'."  
   - Configure to listen for the event `deal.propertyChange` on property `hs_is_closed_won`.  
   - Add HubSpot Developer API credentials.  
   - This node initiates the workflow when a deal is marked won.

2. **Set Template Variables**  
   - Add a **Set** node "Configure Template Variables."  
   - Add string variables:  
     - `company_name` (e.g., "Your Company Name")  
     - `sender_name` (e.g., "Your Sender Name")  
     - `sender_email` (e.g., "your-email@example.com")  
   - Connect output of the trigger node to this node.

3. **Filter Closed Won Deals**  
   - Add an **If** node "If Deal Is Won."  
   - Configure to check if `hs_is_closed_won` property equals boolean true.  
   - Connect from "Configure Template Variables."  
   - Only proceed on the true branch.

4. **Get CSM List from Data Table**  
   - Create an **n8n Data Table** node "Get CSM List."  
   - Set operation to `get` and select your `csm_assignments` Data Table (must be created beforehand).  
   - Connect from the true branch of "If Deal Is Won."

5. **Find Least Busy CSM**  
   - Add a **Code** node "Find Least Busy CSM."  
   - Paste the provided JavaScript code that sorts CSMs by `deal_count` and selects the least busy.  
   - Connect from "Get CSM List."

6. **Get Deal Details**  
   - Add **HubSpot** node "HubSpot: Get Deal Details."  
   - Set resource `deal` and operation `get`.  
   - Use expression to get deal ID: `={{ $("Trigger: Deal Is 'Closed Won'").item.json.body[0].objectId }}`.  
   - Authenticate with OAuth2.  
   - Connect from "Find Least Busy CSM."

7. **Split Contact IDs**  
   - Add a **Code** node "Split Contact IDs."  
   - Use JS code to unpack `associatedVids` from deal details and output each contact ID as separate item.  
   - Connect from "HubSpot: Get Deal Details."

8. **Get Contact Details**  
   - Add **HubSpot** node "HubSpot: Get Contact Details."  
   - Set resource `contact` and operation `get`.  
   - Pass contact ID from incoming item JSON.  
   - Request property `hs_buying_role`.  
   - Authenticate with OAuth2.  
   - Connect from "Split Contact IDs."

9. **Filter Contacts by Role**  
   - Add an **If** node "If Role is 'Champion'."  
   - Check if `hs_buying_role.value` equals `"CHAMPION"`.  
   - Connect from "HubSpot: Get Contact Details."  
   - Proceed only on true branch.

10. **AI Welcome Email Generation**  
    - Add an **AI Agent** node "AI: Write Welcome Email."  
    - Set system message: "You are a professional Customer Success Manager."  
    - Write a prompt instructing to generate a warm, personalized welcome email including variables: sender name/email/company, recipient first/last/email, and content instructions including congratulations, welcome, kickoff call, video and help doc links.  
    - Link AI model output to the **LangChain LM Chat OpenAI** node configured with model `gpt-4o-mini` and OpenAI credentials.  
    - Use a **Structured Output Parser** node set to a manual schema requiring `subject` and `body` fields.  
    - Connect the parser output back to the AI Agent node’s output parser.

11. **Markdown to HTML Conversion**  
    - Add a **Markdown** node "Util: Markdown to HTML."  
    - Set mode to `markdownToHtml`.  
    - Input the email body from AI structured output.  
    - Connect from "AI: Write Welcome Email."

12. **Send Email via Gmail**  
    - Add a **Gmail** node "Gmail: Send Welcome Email."  
    - Configure to send to contact email from contact details.  
    - Use subject from AI output and message as converted HTML.  
    - Authenticate with Gmail OAuth2 credentials.  
    - Connect from "Util: Markdown to HTML."

13. **Assign Contact Owner in HubSpot**  
    - Add **HubSpot** node "HubSpot: Assign Contact Owner."  
    - Update contact owner to the `csm_id` from "Find Least Busy CSM."  
    - Authenticate with OAuth2.  
    - Connect from "Gmail: Send Welcome Email."

14. **Increment CSM Deal Count**  
    - Add a **Data Table** node "Increment CSM Deal Count."  
    - Set operation to `update`.  
    - Filter by row ID from "Find Least Busy CSM."  
    - Update `deal_count` field by incrementing current count + 1 using expression.  
    - Connect from "HubSpot: Assign Contact Owner."

15. **Create n8n Data Table for CSMs**  
    - In n8n, create a Data Table named `csm_assignments`.  
    - Columns:  
      - `csm_id` (String) – HubSpot Owner IDs  
      - `deal_count` (Number) – initialized to 0 for each CSM  
    - Populate with one row per CSM.

16. **Add Credentials**  
    - Add and configure credentials for:  
      - HubSpot Developer API (trigger and other HubSpot nodes)  
      - HubSpot OAuth2 (deal and contact nodes)  
      - OpenAI API (AI nodes)  
      - Gmail OAuth2 (send email node)

17. **Activate Workflow**  
    - Save and activate the workflow.  
    - Test by marking a deal as closed won in HubSpot and verifying email receipt and CSM assignment.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                  |
|------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Workflow author contact: For modifications or support, contact thomas@pollup.net                            | Email contact                                                    |
| More workflows by the author available: https://n8n.io/creators/zeerobug/                                  | Workflow author’s public profile                                 |
| Setup requires creating an n8n Data Table named `csm_assignments` with columns `csm_id` and `deal_count`    | Critical setup step for CSM assignment                           |
| AI email prompt is fully customizable to match company tone and style                                      | Node: AI: Write Welcome Email                                    |
| HubSpot trigger fires on both True and False changes; downstream filter node ensures only True flows continue | Important trigger behavior detail                                |
| Gmail node requires OAuth2 credentials with send email permissions                                        | Credential setup note                                           |

---

**Disclaimer:** The above content is derived exclusively from an automated workflow created with n8n. All data processed are legal and public. The workflow respects current content policies and contains no illegal, offensive, or protected elements.