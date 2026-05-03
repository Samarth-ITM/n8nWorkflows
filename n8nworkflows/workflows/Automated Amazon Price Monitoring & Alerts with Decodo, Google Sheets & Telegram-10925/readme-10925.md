Automated Amazon Price Monitoring & Alerts with Decodo, Google Sheets & Telegram

https://n8nworkflows.xyz/workflows/automated-amazon-price-monitoring---alerts-with-decodo--google-sheets---telegram-10925


# Automated Amazon Price Monitoring & Alerts with Decodo, Google Sheets & Telegram

### 1. Workflow Overview

This workflow automates price monitoring for Amazon products by combining Decodo’s Amazon data extraction, Google Sheets for baseline price storage, and alerting via Telegram, Gmail, and Google Calendar. It is ideal for e-commerce teams or pricing analysts who want to track price fluctuations automatically and receive timely notifications about significant price changes.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Price Check Trigger:** Initiates the workflow on a defined schedule to control the frequency of price checks.
- **1.2 Pull Product Data from Google Sheets:** Reads the list of Amazon product URLs along with baseline prices and alert thresholds from a Google Sheet.
- **1.3 Price Extraction via Decodo:** For each product URL, fetches current pricing and product details using the Decodo node specialized for Amazon.
- **1.4 Price Change Calculation:** Computes the absolute and percentage difference between the current price and the baseline price.
- **1.5 Routing Based on Price Change Magnitude:** Routes the flow to different actions depending on whether the price change is significantly high, normal (stable or slightly increased), or low (price drop).
- **1.6 Alerting & Notification:** Sends alerts to management via Telegram and Google Calendar for price increases, emails stakeholders for price drops, or does nothing if changes are negligible.
- **1.7 Flow Control and Rate Limiting:** Uses Wait nodes to space out API calls and avoid rate limits.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Price Check Trigger

- **Overview:** This block triggers the entire workflow at configured intervals, controlling how often price checks occur.
- **Nodes Involved:** 
  - Schedule Trigger
- **Node Details:**
  - **Schedule Trigger**
    - Type: Schedule Trigger (n8n-nodes-base.scheduleTrigger)
    - Configuration: Runs on a fixed interval (default unspecified but typically every 1–4 hours recommended)
    - Input: None (entry point)
    - Output: Triggers downstream nodes
    - Notes: Interval should be conservative to prevent API rate limits and aligned with business needs.
    - Failure Modes: Misconfiguration may cause the workflow not to trigger or trigger too often.

#### 2.2 Pull Product Data from Google Sheets

- **Overview:** Reads rows from a Google Sheet containing product URLs, baseline prices, and alert parameters.
- **Nodes Involved:**
  - Get row(s) in sheet
  - Loop Over Items (splitInBatches)
- **Node Details:**
  - **Get row(s) in sheet**
    - Type: Google Sheets node
    - Configuration: Reads specified sheet and document ID; uses URL mode for document identification.
    - Credentials: Google Sheets OAuth2
    - Input: Trigger from Schedule Trigger
    - Output: List of product rows with URLs and baseline prices
    - Edge Cases: Missing or incorrect sheet URL, missing columns, or empty rows can cause errors.
  - **Loop Over Items**
    - Type: SplitInBatches
    - Configuration: Splits the list of products into batches for sequential processing.
    - Input: Rows from Google Sheets
    - Output: One product per execution cycle
    - Failure Modes: Large batch sizes can cause API rate limits downstream.

#### 2.3 Price Extraction via Decodo

- **Overview:** Fetches current product details and pricing from Amazon using Decodo for each product URL.
- **Nodes Involved:**
  - Decodo
- **Node Details:**
  - **Decodo**
    - Type: Decodo node (@decodo/n8n-nodes-decodo.decodo)
    - Configuration: Operation set to “amazon”, takes URL dynamically from the current item's JSON
    - Credentials: Decodo API credentials required
    - Input: Product URL from Loop Over Items
    - Output: JSON with Amazon product details including current price, title, and product URL
    - Edge Cases: API failures, invalid URLs, or missing product data may cause undefined outputs or errors.

#### 2.4 Price Change Calculation

- **Overview:** Calculates the difference between the current price and baseline, including percentage change, and prepares data for routing.
- **Nodes Involved:**
  - Calculate Price Changes (Code node)
- **Node Details:**
  - **Calculate Price Changes**
    - Type: Code node (JavaScript)
    - Configuration: Extracts current price, baseline price; calculates difference and percentage change; rounds percentage change up.
    - Key Expressions: Accesses Decodo results and baseline price from previous Loop Over Items node.
    - Input: Decodo output + reference to baseline price
    - Output: JSON with fields: price, baseline, diff, diffPercentage, title, url
    - Edge Cases: Missing baseline price or Decodo results can cause runtime exceptions; no defensive checks present, so adding these is recommended.

#### 2.5 Routing Based on Price Change Magnitude

- **Overview:** Routes the flow into three branches depending on the percentage change: High increase (>10%), Normal (≥0%), or Low (<0%).
- **Nodes Involved:**
  - Price Routing (Switch node)
- **Node Details:**
  - **Price Routing**
    - Type: Switch node
    - Configuration: Evaluates `diffPercentage` field.
      - High: > 10%
      - Normal: ≥ 0%
      - Low: < 0%
    - Input: Calculated price change JSON
    - Output: One output branch per route
    - Edge Cases: Non-numeric or missing `diffPercentage` may cause routing failure or unexpected path.

#### 2.6 Alerting & Notification

- **Overview:** Sends alerts and notifications based on price changes. High increases trigger Telegram messages and Google Calendar invites; low prices trigger rich HTML emails; normal changes do nothing.
- **Nodes Involved:**
  - Send Message to Top Management (Telegram)
  - Set Meeting (Google Calendar)
  - Email Stakeholders (Gmail)
  - No Operation, do nothing (NoOp)
  - Wait (rate limiting)
- **Node Details:**
  - **Send Message to Top Management**
    - Type: Telegram node
    - Configuration: Sends an alert message with product info and price increase details.
    - Credentials: Telegram API
    - Input: High increase branch from Price Routing
    - Output: Triggers Set Meeting node
    - Edge Cases: Telegram API rate limits or invalid chat ID may cause failure.
  - **Set Meeting**
    - Type: Google Calendar node
    - Configuration: Creates a calendar event 2–3 hours from current time, with summary and attendees, describing the price increase.
    - Credentials: Google Calendar OAuth2
    - Input: After Telegram message
    - Output: Triggers Wait node
    - Edge Cases: Calendar permission errors or invalid attendee emails.
  - **Email Stakeholders**
    - Type: Gmail node
    - Configuration: Sends rich HTML email to stakeholders for price decreases, including styled content and links.
    - Credentials: Gmail OAuth2
    - Input: Low price branch from Price Routing
    - Output: Triggers Wait node
    - Edge Cases: Email quota limits, invalid recipient address.
  - **No Operation, do nothing**
    - Type: NoOp node
    - Configuration: Placeholder to do nothing for Normal price changes.
    - Input: Normal price branch from Price Routing
    - Output: Triggers Wait node
  - **Wait**
    - Type: Wait node
    - Configuration: Pauses workflow execution between iterations to avoid API bursts
    - Input: From Email Stakeholders, Set Meeting, or NoOp nodes
    - Output: Back to Loop Over Items for next batch

#### 2.7 Flow Control and Rate Limiting

- **Overview:** Uses Wait nodes to introduce pauses between operations and after completion to avoid hitting API rate limits.
- **Nodes Involved:**
  - Wait
- **Node Details:**
  - **Wait**
    - Type: Wait node
    - Configuration: Default or configured delay (not explicitly set)
    - Role: Ensures API calls do not overload external services; smooths out flow execution.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                      | Input Node(s)             | Output Node(s)                       | Sticky Note                                                                                                            |
|----------------------------|----------------------------|-------------------------------------|---------------------------|------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger           | Triggers workflow on schedule       | None                      | Get row(s) in sheet                | Runs price checks on schedule. Keep interval conservative (1–4 h). Use named schedule for traceability.                 |
| Get row(s) in sheet        | Google Sheets              | Reads product URLs & baseline prices| Schedule Trigger          | Loop Over Items                   | Read product list + baseline prices. Cache sheet ID. Validate columns exist (url, baseline price).                       |
| Loop Over Items            | SplitInBatches             | Processes each product URL individually | Get row(s) in sheet       | Decodo (batch 2), or empty branch |                                                                                                                        |
| Decodo                    | Decodo (Amazon extractor)  | Fetches current Amazon product data | Loop Over Items           | Calculate Price Changes          |                                                                                                                        |
| Calculate Price Changes    | Code                       | Computes price difference & % change | Decodo                    | Price Routing                   | Calculate diff & % change vs baseline. Add defensive checks to prevent exceptions.                                      |
| Price Routing             | Switch                     | Routes flow based on price change    | Calculate Price Changes    | Send Message to Top Management, No Operation, Email Stakeholders |                                                                                                                        |
| Send Message to Top Management | Telegram                | Alerts management about price increases | Price Routing (High)       | Set Meeting                    | Alert top management for High increases. Include product URL & absolute numbers. Consider tracking tags in URL.         |
| Set Meeting               | Google Calendar             | Creates meeting invite for price increase discussion | Send Message to Top Management | Wait                        |                                                                                                                        |
| Email Stakeholders        | Gmail                      | Sends email alerts for price drops   | Price Routing (Low)        | Wait                          | Email stakeholders on price drops. Rich HTML email, optimized subject, mobile friendly.                                  |
| No Operation, do nothing  | NoOp                       | Placeholder for no action on normal changes | Price Routing (Normal)     | Wait                          |                                                                                                                        |
| Wait                      | Wait                       | Pauses workflow to avoid API bursts  | Email Stakeholders, Set Meeting, No Operation | Loop Over Items               | Adds a pause between flows to avoid API bursts.                                                                         |
| Sticky Note (Schedule Trigger) | Sticky Note           | Documentation                       | None                      | None                          | Runs price checks on schedule. Keep interval conservative to avoid rate limits and align with business windows (e.g., 1–4 h). Use named schedule for traceability. |
| Sticky Note1 (Get row(s))  | Sticky Note                 | Documentation                       | None                      | None                          | Read product list + baseline prices. Pulls rows (URL, baseline price, threshold). Cache sheet ID and use ranges to minimize reads. Validate columns exist (url, baseline price (usd)). |
| Sticky Note4 (Calculate)   | Sticky Note                 | Documentation                       | None                      | None                          | Calculate diff & % change vs baseline. Computes current price, baseline, difference and rounded percentage. Add defensive checks for missing baseline, zero baseline, or missing Decodo result to prevent exceptions. |
| Sticky Note5 (Alerts High) | Sticky Note                | Documentation                       | None                      | None                          | Alert top management for High increases. Sends urgent message to management chat when price jump > threshold. Keep copy concise, include product URL & absolute numbers. Consider adding UTM or tracking tag in URL. Creates a short meeting invite when route = High. Put clear agenda in description and required attendees. Use localized timezones and absolute timestamps. |
| Sticky Note6 (Email Low)   | Sticky Note                 | Documentation                       | None                      | None                          | Email stakeholders on price drops. Sends rich HTML email for price decrease notifications. Optimize subject + preheader for scannability and add UTM-tagged CTA links. Test across clients for mobile. |
| Sticky Note7 (Wait)        | Sticky Note                 | Documentation                       | None                      | None                          | Adds a pause between flows to avoid API bursts.                                                                         |
| Sticky Note8 (Workflow overview) | Sticky Note           | Documentation                       | None                      | None                          | Automated price monitoring for Amazon products using Decodo. Workflow automates Amazon price tracking using Decodo to get real-time data and compare it against baseline prices. Suitable for e-commerce teams and pricing analysts. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Type: Schedule Trigger
   - Configure interval (e.g., every 2 hours)
   - Name: "Schedule Trigger"
   - Connect its output to the next node.

2. **Add Google Sheets node to fetch product data**
   - Type: Google Sheets
   - Operation: Read rows from a specific sheet
   - Configure:
     - Document ID or URL of the Google Sheet containing product info
     - Sheet name or GID (e.g., "gid=0")
   - Credentials: Connect Google Sheets OAuth2 credentials
   - Name: "Get row(s) in sheet"
   - Connect input from Schedule Trigger and output to the next node.

3. **Add SplitInBatches node to process each product individually**
   - Type: SplitInBatches
   - Name: "Loop Over Items"
   - Connect input from Google Sheets node
   - Default options (batch size can be 1 for sequential processing)

4. **Add Decodo node to fetch Amazon product data**
   - Type: Decodo (install Decodo n8n integration if needed)
   - Operation: Set to "amazon"
   - URL parameter: Set expression to current item’s `url` field (e.g., `{{$json["url"]}}`)
   - Credentials: Add Decodo API credentials
   - Name: "Decodo"
   - Connect input from second output of SplitInBatches (process batch items)

5. **Add Code node to calculate price changes**
   - Type: Code
   - Language: JavaScript
   - Code: Calculate difference between Decodo price and baseline price, compute percentage change, and prepare output JSON with keys: price, baseline, diff, diffPercentage, title, url.
   - Name: "Calculate Price Changes"
   - Connect input from Decodo node

6. **Add Switch node to route based on price change**
   - Type: Switch
   - Property to check: `diffPercentage` (number)
   - Add rules:
     - High: `diffPercentage > 10`
     - Normal: `diffPercentage >= 0`
     - Low: `diffPercentage < 0`
   - Name: "Price Routing"
   - Connect input from Code node

7. **For High price increase branch:**
   - Add Telegram node:
     - Configure message with price increase details using expressions from previous node
     - Credentials: Telegram API credentials
     - Name: "Send Message to Top Management"
   - Add Google Calendar node:
     - Create calendar event 2–3 hours in the future
     - Summary, description with product details and price increase
     - Add attendees (e.g., management emails)
     - Credentials: Google Calendar OAuth2
     - Name: "Set Meeting"
   - Connect Telegram output to Google Calendar
   - Connect Google Calendar output to Wait node (see step 10)

8. **For Low price drop branch:**
   - Add Gmail node:
     - Send rich HTML email to stakeholders with price drop info and CTA link
     - Configure subject and body with expressions
     - Credentials: Gmail OAuth2
     - Recipient email(s)
     - Name: "Email Stakeholders"
   - Connect output from Price Routing Low branch to this node
   - Connect Gmail output to Wait node

9. **For Normal price change branch:**
   - Add No Operation node (NoOp)
   - Connect output from Price Routing Normal branch to NoOp node
   - Connect NoOp output to Wait node

10. **Add Wait node to pause between batches**
    - Type: Wait node
    - Configure delay as needed to avoid API bursts (e.g., 1–3 seconds or more)
    - Name: "Wait"
    - Connect outputs from Set Meeting, Email Stakeholders, and NoOp nodes to Wait node
    - Connect Wait node output back to the first input of SplitInBatches node ("Loop Over Items") to process next batch

11. **Final setup**
    - Verify all credentials are correctly configured (Decodo, Google Sheets, Gmail, Google Calendar, Telegram)
    - Ensure Google Sheet has columns: `url` and `baseline price (usd)` with valid data
    - Test workflow with one or two products first to verify correct integration and alerting
    - Adjust schedule interval and thresholds as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| This workflow automates Amazon price monitoring using Decodo and multiple notification channels. Suitable for e-commerce teams.  | Sticky Note8 content in workflow                                                                                         |
| Keep schedule intervals conservative (1–4 hours) to avoid API rate limiting and operational overload.                             | Sticky Note (Schedule Trigger)                                                                                           |
| Validate Google Sheet columns exist and cache document ID for efficiency.                                                         | Sticky Note1 (Google Sheets reading)                                                                                     |
| Add defensive checks in code node to avoid failures from missing or zero baseline prices and missing Decodo data.                | Sticky Note4 (Calculate Price Changes)                                                                                   |
| Alert messages include direct product URLs and absolute price numbers; consider adding tracking parameters to URLs for analytics.| Sticky Note5 (High increase alerting)                                                                                    |
| Price drop emails use rich HTML, optimized for mobile and multiple email clients, with clear CTA links.                           | Sticky Note6 (Email Stakeholders)                                                                                        |
| Wait nodes add pauses to prevent API bursts and throttling issues.                                                                 | Sticky Note7 (Wait node)                                                                                                |
| Decodo node requires API credentials; ensure you have a valid Decodo account and API key.                                          | Decodo node documentation                                                                                               |
| Telegram messaging needs a bot with chat ID configured; Google Calendar requires OAuth2 with calendar write permissions.          | Telegram and Google Calendar nodes credential setup                                                                     |

---

**Disclaimer:** The text provided derives exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.