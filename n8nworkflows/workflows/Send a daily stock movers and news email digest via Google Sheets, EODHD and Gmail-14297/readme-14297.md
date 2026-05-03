Send a daily stock movers and news email digest via Google Sheets, EODHD and Gmail

https://n8nworkflows.xyz/workflows/send-a-daily-stock-movers-and-news-email-digest-via-google-sheets--eodhd-and-gmail-14297


# Send a daily stock movers and news email digest via Google Sheets, EODHD and Gmail

# 1. Workflow Overview

This workflow generates and sends a **daily stock market email digest**. It reads a watchlist of US stock tickers from Google Sheets, fetches recent end-of-day price data and recent company news from the EODHD API, computes daily price movement, classifies article sentiment, builds a styled HTML report, and emails it through Gmail.

Typical use cases:
- Personal daily stock watchlist monitoring
- Investor or analyst morning briefings
- Lightweight automated market digest for a small set of tracked US equities

The workflow is organized into the following logical blocks.

## 1.1 Trigger and Runtime Configuration

The workflow starts either on a daily schedule or manually. A configuration node provides the EODHD API key and recipient email address used later in the run.

## 1.2 Watchlist Retrieval from Google Sheets

The workflow loads stock tickers from a Google Sheet. Each row is expected to contain a `ticker` column with one US stock symbol per row.

## 1.3 Market Data and News Collection

For each ticker, the workflow calls EODHD twice:
- once for historical end-of-day price data over roughly the last 14 days
- once for recent news over the last 7 days

Both HTTP nodes allow failures without stopping the full workflow.

## 1.4 Digest Assembly

A Code node regroups all API responses by originating ticker, calculates the latest percentage move versus the previous close, filters and formats news items, applies sentiment labels, and generates the final HTML email body plus subject line.

## 1.5 Email Delivery

The workflow sends the generated digest by Gmail to the configured recipient.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Runtime Configuration

### Overview
This block defines how the workflow starts and centralizes runtime variables. It supports both scheduled execution and manual execution, while keeping the API token and target recipient in one place.

### Nodes Involved
- Schedule Trigger
- When clicking ‘Execute workflow’
- ⚙️ Config

### Node Details

#### Schedule Trigger
- **Type and role:** `n8n-nodes-base.scheduleTrigger`  
  Starts the workflow automatically every day.
- **Configuration choices:**  
  Uses a cron expression: `0 7 * * *`, which means **daily at 07:00**.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **⚙️ Config**.
- **Version-specific requirements:**  
  Uses `typeVersion: 1.2`. Standard schedule trigger behavior.
- **Edge cases or failures:**  
  - Workflow timezone matters; if timezone is not configured as expected in Workflow Settings, the run time may differ.
  - If the workflow is inactive, the schedule does not run.
- **Sub-workflow reference:**  
  None.

#### When clicking ‘Execute workflow’
- **Type and role:** `n8n-nodes-base.manualTrigger`  
  Allows testing or on-demand runs from the editor.
- **Configuration choices:**  
  No parameters.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  No input. Outputs to **⚙️ Config**.
- **Version-specific requirements:**  
  `typeVersion: 1`.
- **Edge cases or failures:**  
  - Manual trigger works only when executed from the n8n editor or supported interface.
- **Sub-workflow reference:**  
  None.

#### ⚙️ Config
- **Type and role:** `n8n-nodes-base.set`  
  Stores static configuration values for downstream nodes.
- **Configuration choices:**  
  Creates two string fields:
  - `api_token`
  - `recipient_email`
- **Key expressions or variables used:**  
  Later referenced via expressions such as:
  - `$('⚙️ Config').first().json.api_token`
  - `$('⚙️ Config').first().json.recipient_email`
- **Input and output connections:**  
  Inputs from **Schedule Trigger** and **When clicking ‘Execute workflow’**. Outputs to **Google Sheets**.
- **Version-specific requirements:**  
  `typeVersion: 3.4`
- **Edge cases or failures:**  
  - Placeholder values must be replaced before production use.
  - Invalid EODHD token causes API authentication or authorization failures later.
  - Invalid email may cause Gmail send failure.
- **Sub-workflow reference:**  
  None.

---

## 2.2 Watchlist Retrieval from Google Sheets

### Overview
This block reads the list of stock symbols to monitor. It provides the base item stream that is later expanded into one API request pair per ticker.

### Nodes Involved
- Google Sheets

### Node Details

#### Google Sheets
- **Type and role:** `n8n-nodes-base.googleSheets`  
  Reads rows from a specific Google Sheets tab.
- **Configuration choices:**  
  - Spreadsheet document: `1k6mDOKB0B6Wt2OOvaAo7oTxpn7Zt-XeiLA0NZlM6VFY`
  - Sheet/tab: `gid=0`
  - Uses Google Sheets OAuth2 credentials
- **Key expressions or variables used:**  
  No dynamic expressions in the configured document or sheet values.
- **Input and output connections:**  
  Input from **⚙️ Config**. Output to **HTTP Request - EOD**.
- **Version-specific requirements:**  
  `typeVersion: 4.4`
- **Edge cases or failures:**  
  - Authentication failure if Google OAuth2 credentials are missing or expired.
  - If the selected sheet is inaccessible or deleted, the node fails.
  - The sheet must contain a column named `ticker`; otherwise downstream API URLs become malformed or empty.
  - Empty rows may still be processed; the code node later skips blank tickers.
- **Sub-workflow reference:**  
  None.

---

## 2.3 Market Data and News Collection

### Overview
This block performs the two EODHD API lookups per ticker. The first fetches historical daily close prices, and the second fetches recent news with sentiment metadata.

### Nodes Involved
- HTTP Request - EOD
- HTTP Request - News

### Node Details

#### HTTP Request - EOD
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Fetches recent end-of-day historical price data for each ticker.
- **Configuration choices:**  
  - URL is built dynamically as:
    `https://eodhd.com/api/eod/{ticker}.US`
  - Query parameters:
    - `api_token`: from **⚙️ Config**
    - `fmt=json`
    - `order=d`
    - `from`: current date minus 14 days
    - `to`: current date
  - `continueOnFail` is enabled
- **Key expressions or variables used:**  
  - URL: `{{ 'https://eodhd.com/api/eod/' + $json.ticker + '.US' }}`
  - API token: `{{ $('⚙️ Config').first().json.api_token }}`
  - Date range generated via JavaScript date expressions
- **Input and output connections:**  
  Input from **Google Sheets**. Output to **HTTP Request - News**.
- **Version-specific requirements:**  
  `typeVersion: 4.2`
- **Edge cases or failures:**  
  - Invalid or missing ticker creates an invalid request path.
  - Free-tier API limits may throttle requests.
  - Market holidays/weekends may reduce the number of returned candles.
  - If only one data point is returned, the digest can show latest close but not daily percentage change.
  - Since `continueOnFail` is enabled, failed items pass through with error payloads rather than aborting the workflow.
- **Sub-workflow reference:**  
  None.

#### HTTP Request - News
- **Type and role:** `n8n-nodes-base.httpRequest`  
  Fetches recent ticker-specific news articles from EODHD.
- **Configuration choices:**  
  - URL: `https://eodhd.com/api/news`
  - Query parameters:
    - `s`: ticker symbol with `.US`
    - `api_token`: from **⚙️ Config**
    - `fmt=json`
    - `limit=3`
    - `from`: current date minus 7 days
  - `continueOnFail` is enabled
- **Key expressions or variables used:**  
  - Symbol: `{{ $('Google Sheets').item.json.ticker + '.US' }}`
  - API token: `{{ $('⚙️ Config').first().json.api_token }}`
  - Dynamic `from` date expression
- **Input and output connections:**  
  Input from **HTTP Request - EOD**. Output to **Build Email HTML**.
- **Version-specific requirements:**  
  `typeVersion: 4.2`
- **Edge cases or failures:**  
  - The expression references the current item from **Google Sheets**, which assumes correct item pairing across execution.
  - Some tickers may have no recent news.
  - API errors, rate limits, or malformed responses are tolerated because `continueOnFail` is enabled.
  - The node requests `limit=3`, but the code later slices to up to 5 items; effectively the practical maximum from this node is 3 unless the node configuration is changed.
- **Sub-workflow reference:**  
  None.

---

## 2.4 Digest Assembly

### Overview
This block consolidates all prior outputs into a single report. It rebuilds ticker-level grouping using paired item metadata, computes movement metrics, formats news and sentiment, tracks failures, and creates the final HTML email body and subject.

### Nodes Involved
- Build Email HTML

### Node Details

#### Build Email HTML
- **Type and role:** `n8n-nodes-base.code`  
  Custom JavaScript transformation node that produces one final item containing `html` and `subject`.
- **Configuration choices:**  
  The node:
  - reads all items from **Google Sheets**, **HTTP Request - EOD**, and **HTTP Request - News**
  - reconstructs per-ticker data by `pairedItem`
  - calculates current price and daily percent change from the two most recent closes
  - filters news to the last 7 days
  - converts sentiment polarity to emoji, label, and color
  - builds a styled HTML digest including:
    - header/date
    - movers table
    - per-ticker news sections
    - warning section for failed tickers
    - footer
- **Key expressions or variables used:**  
  Major variables and logic:
  - `$('Google Sheets').all()`
  - `$('HTTP Request - EOD').all()`
  - `$('HTTP Request - News').all()`
  - helper `getPairedIdx(item)` to support multiple pairedItem formats
  - price change calculation based on the latest two sorted EOD records
  - sentiment thresholds:
    - positive: `polarity > 0.6`
    - negative: `polarity < 0.4`
    - otherwise neutral
  - generated subject:
    `📊 Daily Market Digest — {localized date}`
- **Input and output connections:**  
  Input from **HTTP Request - News**. Output to **Send Email via Gmail**.
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or failures:**  
  - If item pairing is disrupted upstream, data could be attached to the wrong ticker.
  - If EOD or news responses have unexpected schema, the code may produce empty sections or incorrect values.
  - Sentiment labels are rendered as `Positivo`, `Negativo`, and `Neutral`; this is functionally harmless but linguistically mixed for an English email.
  - News limit mismatch: code allows up to 5 items after filtering, but upstream node currently requests 3.
  - HTML is inserted directly from API content without explicit escaping; unusual article titles or summaries containing HTML entities could affect rendering.
  - If every ticker fails, the email still generates, but most sections may be empty or contain only warnings.
- **Sub-workflow reference:**  
  None.

---

## 2.5 Email Delivery

### Overview
This block sends the generated digest to the configured recipient using Gmail OAuth2 credentials.

### Nodes Involved
- Send Email via Gmail

### Node Details

#### Send Email via Gmail
- **Type and role:** `n8n-nodes-base.gmail`  
  Sends the final HTML email.
- **Configuration choices:**  
  - Recipient: from **⚙️ Config**
  - Subject: from **Build Email HTML**
  - Message body: HTML generated by **Build Email HTML**
  - Uses Gmail OAuth2 credentials
- **Key expressions or variables used:**  
  - `{{ $('⚙️ Config').first().json.recipient_email }}`
  - `{{ $json.html }}`
  - `{{ $json.subject }}`
- **Input and output connections:**  
  Input from **Build Email HTML**. No downstream output.
- **Version-specific requirements:**  
  `typeVersion: 2.1`
- **Edge cases or failures:**  
  - Gmail OAuth2 credential expiration or missing scopes can block sending.
  - Gmail sending quotas may apply.
  - Invalid recipient address can cause send failure.
  - Large HTML bodies or problematic markup may render inconsistently across email clients.
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts the workflow automatically every day at 7 AM |  | ⚙️ Config | ## Send daily stock price movers and financial news digest via Gmail<br>Automatically monitor your stock watchlist every morning and receive a formatted email with price changes and curated financial news with sentiment analysis.<br>**How it works:**<br>1. Schedule Trigger fires at 7 AM daily<br>2. Reads tickers from Google Sheets (one ticker per row, column `ticker`)<br>3. Fetches EOD prices + last 7 days of news from EODHD API per ticker<br>4. Calculates daily % change and classifies news sentiment (Positive / Neutral / Negative)<br>5. Sends an HTML digest via Gmail with a movers table and news per ticker<br>**Setup — 4 steps:**<br>1. Open **⚙️ Config** and fill in your `api_token` (EODHD) and `recipient_email`<br>2. Connect **Google Sheets** credential → set your spreadsheet and sheet name (column must be named `ticker`)<br>3. Connect **Gmail** credential<br>4. Set your timezone in **Settings → Workflow Settings**<br>**Requirements:** EODHD account (free tier works) · Google OAuth2 configured in n8n<br><br>## Step 1 — Trigger & config<br>Set your **EODHD API key** and **recipient email** in ⚙️ Config before running.<br>Runs daily at **7 AM UTC**. Change timezone in Workflow Settings.<br>You can also run manually with the "Execute workflow" button. |
| When clicking ‘Execute workflow’ | Manual Trigger | Starts the workflow manually for testing or on-demand runs |  | ⚙️ Config | ## Send daily stock price movers and financial news digest via Gmail<br>Automatically monitor your stock watchlist every morning and receive a formatted email with price changes and curated financial news with sentiment analysis.<br>**How it works:**<br>1. Schedule Trigger fires at 7 AM daily<br>2. Reads tickers from Google Sheets (one ticker per row, column `ticker`)<br>3. Fetches EOD prices + last 7 days of news from EODHD API per ticker<br>4. Calculates daily % change and classifies news sentiment (Positive / Neutral / Negative)<br>5. Sends an HTML digest via Gmail with a movers table and news per ticker<br>**Setup — 4 steps:**<br>1. Open **⚙️ Config** and fill in your `api_token` (EODHD) and `recipient_email`<br>2. Connect **Google Sheets** credential → set your spreadsheet and sheet name (column must be named `ticker`)<br>3. Connect **Gmail** credential<br>4. Set your timezone in **Settings → Workflow Settings**<br>**Requirements:** EODHD account (free tier works) · Google OAuth2 configured in n8n<br><br>## Step 1 — Trigger & config<br>Set your **EODHD API key** and **recipient email** in ⚙️ Config before running.<br>Runs daily at **7 AM UTC**. Change timezone in Workflow Settings.<br>You can also run manually with the "Execute workflow" button. |
| ⚙️ Config | Set | Stores the EODHD API token and destination email | Schedule Trigger, When clicking ‘Execute workflow’ | Google Sheets | ## Send daily stock price movers and financial news digest via Gmail<br>Automatically monitor your stock watchlist every morning and receive a formatted email with price changes and curated financial news with sentiment analysis.<br>**How it works:**<br>1. Schedule Trigger fires at 7 AM daily<br>2. Reads tickers from Google Sheets (one ticker per row, column `ticker`)<br>3. Fetches EOD prices + last 7 days of news from EODHD API per ticker<br>4. Calculates daily % change and classifies news sentiment (Positive / Neutral / Negative)<br>5. Sends an HTML digest via Gmail with a movers table and news per ticker<br>**Setup — 4 steps:**<br>1. Open **⚙️ Config** and fill in your `api_token` (EODHD) and `recipient_email`<br>2. Connect **Google Sheets** credential → set your spreadsheet and sheet name (column must be named `ticker`)<br>3. Connect **Gmail** credential<br>4. Set your timezone in **Settings → Workflow Settings**<br>**Requirements:** EODHD account (free tier works) · Google OAuth2 configured in n8n<br><br>## Step 1 — Trigger & config<br>Set your **EODHD API key** and **recipient email** in ⚙️ Config before running.<br>Runs daily at **7 AM UTC**. Change timezone in Workflow Settings.<br>You can also run manually with the "Execute workflow" button. |
| Google Sheets | Google Sheets | Reads the stock watchlist from the spreadsheet | ⚙️ Config | HTTP Request - EOD | ## Send daily stock price movers and financial news digest via Gmail<br>Automatically monitor your stock watchlist every morning and receive a formatted email with price changes and curated financial news with sentiment analysis.<br>**How it works:**<br>1. Schedule Trigger fires at 7 AM daily<br>2. Reads tickers from Google Sheets (one ticker per row, column `ticker`)<br>3. Fetches EOD prices + last 7 days of news from EODHD API per ticker<br>4. Calculates daily % change and classifies news sentiment (Positive / Neutral / Negative)<br>5. Sends an HTML digest via Gmail with a movers table and news per ticker<br>**Setup — 4 steps:**<br>1. Open **⚙️ Config** and fill in your `api_token` (EODHD) and `recipient_email`<br>2. Connect **Google Sheets** credential → set your spreadsheet and sheet name (column must be named `ticker`)<br>3. Connect **Gmail** credential<br>4. Set your timezone in **Settings → Workflow Settings**<br>**Requirements:** EODHD account (free tier works) · Google OAuth2 configured in n8n<br><br>## Step 2 — Fetch prices & news (EODHD API)<br>Your Google Sheet must have a column named **`ticker`** (lowercase). One US ticker per row (MSFT, META, AMZN…).<br>- **EOD**: fetches last 14 days of closing prices — calculates daily % change<br>- **News**: fetches last 7 days of articles with sentiment polarity score per ticker |
| HTTP Request - EOD | HTTP Request | Fetches recent historical daily close prices per ticker from EODHD | Google Sheets | HTTP Request - News | ## Step 2 — Fetch prices & news (EODHD API)<br>Your Google Sheet must have a column named **`ticker`** (lowercase). One US ticker per row (MSFT, META, AMZN…).<br>- **EOD**: fetches last 14 days of closing prices — calculates daily % change<br>- **News**: fetches last 7 days of articles with sentiment polarity score per ticker |
| HTTP Request - News | HTTP Request | Fetches recent ticker news and sentiment from EODHD | HTTP Request - EOD | Build Email HTML | ## Step 2 — Fetch prices & news (EODHD API)<br>Your Google Sheet must have a column named **`ticker`** (lowercase). One US ticker per row (MSFT, META, AMZN…).<br>- **EOD**: fetches last 14 days of closing prices — calculates daily % change<br>- **News**: fetches last 7 days of articles with sentiment polarity score per ticker |
| Build Email HTML | Code | Rebuilds ticker-level data, computes change %, formats news, and generates HTML email content | HTTP Request - News | Send Email via Gmail | ## Step 3 — Build & send email<br>Groups prices and news by ticker, calculates % change vs previous close, and classifies sentiment:<br>- 😊 **Positive** (polarity > 0.6)<br>- 😐 **Neutral**<br>- 😟 **Negative** (polarity < 0.4)<br>Sends the HTML digest to the email set in ⚙️ Config. |
| Send Email via Gmail | Gmail | Sends the digest email to the configured recipient | Build Email HTML |  | ## Step 3 — Build & send email<br>Groups prices and news by ticker, calculates % change vs previous close, and classifies sentiment:<br>- 😊 **Positive** (polarity > 0.6)<br>- 😐 **Neutral**<br>- 😟 **Negative** (polarity < 0.4)<br>Sends the HTML digest to the email set in ⚙️ Config.<br><br>## Output<br>![txt](https://ik.imagekit.io/agbb7sr41/image.png) |
| Sticky Note | Sticky Note | Canvas documentation for overall workflow purpose and setup |  |  |  |
| Sticky Note1 | Sticky Note | Canvas documentation for trigger and configuration block |  |  |  |
| Sticky Note2 | Sticky Note | Canvas documentation for API fetch block |  |  |  |
| Sticky Note3 | Sticky Note | Canvas documentation for email build and send block |  |  |  |
| Sticky Note4 | Sticky Note | Canvas documentation showing example output image |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.  
   Name it something like: **Daily Financial News & Movers Email Digest**.

2. **Add a Schedule Trigger node.**
   - Node type: **Schedule Trigger**
   - Configure cron schedule: `0 7 * * *`
   - This will run every day at 07:00.
   - In workflow settings, set the timezone you want the 07:00 value to use.

3. **Add a Manual Trigger node.**
   - Node type: **Manual Trigger**
   - Leave default settings.
   - This gives you a second entry point for testing.

4. **Add a Set node and name it `⚙️ Config`.**
   - Node type: **Set**
   - Add two string fields:
     - `api_token` = your EODHD API key
     - `recipient_email` = destination email address
   - Keep the node simple; no computed expressions are required here.

5. **Connect both trigger nodes to `⚙️ Config`.**
   - `Schedule Trigger` → `⚙️ Config`
   - `Manual Trigger` → `⚙️ Config`

6. **Prepare the Google Sheet.**
   - Create a spreadsheet accessible by your Google credential.
   - Add a tab for the watchlist.
   - Ensure there is a column named exactly: `ticker`
   - Add one US stock symbol per row, for example:
     - `MSFT`
     - `META`
     - `AMZN`

7. **Add a Google Sheets node.**
   - Node type: **Google Sheets**
   - Operation: use the standard row-reading action suitable for returning sheet rows as items
   - Select your spreadsheet document
   - Select the target sheet/tab
   - Attach Google Sheets OAuth2 credentials
   - The output should be one item per row, including the `ticker` field

8. **Connect `⚙️ Config` to `Google Sheets`.**

9. **Add an HTTP Request node and name it `HTTP Request - EOD`.**
   - Node type: **HTTP Request**
   - Method: GET
   - URL expression:
     ```js
     {{ 'https://eodhd.com/api/eod/' + $json.ticker + '.US' }}
     ```
   - Enable query parameters and add:
     - `api_token` = `{{ $('⚙️ Config').first().json.api_token }}`
     - `fmt` = `json`
     - `order` = `d`
     - `from` = `{{ new Date(Date.now() - 14 * 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}`
     - `to` = `{{ new Date().toISOString().split('T')[0] }}`
   - Enable **Continue On Fail**
   - This ensures one bad ticker does not stop the entire workflow.

10. **Connect `Google Sheets` to `HTTP Request - EOD`.**

11. **Add a second HTTP Request node and name it `HTTP Request - News`.**
   - Node type: **HTTP Request**
   - Method: GET
   - URL:
     `https://eodhd.com/api/news`
   - Enable query parameters and add:
     - `s` = `{{ $('Google Sheets').item.json.ticker + '.US' }}`
     - `api_token` = `{{ $('⚙️ Config').first().json.api_token }}`
     - `fmt` = `json`
     - `limit` = `3`
     - `from` = `{{ new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}`
   - Enable **Continue On Fail**

12. **Connect `HTTP Request - EOD` to `HTTP Request - News`.**

13. **Add a Code node and name it `Build Email HTML`.**
   - Node type: **Code**
   - Language: JavaScript
   - Paste logic that:
     - loads all rows from `Google Sheets`
     - loads all results from `HTTP Request - EOD`
     - loads all results from `HTTP Request - News`
     - groups them by original ticker/item index
     - calculates latest close and daily percent change
     - filters news to the last 7 days
     - maps sentiment polarity into visual labels
     - builds one HTML string and one subject string
   - The code should return exactly one item with:
     - `html`
     - `subject`

14. **Use the following functional behavior in the Code node.**
   - Skip blank tickers.
   - Mark a ticker as failed if both EOD and news requests fail.
   - For EOD:
     - sort by date descending
     - if at least 2 records exist, compute:
       `((latestClose - previousClose) / previousClose) * 100`
   - For news:
     - keep only articles from the last 7 days
     - derive sentiment using `sentiment.polarity`
       - `> 0.6` → positive
       - `< 0.4` → negative
       - otherwise neutral
   - Generate a final subject like:
     `📊 Daily Market Digest — Mar 27, 2026`

15. **Connect `HTTP Request - News` to `Build Email HTML`.**

16. **Add a Gmail node and name it `Send Email via Gmail`.**
   - Node type: **Gmail**
   - Action: send email
   - Attach Gmail OAuth2 credentials
   - Set:
     - **To** = `{{ $('⚙️ Config').first().json.recipient_email }}`
     - **Subject** = `{{ $json.subject }}`
     - **Message** = `{{ $json.html }}`
   - Ensure the message is sent as HTML if your node version/interface exposes a content type or HTML option. In this exported workflow, the message body is passed directly as HTML.

17. **Connect `Build Email HTML` to `Send Email via Gmail`.**

18. **Configure credentials.**
   - **Google Sheets OAuth2**
     - Must have access to the spreadsheet
     - Reconnect if token expired
   - **Gmail OAuth2**
     - Must allow sending email from the chosen account
   - **EODHD**
     - No n8n credential object is used here; the API token is stored in the `⚙️ Config` Set node

19. **Add optional canvas notes for maintainability.**
   Suggested notes:
   - Trigger/config instructions
   - Google Sheet must contain `ticker`
   - EODHD provides prices and news
   - Output preview image or reference

20. **Test with Manual Trigger.**
   - Replace placeholder values in `⚙️ Config`
   - Ensure the sheet contains a few valid US tickers
   - Run the workflow manually
   - Verify:
     - Google Sheets returns rows
     - EOD and news requests return data
     - Code node outputs one item with `html` and `subject`
     - Gmail sends successfully

21. **Activate the workflow** once validated.
   - The schedule trigger will then send the digest every day at the configured scheduled time.

### Reproduction Notes for Data Expectations

- **Input expected from Google Sheets:**  
  One item per row with at least:
  - `ticker` as string

- **Input expected from EODHD EOD endpoint:**  
  Array-like daily records including:
  - `date`
  - `close`

- **Input expected from EODHD News endpoint:**  
  Article objects commonly including:
  - `title`
  - `date`
  - `link` or `url`
  - `content` or `description`
  - `tags`
  - `sentiment.polarity`

- **Output from the Code node:**  
  One item:
  - `html`: complete email body
  - `subject`: ready-to-send subject line

### Important Build Constraints

- The workflow assumes **US tickers** and appends `.US` automatically.
- The watchlist column name should be exactly **`ticker`**.
- The code uses **paired item tracking** to rebuild per-ticker context; preserve straight-through item flow between row retrieval and API nodes unless you deliberately redesign grouping logic.
- `continueOnFail` should remain enabled on both HTTP nodes to preserve partial success behavior.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Send daily stock price movers and financial news digest via Gmail | Workflow purpose |
| Automatically monitor your stock watchlist every morning and receive a formatted email with price changes and curated financial news with sentiment analysis. | Workflow description |
| Setup: fill in `api_token` and `recipient_email` in `⚙️ Config` before running. | Internal configuration guidance |
| Set your timezone in **Settings → Workflow Settings**. | Scheduling correctness |
| Google Sheet must have a column named `ticker` (lowercase). | Input data requirement |
| EODHD account free tier works. | External service requirement |
| Google OAuth2 must be configured in n8n. | Credential requirement |
| Output preview image | https://ik.imagekit.io/agbb7sr41/image.png |

## Additional Implementation Observations

- There are **two entry points**:
  - scheduled execution
  - manual execution
- There are **no sub-workflows** and no workflow-invoking nodes.
- The workflow is currently **inactive** in the export (`active: false`), so it must be activated after setup.
- There is a minor consistency issue between:
  - News node limit = `3`
  - Code node slice limit = `5`  
  If you want up to 5 articles per ticker in the email, increase the News API `limit` accordingly.
- The generated email is in English overall, but the sentiment labels include `Positivo` and `Negativo`. You may want to standardize them to `Positive` and `Negative` for consistency.