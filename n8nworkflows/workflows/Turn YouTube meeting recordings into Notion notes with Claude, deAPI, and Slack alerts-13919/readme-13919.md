Turn YouTube meeting recordings into Notion notes with Claude, deAPI, and Slack alerts

https://n8nworkflows.xyz/workflows/turn-youtube-meeting-recordings-into-notion-notes-with-claude--deapi--and-slack-alerts-13919


# Turn YouTube meeting recordings into Notion notes with Claude, deAPI, and Slack alerts

# 1. Workflow Overview

This workflow automatically converts newly uploaded YouTube meeting recordings into structured meeting notes. It watches a YouTube channel’s RSS feed, sends each new video URL to deAPI for transcription, extracts transcript text, uses an AI Agent powered by Anthropic Claude to generate structured notes, stores those notes in a Notion database, and finally posts a summary with action items to Slack.

Typical use cases include recurring team meetings, project syncs, remote collaboration records, and content repurposing from recorded calls. The workflow is especially useful when teams already publish recordings to YouTube and want fast, consistent documentation without manual note-taking.

## 1.1 Input Reception and Upload Detection

The workflow starts by polling a YouTube RSS feed for newly published videos. This avoids the need for the YouTube API and works with public RSS endpoints derived from a channel ID.

## 1.2 Video Transcription

Once a new RSS item is detected, the workflow sends the YouTube video URL directly to deAPI. deAPI handles transcription using Whisper Large V3 and returns transcript content as binary data.

## 1.3 Transcript Extraction and AI Structuring

The binary transcript is converted into plain text. That text, together with the video title from the RSS feed, is passed to an AI Agent configured with a system prompt and a structured output parser so the result is returned in a predictable JSON-like schema.

## 1.4 Persistence in Notion

The structured output is mapped into a Notion database page. The workflow assumes a specific set of Notion properties exists for title, summary, action items, decisions, and key topics.

## 1.5 Slack Notification

After the Notion page is created, a Slack message is posted to a configured channel. The message includes the generated title, summary, and action items.

## 1.6 Documentation and In-Canvas Guidance

Several sticky notes are included in the workflow canvas. These provide setup guidance, links to documentation, channel RSS instructions, and node-level context.

---

# 2. Block-by-Block Analysis

## Block 1 — Workflow Documentation and Setup Guidance

### Overview
This block contains only sticky notes. These do not affect execution but are important for understanding the workflow’s purpose, requirements, and configuration expectations.

### Nodes Involved
- Sticky Note - Overview
- Sticky Note - Example
- Sticky Note - Trigger
- Sticky Note - Transcribe
- Sticky Note - Extract
- Sticky Note - AI Agent
- Sticky Note - Notion
- Sticky Note - Slack

### Node Details

#### Sticky Note - Overview
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual documentation node.
- **Configuration choices:** Large overview note summarizing the full workflow, requirements, and support resources.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None during execution; purely visual.
- **Sub-workflow reference:** None.

#### Sticky Note - Example
- **Type and technical role:** Sticky note; provides instructions for finding a YouTube Channel RSS URL.
- **Configuration choices:** Includes example RSS feed format and steps to obtain a Channel ID.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** User may still supply an incorrect channel ID.
- **Sub-workflow reference:** None.

#### Sticky Note - Trigger
- **Type and technical role:** Sticky note; explains the RSS trigger purpose and links to n8n docs.
- **Configuration choices:** Emphasizes setting the Feed URL and that no YouTube API key is needed.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note - Transcribe
- **Type and technical role:** Sticky note; documents deAPI transcription behavior.
- **Configuration choices:** Notes use of Whisper Large V3 and transcript delivery via binary/result URL.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Requires community node `n8n-nodes-deapi`.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note - Extract
- **Type and technical role:** Sticky note; explains the file-to-text extraction step.
- **Configuration choices:** References Extract From File documentation.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note - AI Agent
- **Type and technical role:** Sticky note; describes AI extraction responsibilities.
- **Configuration choices:** Explains expected output categories and mentions structured parsing.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Requires n8n AI/langchain nodes.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note - Notion
- **Type and technical role:** Sticky note; explains Notion persistence behavior.
- **Configuration choices:** Indicates the need to configure a Database ID and map fields.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Compatible with Notion node version shown in workflow.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

#### Sticky Note - Slack
- **Type and technical role:** Sticky note; explains Slack alert behavior.
- **Configuration choices:** Notes that Slack can be swapped for other notification systems.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Requires Slack OAuth2 credentials.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.

---

## Block 2 — YouTube Upload Detection

### Overview
This block detects new YouTube uploads by polling a channel RSS feed. It is the workflow’s entry point and supplies metadata such as the video title and link.

### Nodes Involved
- YouTube RSS Trigger

### Node Details

#### YouTube RSS Trigger
- **Type and technical role:** `n8n-nodes-base.rssFeedReadTrigger`; polling trigger node.
- **Configuration choices:**  
  - Feed URL is set to a YouTube videos RSS endpoint using a channel ID placeholder.  
  - Polling is configured to run every minute.
- **Key expressions or variables used:**  
  - `feedUrl = https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID`
- **Input and output connections:**  
  - No input; this is an entry point.  
  - Outputs to **deAPI Transcribe Video**.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - Invalid or missing channel ID results in feed retrieval failure or empty results.  
  - Private/unavailable feed content may not appear in RSS.  
  - Very frequent polling may be unnecessary depending on use case.  
  - Duplicate handling depends on trigger behavior/state persistence in n8n.
- **Sub-workflow reference:** None.

---

## Block 3 — Video Transcription

### Overview
This block sends the YouTube video URL to deAPI for transcription. It avoids file download and relies on deAPI’s hosted transcription capability.

### Nodes Involved
- deAPI Transcribe Video

### Node Details

#### deAPI Transcribe Video
- **Type and technical role:** `n8n-nodes-deapi.deapi`; community node for deAPI transcription.
- **Configuration choices:**  
  - Resource: `video`  
  - Operation: `transcribe`  
  - Video URL is taken from the RSS item’s `link` field.  
  - Wait timeout is set to 600 seconds, so the node can wait for deAPI processing to complete.
- **Key expressions or variables used:**  
  - `videoUrl = {{ $json.link }}`
- **Input and output connections:**  
  - Input from **YouTube RSS Trigger**  
  - Output to **Extract Transcript Text**
- **Version-specific requirements:**  
  - Requires installation of the community node `n8n-nodes-deapi`.  
  - Requires self-hosted n8n according to the workflow description.  
  - Requires deAPI credentials with API key and webhook secret.
- **Edge cases or potential failure types:**  
  - Invalid deAPI credentials or webhook secret mismatch.  
  - YouTube URL inaccessible to deAPI.  
  - Processing timeout if transcription exceeds 600 seconds.  
  - HTTPS requirement for callbacks/webhooks on the n8n instance.  
  - Changes in deAPI response structure could affect downstream extraction.
- **Sub-workflow reference:** None.

---

## Block 4 — Transcript Text Extraction

### Overview
This block converts deAPI’s returned transcript file content into usable plain text. That text becomes the source material for AI analysis.

### Nodes Involved
- Extract Transcript Text

### Node Details

#### Extract Transcript Text
- **Type and technical role:** `n8n-nodes-base.extractFromFile`; converts file/binary content into text.
- **Configuration choices:**  
  - Operation is set to `text`.  
  - No extra options are configured.
- **Key expressions or variables used:** None explicitly in parameters.
- **Input and output connections:**  
  - Input from **deAPI Transcribe Video**  
  - Output to **AI Agent**
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - If deAPI does not return expected binary data, extraction fails.  
  - Unsupported binary format or corrupted transcript file.  
  - Large transcript content may produce long prompts downstream.
- **Sub-workflow reference:** None.

---

## Block 5 — AI Analysis and Structured Note Generation

### Overview
This block performs the core intelligence of the workflow. The transcript and original video title are passed to an AI Agent, which relies on an Anthropic chat model and a structured output parser to generate consistent meeting notes fields.

### Nodes Involved
- AI Agent
- Anthropic Chat Model
- Structured Output Parser

### Node Details

#### AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates LLM prompting and structured output handling.
- **Configuration choices:**  
  - Prompt type is set to `define`.  
  - Main input text includes:
    - a fixed instruction line
    - the original YouTube title from the trigger node
    - the extracted transcript text from the current item
  - A system message defines the expected meeting note extraction tasks.
  - Structured output is enabled through `hasOutputParser: true`.
- **Key expressions or variables used:**  
  - `{{ $('YouTube RSS Trigger').item.json.title }}` for the YouTube video title  
  - `{{ $json.data }}` for the extracted transcript text
- **Input and output connections:**  
  - Main input from **Extract Transcript Text**  
  - AI language model input from **Anthropic Chat Model**  
  - AI output parser input from **Structured Output Parser**  
  - Main output to **Notion**
- **Version-specific requirements:** Type version 1.7; requires n8n AI/langchain support.
- **Edge cases or potential failure types:**  
  - Prompt may exceed model token limits for very long transcripts.  
  - Model may still return incomplete or low-quality extraction despite parser guidance.  
  - Expression failure if upstream transcript field is missing.  
  - Referencing the trigger node by name means renaming that node later may break expressions if not updated.
- **Sub-workflow reference:** None.

#### Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides the LLM backend.
- **Configuration choices:**  
  - Model selected: `claude-opus-4-6`
  - No additional model options are set.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Outputs via AI language model connection to **AI Agent**
- **Version-specific requirements:**  
  - Type version 1.3  
  - Requires Anthropic API credentials.
- **Edge cases or potential failure types:**  
  - Invalid API key or account quota issues.  
  - Model availability changes by Anthropic.  
  - Cost can increase for long transcripts and premium models like Opus.
- **Sub-workflow reference:** None.

#### Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; constrains model output to a defined JSON schema example.
- **Configuration choices:**  
  - A schema example defines these fields:
    - `title`
    - `summary`
    - `action_items`
    - `decisions`
    - `key_topics`
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Outputs via AI output parser connection to **AI Agent**
- **Version-specific requirements:** Type version 1.2.
- **Edge cases or potential failure types:**  
  - If the model output does not conform closely enough, parsing may fail.  
  - The schema is example-based, so behavior depends on parser implementation and model compliance.  
  - Missing fields may break downstream mappings in Notion or Slack.
- **Sub-workflow reference:** None.

---

## Block 6 — Notion Persistence

### Overview
This block creates a new page in a Notion database using the structured AI output. It assumes the target database has matching property names and compatible property types.

### Nodes Involved
- Notion

### Node Details

#### Notion
- **Type and technical role:** `n8n-nodes-base.notion`; creates a database page in Notion.
- **Configuration choices:**  
  - Resource: `databasePage`  
  - Database ID is configured manually.  
  - The page title and database properties are mapped from `AI Agent` output.
  - Properties configured:
    - `Title|title`
    - `Summary|rich_text`
    - `Action Items|rich_text`
    - `Decisions|rich_text`
    - `Key Topics|rich_text`
- **Key expressions or variables used:**  
  - `{{ $json.output.title }}`
  - `{{ $json.output.summary }}`
  - `{{ $json.output.action_items }}`
  - `{{ $json.output.decisions }}`
  - `{{ $json.output.key_topics }}`
- **Input and output connections:**  
  - Input from **AI Agent**  
  - Output to **Slack**
- **Version-specific requirements:** Type version 2.2; requires Notion credentials and a pre-existing database.
- **Edge cases or potential failure types:**  
  - Database ID invalid or inaccessible.  
  - Property names/types in Notion do not match the configured mappings exactly.  
  - Notion integration not shared with the database.  
  - Empty AI fields may produce incomplete pages.
- **Sub-workflow reference:** None.

---

## Block 7 — Slack Notification

### Overview
This block sends a Slack message summarizing the meeting and listing action items. It uses the AI-generated data, not the Notion page content itself.

### Nodes Involved
- Slack

### Node Details

#### Slack
- **Type and technical role:** `n8n-nodes-base.slack`; posts a message to a Slack channel.
- **Configuration choices:**  
  - Authentication uses OAuth2.  
  - Message is sent to a specific channel ID.  
  - The text body contains:
    - memo emoji
    - AI-generated title
    - summary
    - action items
- **Key expressions or variables used:**  
  - `{{ $('AI Agent').item.json.output.title }}`
  - `{{ $('AI Agent').item.json.output.summary }}`
  - `{{ $('AI Agent').item.json.output.action_items }}`
- **Input and output connections:**  
  - Input from **Notion**  
  - No downstream output
- **Version-specific requirements:** Type version 2.2; requires Slack OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Invalid or expired Slack OAuth token.  
  - Bot lacks permission to post to target channel.  
  - Channel ID incorrect.  
  - If AI output is missing fields, message text may be incomplete or malformed.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | ## Try It Out! Automatically turn YouTube recordings into structured notes, action items, and Slack summaries. This workflow monitors a YouTube channel via RSS for new uploads, transcribes the video, uses AI to extract key topics, action items, and decisions, creates a Notion page, and posts a summary to Slack. |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | Requirements: [deAPI](https://deapi.ai) account for video transcription |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | Requirements: Anthropic account for AI Agent |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | Requirements: Notion workspace with a meeting notes database |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | Requirements: Slack workspace |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | n8n instance must be on **HTTPS** |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | Need Help? [n8n Discord](https://discord.gg/n8n) |
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual overview and setup guidance |  |  | Need Help? [Forum](https://community.n8n.io/) |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Visual instruction for RSS URL discovery |  |  | ### How to get your YouTube Channel RSS URL |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Visual instruction for RSS URL discovery |  |  | `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID` |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Visual instruction for RSS URL discovery |  |  | To find your **Channel ID**: Go to the YouTube channel page → **More** → **Share channel** → **Copy channel ID** |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Visual instruction for RSS URL discovery |  |  | Or find it in the channel URL: `youtube.com/channel/UCxxxxxx` |
| Sticky Note - Example | n8n-nodes-base.stickyNote | Visual instruction for RSS URL discovery |  |  | No YouTube API key needed — RSS feeds are public. |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual documentation for trigger block |  |  | ## 1. RSS Feed Trigger |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual documentation for trigger block |  |  | [Read more about RSS Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.rssfeedread) |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual documentation for trigger block |  |  | Polls a YouTube channel's RSS feed for new video uploads. |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual documentation for trigger block |  |  | Set the **Feed URL** to your channel's RSS URL. |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual documentation for trigger block |  |  | No file size limits — deAPI transcribes directly from the YouTube URL. |
| Sticky Note - Transcribe | n8n-nodes-base.stickyNote | Visual documentation for transcription block |  |  | ## 2. Transcribe Video |
| Sticky Note - Transcribe | n8n-nodes-base.stickyNote | Visual documentation for transcription block |  |  | [deAPI Documentation](https://docs.deapi.ai) |
| Sticky Note - Transcribe | n8n-nodes-base.stickyNote | Visual documentation for transcription block |  |  | Uses **Whisper Large V3** to transcribe the video directly from the YouTube URL. |
| Sticky Note - Transcribe | n8n-nodes-base.stickyNote | Visual documentation for transcription block |  |  | Returns the transcript as binary data via `result_url`. |
| Sticky Note - Extract | n8n-nodes-base.stickyNote | Visual documentation for extraction block |  |  | ## 3. Extract Transcript Text |
| Sticky Note - Extract | n8n-nodes-base.stickyNote | Visual documentation for extraction block |  |  | [Read more about Extract From File](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.extractfromfile) |
| Sticky Note - Extract | n8n-nodes-base.stickyNote | Visual documentation for extraction block |  |  | Converts the binary transcript from deAPI into a text string. |
| Sticky Note - Extract | n8n-nodes-base.stickyNote | Visual documentation for extraction block |  |  | The extracted text is passed to the AI Agent for analysis. |
| Sticky Note - AI Agent | n8n-nodes-base.stickyNote | Visual documentation for AI block |  |  | ## 4. AI-Powered Note Extraction |
| Sticky Note - AI Agent | n8n-nodes-base.stickyNote | Visual documentation for AI block |  |  | [Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Sticky Note - AI Agent | n8n-nodes-base.stickyNote | Visual documentation for AI block |  |  | The AI Agent analyzes the transcript and extracts: Summary, Action items, Decisions, Key topics |
| Sticky Note - AI Agent | n8n-nodes-base.stickyNote | Visual documentation for AI block |  |  | The **Structured Output Parser** ensures consistent JSON output. |
| Sticky Note - Notion | n8n-nodes-base.stickyNote | Visual documentation for Notion block |  |  | ## 5. Save to Notion |
| Sticky Note - Notion | n8n-nodes-base.stickyNote | Visual documentation for Notion block |  |  | [Read more about Notion node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion) |
| Sticky Note - Notion | n8n-nodes-base.stickyNote | Visual documentation for Notion block |  |  | Creates a new page in your Notion meeting notes database. |
| Sticky Note - Notion | n8n-nodes-base.stickyNote | Visual documentation for Notion block |  |  | Configure the **Database ID** and map the output fields to your database properties. |
| Sticky Note - Slack | n8n-nodes-base.stickyNote | Visual documentation for Slack block |  |  | ## 6. Notify via Slack |
| Sticky Note - Slack | n8n-nodes-base.stickyNote | Visual documentation for Slack block |  |  | [Read more about Slack node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack) |
| Sticky Note - Slack | n8n-nodes-base.stickyNote | Visual documentation for Slack block |  |  | Posts a summary with action items to your team's Slack channel. |
| Sticky Note - Slack | n8n-nodes-base.stickyNote | Visual documentation for Slack block |  |  | Swap for Microsoft Teams, email, or any other notification node. |
| YouTube RSS Trigger | n8n-nodes-base.rssFeedReadTrigger | Poll YouTube channel RSS for new uploads |  | deAPI Transcribe Video | ## 1. RSS Feed Trigger |
| YouTube RSS Trigger | n8n-nodes-base.rssFeedReadTrigger | Poll YouTube channel RSS for new uploads |  | deAPI Transcribe Video | [Read more about RSS Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.rssfeedread) |
| YouTube RSS Trigger | n8n-nodes-base.rssFeedReadTrigger | Poll YouTube channel RSS for new uploads |  | deAPI Transcribe Video | Polls a YouTube channel's RSS feed for new video uploads. |
| YouTube RSS Trigger | n8n-nodes-base.rssFeedReadTrigger | Poll YouTube channel RSS for new uploads |  | deAPI Transcribe Video | Set the **Feed URL** to your channel's RSS URL. |
| YouTube RSS Trigger | n8n-nodes-base.rssFeedReadTrigger | Poll YouTube channel RSS for new uploads |  | deAPI Transcribe Video | No file size limits — deAPI transcribes directly from the YouTube URL. |
| deAPI Transcribe Video | n8n-nodes-deapi.deapi | Transcribe YouTube video via deAPI | YouTube RSS Trigger | Extract Transcript Text | ## 2. Transcribe Video |
| deAPI Transcribe Video | n8n-nodes-deapi.deapi | Transcribe YouTube video via deAPI | YouTube RSS Trigger | Extract Transcript Text | [deAPI Documentation](https://docs.deapi.ai) |
| deAPI Transcribe Video | n8n-nodes-deapi.deapi | Transcribe YouTube video via deAPI | YouTube RSS Trigger | Extract Transcript Text | Uses **Whisper Large V3** to transcribe the video directly from the YouTube URL. |
| deAPI Transcribe Video | n8n-nodes-deapi.deapi | Transcribe YouTube video via deAPI | YouTube RSS Trigger | Extract Transcript Text | Returns the transcript as binary data via `result_url`. |
| Extract Transcript Text | n8n-nodes-base.extractFromFile | Convert transcript binary to text | deAPI Transcribe Video | AI Agent | ## 3. Extract Transcript Text |
| Extract Transcript Text | n8n-nodes-base.extractFromFile | Convert transcript binary to text | deAPI Transcribe Video | AI Agent | [Read more about Extract From File](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.extractfromfile) |
| Extract Transcript Text | n8n-nodes-base.extractFromFile | Convert transcript binary to text | deAPI Transcribe Video | AI Agent | Converts the binary transcript from deAPI into a text string. |
| Extract Transcript Text | n8n-nodes-base.extractFromFile | Convert transcript binary to text | deAPI Transcribe Video | AI Agent | The extracted text is passed to the AI Agent for analysis. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate structured meeting notes from transcript | Extract Transcript Text; Anthropic Chat Model; Structured Output Parser | Notion | ## 4. AI-Powered Note Extraction |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate structured meeting notes from transcript | Extract Transcript Text; Anthropic Chat Model; Structured Output Parser | Notion | [Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate structured meeting notes from transcript | Extract Transcript Text; Anthropic Chat Model; Structured Output Parser | Notion | The AI Agent analyzes the transcript and extracts: Summary, Action items, Decisions, Key topics |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate structured meeting notes from transcript | Extract Transcript Text; Anthropic Chat Model; Structured Output Parser | Notion | The **Structured Output Parser** ensures consistent JSON output. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured AI output schema |  | AI Agent | ## 4. AI-Powered Note Extraction |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured AI output schema |  | AI Agent | [Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured AI output schema |  | AI Agent | The AI Agent analyzes the transcript and extracts: Summary, Action items, Decisions, Key topics |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce structured AI output schema |  | AI Agent | The **Structured Output Parser** ensures consistent JSON output. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for AI Agent |  | AI Agent | ## 4. AI-Powered Note Extraction |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for AI Agent |  | AI Agent | [Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent) |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for AI Agent |  | AI Agent | The AI Agent analyzes the transcript and extracts: Summary, Action items, Decisions, Key topics |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backend for AI Agent |  | AI Agent | The **Structured Output Parser** ensures consistent JSON output. |
| Notion | n8n-nodes-base.notion | Create meeting notes page in Notion database | AI Agent | Slack | ## 5. Save to Notion |
| Notion | n8n-nodes-base.notion | Create meeting notes page in Notion database | AI Agent | Slack | [Read more about Notion node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion) |
| Notion | n8n-nodes-base.notion | Create meeting notes page in Notion database | AI Agent | Slack | Creates a new page in your Notion meeting notes database. |
| Notion | n8n-nodes-base.notion | Create meeting notes page in Notion database | AI Agent | Slack | Configure the **Database ID** and map the output fields to your database properties. |
| Slack | n8n-nodes-base.slack | Post meeting summary and action items to Slack | Notion |  | ## 6. Notify via Slack |
| Slack | n8n-nodes-base.slack | Post meeting summary and action items to Slack | Notion |  | [Read more about Slack node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack) |
| Slack | n8n-nodes-base.slack | Post meeting summary and action items to Slack | Notion |  | Posts a summary with action items to your team's Slack channel. |
| Slack | n8n-nodes-base.slack | Post meeting summary and action items to Slack | Notion |  | Swap for Microsoft Teams, email, or any other notification node. |

---

# 4. Reproducing the Workflow from Scratch

1. **Prepare your environment**
   - Use a self-hosted n8n instance.
   - Ensure the n8n instance is reachable over HTTPS.
   - Create accounts and credentials for:
     - deAPI
     - Anthropic
     - Notion
     - Slack

2. **Install the deAPI community node**
   - In n8n, go to **Settings → Community Nodes**.
   - Install `n8n-nodes-deapi`.

3. **Create a new workflow**
   - Name it something like **Meeting Notes Automation**.

4. **Create the trigger node**
   - Add an **RSS Feed Trigger** node.
   - Name it **YouTube RSS Trigger**.
   - Set **Feed URL** to:
     - `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID`
   - Set polling to **Every Minute**.
   - This node is the workflow entry point.

5. **Create the transcription node**
   - Add the **deAPI** node from the installed community package.
   - Name it **deAPI Transcribe Video**.
   - Configure:
     - **Resource:** `video`
     - **Operation:** `transcribe`
     - **Video URL:** `{{ $json.link }}`
     - **Wait Timeout:** `600`
   - Create/select deAPI credentials.
   - Connect **YouTube RSS Trigger → deAPI Transcribe Video**.

6. **Create the text extraction node**
   - Add an **Extract From File** node.
   - Name it **Extract Transcript Text**.
   - Set **Operation** to `text`.
   - Leave default options unless your deAPI output requires adjustment.
   - Connect **deAPI Transcribe Video → Extract Transcript Text**.

7. **Create the AI Agent node**
   - Add an **AI Agent** node.
   - Name it **AI Agent**.
   - Set **Prompt Type** to `Define`.
   - In the main text/prompt field, use:
     - `Here is a transcript of a meeting recording. Please extract structured meeting notes.`
     - Include the video title with `{{ $('YouTube RSS Trigger').item.json.title }}`
     - Include the transcript with `{{ $json.data }}`
   - Enable structured output by turning on the output parser option.
   - In the **System Message**, add instructions equivalent to:
     - Write a concise title
     - Write a 3–5 sentence summary
     - Extract action items with responsible person if mentioned
     - List decisions
     - List key topics
     - Return results in the required JSON format
   - Connect **Extract Transcript Text → AI Agent**.

8. **Create the Anthropic model node**
   - Add an **Anthropic Chat Model** node.
   - Name it **Anthropic Chat Model**.
   - Select the model **claude-opus-4-6**.
   - Add Anthropic API credentials.
   - Connect this node to the **AI Agent** using the **AI Language Model** connection, not the main connection.

9. **Create the structured parser node**
   - Add a **Structured Output Parser** node.
   - Name it **Structured Output Parser**.
   - Provide a schema example with the following fields:
     - `title`
     - `summary`
     - `action_items`
     - `decisions`
     - `key_topics`
   - Example values can be realistic meeting-note strings.
   - Connect this node to the **AI Agent** using the **AI Output Parser** connection.

10. **Create the Notion node**
    - Add a **Notion** node.
    - Name it **Notion**.
    - Configure:
      - **Resource:** Database Page
      - **Database ID:** your target Notion database
    - Add Notion credentials.
    - Map these properties:
      - `Title|title` → `{{ $json.output.title }}`
      - `Summary|rich_text` → `{{ $json.output.summary }}`
      - `Action Items|rich_text` → `{{ $json.output.action_items }}`
      - `Decisions|rich_text` → `{{ $json.output.decisions }}`
      - `Key Topics|rich_text` → `{{ $json.output.key_topics }}`
    - Optionally also set the node title field to `{{ $json.output.title }}`.
    - Connect **AI Agent → Notion**.

11. **Prepare the Notion database**
    - In Notion, create a database for meeting notes.
    - Ensure it contains these properties with compatible types:
      - **Title** → Title
      - **Summary** → Rich text
      - **Action Items** → Rich text
      - **Decisions** → Rich text
      - **Key Topics** → Rich text
    - Share the database with your Notion integration.

12. **Create the Slack node**
    - Add a **Slack** node.
    - Name it **Slack**.
    - Set authentication to **OAuth2**.
    - Choose **Channel** as the destination type.
    - Select the target **Channel ID**.
    - Add Slack OAuth2 credentials.
    - Use a message body equivalent to:
      - `:memo: *{{ $('AI Agent').item.json.output.title }}*`
      - `*Summary:* {{ $('AI Agent').item.json.output.summary }}`
      - `*Action Items:* {{ $('AI Agent').item.json.output.action_items }}`
    - Connect **Notion → Slack**.

13. **Optionally add canvas documentation**
    - Add sticky notes for:
      - overall workflow purpose
      - RSS URL instructions
      - deAPI docs
      - AI node explanation
      - Notion and Slack guidance

14. **Test the RSS trigger**
    - Use a valid YouTube channel RSS URL.
    - Confirm the trigger outputs fields including at least:
      - `title`
      - `link`

15. **Test the transcription step**
    - Execute the deAPI node with a valid RSS item.
    - Confirm transcript data is returned in a format that the Extract From File node can read.

16. **Test transcript extraction**
    - Execute **Extract Transcript Text**.
    - Confirm the output includes plain transcript text, typically in a field such as `data`.

17. **Test AI structuring**
    - Execute the AI Agent path.
    - Confirm the result is structured under `output` with:
      - `title`
      - `summary`
      - `action_items`
      - `decisions`
      - `key_topics`

18. **Test Notion insertion**
    - Confirm the page is created successfully.
    - If it fails, verify:
      - database access
      - property names
      - property types

19. **Test Slack delivery**
    - Confirm the message posts to the selected channel.
    - If it fails, verify:
      - Slack OAuth scopes
      - channel permissions
      - correct channel ID

20. **Activate the workflow**
    - After successful end-to-end testing, activate the workflow.
    - The trigger will then poll the feed every minute for new uploads.

### Credential Configuration Summary

- **deAPI credentials**
  - API key
  - webhook secret
  - HTTPS n8n instance required for callback-based operations

- **Anthropic credentials**
  - Anthropic API key
  - ensure access to the selected Claude model

- **Notion credentials**
  - Notion internal integration or OAuth credentials supported by the node
  - integration must have access to the database

- **Slack credentials**
  - Slack OAuth2
  - app must be authorized to post messages in the chosen channel

### Sub-workflow Setup
This workflow does **not** use sub-workflows and does not invoke any other workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| deAPI account required for transcription | https://deapi.ai |
| deAPI documentation | https://docs.deapi.ai |
| n8n Discord for help | https://discord.gg/n8n |
| n8n community forum for help | https://community.n8n.io/ |
| RSS Trigger documentation | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.rssfeedread |
| Extract From File documentation | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.extractfromfile |
| AI Agent documentation | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent |
| Notion node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.notion |
| Slack node documentation | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack |
| YouTube RSS feed format | `https://www.youtube.com/feeds/videos.xml?channel_id=YOUR_CHANNEL_ID` |
| No YouTube API key is needed because the workflow uses public RSS feeds | Workflow design note |
| deAPI node is noted as unavailable on n8n Cloud in the supplied description | Deployment constraint |
| The workflow is inactive by default in the provided JSON | Operational note |
| Workflow setting `executionOrder` is `v1` | Runtime configuration note |