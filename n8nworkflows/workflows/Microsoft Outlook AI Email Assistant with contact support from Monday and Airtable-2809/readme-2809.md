Microsoft Outlook AI Email Assistant with contact support from Monday and Airtable

https://n8nworkflows.xyz/workflows/microsoft-outlook-ai-email-assistant-with-contact-support-from-monday-and-airtable-2809


# Microsoft Outlook AI Email Assistant with contact support from Monday and Airtable

### 1. Workflow Overview

This workflow, titled **Microsoft Outlook AI Email Assistant with contact support from Monday and Airtable**, automates the process of syncing contacts from Monday.com to Airtable daily and uses AI to categorize and prioritize Outlook emails based on enriched contact data and customizable rules stored in Airtable. It is designed for business users who want to streamline email management by leveraging AI-driven classification and prioritization, integrating CRM contact data, and applying dynamic rules.

The workflow logically divides into two main functional blocks:

- **1.1 Daily Contact Sync (Monday.com → Airtable)**  
  Automatically updates the Airtable Contacts table daily by pulling fresh contact data from Monday.com.

- **1.2 AI Email Categorisation & Prioritisation**  
  Fetches unprocessed Outlook emails, sanitizes their content, enriches them with contact information from Airtable, retrieves categorization rules, and uses an AI agent to assign categories and priorities. It then updates Outlook emails accordingly.

Supporting these are parallel retrievals of rules, categories, and delete rules from Airtable to enable dynamic, real-time decision-making.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Contact Sync (Monday.com → Airtable)

**Overview:**  
This block ensures the Airtable Contacts table is kept up to date by syncing contact data from a Monday.com board once daily.

**Nodes Involved:**  
- Update Contacts Schedule Trigger  
- Monday.com - Get Contacts  
- Airtable - Contacts  
- Sticky Note (CRM Contact List Integration)

**Node Details:**

- **Update Contacts Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the contact sync process on a daily interval (default interval unspecified, but can be configured).  
  - Configuration: Runs automatically based on schedule; no manual input.  
  - Inputs: None  
  - Outputs: Triggers "Monday.com - Get Contacts"  
  - Edge Cases: Scheduler misconfiguration could cause missed syncs.

- **Monday.com - Get Contacts**  
  - Type: Monday.com API Node  
  - Role: Retrieves all contact items from a specified Monday.com board and group.  
  - Configuration: Board ID `1840712625`, Group ID `topics`, operation `getAll`, returns all items.  
  - Inputs: Trigger from schedule node  
  - Outputs: Contact data JSON array  
  - Edge Cases: API rate limits, authentication errors, or board structure changes may cause failures.

- **Airtable - Contacts**  
  - Type: Airtable Node  
  - Role: Upserts contact records into Airtable’s Contacts table, matching on Email to update or insert.  
  - Configuration: Uses base `appNmgIGA4Fhculsn`, table `tbl8gTTEn96uFRDHE` (Contacts), maps Monday.com columns to Airtable fields: Type, Email, Last Name, First Name.  
  - Inputs: Data from Monday.com node  
  - Outputs: Confirmation of upsert operation  
  - Edge Cases: API key expiration, schema mismatch, or data inconsistencies.

- **Sticky Note (CRM Contact List Integration)**  
  - Type: Sticky Note  
  - Role: Documentation note explaining the CRM contact integration purpose and update frequency.  
  - Inputs/Outputs: None

---

#### 2.2 AI Email Categorisation & Prioritisation

**Overview:**  
This block fetches unprocessed Outlook emails, cleans and enriches them with contact data, retrieves categorization rules, and uses an AI agent to assign categories and priorities. It updates Outlook emails with AI-driven categories and importance flags.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Check Mail Schedule Trigger (disabled)  
- Microsoft Outlook23  
- Convert to Markdown  
- Email Messages  
- Loop Over Items  
- Contact  
- Rules  
- Categories  
- Delete Rules  
- Merge  
- AI: Analyse Email  
- OpenAI Chat Model  
- Structured Output Parser  
- Set Category  
- If  
- Set Importance  
- Sticky Notes (multiple for documentation)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the email processing workflow for testing.  
  - Inputs: None  
  - Outputs: Triggers Outlook email fetch and Airtable rules retrieval nodes.  
  - Edge Cases: Manual trigger requires user action.

- **Check Mail Schedule Trigger** (disabled)  
  - Type: Schedule Trigger  
  - Role: Intended to schedule email checking every 15 minutes but currently disabled.  
  - Inputs: None  
  - Outputs: Would trigger Outlook email fetch.  
  - Edge Cases: Disabled, so no effect unless enabled.

- **Microsoft Outlook23**  
  - Type: Microsoft Outlook Node  
  - Role: Fetches up to 10 emails from a specified Outlook folder with filters: emails that are not flagged and have no categories assigned.  
  - Configuration: Filters using OData syntax: `flag/flagStatus eq 'notFlagged' and not categories/any()`, folder specified by ID.  
  - Inputs: Trigger (manual or schedule)  
  - Outputs: Email data including flag, from, importance, subject, body, categories, isRead, etc.  
  - Edge Cases: API limits, authentication errors, folder ID changes.

- **Convert to Markdown**  
  - Type: Markdown Node  
  - Role: Converts the HTML email body content to Markdown format to simplify text for AI processing.  
  - Configuration: Input is the email body content field.  
  - Inputs: Outlook email data  
  - Outputs: Markdown text of email body  
  - Edge Cases: Complex HTML might not convert cleanly.

- **Email Messages**  
  - Type: Set Node  
  - Role: Cleans and sanitizes the email body text by removing HTML tags, markdown links, images, special characters, and excessive whitespace to prepare for AI analysis. Also sets key email fields (subject, importance, sender, from, id).  
  - Configuration: Uses JavaScript expressions with regex replacements for cleaning.  
  - Inputs: Markdown text from previous node  
  - Outputs: Sanitized email data object  
  - Edge Cases: Regex failures or unexpected email formats.

- **Loop Over Items**  
  - Type: SplitInBatches Node  
  - Role: Iterates over each email individually for processing.  
  - Configuration: Default batch size (likely 1) to process emails sequentially.  
  - Inputs: Sanitized emails array  
  - Outputs: Single email per iteration  
  - Edge Cases: Large batch sizes may cause timeouts.

- **Contact**  
  - Type: Airtable Node  
  - Role: Searches Airtable Contacts table for a contact matching the sender’s email address of the current email.  
  - Configuration: Uses filter formula `{Email}='sender_email'` dynamically set per email.  
  - Inputs: Single email from Loop Over Items  
  - Outputs: Contact data if found, empty if not  
  - Edge Cases: No match found, API errors.

- **Rules**  
  - Type: Airtable Node  
  - Role: Retrieves all rules from the Airtable Rules table to be used by the AI agent.  
  - Configuration: Base and table IDs set; operation is search (fetch all).  
  - Inputs: Triggered once per workflow run (executeOnce: true)  
  - Outputs: Rules data array  
  - Edge Cases: API errors, empty rules table.

- **Categories**  
  - Type: Airtable Node  
  - Role: Retrieves all categories from the Airtable Categories table for AI classification.  
  - Configuration: Similar to Rules node, fetches all categories.  
  - Inputs: Triggered once per workflow run  
  - Outputs: Categories data array  
  - Edge Cases: API errors, empty categories.

- **Delete Rules**  
  - Type: Airtable Node  
  - Role: Retrieves delete rules from Airtable that may cause emails to be removed or ignored.  
  - Configuration: Fetches all delete rules.  
  - Inputs: Triggered once per workflow run  
  - Outputs: Delete rules data array  
  - Edge Cases: API errors.

- **Merge**  
  - Type: Merge Node  
  - Role: Combines data streams from Contact, Rules, Categories, and Delete Rules nodes to provide a consolidated dataset for AI processing.  
  - Configuration: Mode "chooseBranch" with 4 inputs, merging all into one output.  
  - Inputs: Contact, Rules, Categories, Delete Rules  
  - Outputs: Merged data for AI agent  
  - Edge Cases: Data mismatch or missing inputs.

- **AI: Analyse Email**  
  - Type: Langchain Agent Node  
  - Role: Uses AI to categorize and prioritize the email based on email content, contact info, and rules/categories.  
  - Configuration:  
    - Prompt includes email JSON, contacts, delete rules, and categories.  
    - System message instructs AI on categories and output format.  
    - Output parser enabled for structured JSON.  
  - Inputs: Merged data from previous node  
  - Outputs: AI classification result JSON with fields: id, subject, category, subCategory, analysis.  
  - Edge Cases: AI response invalid JSON, API limits, prompt errors.

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model Node  
  - Role: Executes the AI model (GPT-4o) with low temperature (0.2) for deterministic output.  
  - Configuration: Model set to "gpt-4o", temperature 0.2.  
  - Inputs: Prompt from AI agent node  
  - Outputs: AI raw response  
  - Edge Cases: API key limits, network errors.

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser Node  
  - Role: Parses AI raw output to ensure it conforms to the defined JSON schema.  
  - Configuration: JSON schema requires id, subject, category, analysis; subCategory optional.  
  - Inputs: AI raw output  
  - Outputs: Validated structured JSON  
  - Edge Cases: Parsing failures if AI output malformed.

- **Set Category**  
  - Type: Microsoft Outlook Node  
  - Role: Updates the Outlook email’s category field with the AI-determined category.  
  - Configuration: Uses email id from AI output to update categories field.  
  - Inputs: AI analysis output  
  - Outputs: Confirmation of update  
  - Edge Cases: Outlook API errors, invalid message id.

- **If**  
  - Type: If Node  
  - Role: Checks if the AI-assigned subCategory equals "Action" to determine if importance should be set high.  
  - Configuration: Condition compares `subCategory` field to "Action".  
  - Inputs: AI output from Set Category node  
  - Outputs: True branch triggers Set Importance; False branch loops back to process next email.  
  - Edge Cases: Missing subCategory field.

- **Set Importance**  
  - Type: Microsoft Outlook Node  
  - Role: Updates Outlook email importance to "High" if conditions met.  
  - Configuration: Uses email id from AI output.  
  - Inputs: True branch from If node  
  - Outputs: Confirmation of update  
  - Edge Cases: API errors.

- **Sticky Notes**  
  - Multiple sticky notes provide inline documentation for blocks: Outlook filters, sanitization, contact matching, rules retrieval, and categorization agent.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                                  | Input Node(s)                          | Output Node(s)                        | Sticky Note                                                                                      |
|------------------------------|----------------------------------|-------------------------------------------------|--------------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------|
| Update Contacts Schedule Trigger | Schedule Trigger                | Initiates daily contact sync                     | None                                 | Monday.com - Get Contacts            |                                                                                                |
| Monday.com - Get Contacts      | Monday.com API Node              | Retrieves contacts from Monday.com board         | Update Contacts Schedule Trigger     | Airtable - Contacts                  |                                                                                                |
| Airtable - Contacts            | Airtable Node                   | Upserts contacts into Airtable                    | Monday.com - Get Contacts             | None                                |                                                                                                |
| When clicking ‘Test workflow’  | Manual Trigger                  | Manual start of email processing workflow        | None                                 | Microsoft Outlook23, Rules, Categories, Delete Rules |                                                                                                |
| Check Mail Schedule Trigger    | Schedule Trigger (disabled)     | Scheduled email fetch trigger (disabled)         | None                                 | Microsoft Outlook23                  |                                                                                                |
| Microsoft Outlook23            | Microsoft Outlook Node          | Fetches unflagged, uncategorized Outlook emails  | When clicking ‘Test workflow’, Check Mail Schedule Trigger | Convert to Markdown                 | ## Outlook Business with filters Filters: `flag/flagStatus eq 'notFlagged' and not categories/any()` |
| Convert to Markdown            | Markdown Node                  | Converts email body HTML to Markdown              | Microsoft Outlook23                  | Email Messages                      | ## Sanitise Email Removes HTML and useless information in preparation for the AI Agent          |
| Email Messages                 | Set Node                       | Cleans and sanitizes email text                    | Convert to Markdown                  | Loop Over Items                    |                                                                                                |
| Loop Over Items                | SplitInBatches Node            | Processes emails one by one                        | Email Messages                      | Contact (branch 2), (branch 1 empty) |                                                                                                |
| Contact                       | Airtable Node                  | Looks up sender in Airtable Contacts               | Loop Over Items                     | Merge                             | ## Match Contact Check if the sender is an existing contact. Contacts loaded from Monday.com    |
| Rules                         | Airtable Node                  | Retrieves rules from Airtable                       | When clicking ‘Test workflow’        | Merge                             | ## Get Rules & Categories Edit the airtables to set your own categories, rules, contacts and/or delete rules. |
| Categories                    | Airtable Node                  | Retrieves categories from Airtable                  | When clicking ‘Test workflow’        | Merge                             |                                                                                                |
| Delete Rules                  | Airtable Node                  | Retrieves delete rules from Airtable                | When clicking ‘Test workflow’        | Merge                             |                                                                                                |
| Merge                        | Merge Node                    | Combines Contact, Rules, Categories, Delete Rules  | Contact, Rules, Categories, Delete Rules | AI: Analyse Email               |                                                                                                |
| AI: Analyse Email             | Langchain Agent Node           | Uses AI to categorize and prioritize emails        | Merge                             | Set Category                     | ## Categorise & Prioritise Emails Agent                                                        |
| OpenAI Chat Model             | Langchain OpenAI Chat Model    | Executes AI model (GPT-4o)                          | AI: Analyse Email (prompt)           | AI: Analyse Email (response)       |                                                                                                |
| Structured Output Parser      | Langchain Output Parser        | Parses AI output to structured JSON                 | OpenAI Chat Model                   | AI: Analyse Email                 |                                                                                                |
| Set Category                 | Microsoft Outlook Node          | Updates Outlook email category                       | AI: Analyse Email                   | If                              | ## Set the category & importance using the output from the agent                               |
| If                           | If Node                       | Checks if subCategory is "Action"                    | Set Category                      | Set Importance (true), Loop Over Items (false) |                                                                                                |
| Set Importance               | Microsoft Outlook Node          | Sets Outlook email importance to High                | If (true branch)                   | Loop Over Items                   |                                                                                                |
| Sticky Note8                 | Sticky Note                   | Title note: Microsoft Outlook AI Email Assistant    | None                             | None                             | # Microsoft Outlook AI Email Assistant                                                         |
| Sticky Note10                | Sticky Note                   | Explains Outlook email filters                       | None                             | None                             | ## Outlook Business with filters Filters: `flag/flagStatus eq 'notFlagged' and not categories/any()` |
| Sticky Note11                | Sticky Note                   | Explains email sanitization                          | None                             | None                             | ## Sanitise Email Removes HTML and useless information in preparation for the AI Agent          |
| Sticky Note12                | Sticky Note                   | Explains rules and categories Airtable setup         | None                             | None                             | ## Get Rules & Categories Edit the airtables to set your own categories, rules, contacts and/or delete rules. |
| Sticky Note                  | Sticky Note                   | Explains CRM contact integration                      | None                             | None                             | ## CRM Contact List Integration For this workflow I am retrieving supplier & client contacts from Monday.com the email assistant has better context to categorise, prioritise and reply to emails. The list is updated daily or you can change the scheduler trigger to update more or less frequently. You could replace this with your own CRM. |
| Sticky Note1                 | Sticky Note                   | Title note for categorisation & prioritisation agent | None                             | None                             | ## Categorise & Prioritise Emails Agent                                                        |
| Sticky Note2                 | Sticky Note                   | Explains setting category and importance in Outlook  | None                             | None                             | ## Set the category & importance using the output from the agent                               |
| Sticky Note3                 | Sticky Note                   | Explains contact matching                              | None                             | None                             | ## Match Contact Check if the sender is an existing contact. Note in this workflow the contacts are dynamically loaded from Monday.com |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node** named `Update Contacts Schedule Trigger`  
   - Set to run daily (default interval can be configured).  
   - This triggers the contact sync process.

2. **Add a Monday.com node** named `Monday.com - Get Contacts`  
   - Operation: `getAll` on `boardItem` resource.  
   - Board ID: `1840712625` (replace with your board).  
   - Group ID: `topics` (replace with your group).  
   - Connect output of `Update Contacts Schedule Trigger` to this node.

3. **Add an Airtable node** named `Airtable - Contacts`  
   - Operation: `upsert` on Contacts table.  
   - Base ID: `appNmgIGA4Fhculsn` (replace with your base).  
   - Table: `tbl8gTTEn96uFRDHE` (Contacts).  
   - Map Monday.com columns to Airtable fields:  
     - Type → Type  
     - Email → Email  
     - Last Name → Extract last word from item name  
     - First Name → Extract first word from item name  
   - Use Email as matching column for upsert.  
   - Connect output of `Monday.com - Get Contacts` to this node.

4. **Create a Manual Trigger node** named `When clicking ‘Test workflow’`  
   - This will manually start the email processing workflow.

5. **Create a Schedule Trigger node** named `Check Mail Schedule Trigger` (optional)  
   - Set to run every 15 minutes (or desired interval).  
   - Initially disable if manual testing is preferred.

6. **Add a Microsoft Outlook node** named `Microsoft Outlook23`  
   - Operation: `getAll` emails.  
   - Limit: 10 emails per run.  
   - Fields: flag, from, importance, replyTo, sender, subject, toRecipients, body, categories, isRead.  
   - Filters:  
     - Custom OData filter: `flag/flagStatus eq 'notFlagged' and not categories/any()`  
     - Folder: specify folder ID for inbox or target folder.  
   - Credentials: Connect your Microsoft 365 Outlook OAuth2 credentials.  
   - Connect outputs of `When clicking ‘Test workflow’` and `Check Mail Schedule Trigger` to this node.

7. **Add a Markdown node** named `Convert to Markdown`  
   - Input: `body.content` from Outlook node.  
   - Converts HTML email body to Markdown.  
   - Connect output of `Microsoft Outlook23` to this node.

8. **Add a Set node** named `Email Messages`  
   - Purpose: Clean and sanitize email text.  
   - Assign fields:  
     - subject: from email JSON  
     - importance: from email JSON  
     - sender: sender email address  
     - from: from email address  
     - body: sanitized text using regex replacements to remove HTML tags, markdown links/images, special characters, multiple spaces, etc.  
     - id: email id  
   - Connect output of `Convert to Markdown` to this node.

9. **Add a SplitInBatches node** named `Loop Over Items`  
   - Processes emails one at a time.  
   - Connect output of `Email Messages` to this node.

10. **Add an Airtable node** named `Contact`  
    - Operation: `search` in Contacts table.  
    - Base and table same as Contacts node above.  
    - Filter formula: `{Email}='{{ $json.from.address }}'` dynamically set per email.  
    - Credentials: Airtable Personal Access Token.  
    - Connect second output branch of `Loop Over Items` to this node.

11. **Add three Airtable nodes** named `Rules`, `Categories`, and `Delete Rules`  
    - Each performs a `search` operation on respective tables in the same Airtable base:  
      - Rules table: `tblMSXbMFKETNToxV`  
      - Categories table: `tbliKDp5PoFNF7YI7`  
      - Delete Rules table: `tbl84EJr7y65ed4zh`  
    - Set `executeOnce` to true for each (fetch once per workflow run).  
    - Connect outputs of `When clicking ‘Test workflow’` to each of these nodes.

12. **Add a Merge node** named `Merge`  
    - Mode: `chooseBranch`  
    - Number of inputs: 4  
    - Connect outputs of `Contact`, `Rules`, `Categories`, and `Delete Rules` to the four inputs of this node.

13. **Add a Langchain Agent node** named `AI: Analyse Email`  
    - Purpose: Use AI to categorize and prioritize email.  
    - Parameters:  
      - Text prompt includes email JSON, contacts, delete rules, categories.  
      - System message instructs AI on categories and output format.  
      - Output parser enabled with JSON schema requiring id, subject, category, analysis; subCategory optional.  
    - Connect output of `Merge` to this node.

14. **Add a Langchain OpenAI Chat Model node** named `OpenAI Chat Model`  
    - Model: `gpt-4o`  
    - Temperature: 0.2  
    - Credentials: OpenAI API key.  
    - Connect AI prompt output from `AI: Analyse Email` to this node.

15. **Add a Langchain Structured Output Parser node** named `Structured Output Parser`  
    - Input schema: JSON schema matching AI output format.  
    - Connect output of `OpenAI Chat Model` to this node.  
    - Connect output of this parser back to `AI: Analyse Email` node’s input parser.

16. **Add a Microsoft Outlook node** named `Set Category`  
    - Operation: `update` email.  
    - Message ID: from AI output `id` field.  
    - Update field: `categories` set to AI output `category`.  
    - Credentials: Microsoft 365 OAuth2.  
    - Connect output of `AI: Analyse Email` to this node.

17. **Add an If node** named `If`  
    - Condition: Check if AI output `subCategory` equals `"Action"`.  
    - Connect output of `Set Category` to this node.

18. **Add a Microsoft Outlook node** named `Set Importance`  
    - Operation: `update` email.  
    - Message ID: from AI output `id`.  
    - Update field: `importance` set to `"High"`.  
    - Credentials: Microsoft 365 OAuth2.  
    - Connect true branch of `If` node to this node.

19. **Connect false branch of `If` node and output of `Set Importance` back to `Loop Over Items`**  
    - This loops processing to next email.

20. **Add Sticky Notes** for documentation at appropriate places to explain filters, sanitization, contact matching, rules, and AI agent.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Microsoft Outlook AI Email Assistant                                                                                              | Workflow title and main purpose.                                                                                |
| Outlook email filters: `flag/flagStatus eq 'notFlagged' and not categories/any()`                                                  | Ensures only fresh, unflagged, uncategorized emails are processed.                                              |
| CRM Contact List Integration: Contacts are retrieved daily from Monday.com to enrich email context.                               | Sticky note explaining CRM integration and update frequency.                                                    |
| Sanitise Email: Removes HTML and useless information to prepare email content for AI processing.                                  | Sticky note describing email cleaning steps.                                                                    |
| Get Rules & Categories: Airtable tables for rules, categories, and delete rules are editable to customize AI behavior.            | Sticky note explaining Airtable customization for rules and categories.                                         |
| OpenAI API Key required: Obtain from https://platform.openai.com/api-keys                                                          | Credential requirement for AI processing.                                                                       |
| Airtable AI Email Assistant Template: https://airtable.com/appuffxqy5HlNYAXJ/shrhb2T92ZMF8FezS/tblP9SIola8yglSc0/viwxAIMM6TvWoahfM | Airtable base template for contacts, rules, categories, and delete rules.                                        |
| Monday.com API Token and Board setup required                                                                                      | Prerequisite for contact sync.                                                                                   |
| Microsoft 365 OAuth2 credentials required for Outlook nodes                                                                        | Prerequisite for Outlook email access and updates.                                                              |

---

This detailed documentation provides a comprehensive understanding of the workflow’s structure, node configurations, data flow, and integration points, enabling advanced users and AI agents to reproduce, modify, or troubleshoot the workflow effectively.