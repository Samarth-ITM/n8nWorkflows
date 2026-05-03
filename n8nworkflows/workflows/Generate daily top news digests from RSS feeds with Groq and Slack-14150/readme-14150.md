Generate daily top news digests from RSS feeds with Groq and Slack

https://n8nworkflows.xyz/workflows/generate-daily-top-news-digests-from-rss-feeds-with-groq-and-slack-14150


# Generate daily top news digests from RSS feeds with Groq and Slack

## 1. Workflow Overview

This workflow generates a daily AI-curated news digest from multiple RSS feeds and posts it to Slack. It is designed for teams that want a concise morning summary of globally important stories without manually reviewing several news sources.

The workflow is organized into three functional blocks:

### 1.1 Trigger & RSS Feed Preparation
The workflow starts on a daily schedule, defines a fixed set of RSS feed URLs, and transforms them into a list that can be processed one by one.

### 1.2 RSS Article Collection & Aggregation
Each RSS feed is read in sequence. The workflow keeps only the latest 10 items per source, then consolidates all selected articles into one payload for downstream AI processing.

### 1.3 AI Selection & Slack Delivery
An AI agent powered by a Groq chat model receives the aggregated article list, selects the most globally impactful stories, and returns structured output. That output is parsed into individual items and formatted into a Slack message containing titles, summaries, and links.

---

## 2. Block-by-Block Analysis

## 2.1 Trigger & RSS Feed Preparation

**Overview:**  
This block launches the workflow every morning and prepares the RSS source list. It converts a single object of feed categories into multiple items so downstream nodes can iterate through them.

**Nodes Involved:**  
- Schedule Trigger2  
- Define News Categories1  
- Prepare RSS Feed List1

### Node: Schedule Trigger2
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger` — time-based entry point.
- **Configuration choices:** Configured to run daily at hour `7`.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none, entry node  
  - Output: Define News Categories1
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**  
  - Time zone behavior depends on the workflow or instance time zone settings.
  - If the server is offline at trigger time, execution may be skipped depending on deployment setup.
- **Sub-workflow reference:** None.

### Node: Define News Categories1
- **Type and technical role:** `n8n-nodes-base.set` — creates static workflow data.
- **Configuration choices:** Defines five string fields containing RSS URLs:
  - `Global_Top_News` → BBC News
  - `Global_Business_News` → BBC Business
  - `Finance_News` → Bloomberg Markets
  - `Technological_News` → TechCrunch
  - `Tech_Trend_News` → The Verge
- **Key expressions or variables used:** Static values only.
- **Input and output connections:**  
  - Input: Schedule Trigger2  
  - Output: Prepare RSS Feed List1
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - Invalid or unreachable feed URLs will not fail here, but later during RSS fetch.
  - If a feed is removed or changes format, downstream parsing may fail.
- **Sub-workflow reference:** None.

### Node: Prepare RSS Feed List1
- **Type and technical role:** `n8n-nodes-base.code` — reshapes the feed object into one item per RSS URL.
- **Configuration choices:** Uses JavaScript to read the first input item and convert all object entries into an array of items shaped like:
  - `{ json: { Rss_Feed: value } }`
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `Object.entries(data)`
- **Input and output connections:**  
  - Input: Define News Categories1  
  - Output: Loop Through RSS Feeds1
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**  
  - If the incoming item is empty, `$input.first()` may be undefined.
  - Any non-string feed values would still be emitted and could fail in the RSS reader.
- **Sub-workflow reference:** None.

---

## 2.2 RSS Article Collection & Aggregation

**Overview:**  
This block iterates over each RSS source, fetches articles, sorts them by publication date, keeps the latest 10 entries, and collects all selected articles into a single combined dataset.

**Nodes Involved:**  
- Loop Through RSS Feeds1  
- Fetch RSS Articles1  
- Filter Latest 10 Articles1  
- Merge All RSS Articles1

### Node: Loop Through RSS Feeds1
- **Type and technical role:** `n8n-nodes-base.splitInBatches` — batch/loop controller.
- **Configuration choices:** Uses default batching behavior with no extra options configured.
- **Key expressions or variables used:** None directly.
- **Input and output connections:**  
  - Input: Prepare RSS Feed List1 and loop-back from Filter Latest 10 Articles1  
  - Outputs:
    - To Fetch RSS Articles1 for per-feed processing
    - To Merge All RSS Articles1 after loop completion
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Misunderstanding loop behavior is common; this node depends on the feedback connection from Filter Latest 10 Articles1.
  - If one iteration fails, later feeds may not be processed.
- **Sub-workflow reference:** None.

### Node: Fetch RSS Articles1
- **Type and technical role:** `n8n-nodes-base.rssFeedRead` — fetches and parses RSS/Atom feeds.
- **Configuration choices:** Reads the URL dynamically from:
  - `={{ $json.Rss_Feed }}`
- **Key expressions or variables used:**  
  - `$json.Rss_Feed`
- **Input and output connections:**  
  - Input: Loop Through RSS Feeds1  
  - Output: Filter Latest 10 Articles1
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - HTTP/network errors
  - Invalid XML or unsupported feed structure
  - Rate limiting or temporary source downtime
  - Missing expected fields such as `isoDate`, `title`, or `link`
- **Sub-workflow reference:** None.

### Node: Filter Latest 10 Articles1
- **Type and technical role:** `n8n-nodes-base.code` — sorts feed items and truncates the list.
- **Configuration choices:**  
  - Reads all incoming items with `$input.all()`
  - Sorts descending by `json.isoDate`
  - Returns only the first 10 items
- **Key expressions or variables used:**  
  - `$input.all()`
  - `new Date(b.json.isoDate).getTime()`
- **Input and output connections:**  
  - Input: Fetch RSS Articles1  
  - Output: Loop Through RSS Feeds1 (loop continuation)
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**  
  - If `isoDate` is missing or malformed, `Date` parsing may produce `NaN` and sorting becomes unreliable.
  - If fewer than 10 articles are present, it simply returns all available items.
- **Sub-workflow reference:** None.

### Node: Merge All RSS Articles1
- **Type and technical role:** `n8n-nodes-base.code` — consolidates all processed feed items into a single articles array.
- **Configuration choices:** Maps each item into a reduced structure containing:
  - `title`
  - `link`
  - `pubDate`
  Then returns one item:
  - `{ json: { articles } }`
- **Key expressions or variables used:**  
  - `$input.all()`
- **Input and output connections:**  
  - Input: Loop Through RSS Feeds1
  - Output: AI Agent2
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**  
  - If upstream items lack `title`, `link`, or `pubDate`, the AI receives incomplete data.
  - This node does not deduplicate articles, so duplicate stories from multiple sources may be included.
- **Sub-workflow reference:** None.

---

## 2.3 AI Selection & Slack Delivery

**Overview:**  
This block sends the aggregated article list to an AI agent backed by Groq, expects a JSON array in the AI response, parses that array into separate items, and composes a final Slack digest message.

**Nodes Involved:**  
- Groq Chat Model2  
- AI Agent2  
- Format News For Slack Message1  
- Post News Digest to Slack1

### Node: Groq Chat Model2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq` — LLM connector used by the AI agent.
- **Configuration choices:** Uses model:
  - `llama-3.3-70b-versatile`
  No advanced options are set.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output via AI language model connection to AI Agent2
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - Missing or invalid Groq credentials
  - API quota or rate-limit issues
  - Model availability changes
  - Response formatting inconsistency if the agent prompt is not strict enough
- **Sub-workflow reference:** None.

### Node: AI Agent2
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent` — orchestrates AI reasoning using the connected Groq model.
- **Configuration choices:** No explicit prompt or options are shown in the JSON beyond default options. It receives article data from the main input and a language model from the AI connector.
- **Key expressions or variables used:** Implicitly consumes the incoming `articles` payload from Merge All RSS Articles1.
- **Input and output connections:**  
  - Main input: Merge All RSS Articles1  
  - AI model input: Groq Chat Model2  
  - Output: Format News For Slack Message1
- **Version-specific requirements:** Type version `3`.
- **Edge cases or potential failure types:**  
  - Because no visible system/user prompt is configured here, output quality and structure may depend on defaults or hidden UI state.
  - If the model returns natural language instead of a JSON array, the next node will fail.
  - Long article lists may increase token usage or exceed model input constraints.
- **Sub-workflow reference:** None.

### Node: Format News For Slack Message1
- **Type and technical role:** `n8n-nodes-base.code` — extracts JSON array content from AI text output and emits one item per selected article.
- **Configuration choices:**  
  - Reads `const text = $json.output`
  - Searches for a JSON array using regex: `/\[[\s\S]*\]/`
  - Throws an error if no array is found
  - Parses the array and returns one item per article
- **Key expressions or variables used:**  
  - `$json.output`
  - `text.match(/\[[\s\S]*\]/)`
  - `JSON.parse(match[0])`
- **Input and output connections:**  
  - Input: AI Agent2  
  - Output: Post News Digest to Slack1
- **Version-specific requirements:** Code node type version `2`.
- **Edge cases or potential failure types:**  
  - If AI output is not valid JSON, `JSON.parse` fails.
  - If the AI wraps JSON with extra brackets elsewhere in the text, regex may capture invalid content.
  - The node assumes each article contains fields later used by Slack: `title`, `Description`, `url`.
- **Sub-workflow reference:** None.

### Node: Post News Digest to Slack1
- **Type and technical role:** `n8n-nodes-base.slack` — posts the final digest to a Slack channel.
- **Configuration choices:**  
  - Authentication: OAuth2
  - Operation: send message to selected channel
  - Channel is configured via `channelId`, but the current JSON shows an empty selected value
  - Message text is generated through a JavaScript expression that:
    - Creates today’s date in `DD-MM-YYYY`
    - Builds a digest header
    - Iterates over all incoming items
    - For each item includes:
      - `title`
      - `Description`
      - linked `url`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `item.json.title`
  - `item.json.Description`
  - `item.json.url`
- **Input and output connections:**  
  - Input: Format News For Slack Message1  
  - Output: none
- **Version-specific requirements:** Type version `2.4`.
- **Edge cases or potential failure types:**  
  - Slack OAuth2 credential missing or expired
  - Empty `channelId` means the node is not fully configured and will not post until a channel is selected
  - If article objects do not include `Description` or `url`, the message may contain blank sections
  - Long digests may exceed Slack message length limits
- **Sub-workflow reference:** None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger2 | Schedule Trigger | Daily entry point at 7 AM |  | Define News Categories1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Schedule Trigger2 | Schedule Trigger | Daily entry point at 7 AM |  | Define News Categories1 | # Step 1 – Workflow Trigger & RSS Feed Preparation Runs every morning and prepares the list of RSS feeds that will be used to collect news articles from multiple categories. |
| Define News Categories1 | Set | Defines static RSS source URLs | Schedule Trigger2 | Prepare RSS Feed List1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Define News Categories1 | Set | Defines static RSS source URLs | Schedule Trigger2 | Prepare RSS Feed List1 | # Step 1 – Workflow Trigger & RSS Feed Preparation Runs every morning and prepares the list of RSS feeds that will be used to collect news articles from multiple categories. |
| Prepare RSS Feed List1 | Code | Converts feed object into one item per RSS URL | Define News Categories1 | Loop Through RSS Feeds1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Prepare RSS Feed List1 | Code | Converts feed object into one item per RSS URL | Define News Categories1 | Loop Through RSS Feeds1 | # Step 1 – Workflow Trigger & RSS Feed Preparation Runs every morning and prepares the list of RSS feeds that will be used to collect news articles from multiple categories. |
| Loop Through RSS Feeds1 | Split In Batches | Iterates through RSS feeds | Prepare RSS Feed List1, Filter Latest 10 Articles1 | Fetch RSS Articles1, Merge All RSS Articles1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Loop Through RSS Feeds1 | Split In Batches | Iterates through RSS feeds | Prepare RSS Feed List1, Filter Latest 10 Articles1 | Fetch RSS Articles1, Merge All RSS Articles1 | # Step 2 – RSS Article Collection & Processing Loops through each RSS feed, fetches the latest articles, filters recent ones, and merges them into a single dataset. |
| Fetch RSS Articles1 | RSS Feed Read | Fetches articles from each RSS source | Loop Through RSS Feeds1 | Filter Latest 10 Articles1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Fetch RSS Articles1 | RSS Feed Read | Fetches articles from each RSS source | Loop Through RSS Feeds1 | Filter Latest 10 Articles1 | # Step 2 – RSS Article Collection & Processing Loops through each RSS feed, fetches the latest articles, filters recent ones, and merges them into a single dataset. |
| Filter Latest 10 Articles1 | Code | Sorts by publication date and keeps latest 10 | Fetch RSS Articles1 | Loop Through RSS Feeds1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Filter Latest 10 Articles1 | Code | Sorts by publication date and keeps latest 10 | Fetch RSS Articles1 | Loop Through RSS Feeds1 | # Step 2 – RSS Article Collection & Processing Loops through each RSS feed, fetches the latest articles, filters recent ones, and merges them into a single dataset. |
| Merge All RSS Articles1 | Code | Aggregates all selected feed items into one articles array | Loop Through RSS Feeds1 | AI Agent2 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Merge All RSS Articles1 | Code | Aggregates all selected feed items into one articles array | Loop Through RSS Feeds1 | AI Agent2 | # Step 2 – RSS Article Collection & Processing Loops through each RSS feed, fetches the latest articles, filters recent ones, and merges them into a single dataset. |
| Groq Chat Model2 | Groq Chat Model | Provides LLM to AI Agent |  | AI Agent2 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Groq Chat Model2 | Groq Chat Model | Provides LLM to AI Agent |  | AI Agent2 | # Step 3 – AI News Selection & Slack Delivery AI analyzes the collected articles, selects the most globally impactful stories, and posts the final news digest to Slack. |
| AI Agent2 | AI Agent | Selects important stories from aggregated articles | Merge All RSS Articles1, Groq Chat Model2 | Format News For Slack Message1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| AI Agent2 | AI Agent | Selects important stories from aggregated articles | Merge All RSS Articles1, Groq Chat Model2 | Format News For Slack Message1 | # Step 3 – AI News Selection & Slack Delivery AI analyzes the collected articles, selects the most globally impactful stories, and posts the final news digest to Slack. |
| Format News For Slack Message1 | Code | Parses AI JSON output into individual digest items | AI Agent2 | Post News Digest to Slack1 | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Format News For Slack Message1 | Code | Parses AI JSON output into individual digest items | AI Agent2 | Post News Digest to Slack1 | # Step 3 – AI News Selection & Slack Delivery AI analyzes the collected articles, selects the most globally impactful stories, and posts the final news digest to Slack. |
| Post News Digest to Slack1 | Slack | Posts final digest message to Slack channel | Format News For Slack Message1 |  | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Post News Digest to Slack1 | Slack | Posts final digest message to Slack channel | Format News For Slack Message1 |  | # Step 3 – AI News Selection & Slack Delivery AI analyzes the collected articles, selects the most globally impactful stories, and posts the final news digest to Slack. |
| Sticky Note7 | Sticky Note | Canvas documentation |  |  | # Daily RSS News Digest to Slack This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. |
| Sticky Note8 | Sticky Note | Canvas documentation for block 1 |  |  | # Step 1 – Workflow Trigger & RSS Feed Preparation Runs every morning and prepares the list of RSS feeds that will be used to collect news articles from multiple categories. |
| Sticky Note9 | Sticky Note | Canvas documentation for block 2 |  |  | # Step 2 – RSS Article Collection & Processing Loops through each RSS feed, fetches the latest articles, filters recent ones, and merges them into a single dataset. |
| Sticky Note10 | Sticky Note | Canvas documentation for block 3 |  |  | # Step 3 – AI News Selection & Slack Delivery AI analyzes the collected articles, selects the most globally impactful stories, and posts the final news digest to Slack. |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   `Generate daily top news digests from RSS feeds with Groq and Slack`.

2. **Add a Schedule Trigger node** named `Schedule Trigger2`.
   - Node type: `Schedule Trigger`
   - Configure it to run every day at hour `7`
   - Confirm your instance time zone is correct

3. **Add a Set node** named `Define News Categories1`.
   - Connect `Schedule Trigger2 -> Define News Categories1`
   - Create five string fields:
     - `Global_Top_News` = `http://feeds.bbci.co.uk/news/rss.xml`
     - `Global_Business_News` = `http://feeds.bbci.co.uk/news/business/rss.xml`
     - `Finance_News` = `https://feeds.bloomberg.com/markets/news.rss`
     - `Technological_News` = `https://techcrunch.com/feed/`
     - `Tech_Trend_News` = `https://www.theverge.com/rss/index.xml`

4. **Add a Code node** named `Prepare RSS Feed List1`.
   - Connect `Define News Categories1 -> Prepare RSS Feed List1`
   - Paste this logic:
     - Read the first input item
     - Convert each key/value pair into one item containing `Rss_Feed`
   - Equivalent code:
     ```javascript
     const data = $input.first().json;

     const result = Object.entries(data).map(([key, value]) => {
       return {
         json: {
           Rss_Feed: value
         }
       };
     });

     return result;
     ```

5. **Add a Split In Batches node** named `Loop Through RSS Feeds1`.
   - Connect `Prepare RSS Feed List1 -> Loop Through RSS Feeds1`
   - Keep default options unless you want custom batch sizing
   - This node will control iteration over the feed list

6. **Add an RSS Feed Read node** named `Fetch RSS Articles1`.
   - Connect `Loop Through RSS Feeds1 -> Fetch RSS Articles1`
   - Set URL to expression:
     ```javascript
     {{ $json.Rss_Feed }}
     ```
   - Leave other options at default unless a feed requires customization

7. **Add a Code node** named `Filter Latest 10 Articles1`.
   - Connect `Fetch RSS Articles1 -> Filter Latest 10 Articles1`
   - Use this code:
     ```javascript
     const allItems = $input.all();

     allItems.sort((a, b) => {
       return new Date(b.json.isoDate).getTime() - new Date(a.json.isoDate).getTime();
     });

     return allItems.slice(0, 10);
     ```

8. **Create the loop-back connection** from `Filter Latest 10 Articles1 -> Loop Through RSS Feeds1`.
   - This is essential
   - Without this connection, the workflow will not continue iterating over all feeds correctly

9. **Add a Code node** named `Merge All RSS Articles1`.
   - Connect the completion output of `Loop Through RSS Feeds1 -> Merge All RSS Articles1`
   - Use this code:
     ```javascript
     const articles = $input.all().map(item => ({
       title: item.json.title,
       link: item.json.link,
       pubDate: item.json.pubDate
     }));

     return [
       {
         json: {
           articles
         }
       }
     ];
     ```

10. **Add a Groq Chat Model node** named `Groq Chat Model2`.
    - Node type: `Groq Chat Model`
    - Set the model to:
      `llama-3.3-70b-versatile`
    - Create or select Groq credentials
    - Credential requirement:
      - A valid Groq API key stored in n8n credentials

11. **Add an AI Agent node** named `AI Agent2`.
    - Connect `Merge All RSS Articles1 -> AI Agent2`
    - Connect `Groq Chat Model2 -> AI Agent2` using the AI language model connection
    - Leave default options if you want to match the JSON exactly

12. **Important implementation note for AI Agent2:**  
    The exported JSON does not show a detailed prompt, but the next node expects the AI output to contain a JSON array. To make the workflow reliable, configure the agent with a clear instruction such as:
    - Review the provided articles
    - Select the most globally impactful stories
    - Return only a JSON array
    - Each item should contain:
      - `title`
      - `Description`
      - `url`

    Example expected structure:
    ```json
    [
      {
        "title": "Story title",
        "Description": "Short summary",
        "url": "https://example.com/article"
      }
    ]
    ```

13. **Add a Code node** named `Format News For Slack Message1`.
    - Connect `AI Agent2 -> Format News For Slack Message1`
    - Use this code:
      ```javascript
      const text = $json.output;

      const match = text.match(/\[[\s\S]*\]/);

      if (!match) {
        throw new Error("No JSON array found in AI output");
      }

      const articles = JSON.parse(match[0]);

      return articles.map(article => ({
        json: article
      }));
      ```

14. **Add a Slack node** named `Post News Digest to Slack1`.
    - Connect `Format News For Slack Message1 -> Post News Digest to Slack1`
    - Node type: `Slack`
    - Authentication: `OAuth2`
    - Operation: send a message to a channel
    - Select the destination channel in `channelId`
    - This field is empty in the provided workflow JSON, so you must choose a channel manually

15. **Configure Slack credentials**.
    - Create Slack OAuth2 credentials in n8n
    - Authorize the Slack workspace
    - Ensure the app has permission to post to the chosen channel
    - If required, invite the app to the target channel

16. **Set the Slack message expression** to build the digest.
    - Use an expression equivalent to:
      ```javascript
      {{
      (() => {
      const today = new Date();
      const formattedDate =
      String(today.getDate()).padStart(2,'0') + '-' +
      String(today.getMonth()+1).padStart(2,'0') + '-' +
      today.getFullYear();

      return `🌍 *Daily Global News Digest - ${formattedDate}*

      📰 Here are today's most important global news:

      ` + $input.all().map(item =>
      `🔹 *${item.json.title}*

      ${item.json.Description}

      <${item.json.url}|Read Full Article>`
        ).join('\n\n────────────────────────────────────────────────\n\n');
      })()
      }}
      ```

17. **Optionally add sticky notes** to match the original canvas structure:
    - One general note describing the overall workflow
    - One note for trigger/feed preparation
    - One note for RSS collection/processing
    - One note for AI selection/Slack delivery

18. **Test the workflow manually**.
    - Execute from the trigger or run downstream nodes manually
    - Confirm that:
      - RSS feeds return items
      - The loop processes all sources
      - The AI returns valid JSON array output
      - Slack receives the final digest

19. **Activate the workflow** once credentials and channel selection are complete.

### Credential setup summary
- **Groq credentials**
  - Needed by `Groq Chat Model2`
  - Must include a valid API key
- **Slack OAuth2 credentials**
  - Needed by `Post News Digest to Slack1`
  - Must have permission to post messages to the target channel

### Input/output expectations
- **Before AI Agent2:** one item containing `articles` array
- **After AI Agent2:** text output containing a JSON array
- **After Format News For Slack Message1:** one item per selected article with:
  - `title`
  - `Description`
  - `url`

### Constraints to preserve
- The AI output must be parseable JSON
- Slack message formatting assumes `Description` and `url` exist
- The loop-back connection is mandatory
- A Slack channel must be selected before production use

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Daily RSS News Digest to Slack: This workflow automatically collects global news from multiple RSS feeds, selects the most important stories using AI, and posts a clean daily news digest to a Slack channel. | Overall workflow purpose |
| How it works: runs every morning, prepares feeds, loops through RSS sources, sorts recent articles, merges them, uses AI to select impactful stories, formats them, and posts them to Slack. | Overall workflow behavior |
| Setup steps in the canvas note: configure Slack credentials and channel, modify RSS URLs if needed, configure AI model credentials, review AI prompt, activate the workflow. | Operational setup guidance |
| Step 1 – Workflow Trigger & RSS Feed Preparation: Runs every morning and prepares the list of RSS feeds that will be used to collect news articles from multiple categories. | Block 1 |
| Step 2 – RSS Article Collection & Processing: Loops through each RSS feed, fetches the latest articles, filters recent ones, and merges them into a single dataset. | Block 2 |
| Step 3 – AI News Selection & Slack Delivery: AI analyzes the collected articles, selects the most globally impactful stories, and posts the final news digest to Slack. | Block 3 |

### Additional implementation observations
- The current JSON shows no selected Slack channel value, so the Slack node is incomplete until a channel is chosen.
- The AI Agent node does not expose a visible prompt in this export, so reliability depends on manually defining strict output instructions.
- There is no deduplication step; the same story may appear multiple times if multiple feeds cover it.
- The workflow depends on RSS fields such as `isoDate`, `title`, `link`, and `pubDate`; feed schema changes may affect behavior.