Daily Space Image Gallery with AI Captions from NASA to Slack & Google Drive

https://n8nworkflows.xyz/workflows/daily-space-image-gallery-with-ai-captions-from-nasa-to-slack---google-drive-10731


# Daily Space Image Gallery with AI Captions from NASA to Slack & Google Drive

### 1. Workflow Overview

This workflow, titled **Daily Space Image Gallery with AI Captions from NASA to Slack & Google Drive**, automates the daily fetching of space-related images from multiple NASA APIs, enriches these images with AI-generated poetic captions in Japanese, and shares a compiled gallery message on Slack while also archiving the message in Google Drive.

The workflow logically divides into these blocks:

- **1.1 Scheduled Trigger & Configuration Initialization:** Starts the workflow daily at 10:00 and loads essential configuration like API keys and Slack channel.
- **1.2 Parallel Image Fetching:** Concurrently retrieves images from three NASA sources: Mars Rover latest photos, NASA Image Library (keyword “nebula”), and NASA EPIC Earth imagery.
- **1.3 Data Formatting:** Normalizes the raw API responses from each source into a unified JSON format with fields like source, title, URL, and explanation.
- **1.4 AI Caption Generation Loop:** Iterates over each formatted image, sending its title and explanation to an AI agent (using OpenAI GPT-4) to generate a short, emotive caption in Japanese.
- **1.5 Message Composition & Distribution:** Combines the AI-generated captions with image data into a formatted Slack message, posts it to Slack, and saves the message text as a document on Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Configuration Initialization

- **Overview:** This block starts the workflow automatically every day at 10:00 AM and sets up configuration parameters including the NASA API key and Slack channel name.
- **Nodes Involved:**  
  - Daily 10:00 - Start Poll  
  - Workflow Configuration

- **Node Details:**

  - **Daily 10:00 - Start Poll**  
    - Type: Schedule Trigger  
    - Role: Triggers the workflow daily at 10:00 AM.  
    - Configuration: Interval set to trigger at hour 10 daily.  
    - Inputs: None (start node)  
    - Outputs: Passes trigger to Workflow Configuration node.  
    - Failure Modes: Misconfigured schedule or node downtime would prevent trigger.

  - **Workflow Configuration**  
    - Type: Set  
    - Role: Defines workflow parameters including NASA API key (`nasaApiKey`) and Slack channel name (`slackChannel`).  
    - Configuration: Two string parameters with placeholders; user must fill these before activation.  
    - Inputs: From Schedule Trigger  
    - Outputs: Feeds into the three NASA API fetch nodes.  
    - Failure Modes: Missing or incorrect API keys will cause downstream API calls to fail with authentication errors.

#### 2.2 Parallel Image Fetching

- **Overview:** This block performs parallel HTTP requests to three different NASA APIs to obtain daily images or photo metadata from Mars Rover, NASA Image Library (searching for “nebula”), and NASA EPIC Earth imagery.
- **Nodes Involved:**  
  - Fetch Mars Rover  
  - Fetch Image Library  
  - Fetch NASA EPIC

- **Node Details:**

  - **Fetch Mars Rover**  
    - Type: HTTP Request  
    - Role: Calls NASA Mars Rover API to get latest photos from Curiosity.  
    - Configuration: URL constructed dynamically with NASA API key from configuration node; retries up to 5 times on failure.  
    - Inputs: From Workflow Configuration  
    - Outputs: JSON response passed to Format Mars Photo node.  
    - Failure Modes: API key invalid, rate limits, or network issues causing request failures.

  - **Fetch Image Library**  
    - Type: HTTP Request  
    - Role: Searches NASA Image Library API with query “nebula” for image media type.  
    - Configuration: Static URL with query parameters; expects JSON response.  
    - Inputs: From Workflow Configuration  
    - Outputs: JSON response passed to Format Library Photo node.  
    - Failure Modes: API downtime, malformed responses.

  - **Fetch NASA EPIC**  
    - Type: HTTP Request  
    - Role: Retrieves latest natural Earth images from NASA EPIC API.  
    - Configuration: URL includes NASA API key from configuration; retries up to 5 times.  
    - Inputs: From Workflow Configuration  
    - Outputs: JSON response passed to Format EPIC Photo node.  
    - Failure Modes: API key issues, empty data sets.

#### 2.3 Data Formatting

- **Overview:** Each formatting node processes raw API data into a standardized JSON object containing source, title, URL, and explanation fields, enabling uniform downstream processing.
- **Nodes Involved:**  
  - Format Mars Photo  
  - Format Library Photo  
  - Format EPIC Photo

- **Node Details:**

  - **Format Mars Photo**  
    - Type: Code (JavaScript)  
    - Role: Extracts the latest Mars rover photo, composes a standardized object including source, title with rover name, photo URL, and camera/date explanation.  
    - Input: Raw JSON from Fetch Mars Rover  
    - Output: Single JSON object with normalized fields.  
    - Edge Cases: No latest photos available → returns empty array (halts downstream processing for this branch).

  - **Format Library Photo**  
    - Type: Code (JavaScript)  
    - Role: Takes first image from search results, truncates description to 200 characters, creates standardized object with source, title, URL, and explanation.  
    - Input: Raw JSON from Fetch Image Library  
    - Output: Single JSON object normalized.  
    - Edge Cases: Empty results or missing fields → returns empty array.

  - **Format EPIC Photo**  
    - Type: Code (JavaScript)  
    - Role: Picks the latest EPIC Earth photo, constructs full image URL from date and image name, sets source, title with date, URL, and caption or fallback explanation.  
    - Input: Raw JSON from Fetch NASA EPIC  
    - Output: Single JSON object standardized.  
    - Edge Cases: Empty or missing data → returns empty array.

#### 2.4 AI Caption Generation Loop

- **Overview:** This block loops over the three standardized image objects, sending each to an AI agent which generates a poetic Japanese caption. The AI agent uses OpenAI GPT-4 via a Langchain agent node.
- **Nodes Involved:**  
  - Merge  
  - Loop Over Items  
  - AI Agent  
  - OpenAI Chat Model3  
  - Replace Me

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines the three formatted image outputs into a single collection of items for batch processing.  
    - Inputs: Outputs from Format EPIC Photo, Format Mars Photo, Format Library Photo  
    - Output: Combined array of three items.  
    - Edge Cases: Missing any input results in fewer items downstream.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates over each image item individually for AI caption generation.  
    - Inputs: From Merge  
    - Outputs: Single item per iteration passed to AI Agent node on the second output; the first output is unused here.

  - **AI Agent**  
    - Type: Langchain Agent (OpenAI GPT-4)  
    - Role: Receives photo title and original explanation, prompts AI to generate a concise, emotive Japanese caption (max 80 characters).  
    - Configuration: System message defines role as “space-specialized copywriter” and instructs output to be caption text only.  
    - Inputs: Single image item from Loop Over Items  
    - Outputs: JSON including original photo data plus AI-generated caption under `output` key.  
    - Edge Cases: AI API rate limits, timeouts, or malformed prompt response.

  - **OpenAI Chat Model3**  
    - Type: Langchain Chat Model (GPT-4 variant)  
    - Role: Language model used by AI Agent to generate captions.  
    - Configuration: Model set to “gpt-4.1-mini”.  
    - Inputs/Outputs: Connected internally to AI Agent node.  
    - Edge Cases: API key/authentication failures, quota exceeded.

  - **Replace Me**  
    - Type: NoOp (No Operation)  
    - Role: Placeholder node used for connection flow; simply passes data forward.  
    - Inputs: From AI Agent  
    - Outputs: Back to Loop Over Items to continue iteration.

#### 2.5 Message Composition & Distribution

- **Overview:** Combines the AI-augmented image data into a formatted Slack message, posts it to Slack, and saves a backup copy as a text document on Google Drive.
- **Nodes Involved:**  
  - Combine for Slack  
  - Post to Slack1  
  - Set: Prepare Summary  
  - Google Drive: Save Summary

- **Node Details:**

  - **Combine for Slack**  
    - Type: Code (JavaScript)  
    - Role: Builds a multi-line Slack message string including all images with their AI captions and URLs, adds emoji reactions guide.  
    - Inputs: Array of image objects with AI captions.  
    - Outputs: JSON object with `slackMessage` string.  
    - Edge Cases: Missing AI captions or incomplete data will affect message quality but not break workflow.

  - **Post to Slack1**  
    - Type: Slack  
    - Role: Posts the assembled message to the configured Slack channel as a message from a specified user.  
    - Configuration: Uses OAuth2 authentication; posts message text from `slackMessage` JSON field.  
    - Inputs: From Combine for Slack  
    - Outputs: None (final action)  
    - Edge Cases: Slack API auth failures, channel misconfiguration, rate limits.

  - **Set: Prepare Summary**  
    - Type: Set  
    - Role: Prepares data for Google Drive by setting document title and content fields.  
    - Configuration: Document title fixed as empty string (can be customized), content from Slack message.  
    - Inputs: From Combine for Slack  
    - Outputs: Feeds Google Drive node.

  - **Google Drive: Save Summary**  
    - Type: Google Drive  
    - Role: Creates a new text document in a specified Google Drive folder with the Slack message content as backup.  
    - Configuration: Target drive is “My Drive”, folder ID specified; operation is createFromText.  
    - Inputs: From Set: Prepare Summary  
    - Outputs: None (final archival step)  
    - Edge Cases: Google Drive authentication or permission errors, quota limits.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                        | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                           |
|-------------------------|----------------------------|-------------------------------------|--------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Daily 10:00 - Start Poll| Schedule Trigger           | Start workflow daily at 10:00 AM     | None                           | Workflow Configuration          |                                                                                                                       |
| Workflow Configuration  | Set                        | Store API keys and Slack channel     | Daily 10:00 - Start Poll        | Fetch Mars Rover, Fetch Image Library, Fetch NASA EPIC |                                                                                                                       |
| Fetch Mars Rover        | HTTP Request               | Fetch latest Mars Rover photos       | Workflow Configuration          | Format Mars Photo              | 🛰️ Fetch Images: These three nodes make parallel API calls to different NASA sources to gather a diverse set of daily space images. |
| Fetch Image Library     | HTTP Request               | Fetch images from NASA Image Library | Workflow Configuration          | Format Library Photo           | 🛰️ Fetch Images                                                                                                       |
| Fetch NASA EPIC         | HTTP Request               | Fetch latest NASA EPIC Earth images  | Workflow Configuration          | Format EPIC Photo              | 🛰️ Fetch Images                                                                                                       |
| Format Mars Photo       | Code                       | Format Mars Rover photo data         | Fetch Mars Rover                | Merge                         | 🛠️ Format Data: Each Code node standardizes API responses into common JSON objects.                                    |
| Format Library Photo    | Code                       | Format Image Library photo data      | Fetch Image Library             | Merge                         | 🛠️ Format Data                                                                                                       |
| Format EPIC Photo       | Code                       | Format NASA EPIC photo data          | Fetch NASA EPIC                | Merge                         | 🛠️ Format Data                                                                                                       |
| Merge                   | Merge                      | Combine all formatted photo outputs  | Format EPIC Photo, Format Mars Photo, Format Library Photo | Loop Over Items               |                                                                                                                       |
| Loop Over Items         | SplitInBatches             | Iterate over each photo item          | Merge                         | AI Agent (batch output)        | 🤖 AI Caption Generation: Loops to send each image to AI for captioning.                                              |
| AI Agent                | Langchain Agent            | Generate poetic Japanese caption      | Loop Over Items                | Replace Me                    | 🤖 AI Caption Generation                                                                                              |
| OpenAI Chat Model3      | Langchain Chat Model       | Provides GPT-4 language model        | Connected internally to AI Agent| AI Agent                      |                                                                                                                       |
| Replace Me              | NoOp                       | Pass data forward                     | AI Agent                      | Loop Over Items               |                                                                                                                       |
| Combine for Slack       | Code                       | Compose Slack message with captions  | AI Agent                      | Post to Slack1, Set: Prepare Summary |                                                                                                                       |
| Post to Slack1          | Slack                      | Post message to Slack channel         | Combine for Slack              | None                         | 🚀 Post & Save: Posts message to Slack; requires OAuth2 credentials connected.                                        |
| Set: Prepare Summary    | Set                        | Prepare document data for Google Drive | Combine for Slack              | Google Drive: Save Summary     | 🚀 Post & Save                                                                                                        |
| Google Drive: Save Summary | Google Drive             | Save Slack message as text document   | Set: Prepare Summary           | None                         | 🚀 Post & Save: Saves a backup of the Slack message to Google Drive; requires credentials.                             |
| Sticky Note             | Sticky Note                | Documentation and explanation node   | None                          | None                         | 🌌 Daily Space Image Gallery with AI Captions (detailed explanation)                                                 |
| Sticky Note 1           | Sticky Note                | Fetch images explanation              | None                          | None                         | 🛰️ Fetch Images                                                                                                       |
| Sticky Note 2           | Sticky Note                | Post and save explanation             | None                          | None                         | 🚀 Post & Save                                                                                                        |
| Sticky Note 3           | Sticky Note                | Format data explanation               | None                          | None                         | 🛠️ Format Data                                                                                                       |
| Sticky Note 4           | Sticky Note                | AI caption generation explanation     | None                          | None                         | 🤖 AI Caption Generation                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Name: `Daily 10:00 - Start Poll`  
   - Trigger: Daily at 10:00 AM (set triggerAtHour to 10)  
   - Connect output to the next node.

2. **Create a Set node for configuration:**  
   - Name: `Workflow Configuration`  
   - Add two string parameters:  
     - `nasaApiKey` (leave empty for user input)  
     - `slackChannel` (default value: "YOUR_SLACK_CHANNEL_NAME")  
   - Connect input from the Schedule Trigger.

3. **Create three HTTP Request nodes for NASA APIs:**

   - **Fetch Mars Rover:**  
     - URL: `https://api.nasa.gov/mars-photos/api/v1/rovers/curiosity/latest_photos?api_key={{ $json.nasaApiKey }}` (use expression referencing workflow config node)  
     - Response format: JSON  
     - Retry on fail: 5 attempts  
     - Connect input from Workflow Configuration.

   - **Fetch Image Library:**  
     - URL: `https://images-api.nasa.gov/search?q=nebula&media_type=image`  
     - Response format: JSON  
     - Connect input from Workflow Configuration.

   - **Fetch NASA EPIC:**  
     - URL: `https://api.nasa.gov/EPIC/api/natural?api_key={{ $json.nasaApiKey }}`  
     - Response format: JSON  
     - Retry on fail: 5 attempts  
     - Connect input from Workflow Configuration.

4. **Create three Code nodes to format each API response:**

   - **Format Mars Photo:**  
     - Paste the JavaScript code to select the latest Mars photo, build a JSON object with fields: `source`, `title`, `url`, `explanation` (refer to code in section 2.3).  
     - Input from Fetch Mars Rover node.

   - **Format Library Photo:**  
     - Paste JavaScript code to select first image from library results, truncate description to 200 characters, build standard JSON object.  
     - Input from Fetch Image Library node.

   - **Format EPIC Photo:**  
     - Paste JavaScript code to select latest EPIC image, build image URL, and standard JSON object.  
     - Input from Fetch NASA EPIC node.

5. **Create a Merge node:**
   - Name: `Merge`  
   - Mode: Combine by SQL (combine all inputs into a single collection)  
   - Number of inputs: 3  
   - Connect inputs from the three format code nodes.

6. **Create a SplitInBatches node:**
   - Name: `Loop Over Items`  
   - Default batch size  
   - Input from Merge node.

7. **Create an AI Agent node (Langchain):**
   - Name: `AI Agent`  
   - Prompt text:  
     ```
     写真のタイトル: {{ $json.title }}
     元の説明文: {{ $json.explanation }}
     ```
   - System message:  
     ```
     あなたは、与えられた写真の情報を元に、その写真が持つ「魂」を短いキャプション（日本語で80文字以内）で表現する、宇宙専門のコピーライターです。

     あなたの仕事は、以下の情報を元に、感情に訴えかける紹介文を生成することです。
     - 写真のタイトル
     - 写真の元々の説明文

     例えば、「火星探査車からの便り」という情報を受け取ったら、「赤い大地に刻まれた、孤独な轍。今日も彼は、人類の夢を乗せて進む。」のように、物語を感じさせる言葉を返してください。

     生成するのは、そのキャプションの本文のみです。他の言葉は一切不要です。
     ```
   - Connect to an OpenAI Chat Model node configured with GPT-4.1-mini model.

8. **Create OpenAI Chat Model node:**
   - Name: `OpenAI Chat Model3`  
   - Model: GPT-4.1-mini  
   - Connect as language model for AI Agent node.

9. **Create a NoOp node:**
   - Name: `Replace Me`  
   - Connect input from AI Agent node and output back to Loop Over Items node to continue processing batches.

10. **Create a Code node to compose Slack message:**
    - Name: `Combine for Slack`  
    - Paste code that loops through all items (with AI captions), concatenates a formatted Slack message (including emoji reactions), and outputs the result as `slackMessage`.  
    - Input from AI Agent node (after loop completes).

11. **Create a Slack node:**
    - Name: `Post to Slack1`  
    - Authentication: OAuth2 (configure Slack credentials)  
    - Channel: Use the configured Slack channel variable.  
    - Text: Expression `={{ $json.slackMessage }}`  
    - Input from Combine for Slack node.

12. **Create a Set node to prepare Google Drive document:**
    - Name: `Set: Prepare Summary`  
    - Assign `documentTitle` (empty string or custom title)  
    - Assign `documentContent` from `slackMessage` field  
    - Input from Combine for Slack node.

13. **Create Google Drive node:**
    - Name: `Google Drive: Save Summary`  
    - Operation: Create from Text  
    - Drive ID: “My Drive”  
    - Folder ID: Folder where to save the summary (configure accordingly)  
    - Name: Fixed string or expression (e.g., “🌌 **本日の宇宙セレクション** 🌌”)  
    - Content: From `documentContent` field  
    - Input from Set node.

14. **Activate the workflow:**
    - Ensure API keys and Slack channel are set in Workflow Configuration node.  
    - Connect credentials for OpenAI, Slack, and Google Drive nodes.  
    - Save and activate workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                 |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| 🌌 Daily Space Image Gallery with AI Captions. This workflow fetches stunning daily images from NASA, generates creative AI captions in Japanese, posts the collection to Slack, and saves a backup in Google Drive.                     | Sticky Note node content                                                                                       |
| 🛰️ Fetch Images: The three parallel HTTP requests to Mars Rover, NASA Image Library, and NASA EPIC APIs fetch diverse daily space images to enrich the gallery.                                                                      | Sticky Note 1                                                                                                  |
| 🛠️ Format Data: Normalizes API responses into a unified JSON format for consistent processing downstream.                                                                                                                           | Sticky Note 3                                                                                                  |
| 🤖 AI Caption Generation: Uses OpenAI GPT-4 model via n8n's Langchain agent node to produce short, poetic captions in Japanese for each image.                                                                                       | Sticky Note 4                                                                                                  |
| 🚀 Post & Save: Posts the compiled message with AI captions to Slack and saves a copy to Google Drive. Requires proper OAuth2 credentials for Slack and Google Drive nodes.                                                         | Sticky Note 2                                                                                                  |
| NASA API documentation: https://api.nasa.gov/                                                                                                                                                                                       | Official NASA API portal                                                                                        |
| OpenAI API documentation: https://platform.openai.com/docs/api-reference                                                                                                                                                            | Official OpenAI API documentation                                                                              |
| Slack API documentation: https://api.slack.com/web                                                                                                                                                                                  | Slack API reference                                                                                            |
| Google Drive API documentation: https://developers.google.com/drive/api                                                                                                                                                             | Google Drive API reference                                                                                      |

---

**Disclaimer:** The supplied text is derived exclusively from an automated workflow created in n8n, an integration and automation tool. This process strictly complies with all current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.