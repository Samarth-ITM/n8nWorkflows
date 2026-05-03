Daily AI News Digest from Hacker News with GPT-5 Summaries and Email Delivery

https://n8nworkflows.xyz/workflows/daily-ai-news-digest-from-hacker-news-with-gpt-5-summaries-and-email-delivery-9384


# Daily AI News Digest from Hacker News with GPT-5 Summaries and Email Delivery

---

### 1. Workflow Overview

This workflow automates the creation and delivery of a daily AI news digest sourced from Hacker News. It retrieves recent stories tagged with "AI," filters them to the last 24 hours, scrapes the full article content, summarizes each article using GPT-5, aggregates summaries into a styled HTML email, and sends it to a predefined recipient.

The workflow is logically divided into the following blocks:

- **1.1 Daily Trigger & Data Fetching:** Initiates the workflow on a daily schedule and fetches Hacker News stories related to AI.
- **1.2 Filtering Recent Stories:** Filters stories to include only those created in the last 24 hours.
- **1.3 Sequential Article Processing:** Processes each story individually to avoid rate limits by scraping, cleaning, and summarizing.
- **1.4 Summarization & Formatting:** Uses GPT-5 to generate concise summaries, formats each item for the email.
- **1.5 Aggregation & Email Delivery:** Combines all summaries into one HTML email and sends it via SMTP.

---

### 2. Block-by-Block Analysis

#### 1.1 Daily Trigger & Data Fetching

- **Overview:**  
  This block triggers the workflow daily and fetches up to 1000 Hacker News stories containing the keyword "AI."

- **Nodes Involved:**  
  - Daily Schedule Trigger  
  - Fetch HN AI Stories

- **Node Details:**

  - **Daily Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Initiates workflow execution once per day (default daily interval).  
    - Configuration: Uses default interval setting for daily execution.  
    - Inputs: None (trigger node).  
    - Outputs: Connects to "Fetch HN AI Stories."  
    - Edge Cases: Misconfigured schedule may cause workflow not to run; time zone considerations may affect timing.

  - **Fetch HN AI Stories**  
    - Type: Hacker News Node  
    - Role: Pulls up to 1000 stories matching "AI" keyword from Hacker News public API.  
    - Configuration:  
      - Resource: "all" stories  
      - Limit: 1000  
      - Additional Filter: Keyword "AI"  
    - Inputs: Trigger from "Daily Schedule Trigger."  
    - Outputs: Passes data to "Filter Last 24 Hours."  
    - Edge Cases: Large data fetch may be rate-limited; API downtime or malformed responses possible.

---

#### 1.2 Filtering Recent Stories

- **Overview:**  
  Filters fetched stories to retain only those created within the last 24 hours, ensuring the digest is timely.

- **Nodes Involved:**  
  - Filter Last 24 Hours

- **Node Details:**

  - **Filter Last 24 Hours**  
    - Type: Filter  
    - Role: Selects stories where `created_at` date is within the last 24 hours.  
    - Configuration:  
      - Condition: `created_at` field greater than or equal to current date minus 1 day (ISO date format).  
      - Uses expression: `{{$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')}}`  
    - Inputs: Stories from "Fetch HN AI Stories."  
    - Outputs: Passes filtered stories to "Loop Over Items."  
    - Edge Cases:  
      - Stories with missing or malformed `created_at` will be excluded.  
      - Timezone differences may affect filtering accuracy.

---

#### 1.3 Sequential Article Processing

- **Overview:**  
  Processes each story sequentially to avoid rate limits and enables detailed content scraping and summarization.

- **Nodes Involved:**  
  - Loop Over Items  
  - Scrape Article URL  
  - Convert to Markdown

- **Node Details:**

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Iterates over each filtered story one at a time.  
    - Configuration: Default batch size (1).  
    - Inputs: Filtered stories from "Filter Last 24 Hours."  
    - Outputs: Two outputs: one to "Combine All Summaries" (aggregating results) and one to "Scrape Article URL" (processing each item).  
    - Edge Cases:  
      - Large number of stories could slow processing.  
      - Batch splitting avoids hitting API or resource limits.

  - **Scrape Article URL**  
    - Type: HTTP Request  
    - Role: Fetches the full HTML content of each story's URL.  
    - Configuration:  
      - URL set dynamically from story's `url` field.  
      - No special headers or proxy configured by default.  
    - Inputs: Single story item from "Loop Over Items."  
    - Outputs: HTML content passed to "Convert to Markdown."  
    - Edge Cases:  
      - Some URLs may be inaccessible or blocked (may require proxy).  
      - HTTP errors, timeouts, or redirects need handling.

  - **Convert to Markdown**  
    - Type: Markdown Node  
    - Role: Converts scraped HTML content into Markdown format suitable for GPT input.  
    - Configuration:  
      - Input HTML from HTTP Request node's `data` field.  
    - Inputs: HTML content from "Scrape Article URL."  
    - Outputs: Markdown text passed to "GPT Summarize Article."  
    - Edge Cases:  
      - Poorly formatted HTML may produce low-quality Markdown.  
      - Empty or missing content will affect summary quality.

---

#### 1.4 Summarization & Formatting

- **Overview:**  
  Uses GPT-5 to generate concise AI news snippets from article Markdown and formats each summary for email.

- **Nodes Involved:**  
  - GPT 5 pro  
  - GPT Summarize Article  
  - Format News Item

- **Node Details:**

  - **GPT 5 pro**  
    - Type: Langchain OpenAI Chat LLM  
    - Role: Provides the GPT-5 language model backend for summarization.  
    - Configuration:  
      - Model: "gpt-5-pro"  
      - Credentials: OpenAI API key assigned.  
    - Inputs: Receives prompt from "GPT Summarize Article."  
    - Outputs: Summarization results forwarded to "GPT Summarize Article."  
    - Edge Cases:  
      - API quota exceeded or invalid credentials cause failures.  
      - Network issues or slow responses can delay workflow.

  - **GPT Summarize Article**  
    - Type: Langchain Chain LLM (prompt node)  
    - Role: Sends prompt to GPT 5 pro to summarize article Markdown into a headline and two sentences.  
    - Configuration:  
      - Prompt text includes instructions to output main point as heading and 2 sentence summary, optionally with link.  
      - Uses expression to inject Markdown content from `{{$json.markdown}}`.  
    - Inputs: Markdown content from "Convert to Markdown."  
    - Outputs: GPT summary text to "Format News Item."  
    - Edge Cases:  
      - Prompt failure or empty input yields poor summaries.  
      - Model misinterpretation possible; prompt tuning may be needed.

  - **Format News Item**  
    - Type: Set  
    - Role: Wraps GPT summary output into an HTML snippet for email formatting.  
    - Configuration:  
      - Creates a new JSON field `news` containing the GPT summary output string.  
    - Inputs: GPT summary from "GPT Summarize Article."  
    - Outputs: Passes formatted news item back to "Loop Over Items" for aggregation.  
    - Edge Cases:  
      - Improper formatting may break email layout.  
      - Missing input data leads to empty news items.

---

#### 1.5 Aggregation & Email Delivery

- **Overview:**  
  Aggregates all formatted news snippets into a single HTML email body and sends the digest via SMTP.

- **Nodes Involved:**  
  - Combine All Summaries  
  - Send Email Digest

- **Node Details:**

  - **Combine All Summaries**  
    - Type: Aggregate  
    - Role: Collects all individual `news` snippets into a single array for the email template.  
    - Configuration:  
      - Aggregates field `news` from incoming items.  
    - Inputs: Receives formatted news snippets from "Loop Over Items."  
    - Outputs: Passes combined array to "Send Email Digest."  
    - Edge Cases:  
      - If no summaries exist, results may be empty.  
      - Large aggregation could impact email size.

  - **Send Email Digest**  
    - Type: Email Send  
    - Role: Sends the final daily AI news digest email with inline CSS styling and responsive design.  
    - Configuration:  
      - Subject includes current date `{{ $now.format('MMMM d, yyyy') }}`.  
      - HTML body embeds aggregated news snippets `{{ $json.news.join('') }}` inside a styled container.  
      - From and To emails must be configured before activation.  
      - Uses Zoho Mail SMTP credentials (or alternative SMTP).  
    - Inputs: Aggregated news array from "Combine All Summaries."  
    - Outputs: None (end node).  
    - Edge Cases:  
      - SMTP misconfiguration causes email delivery failures.  
      - Large email content may be truncated by some clients.  
      - Missing recipient or sender email fields lead to errors.

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                  | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                                                                                                                                |
|-----------------------|-------------------------------|---------------------------------|----------------------------|---------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Daily Schedule Trigger | Schedule Trigger              | Starts workflow daily            | -                          | Fetch HN AI Stories        |                                                                                                                                                                                                            |
| Fetch HN AI Stories    | Hacker News API               | Fetches AI-related stories       | Daily Schedule Trigger      | Filter Last 24 Hours       | 📥 Data Collection: Pulls 1000 stories with "AI" keyword; Uses public API with no auth                                                                                                                      |
| Filter Last 24 Hours   | Filter                       | Filters stories from last 24h    | Fetch HN AI Stories         | Loop Over Items            | 📥 Data Collection: Filters by `created_at` field to keep recent stories; Uses `$now().minus({ days: 1 })`                                                                                                  |
| Loop Over Items        | Split In Batches             | Processes each story sequentially| Filter Last 24 Hours        | Combine All Summaries, Scrape Article URL | 🌐 Content Processing: Prevents rate limiting by sequential processing                                                                                                                                        |
| Scrape Article URL     | HTTP Request                 | Fetches full article HTML content| Loop Over Items             | Convert to Markdown        | 🌐 Content Processing: Scrapes source URL; may need proxy if blocked                                                                                                                                         |
| Convert to Markdown    | Markdown                     | Converts HTML to Markdown        | Scrape Article URL          | GPT Summarize Article      | 🌐 Content Processing: Cleans HTML for LLM input                                                                                                                                                            |
| GPT 5 pro             | Langchain OpenAI Chat LLM    | GPT-5 model for summarization    | GPT Summarize Article       | GPT Summarize Article      | 🤖 AI Summarization: GPT 5 pro node used for generating summaries                                                                                                                                           |
| GPT Summarize Article  | Langchain Chain LLM          | Sends prompt to GPT-5 for summary| Convert to Markdown         | Format News Item           | 🤖 AI Summarization: Generates heading + 2 sentence summary; prompt customizable                                                                                                                            |
| Format News Item       | Set                          | Formats summary as HTML snippet  | GPT Summarize Article       | Loop Over Items            | 🤖 AI Summarization: Prepares summary for aggregation in email                                                                                                                                                |
| Combine All Summaries  | Aggregate                    | Aggregates all summaries         | Loop Over Items             | Send Email Digest          | 🤖 AI Summarization: Merges summaries for email template                                                                                                                                                     |
| Send Email Digest      | Email Send                   | Sends final daily email digest   | Combine All Summaries       | -                         | 📧 Email Delivery: Styled responsive HTML email; update fromEmail/toEmail before activating                                                                                                                  |
| Overview & Setup       | Sticky Note                  | Template overview and instructions| -                          | -                         | Contains detailed setup notes, prerequisites, and troubleshooting instructions                                                                                                                              |
| Note: HN Fetch & Filter| Sticky Note                  | Notes on data collection nodes   | -                          | -                         | Explains fetch and filter logic; advises on customization                                                                                                                                                |
| Note: Scraping & Processing | Sticky Note              | Notes on scraping and conversion | -                          | -                         | Explains scraping and markdown conversion; troubleshooting scraping issues                                                                                                                               |
| Note: AI Summarization | Sticky Note                  | Notes on GPT summarization nodes | -                          | -                         | Details summarization logic and prompt customization                                                                                                                                                     |
| Note: Email Delivery   | Sticky Note                  | Notes on email sending            | -                          | -                         | Details styling, template features, and SMTP setup requirements                                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add “Daily Schedule Trigger” node:**  
   - Type: Schedule Trigger  
   - Set to run daily at desired time (default interval is daily).  
   - No credentials needed.

3. **Add “Fetch HN AI Stories” node:**  
   - Type: Hacker News node  
   - Resource: “all”  
   - Limit: 1000 stories  
   - Additional Fields: Keyword = “AI”  
   - Connect input from “Daily Schedule Trigger.”

4. **Add “Filter Last 24 Hours” node:**  
   - Type: Filter  
   - Condition:  
     - Left value: `{{$json.created_at}}`  
     - Operator: “after or equals”  
     - Right value: `{{$now.minus({ days: 1 }).toFormat('yyyy-MM-dd')}}`  
   - Connect input from “Fetch HN AI Stories.”

5. **Add “Loop Over Items” node:**  
   - Type: Split In Batches  
   - Batch size: default (1)  
   - Connect input from “Filter Last 24 Hours.”

6. **Add “Scrape Article URL” node:**  
   - Type: HTTP Request  
   - URL: `{{$json.url}}` (dynamic)  
   - No special auth or headers required by default.  
   - Connect input from “Loop Over Items” (second output).

7. **Add “Convert to Markdown” node:**  
   - Type: Markdown  
   - Input HTML: `{{$json.data}}` (from HTTP Request)  
   - Connect input from “Scrape Article URL.”

8. **Add “GPT 5 pro” node:**  
   - Type: Langchain OpenAI Chat LLM  
   - Model: “gpt-5-pro” (ensure this model is available)  
   - Credentials: Configure OpenAI API key in n8n credential manager and assign here.

9. **Add “GPT Summarize Article” node:**  
   - Type: Langchain Chain LLM  
   - Prompt Type: Define  
   - Prompt Text:  
     ```
     Summarize the info below so that I can get a quick news snippet of what this is about in the area of AI. 

     Simply output the main point as the heading and then 2 sentences of the summary (maybe with a link)

     ---

     Scraped info below: {{$json.markdown}}
     ```
   - Connect input from “Convert to Markdown.”  
   - Connect “GPT 5 pro” node’s output as language model for this node.

10. **Add “Format News Item” node:**  
    - Type: Set  
    - Assign field:  
      - Name: news  
      - Type: string  
      - Value: `{{$json.output}}` (output from GPT Summarize Article)  
    - Connect input from “GPT Summarize Article.”  
    - Connect output back to “Loop Over Items” (first output) for aggregation.

11. **Back to “Loop Over Items” node:**  
    - Connect first output to “Combine All Summaries.”  
    - Connect second output to “Scrape Article URL.”

12. **Add “Combine All Summaries” node:**  
    - Type: Aggregate  
    - Aggregate field: `news`  
    - Connect input from “Loop Over Items” (first output).

13. **Add “Send Email Digest” node:**  
    - Type: Email Send  
    - Subject: `Daily AI Digest - {{$now.format('MMMM d, yyyy')}}`  
    - To Email: Set recipient email address  
    - From Email: Set sender address, e.g., "AI News <noreply@example.com>"  
    - HTML Body: Use the provided multi-line HTML template with inline CSS, embedding summaries with `{{$json.news.join('')}}`  
    - Credentials: Configure and assign SMTP credentials (e.g., Zoho Mail OAuth2 or SMTP).  
    - Connect input from “Combine All Summaries.”

14. **Final connections:**  
    - Connect “Daily Schedule Trigger” → “Fetch HN AI Stories”  
    - “Fetch HN AI Stories” → “Filter Last 24 Hours”  
    - “Filter Last 24 Hours” → “Loop Over Items”  
    - “Loop Over Items” outputs → “Scrape Article URL” and “Combine All Summaries”  
    - “Scrape Article URL” → “Convert to Markdown” → “GPT Summarize Article” → “Format News Item” → back to “Loop Over Items” for aggregation  
    - “Combine All Summaries” → “Send Email Digest”

15. **Activate workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                               | Context or Link                                                                                                                          |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Workflow fetches AI-related stories from Hacker News, summarizes with GPT-5, and emails a styled daily digest.                                                            | Overview & Setup sticky note in workflow.                                                                                              |
| OpenAI API key required; get from https://platform.openai.com/api-keys                                                                                                     | Setup instructions in Overview & Setup sticky note.                                                                                    |
| SMTP server credentials needed (Zoho Mail example provided); configure in n8n credentials.                                                                                 | Email Delivery sticky note.                                                                                                            |
| Scraping article URLs may require proxy if sites block direct requests.                                                                                                    | Scraping & Processing sticky note.                                                                                                    |
| GPT prompt can be customized for different summary styles or lengths.                                                                                                      | AI Summarization sticky note.                                                                                                         |
| Email template uses inline CSS and responsive design for compatibility with most email clients and mobile devices.                                                        | Email Delivery sticky note.                                                                                                            |
| Troubleshooting tips: check API limits, verify SMTP setup, add proxy for scraping, and adjust keyword or date filters as needed.                                           | Overview & Setup sticky note.                                                                                                          |

---

**Disclaimer:** The provided text is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.

---