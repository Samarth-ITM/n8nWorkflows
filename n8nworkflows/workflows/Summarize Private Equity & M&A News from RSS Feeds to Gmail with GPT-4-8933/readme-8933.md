Summarize Private Equity & M&A News from RSS Feeds to Gmail with GPT-4

https://n8nworkflows.xyz/workflows/summarize-private-equity---m-a-news-from-rss-feeds-to-gmail-with-gpt-4-8933


# Summarize Private Equity & M&A News from RSS Feeds to Gmail with GPT-4

### 1. Workflow Overview

This workflow automates the daily summarization and email delivery of financial news focused on Private Equity and Mergers & Acquisitions (M&A), leveraging RSS feeds and GPT-4 for natural language processing. It is designed for finance professionals and analysts who want quick, readable news digests without manually scanning multiple sources. The workflow performs these major logical blocks:

- **1.1 Scheduled Trigger and RSS Feed Collection:** It starts by triggering at two specific times daily, then concurrently reads three RSS feeds related to Private Equity, M&A, and general finance news.
- **1.2 Feed Merging and Limiting:** The feeds are merged into a single stream, then limited to the most recent 5 items to control volume.
- **1.3 News Formatting:** A JavaScript node formats the merged news items into a structured text block summarizing title, author, date, link, and snippet.
- **1.4 AI Text Rewriting:** The formatted news is processed by an AI language model (GPT-4) which rewrites each news item into a short, engaging 5–6 sentence story using a defined prompt.
- **1.5 Email Delivery:** Finally, the rewritten summaries are sent via Gmail to a configured recipient.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger and RSS Feed Collection

- **Overview:** This block initiates the workflow twice daily and fetches news from three distinct RSS feeds related to Private Equity, M&A, and general finance news.
- **Nodes Involved:** `Schedule Trigger`, `Reuters Private Equity`, `M&A`, `Yahoo Finance`

- **Node Details:**

  - **Schedule Trigger**
    - *Type:* Schedule Trigger
    - *Role:* Starts the workflow automatically at predefined times.
    - *Configuration:* Triggers at 09:00 and 15:00 daily.
    - *Input/Output:* No input; outputs trigger event to three RSS nodes.
    - *Edge cases:* If n8n server time zone differs from expected, trigger times may be off.
    - *Sticky Note:* Explains the trigger times and how to adjust them.

  - **Reuters Private Equity**
    - *Type:* RSS Feed Read
    - *Role:* Reads RSS feed from a New York Times Private Equity topic collection.
    - *Configuration:* URL set to New York Times private equity RSS feed.
    - *Input/Output:* Input from Schedule Trigger; outputs feed items to Merge node.
    - *Edge cases:* RSS feed unavailability or format changes; may return zero items.

  - **M&A**
    - *Type:* RSS Feed Read
    - *Role:* Reads RSS feed from Deal Lawyers blog focused on M&A.
    - *Configuration:* URL set to https://www.deallawyers.com/blog/feed.
    - *Input/Output:* Input from Schedule Trigger; outputs feed items to Merge node.
    - *Edge cases:* Feed downtime or format changes; no items returned.

  - **Yahoo Finance**
    - *Type:* RSS Feed Read
    - *Role:* Reads general finance news from Yahoo Finance RSS.
    - *Configuration:* URL set to https://finance.yahoo.com/news/rssindex.
    - *Input/Output:* Input from Schedule Trigger; outputs feed items to Merge node.
    - *Edge cases:* Feed out of service or format changes.

#### 2.2 Feed Merging and Limiting

- **Overview:** Merges the three incoming RSS feed streams into one combined stream and limits the total number of news items to five.
- **Nodes Involved:** `Merge`, `Limit`

- **Node Details:**

  - **Merge**
    - *Type:* Merge
    - *Role:* Combines three RSS feed outputs into a single stream.
    - *Configuration:* Number of inputs set to 3; merges all items.
    - *Input/Output:* Inputs from `M&A`, `Reuters Private Equity`, `Yahoo Finance`; outputs to `Limit`.
    - *Edge cases:* Unequal input sizes; if one feed is empty, merge still proceeds.

  - **Limit**
    - *Type:* Limit
    - *Role:* Restricts the output to the latest 5 news items.
    - *Configuration:* `maxItems` set to 5.
    - *Input/Output:* Input from `Merge`; outputs limited items to next node.
    - *Edge cases:* If total merged items <5, passes all items as is.

#### 2.3 News Formatting

- **Overview:** Formats the limited news items into a readable structured text block to prepare for AI rewriting.
- **Nodes Involved:** `Code in JavaScript`

- **Node Details:**

  - **Code in JavaScript**
    - *Type:* Code
    - *Role:* Concatenates each news item into a formatted text block with fields: title, author, date, link, and a short snippet.
    - *Configuration:* Custom JavaScript iterates over limited items, extracting properties and assembling the text using template literals.
    - *Key Expressions:* Accesses fields like `title`, `creator` (or fallback), `pubDate`, `link`, `contentSnippet` or `content`.
    - *Input/Output:* Input from `Limit`; outputs JSON with `news_summary` field containing formatted text.
    - *Edge cases:* Missing author fields handled with "Ukjent" (Unknown); missing snippet replaced by placeholder text.
    - *Sticky Note:* Explains purpose and customization of code formatting.

#### 2.4 AI Text Rewriting

- **Overview:** Uses GPT-4 to rewrite the formatted news summaries into short, engaging stories with a professional but simple style.
- **Nodes Involved:** `Basic LLM Chain`, `OpenAI Chat Model`

- **Node Details:**

  - **Basic LLM Chain**
    - *Type:* LangChain LLM Chain
    - *Role:* Defines the prompt and processes input text to generate rewritten news stories.
    - *Configuration:* Custom prompt instructs the AI to rewrite each news item as a short story (5–6 sentences), using HTML formatting with `<h2>` for title and `<p>` for narrative paragraphs.
    - *Key Expressions:* Uses `{{ $json.news_summary }}` as the input text to rewrite.
    - *Input/Output:* Input from `Code in JavaScript`; outputs rewritten HTML stories to Gmail node.
    - *Edge cases:* If input text is empty or malformed, AI output may be poor or error; prompt can be edited for style changes.
    - *Sticky Note:* Notes about AI rewriting style and prompt customization.

  - **OpenAI Chat Model**
    - *Type:* LangChain OpenAI Chat Model
    - *Role:* Provides GPT-4.1-mini language model to the Basic LLM Chain.
    - *Configuration:* Model set to GPT-4.1-mini.
    - *Input/Output:* Input from Basic LLM Chain as the language model; no direct input from workflow nodes.
    - *Edge cases:* Requires valid OpenAI API credentials; quota or network issues may cause failures.

#### 2.5 Email Delivery

- **Overview:** Sends the AI-generated news summaries via Gmail to the configured recipient.
- **Nodes Involved:** `Send a message`

- **Node Details:**

  - **Send a message**
    - *Type:* Gmail node
    - *Role:* Sends an email containing the rewritten news stories.
    - *Configuration:* Uses Gmail OAuth2 credentials; recipient email must be configured by replacing `{{RECIPIENT_EMAIL}}`.
    - *Input/Output:* Input from `Basic LLM Chain`; no output.
    - *Edge cases:* Requires valid Gmail OAuth2 credentials; email sending may fail if quota exceeded or credentials invalid.
    - *Sticky Note:* Instructions to add Gmail credential and set recipient email.

---

### 3. Summary Table

| Node Name           | Node Type                | Functional Role                          | Input Node(s)                  | Output Node(s)             | Sticky Note                                                                                  |
|---------------------|--------------------------|----------------------------------------|-------------------------------|----------------------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger         | Initiates workflow at scheduled times  | —                             | M&A, Reuters Private Equity, Yahoo Finance | 🕒 Schedule Trigger runs at 09:00 and 15:00. Change hours here to adjust update time.       |
| M&A                 | RSS Feed Read            | Reads M&A-related RSS feed              | Schedule Trigger               | Merge                      |                                                                                             |
| Reuters Private Equity | RSS Feed Read           | Reads Private Equity RSS feed           | Schedule Trigger               | Merge                      |                                                                                             |
| Yahoo Finance       | RSS Feed Read            | Reads general finance news RSS feed    | Schedule Trigger               | Merge                      |                                                                                             |
| Merge               | Merge                    | Combines three RSS feeds into one stream | M&A, Reuters Private Equity, Yahoo Finance | Limit                      |                                                                                             |
| Limit               | Limit                    | Limits number of news items to 5       | Merge                         | Code in JavaScript          |                                                                                             |
| Code in JavaScript  | Code                     | Formats news items into readable text  | Limit                         | Basic LLM Chain             | 💻 Code in JavaScript formats feed items into clean text block. Edit code to change layout. |
| Basic LLM Chain     | LangChain LLM Chain      | Rewrites news summaries into stories   | Code in JavaScript             | Send a message              | 🤖 Basic LLM Chain rewrites news into simple 5–6 sentence stories. Edit prompt to change style. |
| OpenAI Chat Model   | LangChain OpenAI Chat Model | Provides GPT-4 model for rewriting    | — (linked to Basic LLM Chain) | Basic LLM Chain             |                                                                                             |
| Send a message      | Gmail                    | Sends rewritten news summaries by email | Basic LLM Chain               | —                          | 📧 Gmail Node sends summaries by email. Add Gmail credential and set recipient email.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**
   - Set the trigger rule to run at 09:00 and 15:00 daily.
   - Position as the start node.

2. **Add Three RSS Feed Read Nodes**
   - Name them `Reuters Private Equity`, `M&A`, and `Yahoo Finance`.
   - Configure URLs:
     - Reuters Private Equity: `https://www.nytimes.com/svc/collections/v1/publish/http://www.nytimes.com/topic/subject/private-equity/rss.xml`
     - M&A: `https://www.deallawyers.com/blog/feed`
     - Yahoo Finance: `https://finance.yahoo.com/news/rssindex`
   - Connect outputs of Schedule Trigger to each RSS node's input.

3. **Add a Merge Node**
   - Set `Number of Inputs` to 3.
   - Connect outputs of the three RSS nodes (`M&A`, `Reuters Private Equity`, `Yahoo Finance`) to inputs 0, 1, and 2 respectively of the Merge node.

4. **Add a Limit Node**
   - Set `maxItems` to 5.
   - Connect the Merge node output to the Limit node input.

5. **Add a Code Node (JavaScript)**
   - Paste the following code to format the news items:

     ```javascript
     const items = $input.all().map(item => item.json);
     const formatted = items.map(i => {
       return `📰 ${i.title}
     Av: ${i.creator || i["dc:creator"] || "Ukjent"}
     Dato: ${i.pubDate}
     Lenke: ${i.link}

     Kort oppsummering:
     ${i.contentSnippet || i.content || "Ingen utdrag tilgjengelig"}

     `;
     }).join("\n---\n\n");
     return [{ json: { news_summary: formatted } }];
     ```

   - Connect the Limit node output to this Code node.

6. **Add a LangChain OpenAI Chat Model Node**
   - Select GPT-4.1-mini as the model.
   - Configure OpenAI API credentials with valid keys.

7. **Add a LangChain Basic LLM Chain Node**
   - Use the following prompt (HTML formatting and instructions included):

     ```
     You are a professional writer who makes complex financial news easy to read and engaging.  
     Rewrite each news item from the RSS feed as a short story of 5–6 lines.  
     {{ $json.news_summary }}

     Use this structure:

     1. <h2>Title</h2>  
        Use the news item’s own title.  

     2. <p>Write a short, continuous story that explains the news. Use a natural narrative style, as if you are explaining it to an interested reader.  
        Complex terms should be briefly explained in parentheses.  
        Do not use subheadings, bullet points, or questions. Only a flowing story of max 6 sentences.</p>  

     3. <p><em>Connection to the news:</em> End with 1–2 sentences that clearly link this story back to the original article.</p>  

     Rules:  
     - Keep the language professional but simple.  
     - No “what does this mean for you” commentary.  
     - Do not use emojis or casual filler.  
     - Always format in HTML (<h2>, <p>).
     ```

   - Connect input from Code node output.
   - Set the LangChain LLM to use the OpenAI Chat Model node.

8. **Add a Gmail Node (Send a message)**
   - Configure with your Gmail OAuth2 credentials.
   - Set the recipient email address in the "To" field by replacing `{{RECIPIENT_EMAIL}}` with the actual email.
   - Connect the output of Basic LLM Chain node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                               |
|------------------------------------------------------------------------------|---------------------------------------------------------------|
| Schedule times can be customized by adjusting the Schedule Trigger node.    | Scheduling details in node “Schedule Trigger” sticky note.    |
| JavaScript formatting code is editable to change text layout or fields.     | See sticky note on “Code in JavaScript” node.                 |
| AI prompt can be modified for different narrative styles or output formats. | See sticky note on “Basic LLM Chain” node.                    |
| Gmail node requires adding OAuth2 credentials and setting recipient email.  | Instructions in sticky note on “Send a message” Gmail node.   |
| Workflow tested with GPT-4.1-mini model for cost-efficient summarization.    | OpenAI Chat Model node configuration.                         |

---

**Disclaimer:** The text and data used in this workflow are sourced solely from publicly available RSS feeds and processed in compliance with content policies. No illegal, offensive, or protected content is included.