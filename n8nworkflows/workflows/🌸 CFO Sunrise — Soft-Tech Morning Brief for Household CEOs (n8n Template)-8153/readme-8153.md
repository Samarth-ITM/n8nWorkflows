🌸 CFO Sunrise — Soft-Tech Morning Brief for Household CEOs (n8n Template)

https://n8nworkflows.xyz/workflows/---cfo-sunrise---soft-tech-morning-brief-for-household-ceos--n8n-template--8153


# 🌸 CFO Sunrise — Soft-Tech Morning Brief for Household CEOs (n8n Template)

### 1. Workflow Overview

The **Serene CFO Sunrise — Soft‑Tech Morning Brief (Household CEO) 🌸** is an automated daily briefing workflow designed to provide household CEOs (primary household financial decision-makers) with a concise and insightful morning report. It consolidates financial data (bank balances, Stripe invoices, Shopify orders) and calendar events, then uses AI to draft a prioritized summary message, which is sent via email and optionally Telegram.

The workflow runs automatically every day at 8:00 AM (configured for America/Los_Angeles timezone) and is structured into the following logical blocks:

- **1.1 Scheduled Trigger and Brand Setup:** Initiates the workflow daily and sets brand-specific parameters.
- **1.2 Time Window Calculation:** Determines the relevant time window for data aggregation.
- **1.3 Data Collection:** Fetches data from various sources: bank balances, Shopify orders, Stripe invoices, and Google Calendar events.
- **1.4 Data Aggregation and AI Processing:** Assembles collected data into a snapshot, then uses OpenAI to generate a top-3 priorities draft.
- **1.5 Message Construction and Delivery:** Builds an HTML soft-formatted message and sends it via email and optionally Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and Brand Setup

- **Overview:**  
  This block triggers the workflow daily at 8 AM and initializes brand-related settings that may affect message styling or other parameters.

- **Nodes Involved:**  
  - Start (Cron @ 08:00)  
  - Brand Settings

- **Node Details:**

  - **Start (Cron @ 08:00):**  
    - *Type:* Cron Trigger  
    - *Role:* Initiates the workflow every day at 08:00 (local timezone America/Los_Angeles).  
    - *Configuration:* Default cron with time fixed to 8:00 AM. No parameters required.  
    - *Input/Output:* No input; output triggers Brand Settings.  
    - *Edge Cases:* Cron misfire if n8n server is down at scheduled time; timezone misalignment if environment timezone changes.

  - **Brand Settings:**  
    - *Type:* Set Node  
    - *Role:* Defines brand-specific variables and settings for downstream nodes.  
    - *Configuration:* No explicit parameters visible; likely sets variables such as brand name, colors, logos, or formatting preferences in expressions or environment variables.  
    - *Input:* From Cron trigger.  
    - *Output:* Passes data to Time Window node.  
    - *Edge Cases:* Misconfiguration could lead to wrong branding or empty variables downstream.

---

#### 2.2 Time Window Calculation

- **Overview:**  
  Calculates the relevant date/time range for which to fetch data (for example, today's date and time boundaries).

- **Nodes Involved:**  
  - Time Window

- **Node Details:**

  - **Time Window:**  
    - *Type:* Function Node  
    - *Role:* Computes and outputs a time window (start and end timestamps) used to filter data from APIs.  
    - *Configuration:* Contains JavaScript code to generate date boundaries (e.g., start of day, current time).  
    - *Input:* Receives brand settings data.  
    - *Output:* Sends time window parameters to all four data collection HTTP/Google Calendar nodes.  
    - *Edge Cases:* Incorrect date calculations due to timezone issues or daylight saving time changes may cause data gaps or overlaps.

---

#### 2.3 Data Collection

- **Overview:**  
  Fetches financial and calendar data from external services using HTTP requests or native nodes.

- **Nodes Involved:**  
  - Bank Balances (HTTP)  
  - Shopify Orders (HTTP)  
  - Stripe Invoices (Open)  
  - Google Calendar — Today

- **Node Details:**

  - **Bank Balances (HTTP):**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves current bank balances from a bank API or intermediary service (e.g., Relay or Plaid).  
    - *Configuration:* Uses HTTP Header Auth via credentials; URL and tokens are stored securely in credentials.  
    - *Input:* Receives time window data.  
    - *Output:* Passes bank data to Assemble Snapshot node.  
    - *Notes:* The node's note mentions swapping URL/provider as needed.  
    - *Edge Cases:* Authentication failures, token expiry, API rate limits, network timeouts.

  - **Shopify Orders (HTTP):**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves Shopify order data for the defined time window.  
    - *Configuration:* Uses HTTP Header Auth credentials; URL and tokens stored in credentials.  
    - *Input:* Receives time window data.  
    - *Output:* Passes Shopify orders data to Assemble Snapshot node.  
    - *Notes:* Native Shopify node can be preferred instead of HTTP.  
    - *Edge Cases:* Authentication errors, API limits, malformed responses.

  - **Stripe Invoices (Open):**  
    - *Type:* HTTP Request  
    - *Role:* Collects Stripe invoice data for the period.  
    - *Configuration:* Uses HTTP Header Auth; credentials store API keys.  
    - *Input:* Receives time window data.  
    - *Output:* Passes invoices data to Assemble Snapshot node.  
    - *Edge Cases:* Authentication failures, API errors, network issues.

  - **Google Calendar — Today:**  
    - *Type:* Google Calendar Node  
    - *Role:* Fetches upcoming calendar events for the day.  
    - *Configuration:* Uses Google OAuth2 credentials. Queries events for the current day/time window.  
    - *Input:* Receives time window data.  
    - *Output:* Sends calendar events data to Assemble Snapshot node.  
    - *Edge Cases:* OAuth token expiry, API quota limits, calendar access permissions.

---

#### 2.4 Data Aggregation and AI Processing

- **Overview:**  
  Aggregates all collected data into a consolidated snapshot, then invokes OpenAI to draft a prioritized "Top 3" summary based on this data.

- **Nodes Involved:**  
  - Assemble Snapshot  
  - Draft Top 3 (OpenAI)

- **Node Details:**

  - **Assemble Snapshot:**  
    - *Type:* Function Node  
    - *Role:* Combines all incoming datasets (bank balances, Shopify orders, Stripe invoices, calendar events) into a structured summary object.  
    - *Configuration:* Custom JavaScript to merge inputs, filter or format them as needed.  
    - *Input:* Receives all data from respective HTTP and Google Calendar nodes.  
    - *Output:* Sends aggregated data to OpenAI node.  
    - *Edge Cases:* Missing or malformed input data could cause errors; careful handling of empty data sets is required.

  - **Draft Top 3 (OpenAI):**  
    - *Type:* OpenAI Node  
    - *Role:* Uses OpenAI API to generate a concise draft of the "Top 3" priorities or insights based on assembled snapshot data.  
    - *Configuration:* Utilizes OpenAI credentials (API key), with prompt templates likely embedding the snapshot data in the prompt.  
    - *Input:* Receives aggregated snapshot data.  
    - *Output:* Passes AI-generated text to message building node.  
    - *Edge Cases:* API quota limits, network timeouts, prompt or response parsing errors.

---

#### 2.5 Message Construction and Delivery

- **Overview:**  
  Constructs a soft HTML email message from the AI draft and sends it via email and optionally Telegram.

- **Nodes Involved:**  
  - Build Message (Soft HTML)  
  - Send Email  
  - Send Telegram (optional)

- **Node Details:**

  - **Build Message (Soft HTML):**  
    - *Type:* Function Node  
    - *Role:* Takes AI draft text and formats it into a styled HTML email body suitable for soft-tech morning briefings.  
    - *Configuration:* Includes HTML templates, possibly inline CSS, and inserts dynamic content from previous nodes.  
    - *Input:* Receives AI draft from OpenAI node.  
    - *Output:* Sends formatted message to email and Telegram nodes.  
    - *Edge Cases:* HTML formatting errors, missing data could cause broken emails.

  - **Send Email:**  
    - *Type:* Email Send Node  
    - *Role:* Sends the constructed email to the household CEO(s).  
    - *Configuration:* Uses SMTP credentials; email parameters such as recipients, subject lines, and message body configured.  
    - *Input:* Receives formatted HTML message.  
    - *Output:* None (end node).  
    - *Edge Cases:* SMTP failures, invalid recipient addresses, email delivery issues.

  - **Send Telegram (optional):**  
    - *Type:* Telegram Node  
    - *Role:* Sends the morning brief message via Telegram messenger as an alternative or complement.  
    - *Configuration:* Uses Telegram Bot credentials; message text likely plain or simplified HTML.  
    - *Input:* Receives formatted message from Build Message node.  
    - *Output:* None (end node).  
    - *Edge Cases:* Bot token invalid, Telegram API limits, chat ID misconfiguration.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                                  | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                  |
|--------------------------|--------------------|-------------------------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Start (Cron @ 08:00)     | Cron Trigger       | Triggers workflow daily at 8 AM                  | —                                 | Brand Settings                    |                                                                                              |
| Brand Settings           | Set                | Defines brand-related variables                   | Start (Cron @ 08:00)              | Time Window                      |                                                                                              |
| Time Window              | Function           | Computes date/time window for data queries       | Brand Settings                   | Bank Balances (HTTP), Shopify Orders (HTTP), Stripe Invoices (Open), Google Calendar — Today |                                                                                              |
| Bank Balances (HTTP)     | HTTP Request       | Fetches bank balances from API                     | Time Window                     | Assemble Snapshot                | Swap URL/provider for Relay/Plaid as needed. Keep tokens in credentials.                     |
| Shopify Orders (HTTP)    | HTTP Request       | Fetches Shopify orders                             | Time Window                     | Assemble Snapshot                | Prefer native Shopify node if you like.                                                    |
| Stripe Invoices (Open)   | HTTP Request       | Fetches Stripe invoices                            | Time Window                     | Assemble Snapshot                |                                                                                              |
| Google Calendar — Today  | Google Calendar    | Fetches today's calendar events                    | Time Window                     | Assemble Snapshot                |                                                                                              |
| Assemble Snapshot        | Function           | Aggregates all data into a snapshot                | Bank Balances, Shopify Orders, Stripe Invoices, Google Calendar | Draft Top 3 (OpenAI)              |                                                                                              |
| Draft Top 3 (OpenAI)     | OpenAI             | Creates AI-generated top 3 summary draft          | Assemble Snapshot               | Build Message (Soft HTML)         |                                                                                              |
| Build Message (Soft HTML)| Function           | Formats the AI draft into an HTML email message   | Draft Top 3 (OpenAI)             | Send Email, Send Telegram (optional) |                                                                                              |
| Send Email               | Email Send         | Sends morning brief email                          | Build Message (Soft HTML)        | —                                 |                                                                                              |
| Send Telegram (optional) | Telegram           | Optionally sends brief via Telegram                | Build Message (Soft HTML)        | —                                 |                                                                                              |
| 🗒️ Sticky — Submission    | Sticky Note        | —                                                 | —                                 | —                                 |                                                                                              |
| 🗒️ Sticky — Brand         | Sticky Note        | —                                                 | —                                 | —                                 |                                                                                              |
| 🗒️ Sticky — Setup & Testing| Sticky Note       | —                                                 | —                                 | —                                 |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node:**  
   - Name: `Start (Cron @ 08:00)`  
   - Type: Cron  
   - Set to trigger every day at 08:00 AM, timezone set to `America/Los_Angeles`.

2. **Create Set Node for Brand Settings:**  
   - Name: `Brand Settings`  
   - Type: Set  
   - Configure brand-related variables (e.g., brandName, logoURL, colors) as needed using expressions or static values.  
   - Connect output of Cron node to this node.

3. **Create Function Node for Time Window Calculation:**  
   - Name: `Time Window`  
   - Type: Function  
   - Implement JavaScript logic to set start and end timestamps for today's date (e.g., start of day 00:00, current time or end of day 23:59).  
   - Connect output of Brand Settings node to this node.

4. **Create HTTP Request Node for Bank Balances:**  
   - Name: `Bank Balances (HTTP)`  
   - Type: HTTP Request  
   - Configure URL for bank API or Relay/Plaid endpoint.  
   - Use HTTP Header Auth credential with API token.  
   - Pass time window parameters as query or body as required by API.  
   - Connect output of Time Window node to this node.

5. **Create HTTP Request Node for Shopify Orders:**  
   - Name: `Shopify Orders (HTTP)`  
   - Type: HTTP Request  
   - Configure Shopify API endpoint for orders with appropriate filters for time window.  
   - Use HTTP Header Auth credential.  
   - Alternatively, use native Shopify node if preferred.  
   - Connect output of Time Window node to this node.

6. **Create HTTP Request Node for Stripe Invoices:**  
   - Name: `Stripe Invoices (Open)`  
   - Type: HTTP Request  
   - Configure Stripe API endpoint for invoices filtered by time window.  
   - Use HTTP Header Auth credential.  
   - Connect output of Time Window node to this node.

7. **Create Google Calendar Node for Today's Events:**  
   - Name: `Google Calendar — Today`  
   - Type: Google Calendar  
   - Use Google OAuth2 credentials.  
   - Set to retrieve events between start and end timestamps calculated in Time Window.  
   - Connect output of Time Window node to this node.

8. **Create Function Node to Assemble Snapshot:**  
   - Name: `Assemble Snapshot`  
   - Type: Function  
   - Write JavaScript code to receive incoming data from all four sources and merge into one structured object (e.g., JSON with keys for bank, orders, invoices, calendar).  
   - Connect outputs of Bank Balances, Shopify Orders, Stripe Invoices, and Google Calendar nodes to this node (all feeding inputs).  
   - Connect output to OpenAI node next.

9. **Create OpenAI Node for Drafting Top 3 Priorities:**  
   - Name: `Draft Top 3 (OpenAI)`  
   - Type: OpenAI  
   - Configure with OpenAI API credentials (API key).  
   - Craft prompt template embedding assembled snapshot data, requesting a concise top 3 priorities list.  
   - Connect output of Assemble Snapshot node to this node.

10. **Create Function Node to Build HTML Message:**  
    - Name: `Build Message (Soft HTML)`  
    - Type: Function  
    - Write code to convert the AI-generated draft into an HTML email body. Include any branding styles or formatting.  
    - Connect output of Draft Top 3 node to this node.

11. **Create Email Send Node:**  
    - Name: `Send Email`  
    - Type: Email Send  
    - Configure SMTP credentials for your email service.  
    - Set recipient(s), subject line, and use the HTML message body from Build Message node as email content.  
    - Connect output of Build Message node to this node.

12. **Create Optional Telegram Node:**  
    - Name: `Send Telegram (optional)`  
    - Type: Telegram  
    - Configure Telegram Bot credentials.  
    - Set chat ID(s) and message text (use formatted or plain text from Build Message node).  
    - Connect output of Build Message node to this node.

13. **Set Workflow Timezone:**  
    - In workflow settings, set timezone to `America/Los_Angeles` for correct time handling.

14. **Add Sticky Notes (Optional):**  
    - Add notes for Submission, Brand, Setup & Testing as reminders or documentation aids.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Use credentials for HTTP Header Auth (Stripe, Shopify, Bank), Google OAuth2, SMTP, Telegram.                   | Credential management best practice; ensures secure API access.                                  |
| Swap URL/provider for Relay/Plaid as needed for bank balances API.                                             | Flexibility in choosing financial data providers.                                               |
| Prefer native Shopify node if you like, for better integration and ease of use.                                | Native nodes often simplify authentication and pagination.                                      |
| Workflow timezone is set to America/Los_Angeles to align with typical household CEO morning routines.          | Important for correct timing of daily briefing trigger and data queries.                         |
| This workflow is designed for household CEOs to receive a soft-tech, AI-enhanced morning briefing integrating financial and calendar info.| Core use case and audience description.                                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.