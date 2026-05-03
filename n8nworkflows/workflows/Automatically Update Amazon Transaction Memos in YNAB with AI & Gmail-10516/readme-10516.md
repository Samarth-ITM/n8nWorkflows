Automatically Update Amazon Transaction Memos in YNAB with AI & Gmail

https://n8nworkflows.xyz/workflows/automatically-update-amazon-transaction-memos-in-ynab-with-ai---gmail-10516


# Automatically Update Amazon Transaction Memos in YNAB with AI & Gmail

---
### 1. Workflow Overview

This workflow automates updating memo fields for Amazon transactions in YNAB (You Need A Budget) by leveraging Gmail email data and AI summarization. It is designed to run on a schedule or via webhook/manual trigger to:

- Retrieve unapproved transactions from YNAB.
- Filter transactions to focus on Amazon purchases with empty memo fields.
- Search Gmail for emails related to each transaction amount within a ±5 day window.
- Use an AI agent to parse and summarize purchase details from the emails.
- Update the YNAB transaction memo with a concise summary or an error message if no valid data is found.
- Incorporate rate limiting to avoid API overload.

The workflow logic is organized into these blocks:

- **1.1 Trigger and Initialization:** Accepts manual, scheduled, or webhook triggers and sets the YNAB budget ID.
- **1.2 Transaction Retrieval and Filtering:** Fetches unapproved transactions, splits them, and filters for relevant Amazon transactions with empty memos.
- **1.3 Batch Processing Loop:** Processes each filtered transaction individually.
- **1.4 Gmail Search and Email Aggregation:** Searches Gmail for emails matching transaction details and aggregates their content.
- **1.5 AI Parsing and Summary Generation:** Uses an AI agent to extract and summarize purchase info from emails.
- **1.6 Memo Update and Error Handling:** Updates YNAB memo field with AI output or error message.
- **1.7 Rate Limiting and Loop Continuation:** Waits before processing the next transaction to avoid rate limiting.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Initialization

- **Overview:** Handles workflow activation via manual trigger, schedule, or webhook and sets required budget ID for YNAB API calls.
- **Nodes Involved:**
  - When clicking ‘Test workflow’
  - Schedule Trigger
  - Webhook
  - BudgetID
  - Sticky Note (Intro)
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **When clicking ‘Test workflow’** | Manual trigger node to start workflow on demand. No parameters configured. Outputs to BudgetID node. |
| **Schedule Trigger** | Triggers workflow every hour. Configured to repeat at hourly intervals. Outputs to BudgetID. |
| **Webhook** | Receives HTTP requests with basic auth for external triggering. Configured with a unique path and basic auth credentials. Outputs to BudgetID. |
| **BudgetID** | Set node assigning the YNAB budget ID string. This is a required parameter for YNAB API calls. The value must be replaced with the user’s own budget ID. |
| **Sticky Note** | Provides an overview and instructions about triggering and workflow purpose. Non-functional. |

- **Edge Cases / Failures:**  
  - Missing or incorrect budget ID will cause API calls to fail downstream.  
  - Webhook authentication failures if credentials are incorrect.  
  - Schedule trigger depends on n8n instance uptime.  
- **Version Requirements:** No special version requirements.  

---

#### 1.2 Transaction Retrieval and Filtering

- **Overview:** Retrieves all unapproved transactions from the specified YNAB budget and filters transactions to Amazon-related purchases with empty memo fields.
- **Nodes Involved:** 
  - Get All Unapproved Transactions
  - Split out Transactions
  - Extract Needed Data
  - Check if Amazon Purchase
  - Check if Memo field is Empty
  - Sticky Notes (Filter explanation)
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **Get All Unapproved Transactions** | HTTP Request node calling YNAB API to GET unapproved transactions for the budget. Uses YNAB API credentials. Query parameter `type=unapproved` restricts results. |
| **Split out Transactions** | SplitOut node that splits the array of transactions into individual items for separate processing. Splits on `data.transactions`. |
| **Extract Needed Data** | Set node extracting and simplifying transaction fields: date, amount (converted to decimal), account name, payee name, category, original payee line, approval status, transaction ID, account ID, and memo. Prepares data for filtering and later use. |
| **Check if Amazon Purchase** | If node checks if the lowercase `payee_name` contains "amazon". Only Amazon transactions proceed. Case sensitive is true, but input lowercased for consistency. |
| **Check if Memo field is Empty** | If node checks if the memo field is empty (string empty). Only transactions with empty memos proceed for AI processing. |
| **Sticky Notes** | Explain filtering logic and usage of budget ID. Non-functional. |

- **Edge Cases / Failures:**  
  - API rate limits or authentication errors may cause transaction retrieval to fail.  
  - Transactions with missing expected fields may cause expression errors.  
  - Transactions with non-Amazon payees or with pre-existing memos are filtered out, ensuring efficiency.  
- **Version Requirements:** YNAB API and credential type version compatible with node version 4.2.  

---

#### 1.3 Batch Processing Loop

- **Overview:** Processes each filtered Amazon transaction individually in batches for controlled execution and to avoid API overload.
- **Nodes Involved:** 
  - Loop Over Transactions
  - Sticky Note (Loop explanation)
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **Loop Over Transactions** | SplitInBatches node iterates over each transaction one at a time, allowing per-transaction processing and rate limiting. No special batch size configured, defaults to 1. |
| **Sticky Note** | Describes looping mechanism and purpose. Non-functional. |

- **Edge Cases / Failures:**  
  - Large numbers of transactions could delay processing.  
  - If batch size modified, rate limits must be adjusted accordingly.  
- **Version Requirements:** None specific.  

---

#### 1.4 Gmail Search and Email Aggregation

- **Overview:** For each transaction, searches Gmail for emails matching the negative transaction amount within a 5-day window before and after the transaction date, aggregates email content for AI analysis.
- **Nodes Involved:** 
  - Search Emails for Transaction Amount
  - Check if emails found
  - Format Emails
  - Join emails together
  - No Emails Found (NoOp)
  - Sticky Note (Email analysis explanation)
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **Search Emails for Transaction Amount** | Gmail node configured with OAuth2 credentials. Searches emails with query `amount * -1` (negated transaction amount) and date window ±5 days around transaction date. Retrieves all matching emails. |
| **Check if emails found** | If node checks if any Gmail messages were found by testing the existence of `id` field in the result. Branches to either formatting or error flow. |
| **Format Emails** | Set node formats each email into a string containing headers `to`, `from`, date, subject, and full HTML content for AI input. |
| **Join emails together** | Aggregate node concatenates all formatted email messages into a single field `messages` for input to AI. |
| **No Emails Found** | NoOp node used as a placeholder to proceed with error memo update if no emails found. |
| **Sticky Note** | Describes email search and aggregation logic. |

- **Edge Cases / Failures:**  
  - Gmail OAuth2 credentials must be valid and authorized.  
  - Emails might be missing expected headers or content, reducing AI accuracy.  
  - Large email content could increase token usage and cost on AI models.  
- **Version Requirements:** Gmail node v2.1+ for OAuth2 and advanced search features.  

---

#### 1.5 AI Parsing and Summary Generation

- **Overview:** Uses a LangChain AI agent with OpenAI GPT-4 mini model to parse aggregated emails and transaction info, extracting and summarizing purchased items and prices into a concise memo string.
- **Nodes Involved:** 
  - Memo AI Agent
  - OpenAI Chat Model
  - Calculator (tool integration)
  - Check if AI Able to parse items
  - Sticky Note (AI usage explanation)
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **Memo AI Agent** | LangChain agent node configured with a detailed system prompt instructing it to identify and summarize purchase details from emails related to the transaction. Input includes transaction info and concatenated email messages. Output is a single-line summary or error phrase. |
| **OpenAI Chat Model** | Provides GPT-4.1-mini model access via OpenAI API credentials. Connected as language model for the AI agent. |
| **Calculator** | LangChain tool node that assists the AI in calculations if needed (e.g., summing prices). Linked as a tool to the AI agent. |
| **Check if AI Able to parse items** | If node checks if AI output contains the phrase "No valid purchase information found." If not found, proceed with memo update; otherwise, send error memo update. |
| **Sticky Note** | Notes on AI agent usage, model options, cost considerations, and prompt design. |

- **Edge Cases / Failures:**  
  - AI may fail to parse items correctly, returning no valid info.  
  - API rate limits or credential issues with OpenAI.  
  - Token limit exceeded if emails are too verbose.  
  - Prompt or AI version changes may affect output format and accuracy.  
- **Version Requirements:** LangChain nodes v1.2+ and OpenAI credentials configured.  

---

#### 1.6 Memo Update and Error Handling

- **Overview:** Updates the memo field of the YNAB transaction with the AI-generated summary or a fixed error message if no valid purchase info was found.
- **Nodes Involved:** 
  - Update Transaction Memo Field
  - Update memo with Error
  - Check Next Transaction (NoOp)
  - Sticky Note (Update explanation)
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **Update Transaction Memo Field** | HTTP Request node sends a PUT request to YNAB API to update the memo field of the specific transaction. Memo is cleaned of special characters and truncated to 499 characters to meet YNAB limits. Uses budget ID and transaction ID from prior nodes. |
| **Update memo with Error** | HTTP Request node updates memo with a hardcoded error message: "No valid purchase information found." This prevents reprocessing of the same transaction repeatedly. |
| **Check Next Transaction** | NoOp node used as a checkpoint before rate limiting wait and next iteration. |
| **Sticky Note** | Explains memo update logic, memo length constraints, and importance of error memo to avoid repeated unnecessary AI calls. |

- **Edge Cases / Failures:**  
  - API authentication or network errors updating memo fields.  
  - Memo exceeding character limits if not properly truncated.  
  - Incorrect transaction or budget IDs causing update failures.  
- **Version Requirements:** YNAB API calls compatible with node v4.2+.  

---

#### 1.7 Rate Limiting and Loop Continuation

- **Overview:** Waits a fixed period between processing transactions to avoid hitting API rate limits, then loops back to process next transaction.
- **Nodes Involved:** 
  - Wait 5 seconds to avoid Rate Limiting
- **Node Details:**

| Node Name | Description |
|-----------|-------------|
| **Wait 5 seconds to avoid Rate Limiting** | Wait node configured to delay workflow execution for 5 seconds before looping back to the batch processor. Helps manage API call frequency, especially for AI and YNAB API. |
| **Sticky Note** | Included in update memo block explaining importance of wait for API limits. |

- **Edge Cases / Failures:**  
  - Wait time should be adjusted based on actual API limits to avoid throttling.  
- **Version Requirements:** None specific.  

---

### 3. Summary Table

| Node Name                          | Node Type                 | Functional Role                           | Input Node(s)                        | Output Node(s)                          | Sticky Note                                                                                                                |
|-----------------------------------|---------------------------|-----------------------------------------|------------------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger            | Manual start trigger                     |                                    | BudgetID                               |                                                                                                                            |
| Schedule Trigger                  | Schedule Trigger          | Scheduled hourly trigger                 |                                    | BudgetID                               |                                                                                                                            |
| Webhook                          | Webhook                   | External webhook trigger with auth      |                                    | BudgetID                               |                                                                                                                            |
| BudgetID                        | Set                       | Set YNAB budget ID                       | When clicking ‘Test workflow’, Webhook, Schedule Trigger | Get All Unapproved Transactions         | "### Update with your budget id" - instructs user to set budget ID from YNAB app URL                                         |
| Get All Unapproved Transactions  | HTTP Request              | Retrieve unapproved transactions from YNAB | BudgetID                          | Split out Transactions                 |                                                                                                                            |
| Split out Transactions           | SplitOut                  | Split transaction array into single transactions | Get All Unapproved Transactions   | Extract Needed Data                    |                                                                                                                            |
| Extract Needed Data              | Set                       | Extract and simplify transaction fields | Split out Transactions             | Check if Amazon Purchase               | "## Filter out unneeded Transactions" - explains filtering logic                                                           |
| Check if Amazon Purchase         | If                        | Filter only Amazon payees                | Extract Needed Data                | Check if Memo field is Empty           |                                                                                                                            |
| Check if Memo field is Empty     | If                        | Filter transactions with empty memo     | Check if Amazon Purchase           | Loop Over Transactions                 |                                                                                                                            |
| Loop Over Transactions           | SplitInBatches            | Process transactions one at a time      | Check if Memo field is Empty       | Search Emails for Transaction Amount (on data), and empty output branch | "## Loop through Transactions" - explains batch processing                                                                  |
| Search Emails for Transaction Amount | Gmail                    | Search Gmail for emails matching amount and date | Loop Over Transactions           | Check if emails found                  | "## Analyze Emails for Transaction Data" - describes email search and aggregation                                            |
| Check if emails found            | If                        | Verify if any emails were found          | Search Emails for Transaction Amount | Format Emails, No Emails Found         |                                                                                                                            |
| Format Emails                   | Set                       | Format emails into strings for AI input | Check if emails found (true branch) | Join emails together                   |                                                                                                                            |
| Join emails together            | Aggregate                 | Combine all email strings into one field | Format Emails                    | Memo AI Agent                        |                                                                                                                            |
| No Emails Found                 | NoOp                      | Placeholder for no emails found branch   | Check if emails found (false branch) | Update memo with Error                 |                                                                                                                            |
| Memo AI Agent                  | LangChain Agent           | AI agent to parse emails and summarize purchases | Join emails together, Loop Over Transactions | Check if AI Able to parse items       | "## Use AI to Extract information" - details AI prompt and model usage; cost and accuracy notes                              |
| OpenAI Chat Model              | LangChain LM Chat OpenAI  | GPT-4 mini model for AI agent            |                                  | Memo AI Agent (ai_languageModel input) |                                                                                                                            |
| Calculator                    | LangChain Tool Calculator | Helper tool for AI calculations           |                                  | Memo AI Agent (ai_tool input)          |                                                                                                                            |
| Check if AI Able to parse items | If                        | Check if AI output contains valid info  | Memo AI Agent                    | Update Transaction Memo Field (true), Update memo with Error (false) |                                                                                                                            |
| Update Transaction Memo Field  | HTTP Request              | Update YNAB transaction memo with AI summary | Check if AI Able to parse items  | Check Next Transaction                 | "## Update Memo Field and Move on to next Transaction" - explains memo update and error handling                             |
| Update memo with Error          | HTTP Request              | Update YNAB memo with error message      | Check if AI Able to parse items, No Emails Found | Check Next Transaction                 |                                                                                                                            |
| Check Next Transaction          | NoOp                      | Placeholder to proceed to next transaction | Update Transaction Memo Field, Update memo with Error | Wait 5 seconds to avoid Rate Limiting |                                                                                                                            |
| Wait 5 seconds to avoid Rate Limiting | Wait                      | Delay to prevent API rate limiting       | Check Next Transaction             | Loop Over Transactions                 | "Check your AI model API limits to set the wait value correctly."                                                           |
| Sticky Note (multiple)          | Sticky Note               | Documentation and explanatory notes      |                                  |                                        | Various notes covering triggers, filtering, email analysis, AI usage, memo updates, and budget ID setup (see section 2).    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".
   - Add a **Schedule Trigger** node named "Schedule Trigger", set to repeat every 1 hour.
   - Add a **Webhook** node named "Webhook":
     - Set HTTP method: POST (default).
     - Set Path to a unique string.
     - Enable Basic Authentication and configure credentials.
   - Connect all three triggers to the next node.

2. **Set Budget ID:**
   - Add a **Set** node named "BudgetID".
   - Create a string field `budgetid` and set its value to your YNAB budget ID (found in your YNAB app URL).
   - Connect all trigger nodes to this node.

3. **Retrieve Unapproved Transactions:**
   - Add an **HTTP Request** node named "Get All Unapproved Transactions".
   - Configure method: GET.
   - URL: `https://api.ynab.com/v1/budgets/{{ $json.budgetid }}/transactions`
   - Add query parameter: `type=unapproved`.
   - Use YNAB API credentials (OAuth2 or personal token).
   - Connect "BudgetID" node to this node.

4. **Split Transactions:**
   - Add a **SplitOut** node named "Split out Transactions".
   - Set field to split: `data.transactions`.
   - Connect from "Get All Unapproved Transactions".

5. **Extract Needed Data:**
   - Add a **Set** node named "Extract Needed Data".
   - Map these fields from incoming JSON:
     - `date` as string: `{{$json.date}}`
     - `amount` as number: `{{ ($json.amount / 1000).toFixed(2) }}`
     - `account_name`, `payee_name`, `category_name`, `import_payee_name_original`, `approved`, `transaction_id` (from `id`), `account_id`, `memo`.
   - Connect from "Split out Transactions".

6. **Filter Amazon Transactions:**
   - Add an **If** node named "Check if Amazon Purchase".
   - Condition: `payee_name.toLowerCase()` contains "amazon".
   - Connect from "Extract Needed Data".

7. **Filter Empty Memo:**
   - Add an **If** node named "Check if Memo field is Empty".
   - Condition: `memo` is empty string.
   - Connect true branch from "Check if Amazon Purchase".

8. **Setup Batch Processing:**
   - Add a **SplitInBatches** node named "Loop Over Transactions".
   - Default batch size 1.
   - Connect true branch from "Check if Memo field is Empty".

9. **Search Gmail for Emails:**
   - Add a **Gmail** node named "Search Emails for Transaction Amount".
   - Set operation: "Get All".
   - Query (`q`): `={{ ($json.amount * -1).toFixed(2) }}`
   - Filters:
     - `receivedAfter`: `={{ $json.date.toDateTime().minus(5,'day') }}`
     - `receivedBefore`: `={{ $json.date.toDateTime().plus(5,'day') }}`
   - Use OAuth2 Gmail credentials.
   - Connect from "Loop Over Transactions".

10. **Check if Emails Found:**
    - Add an **If** node named "Check if emails found".
    - Condition: Check if `$json.id` exists (email found).
    - Connect from "Search Emails for Transaction Amount".

11. **Format Emails:**
    - Add a **Set** node named "Format Emails".
    - Create field `message` combining headers (`to`, `from`), date, subject, and full HTML from each email.
    - Connect true branch from "Check if emails found".

12. **Join Emails:**
    - Add an **Aggregate** node named "Join emails together".
    - Aggregate field: `message` into `messages` (concatenate).
    - Connect from "Format Emails".

13. **No Emails Found Path:**
    - Add a **NoOp** node named "No Emails Found".
    - Connect false branch from "Check if emails found".

14. **AI Agent Setup:**
    - Add a **LangChain agent** node named "Memo AI Agent".
    - In parameters:
      - Text: Compose from transaction info and `$json.messages`.
      - System message: Insert detailed prompt instructing parsing and summarizing purchase info, including format and fallback message.
      - Attach tools: Add Calculator tool node.
      - Use OpenAI Chat model node configured with GPT-4 mini model and your OpenAI API credentials.
    - Connect from "Join emails together" (true branch) and from OpenAI Chat Model (as ai_languageModel), Calculator (as ai_tool).

15. **Check AI Output:**
    - Add an **If** node named "Check if AI Able to parse items".
    - Condition: AI output string does NOT contain "No valid purchase information found."
    - Connect from "Memo AI Agent".

16. **Update Memo Field:**
    - Add an **HTTP Request** node named "Update Transaction Memo Field".
    - Method: PUT.
    - URL: `https://api.ynab.com/v1/budgets/{{ $('BudgetID').item.json.budgetid }}/transactions/{{ $('Loop Over Transactions').item.json.transaction_id }}`
    - Body (JSON):  
      ```json
      {
        "transaction": {
          "memo": "{{ ($json.output || '').replace(/[\\\n\r\t\"]/g, ' ').replace(/\s+/g, ' ').trim().slice(0, 499) }}"
        }
      }
      ```
    - Use YNAB API credentials.
    - Connect true branch from "Check if AI Able to parse items".

17. **Update Memo with Error:**
    - Add an **HTTP Request** node named "Update memo with Error".
    - Method: PUT.
    - URL: Same as above.
    - Body (JSON):  
      ```json
      {
        "transaction": {
          "memo": "No valid purchase information found."
        }
      }
      ```
    - Use YNAB API credentials.
    - Connect false branch from "Check if AI Able to parse items".
    - Also connect from "No Emails Found" node.

18. **Next Transaction Handling:**
    - Add a **NoOp** node named "Check Next Transaction".
    - Connect from both "Update Transaction Memo Field" and "Update memo with Error".

19. **Rate Limiting Wait:**
    - Add a **Wait** node named "Wait 5 seconds to avoid Rate Limiting".
    - Set wait time: 5 seconds.
    - Connect from "Check Next Transaction".

20. **Loop Back:**
    - Connect "Wait 5 seconds to avoid Rate Limiting" back to "Loop Over Transactions" node to process next batch.

21. **Add Sticky Notes:**
    - Add multiple sticky notes to document workflow blocks and instructions as per above descriptions.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow is designed to automatically enhance YNAB transaction memos for Amazon purchases by leveraging Gmail data and AI summarization. | Workflow purpose |
| Budget ID can be found in your YNAB app URL: https://app.ynab.com/XXXXXXXX-XXX-XXX-XXX-XXXXXXX/budget | Sticky Note6 |
| AI prompt designed to extract concise purchase info from verbose Amazon-related emails including order confirmations, ignoring shipping/delivery notes. | Sticky Note5 |
| Use API wait time to avoid hitting OpenAI or YNAB API rate limits; adjust wait based on your subscription limits. | Sticky Note4 |
| Gmail OAuth2 credentials must have permission to search and read emails. | Gmail node context |
| For best AI results, GPT-4 is recommended; cheaper or free alternatives may yield less accurate summaries. | Sticky Note5 |
| The memo field is limited to ~500 characters by YNAB API; AI output is truncated accordingly. | Update Memo explanation |
| Workflow can be triggered manually, on schedule, or via webhook with basic auth for flexibility. | Sticky Note (Trigger and Overview) |

---

**Disclaimer:** The provided text is exclusively sourced from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.