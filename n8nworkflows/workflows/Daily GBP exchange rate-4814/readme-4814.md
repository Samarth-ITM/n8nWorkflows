Daily GBP exchange rate

https://n8nworkflows.xyz/workflows/daily-gbp-exchange-rate-4814


# Daily GBP exchange rate

### 1. Workflow Overview

This workflow is designed to fetch the latest foreign exchange rates relative to the Euro (EUR), transform the data to show rates versus the British Pound (GBP), and distribute this information daily via email. Additionally, it logs the USD rate into a Google Sheet for record-keeping.

The workflow’s logic can be grouped into these blocks:

- **1.1 Trigger and Data Retrieval:** Manual trigger followed by an HTTP request to an exchange rate API.
- **1.2 Data Transformation:** Convert raw API response into an HTML table representing currency rates relative to GBP.
- **1.3 Data Distribution:** Send the HTML table via Gmail email.
- **1.4 Data Logging:** Append the USD exchange rate and date to a Google Sheet for archival.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Retrieval

**Overview:**  
This block initiates the workflow manually and performs an HTTP request to fetch the latest exchange rates from an external API.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- HTTP Request

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Initiates workflow execution manually by user command.  
  - *Configuration:* No parameters; simply triggers the subsequent node.  
  - *Input:* None  
  - *Output:* Triggers HTTP Request node  
  - *Edge Cases:* None typical; if not triggered manually, workflow does not run.

- **HTTP Request**  
  - *Type:* HTTP Request Node  
  - *Role:* Fetches latest exchange rates from Fixer.io API.  
  - *Configuration:*  
    - URL: `http://data.fixer.io/api/latest?access_key=30f059a57833bce0031c7d08bc3825c2&base=EUR&symbols=USD,AUD,CAD`  
    - Method: GET (default)  
    - No additional options set.  
  - *Key Expression:* None; static URL.  
  - *Input:* Trigger from Manual Trigger node  
  - *Output:* JSON response containing date, base currency, and rates object with USD, AUD, CAD rates.  
  - *Edge Cases:*  
    - HTTP errors (e.g., 401 Unauthorized if API key invalid, 429 rate limit exceeded)  
    - Network timeouts or connection issues  
    - API changes affecting response format  
  - *Version:* 4.2

---

#### 2.2 Data Transformation

**Overview:**  
Transforms the raw JSON response into an HTML table listing each currency and its rate relative to GBP (although the API returns rates relative to EUR, so there might be a logical mismatch here).

**Nodes Involved:**  
- Code

**Node Details:**  

- **Code**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Generate an HTML table from the exchange rates JSON.  
  - *Configuration:*  
    - JavaScript code reads `items[0].json` (the HTTP request response).  
    - Builds an HTML table with columns “Currency” and “Rate (vs GBP)”.  
    - Iterates over all currency rates in the `rates` object, creating rows.  
  - *Key Expressions:*  
    - `items[0].json.rates` accessed for currency-rate pairs.  
  - *Input:* Output from HTTP Request node  
  - *Output:* JSON object with a property `htmlTable` containing the generated HTML string.  
  - *Edge Cases:*  
    - If `rates` is undefined or empty, output will be an empty table.  
    - The comment says "Rate (vs GBP})" but the base is EUR, not GBP — possible semantic error or mismatch.  
    - If API response changes structure, code must be updated.  
  - *Version:* 2

---

#### 2.3 Data Distribution

**Overview:**  
Sends the generated HTML table via Gmail to a specified email address.

**Nodes Involved:**  
- send email to adress

**Node Details:**  

- **send email to adress**  
  - *Type:* Gmail Node  
  - *Role:* Sends an email containing the exchange rate table.  
  - *Configuration:*  
    - Send To: `miha.ambroz@n8n.io` (hardcoded recipient)  
    - Subject: `rates`  
    - Message Body: Uses expression to insert `htmlTable` from Code node output: `={{ $json.htmlTable }}`  
    - No additional options configured.  
  - *Credentials:* Gmail OAuth2 authentication required (configured with credential name “Gmail account 2”).  
  - *Input:* Output of Code node (HTML table JSON)  
  - *Output:* Email sent confirmation data  
  - *Edge Cases:*  
    - Authentication failure with Gmail OAuth2  
    - Email delivery failure due to invalid address or quota limits  
    - HTML rendering issues in email clients  
  - *Version:* 2.1

---

#### 2.4 Data Logging

**Overview:**  
Appends a new row into a Google Sheet with the date, USD exchange rate, and base currency for record-keeping.

**Nodes Involved:**  
- Google Sheets

**Node Details:**  

- **Google Sheets**  
  - *Type:* Google Sheets Node  
  - *Role:* Append exchange rate data into a Google Sheet  
  - *Configuration:*  
    - Operation: Append  
    - Sheet Name: “Sheet1” (gid=0)  
    - Document ID: Google Sheet with ID `1cO7Qo4g8o5zaXi54KmqzZYDDHw0yFLJ_-BPyNnLfOG8`  
    - Columns mapped:  
      - Date: from HTTP Request response `date` property  
      - Rate: from HTTP Request response `rates.USD` property  
      - Currency: from HTTP Request response `base` property (EUR)  
    - Mapping Mode: defineBelow (explicit mapping)  
  - *Credentials:* Google Sheets OAuth2 API (credential name “Google Sheets account 2”)  
  - *Input:* Output from HTTP Request node (directly mapped, not from Code node)  
  - *Output:* Append confirmation  
  - *Edge Cases:*  
    - Authentication failure with Google Sheets OAuth2  
    - Invalid document ID or sheet name  
    - API rate limits or quota exceeded  
  - *Version:* 4.6

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role           | Input Node(s)              | Output Node(s)            | Sticky Note                                      |
|---------------------------|----------------------|--------------------------|----------------------------|---------------------------|-------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Workflow start trigger   | —                          | HTTP Request              | This workflow send daily email to desired adress with exchnage rates for GBP. |
| HTTP Request              | HTTP Request         | Fetch exchange rates from Fixer.io API | When clicking ‘Execute workflow’ | Code                      |                                                 |
| Code                      | Code                 | Transform JSON into HTML table | HTTP Request               | send email to adress      |                                                 |
| send email to adress      | Gmail                | Send exchange rates email | Code                       | —                         |                                                 |
| Google Sheets             | Google Sheets        | Log USD exchange rate to sheet | HTTP Request               | —                         |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name it `When clicking ‘Execute workflow’`.
   - No configuration needed.

3. **Add an HTTP Request node:**
   - Name it `HTTP Request`.
   - Set URL to:  
     `http://data.fixer.io/api/latest?access_key=30f059a57833bce0031c7d08bc3825c2&base=EUR&symbols=USD,AUD,CAD`  
   - Method: GET (default).
   - Connect the Manual Trigger node output to this node.

4. **Add a Code node:**
   - Name it `Code`.
   - Set the language to JavaScript.
   - Insert the following code (adapted for clarity):

```javascript
const response = items[0].json;

let html = `<table border="1" cellpadding="5" cellspacing="0">
  <thead>
    <tr>
      <th>Currency</th>
      <th>Rate (vs GBP)</th>
    </tr>
  </thead>
  <tbody>`;

for (const [currency, rate] of Object.entries(response.rates)) {
  html += `<tr><td>${currency}</td><td>${rate}</td></tr>`;
}

html += `</tbody></table>`;

return [{ json: { htmlTable: html } }];
```

   - Connect the HTTP Request node output to this Code node.

5. **Add a Gmail node:**
   - Name it `send email to adress`.
   - Credentials: Select or create Gmail OAuth2 credentials (must have Gmail API enabled, OAuth2 authentication configured).
   - Set “Send To” to the desired recipient email (example: `miha.ambroz@n8n.io`).
   - Set “Subject” to `rates`.
   - Set “Message” to expression: `={{ $json.htmlTable }}` (this inserts the HTML table generated by the Code node).
   - Connect the Code node output to this Gmail node.

6. **Add a Google Sheets node:**
   - Name it `Google Sheets`.
   - Credentials: Select or create Google Sheets OAuth2 credentials.
   - Operation: Append
   - Document ID: Paste your Google Sheet ID (from its URL).
   - Sheet Name: `Sheet1` or the name of the target sheet.
   - Define columns to append:  
     - Date: `={{ $('HTTP Request').item.json.date }}`  
     - Rate: `={{ $('HTTP Request').item.json.rates.USD }}`  
     - Currency: `={{ $('HTTP Request').item.json.base }}`
   - Connect the HTTP Request node output to this Google Sheets node.

7. **Connect the nodes as follows:**

   - `When clicking ‘Execute workflow’` → `HTTP Request` → `Code` → `send email to adress`  
   - `HTTP Request` → `Google Sheets` (parallel branch)

8. **Save and activate the workflow.**

9. **Test the workflow by clicking the manual trigger execute button.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                                 |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This workflow sends daily email with exchange rates for GBP, but the API base is EUR; rates are relative to EUR. | Sticky Note in workflow: “Instruction: This workflow send daily email to desired adress with exchnage rates for GBP.” |
| Gmail node requires OAuth2 credentials for authentication to send emails via Gmail API.                        | Gmail OAuth2 Setup: https://docs.n8n.io/credentials/gmail/                                                     |
| Google Sheets node requires OAuth2 credentials with appropriate scopes to append data to sheets.               | Google Sheets OAuth2 Setup: https://docs.n8n.io/credentials/google-sheets/                                     |
| Fixer.io API requires a valid access key; rate limits and API plan restrictions may apply.                      | Fixer.io API Docs: https://fixer.io/documentation                                                              |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.