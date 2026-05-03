Multi-Account Email Classifier with AI, Gmail, Discord & Google Sheets

https://n8nworkflows.xyz/workflows/multi-account-email-classifier-with-ai--gmail--discord---google-sheets-5789


# Multi-Account Email Classifier with AI, Gmail, Discord & Google Sheets

### 1. Workflow Overview

This workflow implements a **Multi-Account Email Classifier** that integrates Gmail, Discord, Google Sheets, and AI agents to automatically classify incoming emails across multiple Gmail accounts. It filters and categorizes emails based on spam detection and priority levels, updates a centralized spam list stored in Google Sheets, and sends classified email summaries to specific Discord channels for team awareness.

The workflow is logically divided into the following blocks:

- **1.1 Multi-Account Email Reception:** Four Gmail Trigger nodes listen for new emails on distinct Gmail accounts.
- **1.2 Email Metadata Enrichment:** Each incoming email is tagged with metadata including the Discord channel ID and sender email.
- **1.3 Merging Incoming Emails:** Incoming emails from all accounts are merged into a single stream for uniform processing.
- **1.4 AI-Based Email Classification:** An AI agent analyzes the email content, classifies it as Spam or assigns a priority (High, Medium, Low), and extracts a summary with key details.
- **1.5 Spam List Management:** The workflow retrieves and updates a spam list stored in Google Sheets to maintain an evolving email classification database.
- **1.6 Discord Notification:** Classified emails (except Spam) are forwarded to designated Discord channels with rich embeds summarizing the email.
- **1.7 Feedback Webhook & AI Agent:** A webhook receives user feedback from Discord, allowing users to manually classify emails which updates the spam list accordingly.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Multi-Account Email Reception

**Overview:**  
Four Gmail Trigger nodes independently monitor four Gmail accounts for new emails, polling every minute, providing continuous real-time email reception.

**Nodes Involved:**  
- Gmail Trigger (account1@gmail.com)  
- Gmail Trigger1 (account2@gmail.com)  
- Gmail Trigger3 (account3@gmail.com, duplicated in JSON)  
- Gmail Trigger2 (account4@gmail.com)  
- Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3 (labels for accounts)

**Node Details:**

- **Gmail Trigger, Gmail Trigger1, Gmail Trigger2, Gmail Trigger3**  
  - Type: Gmail Trigger  
  - Role: Listen for new emails on respective Gmail accounts  
  - Configuration: Poll every 1 minute, no filters applied (receives all new emails)  
  - Credentials: OAuth2 per Gmail account  
  - Output: New email data objects  
  - Edge Cases: Authentication token expiration, rate limits, empty inboxes, email format variations

- **Sticky Notes**  
  - Role: Visual labels identifying which Gmail account each trigger corresponds to  
  - Content: Email account addresses (e.g., account1@gmail.com)  
  - No execution role

---

#### 1.2 Email Metadata Enrichment

**Overview:**  
Each incoming email is enriched with metadata, specifically a Discord channel ID and the sender's email address, to route classified emails to appropriate Discord channels.

**Nodes Involved:**  
- medium7 (account1)  
- medium6 (account2)  
- medium4 (account3)  
- medium5 (account4)

**Node Details:**

- **mediumX (Set nodes)**  
  - Type: Set  
  - Role: Add two string fields: `discord_channel` (a hardcoded Discord channel ID per account) and `email` (the Gmail account address)  
  - Configuration: Assignments like `discord_channel = "1234567894"` and `email = "account1@gmail.com"`  
  - Input: Email object from respective Gmail Trigger node  
  - Output: Enhanced email object with metadata  
  - Edge Cases: Hardcoded channel IDs require manual updates if Discord channels change; missing or malformed emails may propagate downstream errors

---

#### 1.3 Merging Incoming Emails

**Overview:**  
The four enriched email streams are merged into a single stream to centralize email processing.

**Nodes Involved:**  
- Merge4

**Node Details:**

- **Merge4 (Merge node)**  
  - Type: Merge  
  - Role: Combine email objects from all Gmail accounts into a single output stream  
  - Configuration: Mode "combine" with "combineByPosition" to merge inputs by their position in the input array  
  - Inputs: Four connections from the four mediumX nodes  
  - Output: Unified email stream for downstream AI processing  
  - Edge Cases: If any input is empty or delayed, merge may wait, causing latency; ensure all inputs produce data to prevent blocking

---

#### 1.4 AI-Based Email Classification

**Overview:**  
An AI agent classifies each email as Spam or assigns a priority (High, Medium, Low), summarizes the email, and extracts URLs and images. It uses a spam list from Google Sheets to improve classification accuracy.

**Nodes Involved:**  
- AI Agent2  
- OpenAI Chat Model3  
- Get spam list1  
- Structured Output Parser1  
- If2  
- Send a message (Discord notification)

**Node Details:**

- **AI Agent2 (LangChain Agent)**  
  - Type: AI Agent (LangChain)  
  - Role: Process email content and metadata, classify email according to detailed rules, summarize content, and extract relevant URLs/images  
  - Configuration: Custom system message with detailed classification criteria and priority color mapping; accesses `getSpamList()` tool to retrieve current spam list  
  - Input: Merged email stream with enriched metadata  
  - Output: Structured JSON including classification priority, summary, URLs, and colors  
  - Edge Cases: AI model misclassification, API timeout, malformed email data, missing fields

- **OpenAI Chat Model3**  
  - Type: OpenAI Chat Model (gpt-4o-mini)  
  - Role: Language model used by AI Agent2 for classification and summarization  
  - Credentials: OpenAI API key with appropriate quota and permissions  
  - Edge Cases: API rate limits, network errors

- **Get spam list1**  
  - Type: Google Sheets Tool  
  - Role: Retrieve the spam list spreadsheet containing known spam emails and domains  
  - Configuration: Connects to a specific Google Sheets document and sheet via OAuth2  
  - Edge Cases: OAuth token expiration, spreadsheet access rights, empty or corrupted sheet data

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI Agent2’s raw output into a structured JSON object following a defined schema with fields like from, to, subject, summary, priority, image_url, and action_url  
  - Edge Cases: Parsing errors if AI output does not conform to schema, missing or extra fields

- **If2 (Conditional node)**  
  - Type: If  
  - Role: Filters out emails classified as Spam (no Discord notification sent for Spam)  
  - Condition: Output must not be empty and priority must not equal "Spam"  
  - Edge Cases: Incorrect classification logic may send spam; empty outputs block forwarding

- **Send a message (Discord)**  
  - Type: Discord node  
  - Role: Sends a rich embedded message to the Discord channel associated with the email’s source account  
  - Configuration: Embed includes title (subject), author (from), description (summary), color (priority color), image, and clickable URL  
  - Credentials: OAuth2 for Discord bot with posting permissions  
  - Edge Cases: Discord API rate limits, invalid channel IDs, malformed embeds

---

#### 1.5 Spam List Management

**Overview:**  
The workflow maintains a spam list in Google Sheets that is consulted during classification and updated based on user feedback from Discord.

**Nodes Involved:**  
- Get spam list (used by AI Agent1)  
- update spam list (Google Sheets Tool)  
- Google Sheets OAuth2 credentials shared between these nodes

**Node Details:**

- **Get spam list**  
  - Same as Get spam list1 but used by AI Agent1 (feedback agent)  
  - Role: Retrieve current spam list for classification and update logic

- **update spam list**  
  - Type: Google Sheets Tool (appendOrUpdate)  
  - Role: Add or update entries in the spam list sheet with new spam or legit classifications from user feedback  
  - Configuration: Matches by email column, updates or inserts with classification, domain, labelled_by ("n8n"), and labelled_date (today's date)  
  - Edge Cases: Sheet concurrency, update conflicts, malformed input data

---

#### 1.6 Feedback Webhook & AI Agent

**Overview:**  
A webhook collects manual email classification feedback from Discord users and invokes an AI agent to update the spam list accordingly.

**Nodes Involved:**  
- Webhook  
- AI Agent1  
- OpenAI Chat Model1  
- Get a message in Discord  
- Get spam list (for AI Agent1)  
- update spam list (for AI Agent1)  
- Discord - reply1

**Node Details:**

- **Webhook**  
  - Type: Webhook (POST)  
  - Role: Receives user feedback as JSON POST requests at path `/email-feedback`  
  - Edge Cases: Security (authentication missing?), malformed requests, replay attacks

- **AI Agent1 (LangChain Agent)**  
  - Role: Processes user input, fetches referenced Discord message if needed, retrieves spam list, classifies email as spam or legit, updates spam list in Google Sheets, and prepares a reply message  
  - Tools: `getDiscordMessage(reference)`, `getSpamList()`, `updateSpamList(columns)`  
  - Input: Text content and Discord message reference from webhook JSON body  
  - Output: Classification result and update confirmation message

- **OpenAI Chat Model1**  
  - Used by AI Agent1 for NLP processing and reasoning

- **Get a message in Discord**  
  - Fetches original Discord message referenced by user feedback to extract email and context for classification

- **Discord - reply1**  
  - Sends a reply message to the Discord channel where feedback was received, reporting the classification and update status

- **Get spam list** and **update spam list** nodes are reused here to maintain consistency in spam data management.

- Edge Cases: Discord API permissions, webhook security, invalid references, Google Sheets API failures

---

### 3. Summary Table

| Node Name              | Node Type                     | Functional Role                                         | Input Node(s)                                | Output Node(s)               | Sticky Note                                                                                   |
|------------------------|-------------------------------|--------------------------------------------------------|----------------------------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Gmail Trigger          | Gmail Trigger                 | Listen for new emails on account1@gmail.com            | None                                         | medium7, Merge4               | ## account1@gmail.com                                                                         |
| Gmail Trigger1         | Gmail Trigger                 | Listen for new emails on account2@gmail.com            | None                                         | medium6, Merge4               | ## account2@gmail.com                                                                         |
| Gmail Trigger2         | Gmail Trigger                 | Listen for new emails on account4@gmail.com            | None                                         | medium5, Merge4               | ## account4@gmail.com                                                                         |
| Gmail Trigger3         | Gmail Trigger                 | Listen for new emails on account3@gmail.com            | None                                         | medium4, Merge4               | ## account3@gmail.com                                                                         |
| medium7                | Set                           | Add Discord channel ID and email metadata (account1)   | Gmail Trigger                                | Merge4                       |                                                                                              |
| medium6                | Set                           | Add Discord channel ID and email metadata (account2)   | Gmail Trigger1                               | Merge4                       |                                                                                              |
| medium5                | Set                           | Add Discord channel ID and email metadata (account4)   | Gmail Trigger2                               | Merge4                       |                                                                                              |
| medium4                | Set                           | Add Discord channel ID and email metadata (account3)   | Gmail Trigger3                               | Merge4                       |                                                                                              |
| Merge4                 | Merge                         | Combine emails from all accounts                        | medium7, medium6, medium5, medium4           | AI Agent2                   |                                                                                              |
| AI Agent2              | LangChain AI Agent            | Classify and summarize incoming emails                  | Merge4                                       | If2                         |                                                                                              |
| OpenAI Chat Model3     | OpenAI Chat Model             | Provide NLP capabilities for AI Agent2                  | AI Agent2                                    | AI Agent2                   |                                                                                              |
| Get spam list1         | Google Sheets Tool            | Retrieve spam list for classification                   | AI Agent2 (ai_tool)                          | AI Agent2                   |                                                                                              |
| Structured Output Parser1 | LangChain Output Parser       | Parse AI Agent2 outputs into structured JSON            | AI Agent2                                    | If2                         |                                                                                              |
| If2                    | If                            | Filter out Spam emails from sending Discord messages    | AI Agent2                                    | Send a message              |                                                                                              |
| Send a message         | Discord                       | Post classified email summary to Discord channel        | If2                                          | None                        |                                                                                              |
| Webhook                | Webhook                       | Receive email classification feedback from Discord      | None                                         | AI Agent1                   |                                                                                              |
| AI Agent1              | LangChain AI Agent            | Process feedback, update spam list, reply to Discord    | Webhook                                      | Discord - reply1            |                                                                                              |
| OpenAI Chat Model1     | OpenAI Chat Model             | Provide NLP capabilities for AI Agent1                  | AI Agent1                                    | AI Agent1                   |                                                                                              |
| Get a message in Discord | Discord Tool                 | Retrieve referenced Discord message                      | AI Agent1                                    | AI Agent1                   |                                                                                              |
| Get spam list          | Google Sheets Tool            | Retrieve spam list for AI Agent1                         | AI Agent1 (ai_tool)                          | AI Agent1                   |                                                                                              |
| update spam list       | Google Sheets Tool            | Append or update spam classification in Google Sheets  | AI Agent1 (ai_tool)                          | AI Agent1                   |                                                                                              |
| Discord - reply1       | Discord                       | Send reply message with classification outcome          | AI Agent1                                    | None                        |                                                                                              |
| Sticky Note2           | Sticky Note                  | Label for account1@gmail.com                             | None                                         | None                        | ## account1@gmail.com                                                                         |
| Sticky Note            | Sticky Note                  | Label for account2@gmail.com                             | None                                         | None                        | ## account2@gmail.com                                                                         |
| Sticky Note3           | Sticky Note                  | Label for account3@gmail.com                             | None                                         | None                        | ## account3@gmail.com                                                                         |
| Sticky Note1           | Sticky Note                  | Label for account4@gmail.com                             | None                                         | None                        | ## account4@gmail.com                                                                         |
| Sticky Note4           | Sticky Note                  | Label for "Update spam list" section                     | None                                         | None                        | ## Update spam list                                                                           |
| Sticky Note5           | Sticky Note                  | Label for "Filter incoming message / Send important emails only" | None                                         | None                        | ## Filter incoming message\n### Send important emails only                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger nodes for each account:**  
   - Create 4 Gmail Trigger nodes, each configured with OAuth2 credentials for a distinct Gmail account (account1@gmail.com to account4@gmail.com).  
   - Set polling interval to every 1 minute, no filters.  

2. **Add Set nodes to enrich metadata:**  
   - For each Gmail Trigger node, add a Set node named `mediumX` (medium7 for account1, medium6 for account2, medium4 for account3, medium5 for account4).  
   - In each Set node, add two string fields:  
     - `discord_channel` with the respective Discord channel ID (e.g., "1234567894" for account1).  
     - `email` with the Gmail account email address (e.g., "account1@gmail.com").  
   - Connect respective Gmail Trigger outputs to these Set nodes.

3. **Merge enriched emails:**  
   - Add a Merge node named `Merge4`.  
   - Set mode to "combine" and combineBy "position".  
   - Connect all four `mediumX` nodes as inputs to `Merge4`.

4. **Configure AI Agent for email classification:**  
   - Add a LangChain Agent node named `AI Agent2`.  
   - Configure the system message with detailed classification logic (spam detection rules, priority levels, summary instructions, color codes).  
   - Use `getSpamList()` tool to fetch spam data (calls Google Sheets).  
   - Input expression uses fields from `Merge4` (email fields and content).  
   - Connect `Merge4` output to `AI Agent2` input.

5. **Add OpenAI Chat Model for AI Agent2:**  
   - Add OpenAI Chat Model node (`OpenAI Chat Model3`) with model `gpt-4o-mini`.  
   - Provide valid OpenAI API credentials.  
   - Connect this to `AI Agent2` as the language model provider.

6. **Add Google Sheets tool to get spam list:**  
   - Add Google Sheets Tool node (`Get spam list1`).  
   - Configure with the spam list Google Sheets document ID and sheet name (`gid=0`).  
   - Connect as AI tool input to `AI Agent2`.  
   - Provide Google Sheets OAuth2 credentials.

7. **Parse AI Agent2 output:**  
   - Add a LangChain Structured Output Parser (`Structured Output Parser1`).  
   - Provide a JSON schema example matching expected AI output fields.  
   - Connect AI Agent2 output to this parser.

8. **Filter spam emails:**  
   - Add an If node (`If2`).  
   - Condition: Output is not empty AND `priority` field is not "Spam".  
   - Connect parser output to `If2`.

9. **Send classified emails to Discord:**  
   - Add a Discord node (`Send a message`).  
   - Configure with OAuth2 credentials for your Discord bot.  
   - Set message embed content using fields from parsed AI output (summary, priority color, image URL, action URL).  
   - Use the `discord_channel` from the merged email metadata as the channel ID.  
   - Connect `If2` true branch to this node.

10. **Set up feedback webhook:**  
    - Add a Webhook node (`Webhook`) with POST method and path `/email-feedback`.  
    - No authentication configured here by default (consider adding).  

11. **Configure AI Agent1 for feedback processing:**  
    - Add LangChain Agent node (`AI Agent1`).  
    - System message: explains tools available (getDiscordMessage, getSpamList, updateSpamList) and behavior for classifying emails based on user input.  
    - Input expression uses webhook JSON body content and Discord message references.  
    - Connect Webhook output to `AI Agent1`.

12. **Add OpenAI Chat Model for AI Agent1:**  
    - Add OpenAI Chat Model node (`OpenAI Chat Model1`) using `gpt-4o-mini`.  
    - Connect to AI Agent1 as language model provider.

13. **Add Discord Tool to get referenced message:**  
    - Add Discord Tool node (`Get a message in Discord`).  
    - Configure with OAuth2 credentials and inputs from AI Agent1’s tool parameters to fetch original Discord message.

14. **Add Google Sheets tools for feedback agent:**  
    - Add `Get spam list` node to supply spam list to AI Agent1.  
    - Add `update spam list` node configured with appendOrUpdate operation on the spam list sheet.  
    - Map columns: email, domain, Classification, labelled_by ("n8n"), labelled_date (current date).  
    - Connect as AI tools inputs to AI Agent1.

15. **Send reply to Discord after feedback:**  
    - Add Discord node (`Discord - reply1`) with OAuth2 credentials.  
    - Configure to post reply to the same guild and channel as the feedback webhook, embedding AI Agent1’s output message.  
    - Connect AI Agent1 output to this node.

16. **Connect all nodes according to described flow:**  
    - Gmail Triggers → mediumX nodes → Merge4 → AI Agent2 → Structured Output Parser → If → Discord Send a message  
    - Webhook → AI Agent1 → Discord Reply  
    - AI Agents have their OpenAI and Google Sheets nodes connected as AI tools and language models accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses multiple Gmail accounts to enable multi-account email classification and routing to different Discord channels. | Core design principle of workflow.                                                              |
| The AI agents are based on LangChain with custom system prompts leveraging OpenAI’s gpt-4o-mini model for NLP tasks.     | Requires OpenAI API access with appropriate quotas and permissions.                              |
| Spam list management is centralized in a Google Sheets document enabling easy manual review and updates.                | Google Sheets document ID: `1iOYH829GJ-ytTlmz0Zsl875Efn1qyrwuv6Rx83N1QJU`                         |
| Discord integration requires a bot with OAuth2 credentials, capable of reading and posting messages in configured channels. | The Discord bot must have permissions for the specific guilds and channels used.                 |
| Webhook for feedback does not implement authentication; consider securing it with secrets or IP filtering in production.| Webhook path: `/email-feedback`                                                                  |
| Sticky notes are used solely for organization and visual clarity within the n8n editor.                                  | They do not affect execution.                                                                    |

---

**Disclaimer:** The provided content comes exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.