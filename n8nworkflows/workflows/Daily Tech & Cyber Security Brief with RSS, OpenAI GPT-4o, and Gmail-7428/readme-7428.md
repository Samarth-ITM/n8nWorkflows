Daily Tech & Cyber Security Brief with RSS, OpenAI GPT-4o, and Gmail

https://n8nworkflows.xyz/workflows/daily-tech---cyber-security-brief-with-rss--openai-gpt-4o--and-gmail-7428


# Daily Tech & Cyber Security Brief with RSS, OpenAI GPT-4o, and Gmail

### 1. Workflow Overview

This workflow automates the creation and delivery of a daily cybersecurity and technology news brief. It fetches news items from multiple RSS feeds, consolidates and deduplicates them, filters to recent stories from the last 24 hours, summarizes and structures the content using OpenAI’s GPT-4o-mini model, then emails the final brief via Gmail. The workflow is designed for cybersecurity or tech professionals, such as a VP of Cybersecurity, who want a concise, curated daily digest of relevant news.

Logical blocks:

- **1.1 Input Reception**: Scheduled trigger initiates the workflow daily; multiple RSS feed readers fetch news items from various security and tech sources.
- **1.2 Data Aggregation & Cleaning**: Merge all feed items, remove duplicate links, filter for items published within the last 24 hours.
- **1.3 Content Processing**: Format and prepare the consolidated news items, send them to OpenAI GPT-4o-mini for summarization and categorization into predefined groups, parse the AI response.
- **1.4 Delivery**: Send the final formatted email brief via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block triggers the workflow daily at a scheduled time and gathers news from six different RSS feeds covering cybersecurity and tech news.

**Nodes Involved:**  
- Schedule Trigger  
- Bleeping Computer (RSS Feed)  
- CISA GOV (RSS Feed)  
- HNRSS (Hacker News RSS with filtering)  
- Feedburner (RSS Feed)  
- Ars Technica (RSS Feed)  
- Techcrunch (RSS Feed)  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow daily at 10:00 AM (local timezone assumed).  
  - Configuration: Cron set to trigger at hour 10 every day.  
  - Inputs: None  
  - Outputs: Triggers all RSS Feed nodes in parallel.  
  - Potential Failures: Scheduler misconfiguration or n8n runtime issues could cause missed triggers.

- **RSS Feed Nodes (Bleeping Computer, CISA GOV, HNRSS, Feedburner, Ars Technica, Techcrunch)**  
  - Type: RSS Feed Read  
  - Role: Fetch latest news items from configured URLs.  
  - Configuration: Each node set to read a specific RSS feed URL; some have SSL verification enabled, some disabled as appropriate.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: News items in JSON format, including title, link, publication date, description, etc.  
  - Edge Cases: Feed unavailability, SSL errors, malformed feed data, or no new items.  
  - Notes: The HNRSS feed applies a filter for items with at least 150 points, focusing on popular stories.

---

#### 1.2 Data Aggregation & Cleaning

**Overview:**  
This block consolidates all incoming feed items into one stream, removes duplicate stories based on their URLs, and filters for stories published within the last 24 hours.

**Nodes Involved:**  
- Merge  
- Remove Duplicates  
- If (Date Filter)  
- Limit  
- No New Items (Code)  

**Node Details:**

- **Merge**  
  - Type: Merge (Append mode)  
  - Role: Combines outputs from all six RSS feeds into one unified list.  
  - Configuration: Number of inputs = 6 (one per RSS feed).  
  - Inputs: All RSS Feed nodes.  
  - Outputs: Combined list of news items.  
  - Edge Cases: Empty inputs, unbalanced inputs, delayed feed responses.

- **Remove Duplicates**  
  - Type: Remove Duplicates  
  - Role: Deduplicates news items based on the `link` field to avoid repeated stories.  
  - Configuration: Compare selected fields — field `link`.  
  - Inputs: Output of Merge node.  
  - Outputs: Deduplicated list of stories.  
  - Failure Modes: If `link` field missing or malformed, deduplication may fail or miss duplicates.

- **If (Date Filter)**  
  - Type: If Condition  
  - Role: Filters stories published within the last 24 hours.  
  - Configuration: Checks if publication date (`isoDate`, `pubDate`, or `date`) is later than current time minus 24 hours.  
  - Inputs: Deduplicated stories.  
  - Outputs:  
    - True branch: Stories newer than 24 hours.  
    - False branch: Older or empty.  
  - Edge Cases: Missing or improperly formatted date fields can cause expression to fail or reject valid items.

- **Limit**  
  - Type: Limit  
  - Role: Restricts the number of stories processed further to a maximum of 25.  
  - Configuration: Keep exactly 25 items.  
  - Inputs: True output of If node.  
  - Outputs: Up to 25 most recent stories.  
  - Notes: Prevents overloading AI prompt or email with too many stories.

- **No New Items (Code)**  
  - Type: Code  
  - Role: Handles case where no recent stories were found; generates a fallback message for the email.  
  - Code Summary: If no items, output a JSON object indicating empty with subject and message saying no notable headlines in last 24h. Otherwise passes items through unchanged.  
  - Inputs: False output of If node.  
  - Outputs: Either fallback message or input items unchanged.  
  - Edge Cases: Requires items array presence; if malformed input, may error.

---

#### 1.3 Content Processing

**Overview:**  
This block formats the aggregated stories, sends them to OpenAI GPT-4o-mini model for summarization and categorization, then parses the AI-generated JSON response.

**Nodes Involved:**  
- Code1  
- Message a model (OpenAI)  
- Code2  

**Node Details:**

- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Transforms the array of news items into a structured JSON array with standardized fields (`source`, `title`, `link`, `published`, `snippet`).  
  - Key Logic: Extracts host from URL, normalizes publication date and snippet. Outputs a single JSON object with a `stories` array containing all items.  
  - Inputs: Up to 25 items from Limit or fallback item from No New Items.  
  - Outputs: One item with a JSON property `stories` containing the array.  
  - Edge Cases: URL parsing failures handled with try/catch; missing fields default to empty or 'unknown'.

- **Message a model (OpenAI GPT-4o-mini)**  
  - Type: OpenAI (LangChain node)  
  - Role: Sends prompt and stories JSON to GPT-4o-mini to generate a concise daily brief grouped by categories with strict JSON output.  
  - Configuration:  
    - Model: gpt-4o-mini  
    - Max tokens: 1600  
    - System message: Instructions for summarization style, output format, and grouping categories.  
    - User message: Injected JSON stringified stories from Code1.  
  - Inputs: Output from Code1.  
  - Outputs: AI-generated message with raw content expected to be JSON formatted as per instructions.  
  - Failure Modes: API key/credential issues, model throttling, malformed input/output, parsing errors downstream.

- **Code2**  
  - Type: Code (JavaScript)  
  - Role: Cleans and parses the AI output content string into a JSON object with keys: `subject`, `html`, `text`.  
  - Key Logic: Strips any markdown JSON code fences, tries JSON.parse, falls back to raw content with a warning subject if parse fails.  
  - Inputs: AI response from Message a model.  
  - Outputs: Parsed JSON object for email sending.  
  - Edge Cases: Malformed JSON from AI, unexpected content formats.

---

#### 1.4 Delivery

**Overview:**  
Sends the final summarized daily brief as an email via Gmail.

**Nodes Involved:**  
- Send a message (Gmail)  

**Node Details:**

- **Send a message**  
  - Type: Gmail  
  - Role: Sends the email containing the summary brief to the configured recipient.  
  - Configuration:  
    - Recipient email: Configured email address (default `test@gmail.com` in example).  
    - Subject: Uses parsed `subject` from AI output or defaults to "Daily Cyber & Tech Brief".  
    - Message body: Uses parsed `html` from AI output or a fallback message.  
    - Credentials: Gmail OAuth2 credentials configured.  
  - Inputs: Parsed JSON from Code2.  
  - Outputs: None (terminal node).  
  - Failure Modes: Auth errors, quota limits, network issues.

---

### 3. Summary Table

| Node Name         | Node Type                  | Functional Role                         | Input Node(s)                     | Output Node(s)           | Sticky Note                                                                                          |
|-------------------|----------------------------|---------------------------------------|----------------------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger  | Schedule Trigger            | Initiates workflow daily at 10:00 AM  | None                             | Bleeping Computer, CISA GOV, HNRSS, Feedburner, Ars Technica, Techcrunch | **Schedule** — Set the Cron time/timezone for when you want the daily brief.                        |
| Bleeping Computer | RSS Feed Read               | Fetch news from Bleeping Computer RSS | Schedule Trigger                 | Merge                    | **Edit Feeds** — Open each RSS node and paste your preferred free sources (CISA, BleepingComputer, etc.). |
| CISA GOV          | RSS Feed Read               | Fetch news from CISA cybersecurity RSS| Schedule Trigger                 | Merge                    | (See above)                                                                                        |
| HNRSS             | RSS Feed Read               | Fetch filtered Hacker News frontpage  | Schedule Trigger                 | Merge                    | (See above)                                                                                        |
| Feedburner        | RSS Feed Read               | Fetch news from SecurityWeek RSS       | Schedule Trigger                 | Merge                    | (See above)                                                                                        |
| Ars Technica      | RSS Feed Read               | Fetch news from Ars Technica security feed | Schedule Trigger             | Merge                    | (See above)                                                                                        |
| Techcrunch        | RSS Feed Read               | Fetch news from Techcrunch security feed | Schedule Trigger               | Merge                    | (See above)                                                                                        |
| Merge             | Merge                      | Combine all feed items into one list  | Bleeping Computer, CISA GOV, HNRSS, Feedburner, Ars Technica, Techcrunch | Remove Duplicates        |                                                                                                    |
| Remove Duplicates | Remove Duplicates           | Remove duplicate stories by link      | Merge                           | If                       |                                                                                                    |
| If                | If Condition               | Filter stories published last 24h     | Remove Duplicates               | Limit (true branch), No New Items (false branch) |                                                                                                    |
| Limit             | Limit                      | Limit stories to max 25                | If                             | Code1                    |                                                                                                    |
| No New Items      | Code                       | Generate fallback message if no stories| If                            | Code1                    |                                                                                                    |
| Code1             | Code                       | Format stories into standardized array| Limit, No New Items             | Message a model           |                                                                                                    |
| Message a model   | OpenAI GPT-4o-mini (LangChain) | Summarize and categorize stories via AI | Code1                         | Code2                    | **Delivery** — Pick a lightweight OpenAI model; add optional dedupe to avoid repeat links.         |
| Code2             | Code                       | Parse AI response JSON safely         | Message a model                | Send a message            | (See above)                                                                                        |
| Send a message    | Gmail                      | Send email with daily brief            | Code2                         | None                     | (See above)                                                                                        |
| Sticky Note       | Sticky Note                | Instruction to edit feeds              | None                          | None                     | **Edit Feeds** — Open each RSS node and paste your preferred free sources (CISA, BleepingComputer, etc.). |
| Sticky Note1      | Sticky Note                | Instruction about schedule             | None                          | None                     | **Schedule** — Set the Cron time/timezone for when you want the daily brief.                        |
| Sticky Note2      | Sticky Note                | Instruction about delivery             | None                          | None                     | **Delivery** — Update the Gmail \"To\" address and pick a lightweight OpenAI model; add optional second dedupe to avoid repeat links. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set Cron schedule to trigger daily at 10:00 AM (adjust timezone as needed).

3. **Add six RSS Feed Read nodes, one for each source:**  
   - Bleeping Computer: https://www.bleepingcomputer.com/feed/  
   - CISA GOV: https://www.cisa.gov/cybersecurity-advisories/all.xml  
   - HNRSS (Hacker News): https://hnrss.org/frontpage?points=150 (filters stories with 150+ points)  
   - Feedburner (SecurityWeek): https://feeds.feedburner.com/securityweek  
   - Ars Technica Security: https://arstechnica.com/security/feed/  
   - Techcrunch Security: https://techcrunch.com/category/security/feed/  
   - For each, set SSL verification as needed (usually enabled, except for those with invalid certs).  
   - Connect the Schedule Trigger node to each RSS Feed node.

4. **Add a Merge node:**  
   - Set mode to Append.  
   - Set number of inputs to 6.  
   - Connect each RSS Feed node to one input of the Merge node.

5. **Add a Remove Duplicates node:**  
   - Configure to remove duplicates by comparing the `link` field.  
   - Connect Merge node output to this node.

6. **Add an If node to filter recent stories:**  
   - Condition: Evaluate if publication date is within last 24 hours:  
     `new Date($json.isoDate || $json.pubDate || $json.date || 0) > new Date(Date.now() - 24*60*60*1000)`  
   - Connect Remove Duplicates node output to the If node.

7. **Add a Limit node:**  
   - Set max items to 25.  
   - Connect the True output of the If node to the Limit node.

8. **Add a Code node named "No New Items":**  
   - Paste JavaScript to detect empty input and output a fallback message, else pass input unchanged:  
     ```js
     if (items.length === 0) {
       return [{
         json: {
           empty: true,
           subject: `Daily Cyber & Tech Brief — No major updates`,
           html: `<p>No notable headlines in the last 24h from your sources.</p>`,
           text: `No notable headlines in the last 24h.`
         }
       }];
     }
     return items;
     ```  
   - Connect the False output of the If node to "No New Items" node.  
   - Connect "No New Items" node output and Limit node output into a single Code node (next step).

9. **Add a Code node named "Code1":**  
   - Purpose: Format all input items into a single JSON array of stories with fields: source, title, link, published, snippet.  
   - Use the provided JavaScript code:  
     ```js
     const stories = $input.all().map(i => {
       const j = i.json;
       const link = j.link || '';
       let host = 'unknown';
       try { host = link ? new URL(link).hostname : 'unknown'; } catch(e) {}
     
       return {
         source: j.source || j.feed || host,
         title: j.title || '',
         link,
         published: j.isoDate || j.pubDate || j.date || null,
         snippet: j.contentSnippet || j.description || '',
       };
     });
     return [{ json: { stories } }];
     ```  
   - Connect outputs of Limit and No New Items nodes to this Code1 node (merge inputs).

10. **Add an OpenAI (LangChain) node named "Message a model":**  
    - Credentials: Configure OpenAI API credentials.  
    - Model: Select `gpt-4o-mini` (or equivalent lightweight GPT-4o model).  
    - Max tokens: 1600.  
    - Messages setup:  
      - System message:  
        ```
        You are a security editor writing a crisp daily brief for a VP of Cybersecurity.
        Group stories into: Vulnerabilities, Breaches/Ransomware, Cloud/SaaS, Policy/Regulation, Startups/Funding, Research.
        For each story: 1–2 sentence summary + one “Why it matters”.
        Extract CVEs/vendors if present. Be concise, no hype.
        Output STRICT JSON with keys: subject, html, text. Do not include code fences.
        ```  
      - User message:  
        ```
        Today's items (JSON array):
        {{ JSON.stringify($json.stories) }}
        ```  
    - Connect Code1 node output to this node.

11. **Add a Code node named "Code2":**  
    - Purpose: Parse the AI output content string to JSON safely, stripping any markdown fences.  
    - Use the provided JavaScript code:  
      ```js
      let c = $json?.message?.content ?? '';
      if (typeof c !== 'string') c = String(c || '');
      c = c.replace(/^```(?:json)?\s*/i, '').replace(/\s*```$/,'');
      let out;
      try {
        out = JSON.parse(c);
      } catch (e) {
        out = {
          subject: 'Daily Cyber & Tech Brief (parse issue)',
          html: `<pre>${c.replace(/[<>&]/g, s => ({'<':'&lt;','>':'&gt;','&':'&amp;'}[s]))}</pre>`,
          text: c
        };
      }
      return [{ json: out }];
      ```  
    - Connect output of "Message a model" node to this node.

12. **Add a Gmail node named "Send a message":**  
    - Credentials: Configure Gmail OAuth2 credentials.  
    - Parameters:  
      - Send To: enter recipient email address (e.g., your email).  
      - Subject: `={{ $json.subject || 'Daily Cyber & Tech Brief' }}`  
      - Message: `={{ $json.html || '<p>(empty)</p>' }}`  
    - Connect Code2 output to this node.

13. **Optional:** Add sticky notes with instructions on editing feeds, scheduling, and delivery settings for user guidance.

14. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                   |
|------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Use lightweight OpenAI models like GPT-4o-mini to reduce cost and increase speed while maintaining quality summaries.  | Delivery block recommendation                                    |
| The Hacker News RSS feed is filtered to only include stories with 150+ points to ensure relevance and popularity.      | hnRSS node configuration                                         |
| For best results, update RSS sources periodically to reflect trusted and preferred cybersecurity news outlets.          | Sticky Note instructions for feeds                               |
| Scheduling time can be adjusted to match recipient’s timezone or preferred reading time.                               | Sticky Note about scheduling                                     |
| Gmail node requires OAuth2 credentials with appropriate scopes to send emails on your behalf.                          | Gmail OAuth2 credential setup                                    |
| The workflow includes error handling for empty news days with a friendly fallback email.                               | No New Items code node                                            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.