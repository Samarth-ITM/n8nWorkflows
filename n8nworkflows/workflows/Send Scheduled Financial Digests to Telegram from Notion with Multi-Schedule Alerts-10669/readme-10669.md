Send Scheduled Financial Digests to Telegram from Notion with Multi-Schedule Alerts

https://n8nworkflows.xyz/workflows/send-scheduled-financial-digests-to-telegram-from-notion-with-multi-schedule-alerts-10669


# Send Scheduled Financial Digests to Telegram from Notion with Multi-Schedule Alerts

### 1. Workflow Overview

This workflow automates the process of sending scheduled financial digests to a Telegram chat by aggregating and summarizing data from multiple Notion databases related to personal finance. It targets users managing personal finance data in Notion, providing them with daily, weekly, and monthly summarized updates via Telegram.

The workflow is logically organized into four main blocks:

- **1.1 Triggering Mechanism:** This block consists of schedule triggers set at daily, weekly, and monthly intervals, initiating the workflow execution.
- **1.2 Data Retrieval from Notion:** This block queries various Notion databases to fetch financial transactions, budgets, income, funds, invoices, obligations, and asset/liability liquidity data.
- **1.3 Data Processing and Message Formatting:** Using Set and Code nodes, this block processes raw Notion data, reshapes it, aggregates sums, and formats human-readable messages.
- **1.4 Message Dispatch:** This final block sends the formatted financial summaries to a designated Telegram chat.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering Mechanism

**Overview:**  
Initiates the workflow based on multiple schedules: daily, weekly, and monthly (start and end of month).

**Nodes Involved:**  
- Daily Trigger  
- Weekly Trigger  
- Monthly Start Trigger  
- Monthly End Trigger

**Node Details:**

- **Daily Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs every day (interval setting with no specific time restriction)  
  - Input: None (trigger)  
  - Output: Triggers nodes "Get Debit Transactions" and "Get Invoices"  
  - Edge cases: Misconfiguration could cause multiple runs per day or none if the n8n instance is down.

- **Weekly Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs weekly at 1 AM on the specified day of the week  
  - Input: None (trigger)  
  - Output: Triggers multiple Notion data retrieval nodes for weekly summaries  
  - Edge cases: Time zone differences might cause unexpected trigger times.

- **Monthly Start Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs monthly at the start of the month (default day 1)  
  - Output: Triggers "Get Liquidity" and "Get Semi Liquid" nodes  
  - Edge cases: None significant, but ensure n8n runs continuously.

- **Monthly End Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Runs monthly on the 28th day, intended as month-end trigger  
  - Output: Triggers nodes for end-of-month financial summaries  
  - Edge cases: Months with fewer than 28 days may cause issues if extended beyond.

---

#### 1.2 Data Retrieval from Notion

**Overview:**  
This block retrieves various categorized financial data from Notion databases using filters tailored to date ranges, transaction types, and other criteria.

**Nodes Involved:**  
- Get Debit Transactions  
- Get Invoices  
- Get Debit Transactions1 (weekly)  
- Get Financial Obligations  
- Get Monthly Budget Left  
- Get Monthly Budget Spent  
- Get Income This Month  
- Get Income This Month1 (monthly end)  
- Get Debit Transactions2 (monthly end)  
- Fund Amount Left  
- Get Liquidity  
- Get Semi Liquid

**Node Details:**

- **Get Debit Transactions**  
  - Type: Notion node (databasePage, getAll operation)  
  - Filter: Created time on or after today’s start; Type = Debit  
  - Database: Financial Transaction database  
  - Output: All debit transactions for today  
  - Edge cases: API rate limits, data format changes in Notion.

- **Get Invoices**  
  - Type: Notion node  
  - Filter: Type = Invoice  
  - Output: All invoices currently in the database  
  - Edge cases: Large data sets could lead to slow performance.

- **Get Debit Transactions1**  
  - Similar to "Get Debit Transactions" but for the current week (startOf('week'))  
  - Used for weekly summaries.

- **Get Financial Obligations**  
  - Filter: Next Date on or before now; Type = Financial; Repeat Type != off  
  - Database: Scheduler  
  - Output: Financial obligations due now or earlier, repeating obligations only  
  - Edge cases: Date parsing mismatches.

- **Get Monthly Budget Left**  
  - Filter: Currently Applicable = true; Payment Schedule Type = Monthly; Monthly Budget Left > 0  
  - Database: Budget  
  - Output: Budgets still available this month.

- **Get Monthly Budget Spent**  
  - Filter: Currently Applicable = true; Payment Schedule Type = Monthly; Monthly Budget Left <= 0  
  - Output: Budgets already spent or exceeded.

- **Get Income This Month**  
  - Filter: Created time on or after start of month  
  - Database: Income  
  - Output: Income transactions for the month.

- **Get Income This Month1**  
  - Duplicate of above for monthly end trigger.

- **Get Debit Transactions2**  
  - Filter: Created time on or after start of month; Type = Debit  
  - For monthly end summaries.

- **Fund Amount Left**  
  - No specific filter, fetches all funds data from Funds database.

- **Get Liquidity**  
  - Filter: Liquidity = Liquid  
  - Fetches liquid assets and liabilities.

- **Get Semi Liquid**  
  - Filter: Liquidity = Semi Liquid  
  - Fetches semi-liquid assets/liabilities.

**Common Edge Cases:**  
- Notion API limits or downtime  
- Missing or renamed properties in Notion databases  
- Date filter misalignment due to timezones or format

---

#### 1.3 Data Processing and Message Formatting

**Overview:**  
This block processes raw Notion data results: reshapes fields, aggregates sums, and formats textual summaries for human reading and Telegram delivery.

**Nodes Involved:**  
- Set Fields, Set Fields1, Set Fields2, Set Fields3, Set Fields4, Set Fields5  
- Items to Message, Items to Message1, Items to Message2, Items to Message3, Items to Message4, Items to Message5, Items to Message6, Items to Message7, Items to Message8, Items to Message9, Items to Message10, Items to Message11  
- Summarize, Summarize1, Summarize2, Summarize3, Summarize4, Summarize5

**Node Details:**

- **Set Fields Nodes**  
  - Type: Set  
  - Role: Rename and extract specific Notion properties into simplified JSON fields (e.g., `property_next_date.start` to `next-date`, `property_cost` to `cost`, `property_monthly_budget_left` to `budget_left`, `property_balance` to `balance`, etc.)  
  - Input: Output from Notion nodes  
  - Output: Modified JSON with simpler fields for downstream code nodes  
  - Edge cases: Missing properties cause empty or undefined fields.

- **Summarize Nodes**  
  - Type: Summarize  
  - Role: Aggregate sums of numeric properties such as `property_cost` or `property_balance`  
  - Input: Filtered Notion data  
  - Output: Single item with sum value for use in message formatting

- **Items to Message Nodes**  
  - Type: Code  
  - Role: Generate formatted message strings summarizing the data  
  - Logic: Loop over input items or use summarized values to create human-readable text blocks such as:  
    - "Total Expenses Today - Rs X"  
    - "Monthly Budget Left - <list>"  
    - "Invoices still to pay - <list>"  
    - "Funds Info - spent/needed"  
    - "Liquidity Balance - <list>"  
  - Output: JSON with a `message` field containing the formatted text  
  - Edge cases: Empty input arrays lead to empty or minimal messages; code errors if fields missing.

---

#### 1.4 Message Dispatch

**Overview:**  
Sends the composed textual financial digest messages to a Telegram chat.

**Nodes Involved:**  
- Send Message4 (single Telegram node used for all message dispatches)

**Node Details:**

- **Send Message4**  
  - Type: Telegram  
  - Configuration:  
    - Text: Expression referencing `message` field from previous nodes  
    - chatId: Dynamic input parameter (`chat_id`)  
    - Additional Fields: Attribution set to false (no bot signature appended)  
  - Credential: Uses Telegram Bot credentials named "Accountant AI"  
  - Input: Receives formatted message JSON from any "Items to Message" node  
  - Output: Telegram message sent confirmation  
  - Edge cases: Invalid chat ID, Telegram API rate limits, bot permissions, network failures.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                       | Input Node(s)                         | Output Node(s)                      | Sticky Note                                       |
|-------------------------|---------------------------|------------------------------------|-------------------------------------|-----------------------------------|--------------------------------------------------|
| Daily Trigger           | Schedule Trigger          | Daily workflow start trigger       | None                                | Get Debit Transactions, Get Invoices | ## Step 1 - Trigger                               |
| Weekly Trigger          | Schedule Trigger          | Weekly workflow start trigger      | None                                | Get Debit Transactions1, Get Financial Obligations, Get Monthly Budget Left, Get Monthly Budget Spent, Get Income This Month | ## Step 1 - Trigger                               |
| Monthly Start Trigger   | Schedule Trigger          | Monthly start trigger              | None                                | Get Liquidity, Get Semi Liquid     | ## Step 1 - Trigger                               |
| Monthly End Trigger     | Schedule Trigger          | Monthly end trigger                | None                                | Get Debit Transactions2, Fund Amount Left, Get Income This Month1 | ## Step 1 - Trigger                               |
| Get Debit Transactions  | Notion                    | Get today’s debit transactions    | Daily Trigger                      | Summarize                         | ## Step 2 - Get Notion Databases                  |
| Get Invoices            | Notion                    | Get invoices                      | Daily Trigger                      | Set Fields3                      | ## Step 2 - Get Notion Databases                  |
| Get Debit Transactions1 | Notion                    | Get this week’s debit transactions | Weekly Trigger                    | Summarize1                       | ## Step 2 - Get Notion Databases                  |
| Get Financial Obligations | Notion                  | Get financial obligations          | Weekly Trigger                    | Set Fields2                      | ## Step 2 - Get Notion Databases                  |
| Get Monthly Budget Left | Notion                    | Get monthly budgets left           | Weekly Trigger                    | Set Fields                      | ## Step 2 - Get Notion Databases                  |
| Get Monthly Budget Spent | Notion                   | Get monthly budgets spent          | Weekly Trigger                    | Set Fields1                     | ## Step 2 - Get Notion Databases                  |
| Get Income This Month   | Notion                    | Get income for the month           | Weekly Trigger                    | Summarize3                      | ## Step 2 - Get Notion Databases                  |
| Get Income This Month1  | Notion                    | Duplicate monthly income fetch     | Monthly End Trigger               | Summarize4                      | ## Step 2 - Get Notion Databases                  |
| Get Debit Transactions2 | Notion                    | Get monthly debit transactions     | Monthly End Trigger               | Summarize2                      | ## Step 2 - Get Notion Databases                  |
| Fund Amount Left        | Notion                    | Get fund amounts left              | Monthly End Trigger               | Set Fields4                     | ## Step 2 - Get Notion Databases                  |
| Get Liquidity           | Notion                    | Get liquid assets/liabilities     | Monthly Start Trigger             | Set Fields5                     | ## Step 2 - Get Notion Databases                  |
| Get Semi Liquid         | Notion                    | Get semi-liquid assets/liabilities | Monthly Start Trigger             | Summarize5                      | ## Step 2 - Get Notion Databases                  |
| Set Fields              | Set                       | Map monthly budget left fields    | Get Monthly Budget Left           | Items to Message4                | ## Step 3 - Data Manipulation                      |
| Set Fields1             | Set                       | Map monthly budget spent fields   | Get Monthly Budget Spent          | Items to Message5                | ## Step 3 - Data Manipulation                      |
| Set Fields2             | Set                       | Map financial obligations fields  | Get Financial Obligations         | Items to Message2                | ## Step 3 - Data Manipulation                      |
| Set Fields3             | Set                       | Map invoice fields                | Get Invoices                     | Items to Message6                | ## Step 3 - Data Manipulation                      |
| Set Fields4             | Set                       | Map fund amount left fields       | Fund Amount Left                 | Items to Message7                | ## Step 3 - Data Manipulation                      |
| Set Fields5             | Set                       | Map liquidity balance fields      | Get Liquidity                   | Items to Message10               | ## Step 3 - Data Manipulation                      |
| Summarize               | Summarize                 | Sum today’s debit transactions cost | Get Debit Transactions           | Items to Message                 | ## Step 3 - Data Manipulation                      |
| Summarize1              | Summarize                 | Sum weekly debit transactions cost | Get Debit Transactions1          | Items to Message1                | ## Step 3 - Data Manipulation                      |
| Summarize2              | Summarize                 | Sum monthly debit transactions cost | Get Debit Transactions2          | Items to Message3                | ## Step 3 - Data Manipulation                      |
| Summarize3              | Summarize                 | Sum monthly income received       | Get Income This Month            | Items to Message8                | ## Step 3 - Data Manipulation                      |
| Summarize4              | Summarize                 | Sum monthly income received (duplicate) | Get Income This Month1          | Items to Message9                | ## Step 3 - Data Manipulation                      |
| Summarize5              | Summarize                 | Sum semi-liquid balance            | Get Semi Liquid                 | Items to Message11               | ## Step 3 - Data Manipulation                      |
| Items to Message        | Code                      | Format today’s debit expenses message | Summarize                      | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message1       | Code                      | Format weekly debit expenses message | Summarize1                     | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message2       | Code                      | Format financial obligations message | Set Fields2                    | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message3       | Code                      | Format monthly debit expenses message | Summarize2                     | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message4       | Code                      | Format monthly budget left message  | Set Fields                      | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message5       | Code                      | Format monthly budget spent message | Set Fields1                    | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message6       | Code                      | Format invoices to pay message     | Set Fields3                    | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message7       | Code                      | Format funds info message          | Set Fields4                    | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message8       | Code                      | Format total income this month message | Summarize3                   | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message9       | Code                      | Format total income message (duplicate) | Summarize4                   | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message10      | Code                      | Format liquidity balance message   | Set Fields5                    | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Items to Message11      | Code                      | Format semi-liquid balance message | Summarize5                   | Send Message4                   | ## Step 3 - Data Manipulation                      |
| Send Message4           | Telegram                  | Send formatted messages to Telegram | Any "Items to Message" node    | None                           | ## Step 4 - Send Message                           |
| Sticky Note(s)          | Sticky Note               | Workflow documentation and comments | None                         | None                           | Multiple notes covering all workflow steps        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node named "Daily Trigger". Configure it to run **every day**.
   - Add a **Schedule Trigger** node named "Weekly Trigger". Configure it to run **weekly at 1 AM** on the desired weekday.
   - Add a **Schedule Trigger** node named "Monthly Start Trigger". Configure it to run **monthly on day 1**.
   - Add a **Schedule Trigger** node named "Monthly End Trigger". Configure it to run **monthly on day 28**.

2. **Create Notion Nodes for Data Retrieval:**
   - Create **Notion** nodes for each data query:
     - "Get Debit Transactions": Filter on `Created time` ≥ start of today and `Type` = `Debit`.
     - "Get Invoices": Filter on `Type` = `Invoice`.
     - "Get Debit Transactions1": Filter on `Created time` ≥ start of this week and `Type` = `Debit`.
     - "Get Financial Obligations": Filter on `Next Date` ≤ now, `Type` = `Financial`, `Repeat Type` != `off`.
     - "Get Monthly Budget Left": Filter on `Currently Applicable` = true, `Payment Schedule Type` = Monthly, `Monthly Budget Left` > 0.
     - "Get Monthly Budget Spent": Filter on `Currently Applicable` = true, `Payment Schedule Type` = Monthly, `Monthly Budget Left` ≤ 0.
     - "Get Income This Month": Filter on `Created time` ≥ start of month.
     - "Get Income This Month1": Same as above for monthly end.
     - "Get Debit Transactions2": Filter on `Created time` ≥ start of month and `Type` = `Debit`.
     - "Fund Amount Left": No filters; fetch all.
     - "Get Liquidity": Filter on `Liquidity` = Liquid.
     - "Get Semi Liquid": Filter on `Liquidity` = Semi Liquid.
   - Configure each Notion node with your Notion Integration credentials and appropriate database IDs.
   - Make sure all nodes are set to return all results.

3. **Create Set Nodes to Reshape Data:**
   - For each data category, create **Set** nodes that map Notion properties to simplified fields for easier processing, e.g.:
     - `name`: `{{ $json.name }}`  
     - `next-date`: `{{ $json.property_next_date.start }}`  
     - `cost`: `{{ $json.property_cost }}`  
     - `budget_left`: `{{ $json.property_monthly_budget_left }}`  
     - `balance`: `{{ $json.property_balance }}`  
     - `fund_left`, `fund_needed`, `fund_spent` as relevant.
   - Name each Set node clearly (e.g., "Set Fields", "Set Fields1", etc.)

4. **Create Summarize Nodes for Aggregation:**
   - For each numeric dataset (expenses, income, balances), create **Summarize** nodes configured to sum the appropriate property, such as `property_cost` or `property_balance`.
   - Connect the respective Notion nodes to these Summarize nodes.

5. **Create Code Nodes for Message Formatting:**
   - Create **Code** nodes named "Items to Message", "Items to Message1", etc.  
   - Implement JavaScript to generate formatted text messages. Example logic:  
     - Start with a header like "Total Expenses Today - Rs X"  
     - Loop through items to list details (name, date, cost, etc.)  
     - Return a JSON object with a `message` field containing the final string.

6. **Create Telegram Node to Send Messages:**
   - Add a **Telegram** node named "Send Message4".  
   - Configure it with your Telegram Bot credentials (named "Accountant AI").  
   - Set the `chatId` to a variable or hardcode your target chat ID.  
   - Set the message text to `{{$json.message}}`.  
   - Disable attribution.

7. **Connect Nodes:**
   - Connect each Schedule Trigger node to the appropriate Notion data retrieval nodes.
   - Connect Notion nodes to Summarize or Set nodes as appropriate.
   - Connect Summarize/Set nodes to Code nodes for message formatting.
   - Connect all Code nodes to the single Telegram node to send messages.
   - Ensure the flow matches the dependency graph shown in the summary table.

8. **Credentials Setup:**
   - Create Notion API credentials with your integration token, ensuring access to all relevant databases.
   - Create Telegram API credentials using your bot token.
   - Ensure the chat ID is accessible as a variable or parameter within the workflow.

9. **Validation and Testing:**
   - Test each trigger independently to verify data retrieval, aggregation, formatting, and message sending.
   - Monitor for API errors or empty data sets and handle gracefully (e.g., conditionally skip sending empty messages).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------|
| The workflow is designed around a specific Notion Personal Finance System template. Adjust Notion database property names and filters if your setup differs.                                                                                                                                                                                                                                       | [Personal Finance System Notion Template](https://www.notion.so/templates/personal-finance-system)                                                |
| Telegram Bot creation instructions: Use BotFather in Telegram to create a bot, obtain bot token, and get your chat ID to configure message sending.                                                                                                                                                                                                                                              | Telegram Bot API documentation                                                                                                                    |
| To avoid errors where date fields expect strings but receive objects, convert date variables using `{{$now.toISO()}}` or parse Notion date strings with `DateTime.fromISO(...)` in expressions or code nodes.                                                                                                                                                                                         | n8n Expression and date handling best practices                                                                                                  |
| You can customize the workflow by changing date ranges, adding currency symbols, or replacing summary messages with tables or other formats. Telegram node can also be swapped easily with Slack or Email nodes for alternative notification channels.                                                                                                                                               | n8n documentation for nodes and expressions                                                                                                      |
| The workflow uses a single Telegram node to send all messages, which consolidates multiple message strings generated from different financial summaries. You can extend this to send separate messages or batch them differently as needed.                                                                                                                                                           | n8n Telegram node configuration                                                                                                                  |
| Sticky notes in the workflow provide detailed inline documentation and setup instructions; refer to them for additional context during manual recreation or modification.                                                                                                                                                                                                                           | Sticky notes content is integrated in section 3 and workflow overview                                                                           |

---

**Disclaimer:** The provided text exclusively originates from an n8n automation workflow. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.