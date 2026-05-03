Weekly Notion Journal & Task Summarization with GPT-4.1 to Discord

https://n8nworkflows.xyz/workflows/weekly-notion-journal---task-summarization-with-gpt-4-1-to-discord-7032


# Weekly Notion Journal & Task Summarization with GPT-4.1 to Discord

### 1. Workflow Overview

This workflow automates the weekly summarization of your Notion journal entries and task completions, leveraging GPT-4.1 to generate a concise summary and then posts the output to Discord and saves it back into Notion. It is designed to support personal productivity tracking by aggregating daily journals from the past week and providing an overview of completed tasks, with congratulatory feedback based on task count.

**Target Use Cases:**  
- Weekly reflection and progress tracking for personal or professional journals stored in Notion.  
- Automated summarization using AI to reduce manual review effort.  
- Integration with Discord for team or personal notifications.  
- Archiving weekly summaries back into Notion for historical record keeping.

**Logical Blocks:**  
- **1.1 Triggering Mechanisms:** Schedule-based and manual triggers to start the workflow.  
- **1.2 Journal Retrieval & Processing:** Fetches daily journal pages from Notion and their child content blocks, then aggregates the textual content.  
- **1.3 AI Summarization:** Sends the aggregated journal text to GPT-4.1 for summary generation.  
- **1.4 Summary Distribution:** Posts the AI-generated summary to Discord and saves it back to Notion as a new weekly summary entry.  
- **1.5 Task Retrieval & Summary:** Retrieves completed tasks from Notion for the past week, calculates totals with an emotive message, and posts the summary to Discord.

---

### 2. Block-by-Block Analysis

#### 1.1 Triggering Mechanisms

**Overview:**  
This block initiates the workflow either on a scheduled weekly basis or manually by user execution.

**Nodes Involved:**  
- `Schedule Trigger`  
- `When clicking ‘Execute workflow’`

**Node Details:**  

- **Schedule Trigger**  
  - **Type:** Schedule Trigger  
  - **Role:** Automatically triggers the workflow every 7 days at 16:00 (4 PM) based on configured timezone (America/New_York).  
  - **Configuration:** Interval set to 7 days; trigger hour at 16.  
  - **Inputs:** None (time-based)  
  - **Outputs:** Starts `Get Daily Journals` and `Get Tasks` nodes.  
  - **Failure Types:** Timezone misconfiguration, scheduling conflicts.  
  - **Version:** 1.2  

- **When clicking ‘Execute workflow’**  
  - **Type:** Manual Trigger  
  - **Role:** Allows manual execution of the workflow on demand.  
  - **Configuration:** Default manual trigger without parameters.  
  - **Inputs:** None  
  - **Outputs:** Starts `Get Daily Journals` and `Get Tasks` nodes.  
  - **Failure Types:** None typical, manual usage-dependent.  
  - **Version:** 1  

---

#### 1.2 Journal Retrieval & Processing

**Overview:**  
Fetches daily journal pages created or edited within the past week from Notion, retrieves their child blocks (content), and aggregates the collected text for further AI processing.

**Nodes Involved:**  
- `Get Daily Journals`  
- `Get many child blocks`  
- `Aggregate`  
- `Combined Summary for LLM`

**Node Details:**  

- **Get Daily Journals**  
  - **Type:** Notion Node (databasePage/getAll)  
  - **Role:** Retrieves up to 5 journal entries from the configured Notion database filtered by pages tagged as `Journal` and last edited within the past week.  
  - **Configuration:** Filter by property `Type` equals `Journal`; last edited time in past week; limit 5 entries.  
  - **Inputs:** Trigger nodes (`Schedule Trigger` or manual).  
  - **Outputs:** Page metadata containing journal entries.  
  - **Failure Types:** Notion API auth errors, incorrect database or filter configuration, rate limits.  
  - **Version:** 2.2  

- **Get many child blocks**  
  - **Type:** Notion Node (block/getAll)  
  - **Role:** For each journal page retrieved, fetches all child blocks (content elements like text, headings).  
  - **Configuration:** Uses the page ID from `Get Daily Journals` output to fetch child blocks.  
  - **Inputs:** Output of `Get Daily Journals`.  
  - **Outputs:** Detailed content blocks of each journal page.  
  - **Failure Types:** Missing permissions on pages, API timeouts, malformed IDs.  
  - **Version:** 2.2  

- **Aggregate**  
  - **Type:** Aggregate Node  
  - **Role:** Combines the `content` field from all child blocks into a single aggregated text blob.  
  - **Configuration:** Aggregates by `content` field.  
  - **Inputs:** Output from `Get many child blocks`.  
  - **Outputs:** Single combined text string.  
  - **Failure Types:** Empty content arrays, missing `content` fields.  
  - **Version:** 1  

- **Combined Summary for LLM**  
  - **Type:** Code Node (JavaScript)  
  - **Role:** Concatenates all aggregated content strings into one message to send to the AI model.  
  - **Configuration:** Iterates over all inputs, concatenates non-empty `content` fields into a single string under `message`.  
  - **Inputs:** Output from `Aggregate`.  
  - **Outputs:** JSON object with combined journal content in `message`.  
  - **Failure Types:** Empty inputs, undefined `content` fields, JS runtime errors.  
  - **Version:** 2  

---

#### 1.3 AI Summarization

**Overview:**  
Uses the GPT-4.1 model to generate a concise weekly summary of the aggregated journal entries.

**Nodes Involved:**  
- `Generate Summary`

**Node Details:**

- **Generate Summary**  
  - **Type:** OpenAI Node (LangChain)  
  - **Role:** Sends the combined journal text as prompt content to GPT-4.1 and receives a concise summary.  
  - **Configuration:** Model set to `gpt-4.1`; prompt instructs AI to summarize the weekly journal entries; no simplification applied.  
  - **Inputs:** Output from `Combined Summary for LLM`.  
  - **Outputs:** AI-generated summary in `choices[0].message.content`.  
  - **Failure Types:** API key/auth errors, rate limits, prompt formatting issues, model unavailability.  
  - **Version:** 1.8  

---

#### 1.4 Summary Distribution

**Overview:**  
Posts the AI-generated weekly summary to a Discord channel via webhook and saves the summary as a new Notion page with type `Weekly`.

**Nodes Involved:**  
- `Send Summary`  
- `Save Back to Notion`

**Node Details:**  

- **Send Summary**  
  - **Type:** Discord Node  
  - **Role:** Posts the summary message received from GPT-4.1 to a configured Discord webhook.  
  - **Configuration:** Content sourced from AI summary; uses webhook authentication.  
  - **Inputs:** Output from `Generate Summary`.  
  - **Outputs:** Discord message success/failure response.  
  - **Failure Types:** Invalid webhook URL, network errors, Discord rate limits.  
  - **Version:** 2  

- **Save Back to Notion**  
  - **Type:** Notion Node (databasePage/create)  
  - **Role:** Creates a new Notion page in the configured database with the weekly summary content and sets `Type` property to `Weekly`.  
  - **Configuration:** Title is current date formatted as `MMM d, yy` with suffix `Weekly Summary [automated]`; content block contains AI summary text; database and property configured.  
  - **Inputs:** Output from `Generate Summary`.  
  - **Outputs:** New page metadata.  
  - **Failure Types:** Permission errors, invalid database ID, API limits.  
  - **Version:** 2.2  

---

#### 1.5 Task Retrieval & Summary

**Overview:**  
Retrieves completed tasks from Notion for the past week, calculates total tasks, composes an emoji-enhanced congratulatory message, and posts it to Discord.

**Nodes Involved:**  
- `Get Tasks`  
- `To Dos Totals`  
- `To Do Summary`

**Node Details:**  

- **Get Tasks**  
  - **Type:** Notion Node (databasePage/getAll)  
  - **Role:** Fetches all tasks completed within the past week from the specified Notion database.  
  - **Configuration:** Filter on `Completed Date` within past week; limit 100 tasks.  
  - **Inputs:** Trigger nodes (`Schedule Trigger` or manual).  
  - **Outputs:** List of completed task entries.  
  - **Failure Types:** API errors, incorrect database or filter configuration.  
  - **Version:** 2.2  

- **To Dos Totals**  
  - **Type:** Code Node (JavaScript)  
  - **Role:** Counts the number of completed tasks and assigns an emoji based on thresholds to create a summary message.  
  - **Configuration:** Emoji logic based on task count:  
    - >40 tasks: ✨🤩✨  
    - >30 tasks: 🥳  
    - >20 tasks: 🤠  
    - <10 tasks: 🤨  
  - **Inputs:** Output from `Get Tasks`.  
  - **Outputs:** JSON with a `message` string containing the total and emoji.  
  - **Failure Types:** Empty task list, JS code exceptions.  
  - **Version:** 2  

- **To Do Summary**  
  - **Type:** Discord Node  
  - **Role:** Posts the tasks summary message to Discord webhook.  
  - **Configuration:** Content set from `To Dos Totals` message output; uses webhook auth.  
  - **Inputs:** Output from `To Dos Totals`.  
  - **Outputs:** Discord post response.  
  - **Failure Types:** Webhook errors, network issues.  
  - **Version:** 2  

---

### 3. Summary Table

| Node Name               | Node Type                   | Functional Role                     | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                                                  |
|-------------------------|-----------------------------|-----------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Manual start of workflow          | None                            | Get Daily Journals, Get Tasks    |                                                                                                                              |
| Schedule Trigger        | Schedule Trigger            | Scheduled weekly start            | None                            | Get Daily Journals, Get Tasks    |                                                                                                                              |
| Get Daily Journals      | Notion (databasePage/getAll) | Retrieve last week’s journal pages | Schedule Trigger, Manual Trigger | Get many child blocks            | Ensure your Notion connection is configured here as well as the page DB and Type (filter) you want to use when retrieving your daily journals. Likewise ensure your Tasks DB is configured correctly (or simply remove the bottom portion of this workflow) |
| Get many child blocks   | Notion (block/getAll)       | Retrieve content blocks of journals | Get Daily Journals               | Aggregate                       |                                                                                                                              |
| Aggregate              | Aggregate                    | Combine journal content           | Get many child blocks            | Combined Summary for LLM         |                                                                                                                              |
| Combined Summary for LLM | Code                        | Concatenate all content strings   | Aggregate                       | Generate Summary                 | The preconfigured prompt is pretty generic, but feel free to edit to add humor or any other specific instructions you'd like ChatGPT to use when generating your summaries. |
| Generate Summary        | OpenAI (GPT-4.1)            | Generate concise weekly summary   | Combined Summary for LLM         | Send Summary, Save Back to Notion |                                                                                                                              |
| Send Summary            | Discord                     | Post weekly summary to Discord    | Generate Summary                | None                           |                                                                                                                              |
| Save Back to Notion     | Notion (databasePage/create) | Save weekly summary to Notion     | Generate Summary                | None                           |                                                                                                                              |
| Get Tasks               | Notion (databasePage/getAll) | Retrieve completed tasks last week | Schedule Trigger, Manual Trigger | To Dos Totals                  | Ensure your Notion connection is configured here as well as the page DB and Type (filter) you want to use when retrieving your daily journals. Likewise ensure your Tasks DB is configured correctly (or simply remove the bottom portion of this workflow) |
| To Dos Totals           | Code                        | Count tasks and compose summary   | Get Tasks                      | To Do Summary                   | Customize the Weekly Tasks total summary message that gets sent to Discord.                                                   |
| To Do Summary           | Discord                     | Post task completion summary      | To Dos Totals                  | None                           |                                                                                                                              |
| Sticky Note             | Sticky Note                 | Documentation                    | None                            | None                           | ## Notion Weekly Journal AI Summary This workflow will run on a weekly schedule and retrieve your Daily Journal pages for the past week and aggregate them into a ChatGPT generated concise summary. It will save that weekly summary back to your Notion as a new Note in addition to posting to a personal Discord channel. Additionally it will also retrieve all of the Tasks you've completed in the past week and provide a quick total with a congratulatory message to a Discord channel as well. ### Requirements: - You need Notion setup w/ a Notes database - If you want the Discord messages, setup a Discord webhook for your channel as well, or simply delete the Discord nodes. - One of the properties for the Notes db should be `Type` with a value of `Journal` - The contents of your daily Journal pages can be whatever you want - I've found what works best for me is the format of "What was a highlight of the day?", "What was a low point of the day?", and "What decisions did I delegate, delay, or dodge?" - You should also create an additional `Type` for your Weekly summary - in this case I used simply `Weekly` - Automate this to run weekly on your day of choice. I tend to only journal on weekdays so I've set mine up to run every Friday retrieving the past 5 days of Journal entries. |
| Sticky Note1            | Sticky Note                 | Documentation                    | None                           | None                           | The preconfigured prompt is pretty generic, but feel free to edit to add humor or any other specific instructions you'd like ChatGPT to use when generating your summaries. |
| Sticky Note2            | Sticky Note                 | Documentation                    | None                           | None                           | Ensure your Notion connection is configured here as well as the page DB and Type (filter) you want to use when retrieving your daily journals. Likewise ensure your Tasks DB is configured correctly (or simply remove the bottom portion of this workflow) |
| Sticky Note3            | Sticky Note                 | Documentation                    | None                           | None                           | Customize the Weekly Tasks total summary message that gets sent to Discord.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node:  
     - Set interval to every 7 days.  
     - Set trigger hour to 16 (4 PM).  
     - Set timezone to America/New_York.  
   - Add a **Manual Trigger** node named "When clicking ‘Execute workflow’" for manual runs.

2. **Set up Notion Credentials:**  
   - Configure Notion API credentials with required access scopes (read/write to your Notes and Tasks databases).

3. **Get Daily Journals:**  
   - Add a **Notion** node named `Get Daily Journals`.  
   - Set operation to `getAll` on `databasePage`.  
   - Set database to your Notes database ID.  
   - Apply filter:  
     - Property `Type` equals `Journal`.  
     - `Last edited time` is in `past_week`.  
   - Limit results to 5.

4. **Get Child Blocks of Journals:**  
   - Add **Notion** node `Get many child blocks`.  
   - Set operation `getAll` on `block`.  
   - Configure to use `blockId` from each journal page retrieved (expression mode: use `={{ $json.id }}`).

5. **Aggregate Journal Content:**  
   - Add an **Aggregate** node named `Aggregate`.  
   - Aggregate `content` field from all input child blocks.

6. **Combine Aggregated Text:**  
   - Add a **Code** node named `Combined Summary for LLM`.  
   - Use JavaScript to concatenate all `content` fields into a single string property named `message`.

7. **Configure OpenAI GPT-4.1 Node:**  
   - Add **OpenAI (LangChain)** node named `Generate Summary`.  
   - Set model to `gpt-4.1`.  
   - Set prompt message:  
     ```
     Given this blob of text which is the aggregation of the past week of Journal entries I've made, can you give me a concise summary for the week?

     {{ $json.message }}
     ```  
   - Connect input to output of `Combined Summary for LLM`.  
   - Configure OpenAI API credentials.

8. **Send Summary to Discord:**  
   - Add **Discord** node `Send Summary`.  
   - Authenticate via Discord webhook.  
   - Set content to `={{ $json.choices[0].message.content }}` from `Generate Summary` output.

9. **Save Summary Back to Notion:**  
   - Add **Notion** node `Save Back to Notion`.  
   - Configure to create a new page in the same Notes database.  
   - Title: current date formatted as `MMM d, yy` + " Weekly Summary [automated]".  
   - Content block: text set to AI summary content.  
   - Set property `Type` to `Weekly`.

10. **Get Completed Tasks:**  
    - Add a **Notion** node `Get Tasks`.  
    - Operation `getAll` on `databasePage`.  
    - Database set to your Tasks database ID.  
    - Filter: `Completed Date` in `past_week`.  
    - Limit 100.

11. **Calculate Task Totals:**  
    - Add **Code** node `To Dos Totals`.  
    - JavaScript logic: count tasks from input length and assign emoji based on count thresholds.  
    - Return a message string with count and emoji.

12. **Post Task Summary to Discord:**  
    - Add **Discord** node `To Do Summary`.  
    - Authenticate via Discord webhook.  
    - Content set from `To Dos Totals` message output.

13. **Connect Nodes:**  
    - Connect both triggers (`Schedule Trigger` and manual trigger) to `Get Daily Journals` and `Get Tasks`.  
    - Chain `Get Daily Journals` → `Get many child blocks` → `Aggregate` → `Combined Summary for LLM` → `Generate Summary` → both `Send Summary` and `Save Back to Notion`.  
    - Chain `Get Tasks` → `To Dos Totals` → `To Do Summary`.

14. **Add Sticky Notes:** (optional for documentation)  
    - Create sticky notes with instructions and requirements as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ## Notion Weekly Journal AI Summary This workflow will run on a weekly schedule and retrieve your Daily Journal pages for the past week and aggregate them into a ChatGPT generated concise summary. It will save that weekly summary back to your Notion as a new Note in addition to posting to a personal Discord channel. Additionally it will also retrieve all of the Tasks you've completed in the past week and provide a quick total with a congratulatory message to a Discord channel as well. | Main workflow description sticky note                                                          |
| Requirements include having a Notion database with a `Type` property set to `Journal` for daily entries and `Weekly` for summaries. A Discord webhook must be configured to receive messages or else remove Discord nodes. Journals are best formatted with reflective prompts like highlights, low points, or decisions made.                                                                                                                                                                                                 | Requirements from main sticky note                                                              |
| The preconfigured GPT prompt is generic; users are encouraged to customize it for tone, humor, or specific instructions to better fit personal summarization style.                                                                                                                                                                                                                                                                                                                                                       | Sticky Note1 content                                                                             |
| Ensure Notion connection credentials and database IDs are correctly set for both journal entries and tasks. The filtering on `Type` property and date ranges is essential for accurate data retrieval.                                                                                                                                                                                                                                                                                                                     | Sticky Note2 content                                                                             |
| Customize the congratulatory message for the tasks summary sent to Discord by modifying the JavaScript code in the `To Dos Totals` node. Different emojis and thresholds can be adapted.                                                                                                                                                                                                                                                                                                                                   | Sticky Note3 content                                                                             |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.