📰 EU Sustainability News Curation with RSS, GPT, Gmail, ElevenLabs & Telegram

https://n8nworkflows.xyz/workflows/---eu-sustainability-news-curation-with-rss--gpt--gmail--elevenlabs---telegram-10961


# 📰 EU Sustainability News Curation with RSS, GPT, Gmail, ElevenLabs & Telegram

### 1. Workflow Overview

This n8n workflow automates the daily curation and dissemination of EU sustainability-related news by leveraging RSS feeds, AI classification, email delivery, text-to-speech synthesis, and Telegram messaging. It targets users interested in curated, relevant news on EU environmental and climate topics, delivering a digest email and a voice summary every morning at 9:00 am.

The workflow is logically divided into four main blocks:

- **1.1 Fetch and Filter News**: Triggered daily at 9 am, this block fetches the latest news from the EU Council RSS feed, filters out archived (already processed) news to avoid duplicates, and prepares fresh items for processing.

- **1.2 AI Classification of News Items**: Each new news item is classified by a GPT-4o-mini AI model to determine if it is relevant to a predefined sustainability topic based on a detailed topic description.

- **1.3 Digest Construction and Email Sending**: Relevant news items are saved into a datatable, aggregated, and formatted into an attractive HTML email digest that is sent via Gmail.

- **1.4 Voice Summary Generation and Telegram Sending**: The workflow creates a concise 30-second voice summary of the day’s news headlines using an AI agent and ElevenLabs text-to-speech, then sends the audio file via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Fetch and Filter News

**Overview:**  
This block triggers the workflow daily at 9:00 am, fetches the latest news from the EU Council RSS feed, retrieves archived news identifiers, and filters out duplicates.

**Nodes Involved:**  
- Trigger: 09:00 am  
- RSS News EU  
- Archived News  
- List of guid(s) archived  
- Add Archived Guid(s)  
- Avoid Duplicates  

**Node Details:**

- **Trigger: 09:00 am**  
  - Type: Schedule Trigger  
  - Role: Starts workflow every day at 9:00 am  
  - Config: Trigger at hour 9 (local time assumed)  
  - Inputs: None  
  - Outputs: Today's News, RSS News EU, Archived News nodes  
  - Edge cases: Delays in execution or time zone mismatches could affect trigger timing  

- **RSS News EU**  
  - Type: RSS Feed Read  
  - Role: Fetch the latest press releases from the Council of the European Union RSS feed  
  - Config: URL set to "https://www.consilium.europa.eu/en/rss/pressreleases.ashx"  
  - Inputs: Trigger node  
  - Outputs: Add Archived Guid(s) node  
  - Edge cases: RSS feed downtime, invalid feed format, network errors  

- **Archived News**  
  - Type: Data Table (Get Operation)  
  - Role: Retrieves all previously archived news records to identify duplicates  
  - Config: Reads from a pre-configured datatable with columns like guid, title, etc.  
  - Inputs: Trigger node  
  - Outputs: List of guid(s) archived node  
  - Edge cases: Data table access errors, outdated data  

- **List of guid(s) archived**  
  - Type: Aggregate  
  - Role: Extracts all GUIDs from the archived news to build a list of processed news  
  - Config: Aggregates the "guid" field into an array named "guids"  
  - Inputs: Archived News node  
  - Outputs: Add Archived Guid(s) node  
  - Edge cases: Empty archive, aggregation errors  

- **Add Archived Guid(s)**  
  - Type: Merge (Combine By SQL)  
  - Role: Joins the newly fetched RSS news items with the list of archived GUIDs to filter duplicates  
  - Config: Combines incoming items without overwriting  
  - Inputs: RSS News EU (main), List of guid(s) archived (secondary)  
  - Outputs: Avoid Duplicates node, Generate a Digest in HTML node  
  - Edge cases: Mismatched data structures, merge failures  

- **Avoid Duplicates**  
  - Type: If (Condition)  
  - Role: Filters out news items whose GUID is already archived (i.e., duplicates)  
  - Config: Condition checks if current item's GUID is NOT included in archived GUIDs array  
  - Inputs: Add Archived Guid(s) node  
  - Outputs: Add News node (if not duplicate)  
  - Edge cases: Incorrect logic leading to false positives/negatives, expression evaluation errors  

---

#### 2.2 AI Classification of News Items

**Overview:**  
Classifies each new news item using an AI model to determine relevance to a sustainability topic based on a detailed topic description.

**Nodes Involved:**  
- Topic Config  
- Classify News based on a Topic  
- If  
- Add News  

**Node Details:**

- **Topic Config**  
  - Type: Set  
  - Role: Defines the topic name ("SUSTAINABILITY") and a detailed topic description covering climate, green initiatives, renewable energy, etc.  
  - Config: Assigns two string variables: `topic_name` and `topic_description`  
  - Inputs: None (triggered from prior flow)  
  - Outputs: Classify News based on a Topic node  
  - Edge cases: Missing or incomplete topic description affects classification accuracy  

- **Classify News based on a Topic**  
  - Type: OpenAI GPT-4o-mini (via LangChain)  
  - Role: AI classifies news items to determine if they are relevant to the sustainability topic  
  - Config: Uses GPT-4o-mini model with temperature 0.3 for deterministic output  
  - Prompt: Provides topic name, description, news title, content, and categories; expects JSON response with fields `is_relevant`, `relevance_score`, `key_topics_found`, `reasoning`  
  - Inputs: Topic Config node, current news item data  
  - Outputs: If node  
  - Edge cases: API rate limits, model response parsing errors, unexpected output format  

- **If (Relevance Check)**  
  - Type: If (Condition)  
  - Role: Filters only news items where AI classified `is_relevant` as true  
  - Config: Checks JSON path expression `$json.message.content.parseJson().is_relevant == true`  
  - Inputs: Classify News based on a Topic node  
  - Outputs: Add News node (if relevant)  
  - Edge cases: JSON parsing errors, missing fields, false classification  

- **Add News**  
  - Type: Data Table (Add Operation)  
  - Role: Saves relevant news item data (title, link, content, categories, guid, createDate) into the datatable for record-keeping and later digest generation  
  - Config: Maps fields from current news item, date set to current date in yyyy-MM-dd format  
  - Inputs: If node (filtered relevant news)  
  - Outputs: None (terminal for new news save)  
  - Edge cases: Data table write errors, field mapping mismatches  

---

#### 2.3 Digest Construction and Email Sending

**Overview:**  
Aggregates today's saved news, generates an HTML formatted email digest, and sends it via Gmail.

**Nodes Involved:**  
- Today's News  
- Generate a Digest in HTML  
- Send the digest  

**Node Details:**

- **Today's News**  
  - Type: Data Table (Get Operation with filter)  
  - Role: Loads news records from the datatable created today (within last 5 days filter, effectively recent news)  
  - Config: Filter where `createDate` is greater or equal to 5 days ago (to ensure coverage)  
  - Inputs: Trigger: 09:00 am node  
  - Outputs: Generate a Digest in HTML, Aggregate Titles nodes  
  - Edge cases: Empty results, data table access errors  

- **Generate a Digest in HTML**  
  - Type: Code (JavaScript)  
  - Role: Creates an HTML email template embedding all news items with styling, date, categories, and links; formats a professional sustainability news digest  
  - Config: Uses a script that processes all input news items, formats date in en-GB locale, and includes branded footer and links  
  - Inputs: Today's News node  
  - Outputs: Send the digest node  
  - Edge cases: Large number of news causing performance issues, HTML formatting errors  

- **Send the digest**  
  - Type: Gmail  
  - Role: Sends the generated HTML news digest email to a configured recipient  
  - Config: Uses Gmail credentials, subject dynamically set to "EU Sustainability Digest - {date}", email body is the HTML generated  
  - Inputs: Generate a Digest in HTML node  
  - Outputs: None (terminal for email sending)  
  - Edge cases: Gmail API authentication/authorization errors, email quota limits, invalid recipient address  

---

#### 2.4 Voice Summary Generation and Telegram Sending

**Overview:**  
Synthesizes a 30-second voice summary of the day’s news headlines using an AI agent and ElevenLabs text-to-speech, then sends it via Telegram.

**Nodes Involved:**  
- Aggregate Titles  
- Create Daily Voice Summary  
- Generate Voice Message  
- Send Voice Summary  

**Node Details:**

- **Aggregate Titles**  
  - Type: Aggregate  
  - Role: Collects all news titles from today's news into an array for summary generation  
  - Config: Aggregates the "title" field into array named "titles"  
  - Inputs: Today's News node  
  - Outputs: Create Daily Voice Summary node  
  - Edge cases: Empty input, aggregation failures  

- **Create Daily Voice Summary**  
  - Type: LangChain Agent (AI)  
  - Role: Generates a natural language spoken-style summary paragraph synthesizing the day's news headlines grouped into themes, suitable for a 30-second voice note  
  - Config: System message instructs the AI to create a plain-text, coherent summary mentioning "European Union" or "EU" and avoiding bullet points or markdown  
  - Inputs: Aggregate Titles node (JSON array of titles)  
  - Outputs: Generate Voice Message node  
  - Edge cases: AI model errors, unexpected output format  

- **Generate Voice Message**  
  - Type: ElevenLabs Text-to-Speech  
  - Role: Converts the AI-generated summary text into an audio speech file using a selected voice ("Paul")  
  - Config: Voice selected from ElevenLabs voice list, input text is the AI summary output  
  - Inputs: Create Daily Voice Summary node  
  - Outputs: Send Voice Summary node  
  - Edge cases: ElevenLabs API errors, voice selection missing or invalid, audio generation failure  

- **Send Voice Summary**  
  - Type: Telegram  
  - Role: Sends the generated speech audio as a Telegram audio message to a configured chat ID with a caption including the current date  
  - Config: Chat ID "1698247520" (must be updated for user), sends audio as binary, caption includes today's date  
  - Inputs: Generate Voice Message node  
  - Outputs: None (terminal for voice message delivery)  
  - Edge cases: Telegram API authentication errors, invalid chat ID, message sending failure  

---

### 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                 | Input Node(s)                    | Output Node(s)              | Sticky Note                                                                                                                                                                                                                                  |
|---------------------------|--------------------------------|------------------------------------------------|---------------------------------|-----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Trigger: 09:00 am         | Schedule Trigger               | Starts workflow daily at 9 am                   | None                            | Today's News, RSS News EU, Archived News | ## 1. Fetch the latest EU news from RSS                                                                                                                                                                                                     |
| RSS News EU               | RSS Feed Read                 | Fetches EU Council press releases RSS feed     | Trigger: 09:00 am               | Add Archived Guid(s)          | ## 1. Fetch the latest EU news from RSS                                                                                                                                                                                                     |
| Archived News             | Data Table (Get)              | Retrieves archived news to avoid duplicates     | Trigger: 09:00 am               | List of guid(s) archived      | ## 1. Fetch the latest EU news from RSS                                                                                                                                                                                                     |
| List of guid(s) archived  | Aggregate                    | Aggregates archived news GUIDs                   | Archived News                   | Add Archived Guid(s)          | ## 1. Fetch the latest EU news from RSS                                                                                                                                                                                                     |
| Add Archived Guid(s)      | Merge (Combine By SQL)        | Joins new news with archived GUIDs               | RSS News EU, List of guid(s) archived | Avoid Duplicates, Generate a Digest in HTML | ## 1. Fetch the latest EU news from RSS                                                                                                                                                                                                     |
| Avoid Duplicates          | If                           | Filters out news already archived (duplicates)  | Add Archived Guid(s)            | Add News                     | ## 2. Select the relevant news using an AI Classifier                                                                                                                                                                                       |
| Topic Config              | Set                          | Defines sustainability topic name and description| Avoid Duplicates (via flow)     | Classify News based on a Topic | ## 2. Select the relevant news using an AI Classifier                                                                                                                                                                                       |
| Classify News based on a Topic | OpenAI GPT-4o-mini (LangChain) | AI classifies news for topic relevance           | Topic Config                   | If                          | ## 2. Select the relevant news using an AI Classifier                                                                                                                                                                                       |
| If                        | If                           | Filters news where AI classified as relevant    | Classify News based on a Topic  | Add News                     | ## 2. Select the relevant news using an AI Classifier                                                                                                                                                                                       |
| Add News                  | Data Table (Add)             | Saves relevant news to archive                    | If                            | None                        | ## 2. Select the relevant news using an AI Classifier                                                                                                                                                                                       |
| Today's News              | Data Table (Get)             | Retrieves news from last 5 days for digest       | Trigger: 09:00 am               | Generate a Digest in HTML, Aggregate Titles | ## 3. Build the daily digest and send it by email                                                                                                                                                                                           |
| Generate a Digest in HTML | Code (JavaScript)            | Generates HTML formatted email digest             | Today's News                   | Send the digest              | ## 3. Build the daily digest and send it by email                                                                                                                                                                                           |
| Send the digest           | Gmail                        | Sends the HTML email digest                        | Generate a Digest in HTML       | None                        | ## 3. Build the daily digest and send it by email                                                                                                                                                                                           |
| Aggregate Titles          | Aggregate                    | Aggregates news titles for voice summary          | Today's News                   | Create Daily Voice Summary   | ## 4. Send a 30-second voice message summarizing the news using titles                                                                                                                                                                       |
| Create Daily Voice Summary| LangChain Agent (OpenAI)     | Creates spoken-style summary paragraph             | Aggregate Titles               | Generate Voice Message       | ## 4. Send a 30-second voice message summarizing the news using titles                                                                                                                                                                       |
| Generate Voice Message    | ElevenLabs Text-to-Speech    | Converts text summary to audio speech              | Create Daily Voice Summary     | Send Voice Summary           | ## 4. Send a 30-second voice message summarizing the news using titles                                                                                                                                                                       |
| Send Voice Summary        | Telegram                     | Sends generated audio summary to Telegram chat    | Generate Voice Message         | None                        | ## 4. Send a 30-second voice message summarizing the news using titles                                                                                                                                                                       |
| Sticky Note2              | Sticky Note                  | Comment: Step 1 - Fetch EU news                    | None                          | None                        | ## 1. Fetch the latest EU news from RSS                                                                                                                                                                                                     |
| Sticky Note               | Sticky Note                  | Comment: Step 2 - AI classifier for relevance     | None                          | None                        | ## 2. Select the relevant news using an AI Classifier                                                                                                                                                                                       |
| Sticky Note3              | Sticky Note                  | Comment: Step 3 - Build digest & send email       | None                          | None                        | ## 3. Build the daily digest and send it by email                                                                                                                                                                                           |
| Sticky Note4              | Sticky Note                  | Detailed workflow explanation, setup, and customization instructions | None                          | None                        | ## AI-Powered EU News Digest by Topic - Setup instructions and customization guide.                                                                                                                                                         |
| Sticky Note5              | Sticky Note                  | Comment: Step 4 - Send 30-second voice summary    | None                          | None                        | ## 4. Send a 30-second voice message summarizing the news using titles                                                                                                                                                                       |
| Sticky Note1              | Sticky Note                  | YouTube video link: @[youtube](vNavNGRqcK4)       | None                          | None                        | YouTube reference video link                                                                                                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 9:00 am (hour=9).  

2. **Create Data Table Nodes**  
   - **Archived News**: Data Table Get operation retrieving all archived news records.  
   - **Today's News**: Data Table Get operation filtering news created in the last 5 days using `createDate >= {{ $now.minus(5,'days').toFormat('yyyy-MM-dd') }}`.  
   - **Add News**: Data Table Add operation with columns: title, link, content, contentSnippet, guid, categories, createDate. Map fields from input JSON, date as current date.  

3. **Create RSS Feed Read Node**  
   - Name: RSS News EU  
   - URL: https://www.consilium.europa.eu/en/rss/pressreleases.ashx  

4. **Create Aggregate Node**  
   - Name: List of guid(s) archived  
   - Aggregate field: guid aggregated into array "guids".  

5. **Create Merge Node**  
   - Name: Add Archived Guid(s)  
   - Mode: Combine By SQL to merge RSS news with archived GUIDs.  

6. **Create If Node**  
   - Name: Avoid Duplicates  
   - Condition: Keep news where `guid` is NOT in archived `guids` array. Expression: `{{$json.guids.includes($json.guid)}}` is false.  

7. **Create Set Node**  
   - Name: Topic Config  
   - Assign variables:  
     - topic_name = "SUSTAINABILITY"  
     - topic_description = detailed multi-line string describing EU sustainability topics as in the original workflow.  

8. **Create OpenAI Node**  
   - Name: Classify News based on a Topic  
   - Model: GPT-4o-mini  
   - Temperature: 0.3  
   - Message prompt: Provide topic name, topic description, news item fields (title, content, categories) and request a JSON response with `is_relevant` boolean and other details.  

9. **Create If Node**  
   - Name: If (Relevance Check)  
   - Condition: Check if AI response JSON `is_relevant` is true. Use expression parsing the AI response JSON.  

10. **Connect Nodes**  
    - Trigger → Today's News, RSS News EU, Archived News  
    - Archived News → List of guid(s) archived  
    - List of guid(s) archived → Add Archived Guid(s) (secondary input)  
    - RSS News EU → Add Archived Guid(s) (main input)  
    - Add Archived Guid(s) → Avoid Duplicates  
    - Avoid Duplicates (true) → Topic Config  
    - Topic Config → Classify News based on a Topic  
    - Classify News based on a Topic → If (Relevance Check)  
    - If (true) → Add News  

11. **Create Code Node**  
    - Name: Generate a Digest in HTML  
    - JavaScript code: Format all news items from Today's News into a styled HTML email with date, categories, links, and footer as per provided script.  

12. **Create Gmail Node**  
    - Name: Send the digest  
    - Credentials: Set up Gmail OAuth2 credentials  
    - To: Set destination email address  
    - Subject: Dynamic with today's date "EU Sustainability Digest - {date}"  
    - HTML Body: Use output from Generate a Digest in HTML node  

13. **Create Aggregate Node**  
    - Name: Aggregate Titles  
    - Aggregate field: title into array "titles" from Today's News  

14. **Create LangChain Agent Node**  
    - Name: Create Daily Voice Summary  
    - Input: JSON array of titles  
    - System message: Instruct AI to generate a spoken-style summary paragraph (30 seconds) mentioning EU, no bullet points, plain text.  

15. **Create ElevenLabs Node**  
    - Name: Generate Voice Message  
    - Voice: Select preferred voice (e.g., "Paul")  
    - Text input: Output from Create Daily Voice Summary  

16. **Create Telegram Node**  
    - Name: Send Voice Summary  
    - Credentials: Set up Telegram Bot credentials  
    - Chat ID: Set recipient chat ID  
    - Operation: Send audio with caption including today's date  
    - Input: Audio binary from Generate Voice Message  

17. **Connect Nodes**  
    - Today's News → Generate a Digest in HTML, Aggregate Titles  
    - Generate a Digest in HTML → Send the digest  
    - Aggregate Titles → Create Daily Voice Summary  
    - Create Daily Voice Summary → Generate Voice Message  
    - Generate Voice Message → Send Voice Summary  

18. **Add Sticky Notes**  
    - Add informative sticky notes at logical locations describing workflow steps and setup instructions.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| AI-Powered EU News Digest by Topic: Detailed explanation of workflow steps, setup instructions for datatable, OpenAI, Gmail, ElevenLabs, Telegram credentials, and customization options.                                                    | Sticky Note4 content in workflow                 |
| YouTube video reference for workflow overview and demonstration: @[youtube](vNavNGRqcK4)                                                                                                                                                   | Sticky Note1 content in workflow                 |
| Branding in email footer as "🤖 LogiGreen Bot" with links to https://logi-green.com and https://linktr.ee/samirsaci                                                                                                                       | Email digest footer in Generate a Digest code   |
| Use of GPT-4o-mini model for classification and GPT-4.1-mini for voice summary generation via LangChain OpenAI integration nodes.                                                                                                        | OpenAI nodes configuration                       |
| ElevenLabs voice synthesis configured with voice ID "5Q0t7uMcjvnagumLfvZi" (named "Paul")                                                                                                                                                  | ElevenLabs node configuration                     |
| Telegram chat ID "1698247520" must be updated by user to target their own chat or group for voice message delivery.                                                                                                                      | Telegram node configuration                       |

---

This comprehensive analysis and reconstruction guide provides all necessary details to understand, reproduce, or modify the "EU Sustainability News Curation" workflow, ensuring robustness against common failure modes and facilitating integration with external services.