Send Daily Currency Exchange Rate Updates via CurrencyFreaks API and Gmail

https://n8nworkflows.xyz/workflows/send-daily-currency-exchange-rate-updates-via-currencyfreaks-api-and-gmail-5512


# Send Daily Currency Exchange Rate Updates via CurrencyFreaks API and Gmail

### 1. Workflow Overview

This workflow automates the daily retrieval and email distribution of currency exchange rates using the CurrencyFreaks API and Gmail. It is designed for users who want to receive up-to-date exchange rates for a predefined set of currencies at a scheduled interval, typically daily. The workflow includes these logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow at a configured recurring interval.
- **1.2 Configuration Setup**: Assigns the API key and preferred currencies to variables for API request construction.
- **1.3 Data Retrieval**: Calls the CurrencyFreaks API to fetch the latest exchange rates.
- **1.4 Email Preparation**: Sets the recipient email and the email subject dynamically.
- **1.5 Email Sending**: Sends a styled HTML email containing the currency rates via Gmail OAuth2.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow execution automatically at a defined time interval, typically daily, to ensure regular updates.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `Schedule Trigger` (n8n-nodes-base.scheduleTrigger)  
    - Role: Starts the workflow at configured intervals.  
    - Configuration: Default interval set to daily (one empty object in interval array implies daily run).  
    - Expressions/Variables: None.  
    - Input: None (trigger node).  
    - Output: Connects to "Set API Key & Preferred Currencies" node.  
    - Possible Failures: Misconfiguration of schedule could prevent triggering; no authentication required.  
    - Version: 1.2  
    - Sub-Workflow: None.

---

#### 1.2 Configuration Setup

- **Overview:**  
  Defines critical variables such as the API key for CurrencyFreaks and the list of preferred currencies for which exchange rates will be fetched.

- **Nodes Involved:**  
  - Set API Key & Preferred Currencies

- **Node Details:**

  - **Set API Key & Preferred Currencies**  
    - Type: `Set` node (n8n-nodes-base.set)  
    - Role: Stores the API Key and preferred currencies as JSON variables for use in the HTTP request.  
    - Configuration:  
      - API Key: Empty string placeholder, to be filled with a valid CurrencyFreaks API key.  
      - Preferred Currencies: A CSV string `"PKR,GBP,EUR,USD,BDT,INR"` representing currency codes.  
    - Expressions/Variables: None; static assignment.  
    - Input: From Schedule Trigger node.  
    - Output: To "Get Latest Rate" node.  
    - Possible Failures: Empty or incorrect API key will cause API request failures in subsequent nodes.  
    - Version: 3.4  
    - Sub-Workflow: None.

---

#### 1.3 Data Retrieval

- **Overview:**  
  Fetches the latest currency exchange rates from the CurrencyFreaks API using HTTP GET with dynamic parameters for API key and currency symbols.

- **Nodes Involved:**  
  - Get Latest Rate

- **Node Details:**

  - **Get Latest Rate**  
    - Type: `HTTP Request` (n8n-nodes-base.httpRequest)  
    - Role: Calls CurrencyFreaks API endpoint to retrieve the latest exchange rate data.  
    - Configuration:  
      - HTTP Method: GET (default)  
      - URL: Constructed dynamically using expressions referencing stored API Key and Preferred Currencies:  
        ```
        https://api.currencyfreaks.com/v2.0/rates/latest?apikey={{ $json['API Key'] }}&symbols={{ $json['Preferred Currencies'] }}
        ```  
      - No additional headers or authentication (API key passed as query param).  
    - Expressions/Variables: Uses Mustache-style expressions to embed variables from previous node.  
    - Input: From "Set API Key & Preferred Currencies".  
    - Output: To "Set Recipient Email".  
    - Possible Failures:  
      - Invalid API key or exceeding rate limits returns error responses.  
      - Network timeouts or API downtime could cause failures.  
      - Malformed response may cause downstream JSON parsing errors.  
    - Version: 4.2  
    - Sub-Workflow: None.

---

#### 1.4 Email Preparation

- **Overview:**  
  Prepares email metadata including recipient address and subject line before sending the email.

- **Nodes Involved:**  
  - Set Recipient Email  
  - Set E-mail Subject

- **Node Details:**

  - **Set Recipient Email**  
    - Type: `Set` (n8n-nodes-base.set)  
    - Role: Defines the recipient's email address dynamically.  
    - Configuration:  
      - Field: `Recipient Email` (string)  
      - Value: Empty string placeholder to be replaced with an actual recipient email before activation.  
    - Expressions/Variables: None (static assignment).  
    - Input: From "Get Latest Rate".  
    - Output: To "Set E-mail Subject".  
    - Possible Failures: Empty recipient email will cause Gmail node to fail sending.  
    - Version: 3.4  
    - Sub-Workflow: None.

  - **Set E-mail Subject**  
    - Type: `Set` (n8n-nodes-base.set)  
    - Role: Sets the email subject line, reflecting the currencies included in the update.  
    - Configuration:  
      - Field: `Subject`  
      - Value: `"Daily Currency Update: PKR, GBP, EUR, USD, BDT, INR"` (static string)  
    - Input: From "Set Recipient Email".  
    - Output: To "Send a message".  
    - Possible Failures: None expected; static string.  
    - Version: 3.4  
    - Sub-Workflow: None.

---

#### 1.5 Email Sending

- **Overview:**  
  Sends a styled HTML email containing the retrieved currency exchange data to the specified recipient via Gmail using OAuth2 credentials.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**

  - **Send a message**  
    - Type: `Gmail` node (n8n-nodes-base.gmail)  
    - Role: Sends an email with dynamic content generated using data from previous nodes.  
    - Configuration:  
      - `sendTo`: Uses expression referencing `Recipient Email` from "Set Recipient Email" node.  
      - `subject`: Uses expression referencing `Subject` from "Set E-mail Subject".  
      - `message`: A full HTML document embedding the currency update data dynamically:  
        - Date: `{{ $('Get Latest Rate').item.json.date }}`  
        - Base Currency: `{{ $('Get Latest Rate').item.json.base }}`  
        - Exchange Rates: A code block listing currency codes and rates using JavaScript `.map()` and `.join()` expressions.  
      - Options: Default (no attachments or special flags).  
    - Credentials: Gmail OAuth2 credential configured with a connected Gmail account.  
    - Input: From "Set E-mail Subject".  
    - Output: None (end node).  
    - Possible Failures:  
      - Authentication errors if Gmail OAuth2 token expires or is revoked.  
      - Invalid recipient email causing SMTP errors.  
      - Email quota limits on Gmail side.  
    - Version: 2.1  
    - Sub-Workflow: None.

---

### 3. Summary Table

| Node Name                    | Node Type             | Functional Role                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                           |
|------------------------------|-----------------------|---------------------------------|-------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger      | Initiate workflow on schedule   | None                          | Set API Key & Preferred Currencies | #### Daily Currency Update Workflow (n8n) - Trigger: ScheduleTrigger node (configurable interval)                      |
| Set API Key & Preferred Currencies | Set                 | Store API key and currency list | Schedule Trigger              | Get Latest Rate               | - Set Variables: API Key, Preferred Currencies (PKR, GBP, EUR, USD, BDT, INR)                                          |
| Get Latest Rate              | HTTP Request         | Fetch latest exchange rates     | Set API Key & Preferred Currencies | Set Recipient Email          | - HTTP Request: Fetch latest exchange rates from CurrencyFreaks API                                                   |
| Set Recipient Email          | Set                   | Define recipient email address  | Get Latest Rate               | Set E-mail Subject            | - Set Recipient Email                                                                                                  |
| Set E-mail Subject           | Set                   | Define email subject            | Set Recipient Email           | Send a message               | - Set Email Subject                                                                                                    |
| Send a message              | Gmail                 | Send email with currency data   | Set E-mail Subject            | None                         | - Send Email: HTML formatted via Gmail OAuth2 with dynamic rate data (date, base currency, rates)                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval to run daily (default empty object in interval array).  
   - No credentials required.  
   - Position: Start node.

2. **Create Set Node for API Key & Preferred Currencies**  
   - Type: Set  
   - Add two fields:  
     - `API Key` (string): Enter your valid CurrencyFreaks API key here.  
     - `Preferred Currencies` (string): Set to `"PKR,GBP,EUR,USD,BDT,INR"` or desired currency codes separated by commas.  
   - Connect Schedule Trigger → Set API Key & Preferred Currencies.

3. **Create HTTP Request Node to Fetch Exchange Rates**  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL:  
     ```
     https://api.currencyfreaks.com/v2.0/rates/latest?apikey={{ $json["API Key"] }}&symbols={{ $json["Preferred Currencies"] }}
     ```  
   - No authentication needed beyond API key in URL.  
   - Connect Set API Key & Preferred Currencies → Get Latest Rate.

4. **Create Set Node to Define Recipient Email**  
   - Type: Set  
   - Add field:  
     - `Recipient Email` (string): Enter the recipient email address here.  
   - Connect Get Latest Rate → Set Recipient Email.

5. **Create Set Node to Define Email Subject**  
   - Type: Set  
   - Add field:  
     - `Subject` (string): Set to `"Daily Currency Update: PKR, GBP, EUR, USD, BDT, INR"` or customize as needed.  
   - Connect Set Recipient Email → Set E-mail Subject.

6. **Create Gmail Node to Send Email**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials with access to your Gmail account.  
   - Parameters:  
     - `sendTo`: Expression: `={{ $('Set Recipient Email').item.json["Recipient Email"] }}`  
     - `subject`: Expression: `={{ $json.Subject }}`  
     - `message`: Paste the full HTML template including the following dynamic placeholders:  
       - Date: `{{ $('Get Latest Rate').item.json.date }}`  
       - Base Currency: `{{ $('Get Latest Rate').item.json.base }}`  
       - Rates:  
         ```
         {{ Object.entries($('Get Latest Rate').item.json.rates).map(([k, v]) => `${k}: ${v}`).join('\n') }}
         ```  
   - Connect Set E-mail Subject → Send a message.

7. **Test the Workflow**  
   - Fill in all placeholders: API Key, Recipient Email.  
   - Activate the workflow or run manually to verify email delivery with correct data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow’s sticky note summarizes the flow and key points: scheduled trigger, variable setup, HTTP request, email setup, and sending via Gmail OAuth2 with dynamic HTML content.                                                                   | Sticky Note node content inside the workflow JSON.                                             |
| CurrencyFreaks API documentation for reference: https://currencyfreaks.com/api-documentation/                                                                                                                                                         | CurrencyFreaks API official docs.                                                              |
| Gmail OAuth2 credential setup is required to authenticate and send emails securely. Ensure the OAuth2 credentials are up to date and authorized.                                                                                                      | n8n Gmail node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ |
| The email HTML uses inline styles and monospace fonts for clarity of exchange rate presentation, suitable for modern email clients.                                                                                                                  | Email HTML content inside the "Send a message" node.                                           |
| The workflow is inactive by default; ensure to activate it after configuration to enable scheduled runs.                                                                                                                                               | Workflow property `"active": false` in JSON.                                                  |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created using n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.