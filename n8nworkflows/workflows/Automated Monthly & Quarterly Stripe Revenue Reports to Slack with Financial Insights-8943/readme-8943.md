Automated Monthly & Quarterly Stripe Revenue Reports to Slack with Financial Insights

https://n8nworkflows.xyz/workflows/automated-monthly---quarterly-stripe-revenue-reports-to-slack-with-financial-insights-8943


# Automated Monthly & Quarterly Stripe Revenue Reports to Slack with Financial Insights

### 1. Workflow Overview

This workflow automates the generation and delivery of detailed monthly and quarterly revenue reports derived from Stripe payment data, directly posting these insights to a designated Slack channel. It targets business owners, finance teams, and analysts who require regular, automated updates on financial performance, customer transactions, refunds, and risk assessments.

The workflow is logically structured into these key blocks:

- **1.1 Scheduling Triggers:** Define when the monthly and quarterly reports are generated.
- **1.2 Date Range Calculation:** Dynamically determine the reporting period based on the trigger.
- **1.3 Stripe Data Retrieval:** Fetch charges and refunds from the Stripe API within the calculated date range.
- **1.4 Data Merging:** Combine charges and refunds into a single dataset.
- **1.5 Financial Metrics Calculation:** Analyze the merged data to compute revenue, refunds, customer insights, payment method breakdown, refund reasons, risk levels, and growth metrics.
- **1.6 Slack Message Formatting:** Convert computed metrics into a well-structured Slack message with markdown and emojis.
- **1.7 Slack Delivery:** Post the formatted report to the configured Slack channel.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduling Triggers

**Overview:**  
This block uses two cron-based schedule triggers to initiate report generation: one for monthly reports (1st day every month at 9 AM) and one for quarterly reports (1st day every 3 months at 9 AM). Both triggers feed into the same subsequent logic.

**Nodes Involved:**  
- Monthly Schedule (1st day, 9 AM)  
- Quarterly Schedule (1st day every 3 months, 9 AM)  
- Schedule Note (sticky)

**Node Details:**

- **Monthly Schedule (1st day, 9 AM):**  
  - Type: Schedule Trigger  
  - Config: Cron expression `0 9 1 */1 *` (runs at 9:00 AM on the first day every month)  
  - Outputs trigger signal to the "Calculate Date Range" node  
  - Edge cases: Ensure server timezone matches expected scheduling; misconfiguration may cause missed runs.

- **Quarterly Schedule (1st day every 3 months, 9 AM):**  
  - Type: Schedule Trigger  
  - Config: Cron expression `0 9 1 */3 *` (runs at 9:00 AM on the first day every 3 months)  
  - Outputs trigger signal to the "Calculate Date Range" node  
  - Edge cases: Same as monthly trigger, plus potential confusion if quarterly months misaligned.

- **Schedule Note:**  
  - Type: Sticky Note  
  - Purpose: Documents schedule configuration for user clarity.

---

#### 2.2 Date Range Calculation

**Overview:**  
Determines whether the report is monthly or quarterly based on current date and calculates the appropriate start and end timestamps for Stripe API queries.

**Nodes Involved:**  
- Calculate Date Range  
- Date Range Note (sticky)

**Node Details:**

- **Calculate Date Range:**  
  - Type: Code  
  - Logic:
    - Reads current date and month.
    - Determines if current month corresponds to a quarter start.
    - Calculates start and end dates for previous month (monthly) or previous quarter (quarterly).
    - Converts dates to Unix timestamps suitable for Stripe API filtering.
    - Outputs JSON with `startDate`, `endDate`, `period` ('monthly' or 'quarterly'), and formatted date strings.
  - Inputs: Trigger from schedule nodes.
  - Outputs: JSON data with date range info, passed to Stripe data nodes.
  - Edge cases: Handle year transitions correctly (e.g., January’s previous month is December last year).
  - Version: Uses n8n Code node v2 syntax.

- **Date Range Note:**  
  - Type: Sticky Note  
  - Purpose: Explains date range logic and output format.

---

#### 2.3 Stripe Data Retrieval

**Overview:**  
Fetches payment charges and refund data from Stripe using the date range determined earlier.

**Nodes Involved:**  
- Get Stripe Charges  
- Get Stripe Refunds  
- Stripe Data Note (sticky)

**Node Details:**

- **Get Stripe Charges:**  
  - Type: Stripe  
  - Resource: Charge  
  - Operation: getAll  
  - Parameters: Returns all charges without limit (returnAll=true)  
  - Authentication: Uses stored Stripe API credentials  
  - Inputs: Receives date range from "Calculate Date Range" (note: date filtering is applied within the code logic or Stripe API parameters if configured, though not explicitly shown)  
  - Outputs: List of charge objects  
  - Edge cases: Stripe API rate limits or credential errors; large data volumes may cause timeouts.

- **Get Stripe Refunds:**  
  - Type: HTTP Request  
  - Method: GET to `https://api.stripe.com/v1/refunds`  
  - Query parameters: limit=100 (pagination not explicitly handled)  
  - Authentication: Predefined Stripe API credentials  
  - Inputs: Date range from "Calculate Date Range" (not explicitly used in query, possible enhancement needed)  
  - Outputs: List of refund objects  
  - Edge cases: Pagination not handled; may miss refunds if >100 in period; API errors; credential issues.

- **Stripe Data Note:**  
  - Type: Sticky Note  
  - Purpose: Describes the purpose of charges and refunds data retrieval.

---

#### 2.4 Data Merging

**Overview:**  
Combines the charges and refunds datasets into a single stream for unified analysis.

**Nodes Involved:**  
- Merge  
- Merge Note (sticky)

**Node Details:**

- **Merge:**  
  - Type: Merge  
  - Mode: Default (likely Append)  
  - Inputs:  
    - Main input: Outputs from "Get Stripe Charges" (index 0)  
    - Secondary input: Outputs from "Get Stripe Refunds" (index 1)  
  - Outputs: Combined dataset to "Calculate Financial Metrics" node  
  - Edge cases: Data format inconsistencies; node expects consistent JSON structure.

- **Merge Note:**  
  - Type: Sticky Note  
  - Purpose: Explains merging of data streams.

---

#### 2.5 Financial Metrics Calculation

**Overview:**  
Processes merged Stripe data to compute key financial metrics, customer insights, payment method breakdowns, refund reasons, risk levels, and growth estimates.

**Nodes Involved:**  
- Calculate Financial Metrics  
- Financial Metrics Note (sticky)

**Node Details:**

- **Calculate Financial Metrics:**  
  - Type: Code  
  - Logic:
    - Separates charges and refunds from merged data.
    - Filters successful, paid, and non-refunded charges.
    - Calculates totals: revenue, refunds, net revenue.
    - Aggregates customer revenue and identifies top 3 customers.
    - Breaks down payment methods by card brand.
    - Computes average transaction value.
    - Estimates monthly recurring revenue (MRR) and annual recurring revenue (ARR).
    - Performs risk analysis based on Stripe risk scores (low, medium, high).
    - Analyzes refund reasons frequency.
    - Calculates refund rates.
    - Outputs a comprehensive JSON object with all metrics and metadata.
  - Inputs: Combined charges/refunds data from "Merge".
  - Outputs: Structured financial metrics JSON for Slack formatting.
  - Edge cases: Missing or incomplete data fields; zero transactions; division by zero; incorrect data types.
  - Version: Uses n8n Code node v2 syntax.

- **Financial Metrics Note:**  
  - Type: Sticky Note  
  - Purpose: Documents the financial analysis performed.

---

#### 2.6 Slack Message Formatting

**Overview:**  
Transforms the calculated metrics into a readable, emoji-enhanced Slack message with currency and percentage formatting, organized into thematic sections.

**Nodes Involved:**  
- Format Slack Message  
- Slack Format Note (sticky)

**Node Details:**

- **Format Slack Message:**  
  - Type: Code  
  - Logic:
    - Extracts data from input JSON.
    - Defines formatting helpers for currency and percentage.
    - Constructs message sections: title, financial summary, growth, transactions, top customers, payment methods, risk, refunds.
    - Uses Markdown and emojis for clarity and emphasis.
    - Includes metadata summary at the end.
    - Outputs Slack message text and blocks JSON structure for Slack API.
  - Inputs: Financial metrics JSON from "Calculate Financial Metrics".
  - Outputs: Slack message payload to send.
  - Edge cases: Empty or missing data arrays; formatting errors; large customer lists truncated to top 3.

- **Slack Format Note:**  
  - Type: Sticky Note  
  - Purpose: Explains Slack message styling and formatting.

---

#### 2.7 Slack Delivery

**Overview:**  
Sends the formatted revenue report message to a specified Slack channel with markdown enabled.

**Nodes Involved:**  
- Send To Slack  
- Slack Delivery Note (sticky)

**Node Details:**

- **Send To Slack:**  
  - Type: Slack  
  - Parameters:  
    - Text: Uses expression to insert the generated Slack message text.  
    - Channel: Selects channel by channel ID (`C09H21LK9BJ`)  
    - Other options: Markdown enabled for formatting  
  - Credentials: Uses configured Slack API credentials  
  - Inputs: Receives formatted message from "Format Slack Message"  
  - Outputs: Slack API response  
  - Edge cases: Invalid or unauthorized channel ID; Slack API rate limits; credential expiration.

- **Slack Delivery Note:**  
  - Type: Sticky Note  
  - Purpose: Highlights the need to update channel ID and markdown usage.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                         | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                                      |
|---------------------------|-------------------|---------------------------------------|-----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------|
| Workflow Description      | Sticky Note       | Workflow purpose and features overview | -                                 | -                                | ## 📊 Stripe Financial Reporting Workflow ...                                                                    |
| Setup Instructions        | Sticky Note       | Setup instructions for credentials and config | -                                 | -                                | ## ⚙️ Setup Instructions ...                                                                                      |
| Schedule Note             | Sticky Note       | Documents scheduling configuration     | -                                 | -                                | ## 🗓️ Schedule Configuration ...                                                                                  |
| Monthly Schedule (1st day, 9 AM) | Schedule Trigger | Monthly report trigger                  | -                                 | Calculate Date Range             | ## 🗓️ Schedule Configuration ...                                                                                  |
| Quarterly Schedule (1st day every 3 months, 9 AM) | Schedule Trigger | Quarterly report trigger                | -                                 | Calculate Date Range             | ## 🗓️ Schedule Configuration ...                                                                                  |
| Calculate Date Range      | Code              | Computes report date range and period  | Monthly Schedule, Quarterly Schedule | Get Stripe Charges, Get Stripe Refunds | ## 📅 Date Range Logic ...                                                                                          |
| Date Range Note           | Sticky Note       | Explains date range calculation         | -                                 | -                                | ## 📅 Date Range Logic ...                                                                                          |
| Get Stripe Charges        | Stripe            | Fetch charges data from Stripe          | Calculate Date Range               | Merge                           | ## 💳 Stripe Data Collection ...                                                                                   |
| Get Stripe Refunds        | HTTP Request      | Fetch refunds data from Stripe          | Calculate Date Range               | Merge                           | ## 💳 Stripe Data Collection ...                                                                                   |
| Stripe Data Note          | Sticky Note       | Describes Stripe data retrieval          | -                                 | -                                | ## 💳 Stripe Data Collection ...                                                                                   |
| Merge                     | Merge             | Combines charges and refunds             | Get Stripe Charges, Get Stripe Refunds | Calculate Financial Metrics     | ## 🔄 Data Merger ...                                                                                               |
| Merge Note                | Sticky Note       | Explains data merging step               | -                                 | -                                | ## 🔄 Data Merger ...                                                                                               |
| Calculate Financial Metrics | Code            | Processes data to compute financial KPIs | Merge                            | Format Slack Message             | ## 📊 Financial Analysis Engine ...                                                                                 |
| Financial Metrics Note    | Sticky Note       | Describes financial metric calculations  | -                                 | -                                | ## 📊 Financial Analysis Engine ...                                                                                 |
| Format Slack Message      | Code              | Formats metrics into Slack message       | Calculate Financial Metrics        | Send To Slack                   | ## 💬 Slack Message Formatting ...                                                                                  |
| Slack Format Note         | Sticky Note       | Details Slack message styling             | -                                 | -                                | ## 💬 Slack Message Formatting ...                                                                                  |
| Send To Slack             | Slack             | Posts the message to Slack                | Format Slack Message               | -                              | ## 🚀 Slack Delivery ...                                                                                            |
| Slack Delivery Note       | Sticky Note       | Advises on Slack channel config           | -                                 | -                                | ## 🚀 Slack Delivery ...                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Nodes:**
   - Create a **Schedule Trigger** node named "Monthly Schedule (1st day, 9 AM)".
     - Set the cron expression to `0 9 1 */1 *` (runs at 9 AM on the first day each month).
   - Create another **Schedule Trigger** node named "Quarterly Schedule (1st day every 3 months, 9 AM)".
     - Set the cron expression to `0 9 1 */3 *` (runs at 9 AM on the first day every 3 months).

2. **Add Code Node for Date Range Calculation:**
   - Add a **Code** node named "Calculate Date Range".
   - Use JavaScript code to:
     - Get current date and month.
     - Determine if the report is quarterly or monthly based on month.
     - Calculate start and end dates for the relevant period.
     - Convert dates to Unix timestamps.
     - Output JSON with `startDate`, `endDate`, `period`, and formatted date strings.
   - Connect both Schedule Trigger nodes’ outputs to this node’s input.

3. **Configure Stripe Credentials:**
   - In n8n, add your Stripe API key as a credential named (e.g.) "Stripe account".
   - Ensure the key has permissions to read charges and refunds.

4. **Create Stripe Node to Get Charges:**
   - Create a **Stripe** node named "Get Stripe Charges".
   - Set Resource to "Charge", Operation to "Get All".
   - Enable "Return All" to true.
   - Attach your Stripe credentials.
   - Connect output of "Calculate Date Range" to this node.
   - (Optional) If available, configure filters or date constraints using the timestamp outputs; otherwise, this may be handled in code later.

5. **Create HTTP Request Node to Get Refunds:**
   - Create an **HTTP Request** node named "Get Stripe Refunds".
   - Set method to GET.
   - URL: `https://api.stripe.com/v1/refunds`
   - Add query parameter `limit=100` (pagination not implemented).
   - Use the same Stripe API credentials.
   - Connect output of "Calculate Date Range" to this node.

6. **Merge Charges and Refunds:**
   - Add a **Merge** node named "Merge".
   - Connect "Get Stripe Charges" output to Merge input 0.
   - Connect "Get Stripe Refunds" output to Merge input 1.

7. **Add Code Node for Financial Metrics:**
   - Create a **Code** node named "Calculate Financial Metrics".
   - Write JavaScript to:
     - Separate charges and refunds from merged data.
     - Filter successful and non-refunded charges.
     - Calculate total revenue, refunds, net revenue.
     - Aggregate customer revenue and identify top customers.
     - Compute payment method breakdown.
     - Calculate average transaction value.
     - Estimate MRR and ARR.
     - Analyze risk levels and refund reasons.
     - Output a structured JSON with all metrics and metadata.
   - Connect output of "Merge" to this node.

8. **Add Code Node for Slack Message Formatting:**
   - Create a **Code** node named "Format Slack Message".
   - Write JavaScript to format the JSON from "Calculate Financial Metrics" into a Slack message:
     - Include emojis, markdown, currency and percentage formatting.
     - Organize message into sections (financial summary, top customers, payment methods, risk, refunds).
     - Output JSON with `text` and `blocks` for Slack API.
   - Connect output of "Calculate Financial Metrics" to this node.

9. **Configure Slack Credentials:**
   - In n8n, add your Slack API token as a credential named (e.g.) "Slack account".
   - Ensure the token has permission to post messages to channels.

10. **Create Slack Node to Send Message:**
    - Add a **Slack** node named "Send To Slack".
    - Set "Text" parameter using an expression to insert `{{$json["text"]}}`.
    - Select channel by ID (update to your Slack channel ID).
    - Enable markdown formatting.
    - Attach Slack credentials.
    - Connect output of "Format Slack Message" to this node.

11. **Add Sticky Notes as Documentation:**
    - Add sticky notes with setup instructions, schedule configuration, Stripe data retrieval explanation, data merge explanation, financial metrics summary, Slack formatting description, and Slack delivery notes.
    - Position and size them for clarity.

12. **Activate Workflow and Test:**
    - Run manually to test data retrieval, processing, and Slack message delivery.
    - Adjust parameters, credentials, or channel IDs as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| The workflow automatically detects whether to run monthly or quarterly reports based on date.   | Date range logic in "Calculate Date Range" node                                                    |
| Stripe API refunds query uses a fixed limit of 100; consider implementing pagination for large data sets. | Stripe API Refunds node may miss refunds beyond 100 per period                                      |
| Slack channel ID must be updated to your target channel to ensure message delivery.             | See "Slack Delivery Note" sticky node                                                               |
| Currency is formatted as USD, modify currency codes in formatting functions if needed.          | Formatting in "Format Slack Message" node                                                           |
| Risk analysis categorizes transactions into Low, Medium, High based on Stripe risk scores.      | See "Calculate Financial Metrics" node                                                             |
| Manual testing before enabling schedule triggers is recommended to verify credentials and config. | Setup Instructions sticky note                                                                       |
| The workflow provides detailed financial insights including refund reasons and customer rankings.| Useful for finance teams and business owners                                                        |
| For more advanced Stripe data handling (pagination, filtering), consider extending the HTTP requests or adding looping logic. | Enhancements for scalability and completeness                                                      |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.