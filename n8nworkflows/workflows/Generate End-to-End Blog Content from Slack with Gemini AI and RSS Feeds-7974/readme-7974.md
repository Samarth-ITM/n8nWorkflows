Generate End-to-End Blog Content from Slack with Gemini AI and RSS Feeds

https://n8nworkflows.xyz/workflows/generate-end-to-end-blog-content-from-slack-with-gemini-ai-and-rss-feeds-7974


# Generate End-to-End Blog Content from Slack with Gemini AI and RSS Feeds

### 1. Workflow Overview

This workflow automates the generation of end-to-end blog content focused on the hotel and hospitality industry by integrating Slack commands, RSS news feeds, and Google Gemini AI (PaLM) language models. It is designed to work interactively via Slack, allowing users to trigger content generation workflows through simple commands and replies.

The workflow is logically divided into these blocks:

- **1.1 Slack Command Reception and Parsing**: Listens for incoming messages in a specific Slack channel, parses commands and user inputs, and routes processing based on command type.
- **1.2 RSS Feed Aggregation and Normalization**: Reads multiple hospitality-related RSS feeds, merges and deduplicates article headlines and snippets into a compact format suitable for AI input.
- **1.3 Topic Clustering with Gemini AI**: Sends aggregated headlines to Google Gemini AI to cluster them into relevant trending topics, returning paraphrased topic headlines.
- **1.4 Slack Topic Interaction**: Posts the clustered topic list back into Slack and waits for the user to select a topic by number.
- **1.5 Topic Selection and Draft Generation**: Fetches Slack thread replies to identify the user’s topic selection, then prompts Google Gemini AI to generate a concise, LinkedIn-style blog post draft based on the chosen topic and recent headlines.
- **1.6 Draft Posting**: Posts the generated draft back into the Slack thread for user review.

---

### 2. Block-by-Block Analysis

#### 2.1 Slack Command Reception and Parsing

**Overview:**  
This block listens to Slack channel messages and normalizes incoming events into structured commands with parameters, filtering out bot/system messages. It routes commands to different workflow branches for further processing.

**Nodes Involved:**  
- Slack: Incoming messages  
- Code: parse_slack_command  
- Switch: route_by_command

**Node Details:**  

- **Slack: Incoming messages**  
  - *Type:* Slack Trigger  
  - *Role:* Entry point listening for new messages in a specific Slack channel (ID: `C09C3Q4QCF3`).  
  - *Configuration:* Trigger on “message” events, uses Slack Bot OAuth2 credentials.  
  - *Input/Output:* No input; outputs raw Slack event JSON.  
  - *Failure modes:* Auth errors if Slack token invalid; webhook issues; filtering may miss messages if channel ID changes.  
  - *Notes:* Webhook ID provided for Slack event subscription.

- **Code: parse_slack_command**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Normalize Slack event payload and parse commands (`start`, `stop`, `done`, `revise[...]`, `gen N`, or numeric picks) and extract relevant metadata (user, channel, thread timestamp).  
  - *Key expressions:* Regex parsing for commands, stripping @mentions, filtering out bot messages.  
  - *Input/Output:* Input raw Slack event JSON; outputs normalized command object with fields: `cmd`, `pick`, `notes`, `channel`, `ts`, `thread_ts`, `user`, `text`.  
  - *Failure modes:* Returns empty if event is not a user message or if parsing fails; guarded against missing event fields.

- **Switch: route_by_command**  
  - *Type:* Switch  
  - *Role:* Routes workflow based on `cmd` field to different processing branches:  
    - `start` → triggers RSS feed reads  
    - `gen` → triggers Slack thread reply fetch  
    - `revise` and `done` commands handled downstream (not detailed here)  
  - *Input/Output:* Input normalized command JSON; outputs to multiple branches.  
  - *Failure modes:* No default branch; unknown commands result in no output.

---

#### 2.2 RSS Feed Aggregation and Normalization

**Overview:**  
Reads multiple hospitality industry RSS feeds, merges them, deduplicates articles, and compiles a concise textual summary for AI ingestion.

**Nodes Involved:**  
- RSS Read2  
- RSS Read1  
- RSS Read  
- Merge: all_feeds  
- Code: make_headline_payload

**Node Details:**  

- **RSS Read2, RSS Read1, RSS Read**  
  - *Type:* RSS Feed Read nodes  
  - *Role:* Fetch latest articles from three distinct RSS sources:  
    - RSS Read2: hotel marketing topics from Google News (last 7 days)  
    - RSS Read1: hotel technology topics from Google News (last 7 days)  
    - RSS Read: Skift hospitality news feed  
  - *Configuration:* URLs specified, no special options.  
  - *Input/Output:* No input; outputs arrays of feed items.  
  - *Failure modes:* Network issues, invalid URLs, feed format changes.

- **Merge: all_feeds**  
  - *Type:* Merge  
  - *Role:* Combines the three RSS feed outputs into a single dataset appending items.  
  - *Input/Output:* Inputs from the three RSS nodes; outputs combined list.  
  - *Failure modes:* Mismatched input sizes or empty feeds.

- **Code: make_headline_payload**  
  - *Type:* Code node  
  - *Role:* Deduplicates merged feed items by normalized title, strips HTML from descriptions, truncates text, limits to max 60 articles, and compiles a numbered text list for AI prompt.  
  - *Key expressions:* Custom JS functions to strip HTML, truncate text, pick first existing field, deduplication by lowercase title.  
  - *Input/Output:* Input all merged feed items; outputs structured JSON with arrays: `articles` (objects with title, desc, link), `headlines` (titles only), `articleText` (numbered list string), and `articleCount`.  
  - *Failure modes:* Missing or malformed feed fields; empty input results in empty arrays.

---

#### 2.3 Topic Clustering with Gemini AI

**Overview:**  
Sends the compact article text to Google Gemini AI to cluster and paraphrase the top trending topics in the hospitality space.

**Nodes Involved:**  
- Gemini: cluster_to_topics  
- Code: parse_or_pass_topics

**Node Details:**  

- **Gemini: cluster_to_topics**  
  - *Type:* Google Gemini AI (LangChain node)  
  - *Role:* Uses Gemini 1.5 Flash model to analyze recent hospitality headlines and generate 10–30 paraphrased topic headlines as JSON.  
  - *Configuration:* Prompt instructs selecting diverse themes, paraphrasing headlines concisely without emojis or numbering, returning strict JSON with `topics` array.  
  - *Input/Output:* Input `articleText` from previous code node; outputs AI-generated JSON text.  
  - *Failure modes:* API key issues, rate limits, prompt parsing errors, malformed JSON from model.

- **Code: parse_or_pass_topics**  
  - *Type:* Code node  
  - *Role:* Parses the raw Gemini JSON output string, normalizes it into an array of objects `{ topic: string }`, deduplicates and caps topics. Handles errors gracefully.  
  - *Key expressions:* Regex to remove Markdown code fences, tries to parse JSON robustly, fallback extraction if needed.  
  - *Input/Output:* Input raw Gemini output; outputs normalized `topics` array with `mode: 'topics'`.  
  - *Failure modes:* Failure to parse malformed JSON; logs error but continues with empty topics.

---

#### 2.4 Slack Topic Interaction

**Overview:**  
Posts the list of clustered trending topics back into Slack for the user to review and select a topic number for content generation.

**Nodes Involved:**  
- Slack: post_topic_list

**Node Details:**  

- **Slack: post_topic_list**  
  - *Type:* Slack Post Message  
  - *Role:* Posts a formatted message listing up to 30 trending hotel topics with numbered lines, instructing the user to reply with a topic number or commands (`revise[...]`, `done`).  
  - *Configuration:* Posts in the same Slack channel and thread as the user’s original command, uses Slack Bot credentials.  
  - *Input/Output:* Input normalized topics from previous code node; outputs Slack post confirmation.  
  - *Failure modes:* Slack API errors, rate limiting, invalid channel or thread_ts.

---

#### 2.5 Topic Selection and Draft Generation

**Overview:**  
Fetches Slack thread replies to identify the user’s topic choice by number, resolves the chosen topic, and generates a LinkedIn-style blog post draft using Gemini AI.

**Nodes Involved:**  
- Slack: fetch_thread_replies  
- Code: pick_topic_from_thread  
- Gemini: write_draft

**Node Details:**  

- **Slack: fetch_thread_replies**  
  - *Type:* Slack API node (fetch replies)  
  - *Role:* Retrieves all messages in the Slack thread where the topic list was posted, including user replies with topic numbers.  
  - *Configuration:* Uses original message’s thread timestamp and channel ID.  
  - *Input/Output:* Input thread_ts from command parsing; outputs array of thread replies.  
  - *Failure modes:* Slack API errors, empty thread, permission issues.

- **Code: pick_topic_from_thread**  
  - *Type:* Code node  
  - *Role:* Parses user's numeric reply from the thread messages, matches it to the original posted topic list, validates selection, and outputs the chosen topic for draft generation.  
  - *Key expressions:* Parses numbers from Slack messages, searches thread for topic list marker, extracts topics from numbered list, clamps pick to valid range.  
  - *Input/Output:* Input Slack thread messages and user pick; outputs chosen topic text and metadata.  
  - *Failure modes:* No numeric pick found, unable to find topic list message, parsing errors, user reply outside valid range.

- **Gemini: write_draft**  
  - *Type:* Google Gemini AI (LangChain node)  
  - *Role:* Generates a 250–350 word LinkedIn-style post about the chosen topic, using recent headlines as context. The prompt instructs a strong hook, analytical tone, bullets for observations, and a soft CTA, with no hashtags or emojis.  
  - *Configuration:* Uses Gemini 1.5 Flash model, dynamically injects chosen topic and article text.  
  - *Input/Output:* Input topic and article text; outputs draft text content.  
  - *Failure modes:* API errors, long prompt truncation, malformed responses.

---

#### 2.6 Draft Posting

**Overview:**  
Posts the AI-generated blog post draft back into the original Slack thread for user review.

**Nodes Involved:**  
- Slack: post_draft

**Node Details:**  

- **Slack: post_draft**  
  - *Type:* Slack Post Message  
  - *Role:* Posts the generated draft text into the Slack thread where the conversation originated, allowing users to read and further interact.  
  - *Configuration:* Uses Slack Bot credentials, posts in the same channel and thread as the original command.  
  - *Input/Output:* Input draft text from Gemini; outputs Slack post confirmation.  
  - *Failure modes:* Slack API errors, rate limiting, missing thread_ts causing posts to the wrong context.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                           | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                   |
|----------------------------|----------------------------------|-----------------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| Slack: Incoming messages    | Slack Trigger                    | Entry point: listens for Slack messages | None                            | Code: parse_slack_command       |                                                                                                              |
| Code: parse_slack_command   | Code (JS)                       | Parse and normalize Slack commands      | Slack: Incoming messages        | Switch: route_by_command        |                                                                                                              |
| Switch: route_by_command    | Switch                          | Route workflow based on command          | Code: parse_slack_command       | RSS Read2, RSS Read1, RSS Read, Slack: fetch_thread_replies |                                                                                                              |
| RSS Read2                  | RSS Feed Read                   | Fetch hotel marketing RSS feed           | Switch: route_by_command        | Merge: all_feeds                |                                                                                                              |
| RSS Read1                  | RSS Feed Read                   | Fetch hotel technology RSS feed          | Switch: route_by_command        | Merge: all_feeds                |                                                                                                              |
| RSS Read                   | RSS Feed Read                   | Fetch Skift hospitality news feed        | Switch: route_by_command        | Merge: all_feeds                |                                                                                                              |
| Merge: all_feeds           | Merge                          | Combine RSS feed items into one list     | RSS Read2, RSS Read1, RSS Read  | Code: make_headline_payload     |                                                                                                              |
| Code: make_headline_payload | Code (JS)                      | Deduplicate and prepare article text for AI | Merge: all_feeds               | Gemini: cluster_to_topics       |                                                                                                              |
| Gemini: cluster_to_topics  | Google Gemini AI (LangChain)    | Cluster headlines into paraphrased topics | Code: make_headline_payload     | Code: parse_or_pass_topics      |                                                                                                              |
| Code: parse_or_pass_topics | Code (JS)                      | Parse and normalize Gemini JSON output   | Gemini: cluster_to_topics       | Slack: post_topic_list          |                                                                                                              |
| Slack: post_topic_list     | Slack Post Message             | Post trending topic list to Slack thread | Code: parse_or_pass_topics      | None                           | Posts up to 30 topics with instructions; replies should be topic number or commands.                          |
| Slack: fetch_thread_replies | Slack API (fetch replies)       | Fetch thread replies to identify topic pick | Switch: route_by_command (gen) | Code: pick_topic_from_thread   |                                                                                                              |
| Code: pick_topic_from_thread | Code (JS)                      | Parse user’s numeric topic pick from thread | Slack: fetch_thread_replies    | Gemini: write_draft            | Returns error if no pick or list message found in thread.                                                    |
| Gemini: write_draft        | Google Gemini AI (LangChain)    | Generate blog post draft on selected topic | Code: pick_topic_from_thread    | Slack: post_draft              |                                                                                                              |
| Slack: post_draft          | Slack Post Message             | Post generated draft back to Slack thread | Gemini: write_draft             | None                           | Posts draft text in the original Slack thread for review.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Slack Trigger node**  
   - Type: Slack Trigger  
   - Configuration: Trigger on "message" events in channel ID `C09C3Q4QCF3`  
   - Credentials: Slack Bot OAuth2 token  
   - Position: Entry point of workflow.

2. **Add Code node "parse_slack_command"**  
   - Type: Code (JavaScript)  
   - Paste provided JS to normalize Slack events, parse commands (`start`, `stop`, `done`, `revise[...]`, `gen N`, numeric picks).  
   - Connect Slack Trigger output to this code node.

3. **Add Switch node "route_by_command"**  
   - Type: Switch  
   - Configure rules on `$json.cmd` string:  
     - `start` → path 1  
     - `gen` → path 2  
     - `revise` → (not in this workflow’s JSON outputs)  
     - `done` → (not in this workflow’s JSON outputs)  
   - Connect Code node output to Switch input.

4. **Add three RSS Feed Read nodes**  
   - RSS Read2: URL = `https://news.google.com/rss/search?q=hotel+marketing+OR+hospitality+marketing+when:7d&hl=en-US&gl=US&ceid=US:en`  
   - RSS Read1: URL = `https://news.google.com/rss/search?q=hotel+technology+OR+hospitality+tech+when:7d&hl=en-US&gl=US&ceid=US:en`  
   - RSS Read: URL = `https://skift.com/feed/`  
   - Connect outputs to Merge node inputs as per order.

5. **Add Merge node "all_feeds"**  
   - Type: Merge  
   - Number of inputs: 3  
   - Mode: Append  
   - Connect RSS Read nodes to each input.

6. **Add Code node "make_headline_payload"**  
   - Paste JS code that deduplicates and compiles headlines, descriptions, and links into arrays and a text block.  
   - Connect Merge output to this node.

7. **Add Gemini AI node "cluster_to_topics"**  
   - Type: Google Gemini AI via LangChain  
   - Model: `models/gemini-1.5-flash`  
   - Prompt: instruct Gemini to cluster headlines into 10–30 paraphrased topics, strict JSON output.  
   - Connect `make_headline_payload` output to this node.  
   - Credentials: Google Palm API with valid API key.

8. **Add Code node "parse_or_pass_topics"**  
   - Paste provided JS to parse Gemini output JSON, normalize topics array, deduplicate.  
   - Connect Gemini output to this node.

9. **Add Slack Post Message node "post_topic_list"**  
   - Text: Format topics as numbered list with instructions to reply with a number or commands.  
   - Post to same channel and thread as original command (use thread_ts from parsed Slack command).  
   - Credentials: Slack Bot OAuth2.  
   - Connect Code parse_or_pass_topics output to this node.

10. **From Switch "route_by_command" gen branch, add Slack API node "fetch_thread_replies"**  
    - Operation: fetch replies in thread using thread_ts and channel from parsed Slack command.  
    - Credentials: Slack Bot OAuth2.

11. **Add Code node "pick_topic_from_thread"**  
    - Paste JS to parse user’s numeric reply in thread, identify topic selection from posted list.  
    - Connect fetch_thread_replies output to this node.

12. **Add Gemini AI node "write_draft"**  
    - Model: `models/gemini-1.5-flash`  
    - Prompt: Write a 250–350 word LinkedIn-style post about the chosen topic, with hooks, bullets, and CTA, using recent headlines as context.  
    - Credentials: Google Palm API.  
    - Connect pick_topic_from_thread output to this node.

13. **Add Slack Post Message node "post_draft"**  
    - Text: Use generated draft text from Gemini output.  
    - Post to same channel and thread as original command.  
    - Credentials: Slack Bot OAuth2.  
    - Connect Gemini write_draft output to this node.

14. **Activate workflow and test by posting commands in Slack channel:**  
    - `start` to fetch feeds and post topics  
    - Reply with number (e.g. `2`) to generate draft  
    - Use `revise[...]` or `done` commands as needed (additional handling may be required outside given nodes).

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                                                     |
|----------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates hotel industry blog content generation with Gemini AI and Slack interaction                 | Workflow title: "Generate End-to-End Blog Content from Slack with Gemini AI and RSS Feeds"                                         |
| Uses Google Gemini 1.5 Flash model for clustering and content generation                                       | Gemini AI nodes configured with `models/gemini-1.5-flash` model ID                                                                 |
| Slack bot requires OAuth2 credentials with permissions to read channel messages and post messages in threads   | Slack Bot credentials needed with webhook subscriptions for message events                                                         |
| Handles user commands: `start`, `gen N`, `revise[...]`, and `done` (though revise/done commands require further implementation outside this workflow) | Command parsing logic in Code: parse_slack_command node                                                                            |
| RSS feeds used are public hospitality industry news sources (Google News and Skift)                            | RSS URLs configured in RSS Read nodes                                                                                            |
| Workflow includes robust error handling in code nodes to manage malformed inputs and AI JSON parsing           | See Code nodes for defensive programming and error logging                                                                         |

---

This documentation fully describes the workflow logic, nodes, configurations, and integration points enabling reproduction, modification, and error anticipation for both human operators and AI agents.