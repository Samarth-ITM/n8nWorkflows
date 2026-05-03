Manage Personal Expenses with Webhooks and Google Sheets Automated Tracker

https://n8nworkflows.xyz/workflows/manage-personal-expenses-with-webhooks-and-google-sheets-automated-tracker-5517


# Manage Personal Expenses with Webhooks and Google Sheets Automated Tracker

---

### 1. Workflow Overview

This workflow, titled **"Personal expense tracker"**, provides a complete automated system to manage personal expenses through a REST API and Google Sheets. It targets individuals or small teams who want to log expenses via external apps or forms and maintain an automatically updated spreadsheet with summaries.

The workflow is structured into two major logical blocks:

- **1.1 Expense Input and Processing**:  
  Handles receiving expense data via a POST webhook, validates and formats the input, stores it into a Google Sheets document, and returns an immediate success or error response.

- **1.2 Daily Summary Automation**:  
  Runs once daily via a scheduled trigger to read all expenses recorded for the current day, calculates totals and category breakdowns, and prepares a summary report (which could be extended for notifications).

Each block consists of a sequence of nodes working together to ensure data integrity, persistence, and timely feedback.


---

### 2. Block-by-Block Analysis

#### 2.1 Expense Input and Processing

**Overview:**  
This block receives expense data through an HTTP webhook, validates and formats it, appends the data to a Google Sheet, calculates a summary for the added expense, and sends a real-time success or error response.

**Nodes Involved:**  
- Expense Input Webhook  
- Validate and Format Expense Data  
- Save Expense to Google Sheets  
- Calculate Monthly Summary  
- Send Success Response  
- Send Error Response  

**Node Details:**

- **Expense Input Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point for expense data submitted via HTTP POST  
  - *Configuration:*  
    - HTTP method: POST  
    - Path: `/add-expense`  
    - Response mode: Uses RespondToWebhook nodes downstream  
  - *Input:* External HTTP POST requests with JSON payload  
  - *Output:* Passes received JSON to next node  
  - *Potential Failures:* Invalid HTTP method, malformed JSON, webhook not activated  
  - *Notes:* Designed for integration with mobile apps, web forms, or API clients  

- **Validate and Format Expense Data**  
  - *Type:* Function  
  - *Role:* Validate input fields and apply defaults  
  - *Configuration:*  
    - Extracts `date`, `category`, `description`, `amount`, `payment_method` from webhook payload  
    - Defaults: current date if missing; category "Other"; description "No description"; payment_method "Cash"  
    - Validates `amount` > 0, throws error if invalid  
    - Validates category against predefined list, defaults to "Other" if invalid  
    - Formats amount to two decimals  
  - *Expressions:* Uses `$input.first().json.body` to access webhook JSON body  
  - *Input:* JSON from webhook node  
  - *Output:* Validated and normalized expense JSON  
  - *Potential Failures:* Throws error on invalid amount, leading to error response branch  

- **Save Expense to Google Sheets**  
  - *Type:* Google Sheets  
  - *Role:* Append validated expense to "Expenses" sheet in Google Sheets document  
  - *Configuration:*  
    - Operation: append  
    - Sheet: "Expenses"  
    - Document ID: placeholder `"REPLACE_WITH_YOUR_SPREADSHEET_ID"` (must be replaced with actual ID)  
    - Cell format: USER_ENTERED (keeps Google Sheets formatting)  
  - *Input:* Validated expense JSON  
  - *Output:* Passes data to next node  
  - *Potential Failures:* Credential errors, incorrect spreadsheet ID, API rate limits, network issues  
  - *Requirements:* Google Sheets OAuth2 credentials configured  

- **Calculate Monthly Summary**  
  - *Type:* Function  
  - *Role:* Create a JSON summary of the current expense including date, category, and amount added  
  - *Configuration:*  
    - Extracts current expense details from validation node (uses `$()` to reference node by name)  
    - Computes current month and year  
    - Builds JSON response with success message and details  
  - *Input:* Output from Google Sheets node (implicitly uses validated expense data)  
  - *Output:* Summary JSON for response  
  - *Potential Failures:* Expression errors if referenced node output missing or malformed data  

- **Send Success Response**  
  - *Type:* RespondToWebhook  
  - *Role:* Sends HTTP 200 response with JSON success summary to the original webhook caller  
  - *Configuration:* Default (outputs data passed to it)  
  - *Input:* Summary JSON from Monthly Summary node  
  - *Output:* HTTP response to client  
  - *Potential Failures:* Network issues while responding  

- **Send Error Response**  
  - *Type:* RespondToWebhook  
  - *Role:* Sends HTTP 400 or error response when validation fails (e.g., invalid amount)  
  - *Configuration:* Default error response node  
  - *Input:* Triggered on validation errors thrown in Function node  
  - *Output:* HTTP error response to client  
  - *Potential Failures:* Network issues while responding  

---

#### 2.2 Daily Summary Automation

**Overview:**  
This block runs daily at 8 PM, reads all expense entries from Google Sheets, filters expenses recorded for the current day, calculates total spending and breakdown by category, and prepares a JSON summary for further use.

**Nodes Involved:**  
- Daily Summary Schedule  
- Read Today's Expenses from Sheet  
- Calculate Daily Total  

**Node Details:**

- **Daily Summary Schedule**  
  - *Type:* Cron  
  - *Role:* Triggers the daily summary workflow at a set time  
  - *Configuration:* Default cron settings (should be set to run daily at 8:00 PM, although not explicitly configured in JSON)  
  - *Input:* None (trigger)  
  - *Output:* Starts the scheduled workflow branch  
  - *Potential Failures:* Cron misconfiguration, workflow inactive  

- **Read Today's Expenses from Sheet**  
  - *Type:* Google Sheets  
  - *Role:* Reads all data from "Expenses" sheet in Google Sheets  
  - *Configuration:*  
    - Operation: read (default)  
    - Sheet: "Expenses"  
    - Document ID: placeholder `"REPLACE_WITH_YOUR_SPREADSHEET_ID"` (must be replaced)  
  - *Input:* Trigger from cron node  
  - *Output:* Array of all expense entries as JSON objects  
  - *Potential Failures:* Credential errors, spreadsheet ID errors, empty sheet, API limits  

- **Calculate Daily Total**  
  - *Type:* Function  
  - *Role:* Filters expenses for the current date and calculates total and category-wise breakdown  
  - *Configuration:*  
    - Extracts current date as ISO string (YYYY-MM-DD)  
    - Filters expenses matching the current date  
    - Sums amounts overall and per category  
    - Returns JSON with date, total expenses, count, category breakdown, and list of today's expenses  
  - *Input:* JSON array from Google Sheets read node  
  - *Output:* JSON summary object of daily expenses  
  - *Potential Failures:* Empty or malformed data, date comparison issues  

---

### 3. Summary Table

| Node Name                      | Node Type           | Functional Role                                     | Input Node(s)                      | Output Node(s)                 | Sticky Note                                                                                                  |
|-------------------------------|---------------------|----------------------------------------------------|----------------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------|
| Expense Input Webhook          | Webhook             | Receives expense data via POST API                  | –                                | Validate and Format Expense Data | ## Step 1: API Input  
**Webhook** receives expense data via POST request  
Endpoint: `/webhook/add-expense`  
Method: POST  
Format: JSON  
*Perfect for mobile apps, web forms, or direct API calls* |
| Validate and Format Expense Data | Function            | Validates and normalizes expense input               | Expense Input Webhook             | Save Expense to Google Sheets  | ## Step 2: Data Validation  
Function Node validates and cleans data:  
• Checks amount > 0  
• Validates categories  
• Sets defaults for missing fields  
• Formats numbers properly  
*Ensures data consistency*                     |
| Save Expense to Google Sheets  | Google Sheets       | Appends validated expense to Google Sheets           | Validate and Format Expense Data | Calculate Monthly Summary       | ## Step 3: Google Sheets Storage  
Google Sheets Node appends expense to spreadsheet  
Sheet: 'Expenses'  
Columns: Date | Category | Description | Amount | Payment Method  
*Your permanent expense database*              |
| Calculate Monthly Summary      | Function            | Creates monthly summary JSON and success message     | Save Expense to Google Sheets    | Send Success Response           | ## Step 4: API Response  
Response Nodes return JSON with:  
✅ Success: Expense details + confirmation  
❌ Error: Validation errors + field requirements  
*Immediate feedback for calling applications* |
| Send Success Response          | RespondToWebhook    | Sends HTTP success response to API caller             | Calculate Monthly Summary        | –                             | See Step 4 sticky note above                                                                                 |
| Send Error Response            | RespondToWebhook    | Sends HTTP error response on validation failure       | Validate and Format Expense Data (on error) | –                   | See Step 4 sticky note above                                                                                 |
| Daily Summary Schedule         | Cron                | Triggers daily summary calculation                     | –                                | Read Today's Expenses from Sheet | ## 📊 Automated Daily Summary  
Cron Trigger runs daily at 8:00 PM to:  
• Read all today's expenses from Google Sheets  
• Calculate total spending and category breakdown  
• Generate summary report  
*Stay informed about your daily spending habits automatically* |
| Read Today's Expenses from Sheet | Google Sheets       | Reads all expense data from Google Sheets              | Daily Summary Schedule           | Calculate Daily Total          | Same as above                                                                                                |
| Calculate Daily Total          | Function            | Filters today's expenses and calculates totals         | Read Today's Expenses from Sheet | –                             | Same as above                                                                                                |
| Main Workflow Explanation     | Sticky Note         | Provides detailed high-level documentation and setup   | –                                | –                             | # 💰 Personal Expense Tracker API  
Complete expense tracking system with webhook API, validation, storage, and daily summaries.             |
| Step 1 - API Input             | Sticky Note         | Explains webhook input step                             | –                                | –                             | Same as Expense Input Webhook sticky note                                                                   |
| Step 2 - Data Validation       | Sticky Note         | Explains validation logic                               | –                                | –                             | Same as Validate and Format Expense Data sticky note                                                        |
| Step 3 - Google Sheets Storage | Sticky Note         | Explains Google Sheets storage                          | –                                | –                             | Same as Save Expense to Google Sheets sticky note                                                           |
| Step 4 - API Response          | Sticky Note         | Explains API response handling                          | –                                | –                             | Same as Calculate Monthly Summary sticky note                                                               |
| Daily Summary Automation       | Sticky Note         | Explains daily summary automation                       | –                                | –                             | Same as Daily Summary Schedule sticky note                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Workflow in n8n and Name It**  
   Set the workflow name to "Personal expense tracker".

2. **Add Webhook Node: "Expense Input Webhook"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `add-expense`  
   - Response Mode: Select "Respond to Webhook" node downstream  
   - Purpose: Entry point for expense data API  

3. **Add Function Node: "Validate and Format Expense Data"**  
   - Connect from "Expense Input Webhook"  
   - Paste this JavaScript code to validate and format input:

    ```javascript
    const body = $input.first().json.body || {};

    const expense = {
      date: body.date || new Date().toISOString().split('T')[0],
      category: body.category || 'Other',
      description: body.description || 'No description',
      amount: parseFloat(body.amount) || 0,
      payment_method: body.payment_method || 'Cash'
    };

    if (expense.amount <= 0) {
      throw new Error('Amount must be greater than 0');
    }

    const validCategories = ['Food', 'Transport', 'Shopping', 'Bills', 'Entertainment', 'Health', 'Other'];
    if (!validCategories.includes(expense.category)) {
      expense.category = 'Other';
    }

    expense.amount = Math.round(expense.amount * 100) / 100;

    return { json: expense };
    ```
   - Purpose: Ensures data integrity and applies defaults  

4. **Add Google Sheets Node: "Save Expense to Google Sheets"**  
   - Connect from "Validate and Format Expense Data"  
   - Operation: Append  
   - Sheet Name: `Expenses`  
   - Document ID: Replace `"REPLACE_WITH_YOUR_SPREADSHEET_ID"` with your actual Google Sheets ID  
   - Cell Format: USER_ENTERED  
   - Credentials: Configure OAuth2 for Google Sheets  
   - Purpose: Store validated expense data  

5. **Add Function Node: "Calculate Monthly Summary"**  
   - Connect from "Save Expense to Google Sheets"  
   - Paste this JavaScript code:

    ```javascript
    const currentExpense = $('Validate and Format Expense Data').first().json;
    const currentDate = new Date(currentExpense.date);
    const currentMonth = currentDate.getMonth() + 1;
    const currentYear = currentDate.getFullYear();

    const summary = {
      expense_added: currentExpense,
      monthly_summary: {
        month: currentMonth,
        year: currentYear,
        category: currentExpense.category,
        amount_added: currentExpense.amount,
        date_updated: new Date().toISOString()
      },
      success: true,
      message: `Expense of $${currentExpense.amount} for ${currentExpense.category} has been recorded successfully.`
    };

    return { json: summary };
    ```
   - Purpose: Prepare success response with summary info  

6. **Add RespondToWebhook Node: "Send Success Response"**  
   - Connect from "Calculate Monthly Summary"  
   - Default configuration to send JSON response back to API client  

7. **Add RespondToWebhook Node: "Send Error Response"**  
   - No direct connection; configure it to trigger on errors from "Validate and Format Expense Data" node (Use the error workflow feature in n8n)  
   - Default configuration to send error details back to API client  

8. **Set Up Error Workflow**  
   - In "Validate and Format Expense Data" node settings, enable error handling to route to "Send Error Response" node upon exceptions  

9. **Add Cron Node: "Daily Summary Schedule"**  
   - Type: Cron  
   - Set to trigger daily at 8:00 PM (adjust time as needed)  

10. **Add Google Sheets Node: "Read Today's Expenses from Sheet"**  
    - Connect from "Daily Summary Schedule"  
    - Operation: Read  
    - Sheet Name: `Expenses`  
    - Document ID: Same Google Sheets ID as above  
    - Credentials: Same OAuth2 Google Sheets credentials  

11. **Add Function Node: "Calculate Daily Total"**  
    - Connect from "Read Today's Expenses from Sheet"  
    - Paste this JavaScript code:

    ```javascript
    const today = new Date().toISOString().split('T')[0];
    const allExpenses = $input.all()[0].json || [];

    const todayExpenses = allExpenses.filter(expense => expense.date === today);

    const categoryTotals = {};
    let totalToday = 0;

    todayExpenses.forEach(expense => {
      const amount = parseFloat(expense.amount || 0);
      totalToday += amount;

      if (!categoryTotals[expense.category]) {
        categoryTotals[expense.category] = 0;
      }
      categoryTotals[expense.category] += amount;
    });

    return {
      json: {
        date: today,
        total_expenses: Math.round(totalToday * 100) / 100,
        expense_count: todayExpenses.length,
        category_breakdown: categoryTotals,
        expenses: todayExpenses
      }
    };
    ```
    - Purpose: Calculate daily expense totals and breakdown  

12. **Activate the workflow** to start listening for webhook requests and scheduled triggers.

13. **Google Sheets Setup**  
    - Create a Google Sheets spreadsheet with a sheet named `Expenses`  
    - Define columns in first row: Date | Category | Description | Amount | Payment Method  
    - Share and authorize Google Sheets OAuth2 credentials accordingly  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The workflow includes detailed sticky notes explaining each major step, API usage, and setup instructions. These notes serve as in-workflow documentation for maintainers and users.                                                  | Sticky Notes nodes within the workflow                                                              |
| API POST endpoint: `/webhook/add-expense` accepts JSON payloads like `{ "amount": 25.50, "category": "Food", "description": "Lunch at cafe", "payment_method": "Credit Card" }`.                                                          | Main Workflow Explanation sticky note                                                               |
| Categories supported: Food, Transport, Shopping, Bills, Entertainment, Health, Other. Invalid categories default to "Other".                                                                                                         | Data Validation function node and sticky notes                                                      |
| Replace all instances of `"REPLACE_WITH_YOUR_SPREADSHEET_ID"` with your actual Google Sheets document ID for correct operation.                                                                                                    | Google Sheets nodes                                                                                   |
| Google Sheets credentials must be set up with OAuth2 and authorized to access the target spreadsheet.                                                                                                                               | Google Sheets nodes and setup instructions                                                          |
| The daily summary currently calculates totals but does not send notifications; this can be extended by adding email or messaging nodes after "Calculate Daily Total".                                                                | Daily Summary Automation sticky note                                                                |
| This workflow is ideal for personal finance tracking, mobile app integration, family expense sharing, or small business expense logging, providing validation, error handling, and real-time feedback.                                | Main Workflow Explanation sticky note                                                               |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.

---