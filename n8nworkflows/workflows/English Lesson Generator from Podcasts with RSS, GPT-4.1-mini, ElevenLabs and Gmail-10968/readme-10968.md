English Lesson Generator from Podcasts with RSS, GPT-4.1-mini, ElevenLabs and Gmail

https://n8nworkflows.xyz/workflows/english-lesson-generator-from-podcasts-with-rss--gpt-4-1-mini--elevenlabs-and-gmail-10968


# English Lesson Generator from Podcasts with RSS, GPT-4.1-mini, ElevenLabs and Gmail

### 1. Workflow Overview

This workflow automates the creation and distribution of an English language learning email based on the latest episode from the BBC "6 Minute English" podcast. It is designed for intermediate English learners (B1-B2 level) and leverages RSS feed data, HTML parsing, AI-powered content generation (GPT-4.1-mini), audio synthesis (ElevenLabs), and Gmail for email delivery.

The workflow is logically divided into the following blocks:

- **1.1 Weekly Trigger & RSS Feed Retrieval:** Scheduled trigger initiates the process weekly; RSS feed is read to get recent podcast episodes.
- **1.2 Archive Check & Duplicate Filtering:** Previously sent episode identifiers are checked to avoid resending old content.
- **1.3 Episode Selection & Content Extraction:** Latest unsent episode is selected; its webpage HTML is fetched and parsed to extract metadata, transcript, vocabulary, and resource links.
- **1.4 AI Content Generation:** Three AI nodes generate a hook message, vocabulary practice exercises, and discussion questions based on the episode content.
- **1.5 Audio Generation:** Vocabulary words are aggregated and sent to ElevenLabs to produce a slow-paced audio track for vocabulary learning.
- **1.6 Email Assembly & Sending:** All generated content is merged into a styled HTML email, attaching the vocabulary audio, and sent via Gmail.

### 2. Block-by-Block Analysis

#### 2.1 Weekly Trigger & RSS Feed Retrieval

- **Overview:**  
  Initiates the workflow every Sunday at 20:00 to fetch the latest episodes from the BBC "6 Minute English" RSS feed.

- **Nodes Involved:**  
  - Trigger Sunday 20:00  
  - Podcasts from RSS Feed

- **Node Details:**

  - **Trigger Sunday 20:00**  
    - Type: Schedule Trigger  
    - Configuration: Runs weekly on Sundays at 20:00 hours.  
    - Inputs: None (start trigger)  
    - Outputs: Triggers "Podcasts from RSS Feed" and "Archived Podcasts" nodes.  
    - Edge Cases: If the n8n instance is offline at trigger time, the run may be missed or delayed.

  - **Podcasts from RSS Feed**  
    - Type: RSS Feed Read  
    - Configuration: Reads BBC 6 Minute English RSS feed URL (`https://feeds.bbci.co.uk/learningenglish/english/features/6-minute-english/rss`).  
    - Inputs: Trigger from schedule node  
    - Outputs: Provides array of recent podcast episodes with metadata.  
    - Edge Cases: RSS feed downtime or format changes; network errors.

#### 2.2 Archive Check & Duplicate Filtering

- **Overview:**  
  Retrieves previously archived podcast GUIDs and merges them with the current RSS feed GUIDs to filter out already sent episodes.

- **Nodes Involved:**  
  - Archived Podcasts  
  - Unique guid(s)  
  - Combine with archived guid(s)  
  - Filter out duplicates

- **Node Details:**

  - **Archived Podcasts**  
    - Type: Data Table  
    - Configuration: Retrieves archived records containing GUIDs of previously processed episodes.  
    - Inputs: Trigger from "Trigger Sunday 20:00" node  
    - Outputs: List of archived GUIDs.  
    - Edge Cases: Data Table misconfiguration or empty archive on first run.

  - **Unique guid(s)**  
    - Type: Aggregate  
    - Configuration: Aggregates unique GUIDs from archived podcasts, outputting a list under the field `guids`.  
    - Inputs: Data from "Archived Podcasts"  
    - Outputs: Unique GUID list.  
    - Edge Cases: Empty input produces empty list.

  - **Combine with archived guid(s)**  
    - Type: Merge (Combine by SQL)  
    - Configuration: Combines current RSS feed items with archived GUIDs for comparison.  
    - Inputs: From "Podcasts from RSS Feed" and "Unique guid(s)"  
    - Outputs: Combined data for filtering.  
    - Edge Cases: Merge failure if inputs are missing.

  - **Filter out duplicates**  
    - Type: Filter  
    - Configuration: Filters out episodes whose GUID exists in the archived GUID list (`{{$json.guids.includes($json.guid)}}` is false to pass).  
    - Inputs: Combined data  
    - Outputs: Passes only new, unsent episodes.  
    - Edge Cases: False negatives if GUIDs are inconsistent; expression failures if fields missing.

#### 2.3 Episode Selection & Content Extraction

- **Overview:**  
  Selects the latest unsent episode, fetches its HTML content, and parses the page to extract useful content such as description, transcript, vocabulary, and download links.

- **Nodes Involved:**  
  - Latest Episode  
  - Extract Podcast Information  
  - Parse HTML

- **Node Details:**

  - **Latest Episode**  
    - Type: Limit  
    - Configuration: Limits data to only the newest episode after filtering.  
    - Inputs: Filtered unsent episodes from "Filter out duplicates"  
    - Outputs: Single episode data.  
    - Edge Cases: Empty input results in no output.

  - **Extract Podcast Information**  
    - Type: HTTP Request  
    - Configuration: Fetches the webpage content (HTML) from the episode URL (`{{$json.link}}`).  
    - Inputs: Latest episode URL  
    - Outputs: Raw HTML content.  
    - Edge Cases: HTTP errors, page not found, or content changes.

  - **Parse HTML**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Cleans HTML text to remove tags and decode entities.  
      - Extracts episode metadata: ID, code, title, description.  
      - Extracts vocabulary list (words and definitions) from "Vocabulary" section.  
      - Extracts transcript paragraphs, filtering out notes.  
      - Extracts download links for PDF, audio, transcript, and podcast.  
    - Inputs: HTML from "Extract Podcast Information"  
    - Outputs: Structured JSON with episode metadata, transcript, vocabulary items, and download links.  
    - Edge Cases: HTML structure changes can break extraction regex; empty or missing sections handled gracefully.

#### 2.4 AI Content Generation

- **Overview:**  
  Uses three AI language model nodes (GPT-4.1-mini) to produce an engaging email hook, practice exercises, and discussion questions based on the parsed episode content.

- **Nodes Involved:**  
  - Model: Email Hook  
  - Model: Practice Exercises  
  - Model: Discussion Questions

- **Node Details:**

  - **Model: Email Hook**  
    - Type: Langchain OpenAI Chat Model  
    - Configuration:  
      - Prompt instructs to create a 3-paragraph hook message with emojis, motivating students to listen.  
      - Uses episode title, description, and vocabulary words in the prompt.  
      - Max tokens: 300; temperature: 0.7  
    - Inputs: Parsed episode JSON  
    - Outputs: Hook message text (HTML paragraphs)  
    - Edge Cases: AI API errors, rate limits.

  - **Model: Practice Exercises**  
    - Type: Langchain OpenAI Chat Model  
    - Configuration:  
      - Prompt to create fill-in-the-blank sentences for each vocabulary word.  
      - Sentences use blanks ("_______") for learners to fill.  
      - Max tokens: 400; temperature: 0.8  
    - Inputs: Parsed episode JSON  
    - Outputs: Practice sentences (plain text, one per line)  
    - Edge Cases: AI API errors, incomplete sentences.

  - **Model: Discussion Questions**  
    - Type: Langchain OpenAI Chat Model  
    - Configuration:  
      - Prompt to generate 5 open-ended, engaging discussion questions using vocabulary and episode content.  
      - Emphasizes personal opinions, critical thinking, and real-life connection.  
      - Max tokens: 400; temperature: 0.9  
    - Inputs: Parsed episode JSON  
    - Outputs: Discussion questions (plain text)  
    - Edge Cases: AI API errors, inappropriate phrasing.

#### 2.5 Audio Generation

- **Overview:**  
  Aggregates vocabulary words and sends them to ElevenLabs to generate a slow-paced audio pronunciation track for learners.

- **Nodes Involved:**  
  - Split Out Vocabulary  
  - Combine Words  
  - Generate Audio Transcript

- **Node Details:**

  - **Split Out Vocabulary**  
    - Type: Split Out  
    - Configuration: Splits the vocabulary items array into individual items for processing.  
    - Inputs: Parsed episode JSON from "Parse HTML"  
    - Outputs: Individual vocabulary word objects.  
    - Edge Cases: Empty vocabulary array.

  - **Combine Words**  
    - Type: Aggregate  
    - Configuration: Aggregates all vocabulary words into a single array under `words`.  
    - Inputs: Split vocabulary items  
    - Outputs: Aggregated words list.  
    - Edge Cases: Aggregation failure if inputs missing.

  - **Generate Audio Transcript**  
    - Type: ElevenLabs Text-to-Speech  
    - Configuration:  
      - Text: comma-separated vocabulary words.  
      - Voice: Selected voice ID "5Q0t7uMcjvnagumLfvZi" (Paul).  
      - Voice settings adjusted for stability, similarity boost, style, speaker boost, and speed (0.8 for slower pace).  
    - Inputs: Aggregated words list  
    - Outputs: Audio file (binary)  
    - Edge Cases: API authentication failure, request limits, audio generation errors.

#### 2.6 Email Assembly & Sending

- **Overview:**  
  Combines the AI-generated hook, exercises, discussion questions, and audio into an HTML email and sends it via Gmail.

- **Nodes Involved:**  
  - Merge All Sections  
  - Prepare Email  
  - Combine Email + Attachment  
  - Send Email

- **Node Details:**

  - **Merge All Sections**  
    - Type: Merge  
    - Configuration: Combines outputs from Email Hook, Practice Exercises, and Discussion Questions into one data object.  
    - Inputs: Outputs of the three AI nodes  
    - Outputs: Combined JSON for email preparation.  
    - Edge Cases: Missing any input breaks merge.

  - **Prepare Email**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Builds a complete HTML email template with inline CSS styling.  
      - Inserts episode title, description, hook message, resource links (audio, transcript, podcast), vocabulary list, practice exercises, and discussion questions.  
      - Formats practice exercises and discussion questions as lists.  
      - Adds a branded footer with links and episode metadata.  
    - Inputs: Combined AI content and parsed episode JSON  
    - Outputs: JSON with `html` email body, `subject`, and episode ID.  
    - Edge Cases: Missing data fields cause empty sections; complex HTML may have rendering issues in some email clients.

  - **Combine Email + Attachment**  
    - Type: Merge  
    - Configuration: Combines the prepared email JSON and the generated audio binary into one payload.  
    - Inputs: From "Prepare Email" and "Generate Audio Transcript"  
    - Outputs: Email content with audio attachment.  
    - Edge Cases: Attachment mismatches or size limits.

  - **Send Email**  
    - Type: Gmail Node  
    - Configuration:  
      - Sends email to configured recipient (requires setup).  
      - Uses HTML content as the message body.  
      - Subject built from episode title.  
      - Attaches vocabulary audio file.  
    - Inputs: Combined email and audio attachment  
    - Outputs: Email send confirmation.  
    - Edge Cases: Authentication errors, quota limits, incorrect recipient address.

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                              | Input Node(s)                    | Output Node(s)                        | Sticky Note                                                                                   |
|---------------------------|--------------------------------|----------------------------------------------|---------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------|
| Trigger Sunday 20:00       | Schedule Trigger                | Weekly trigger to start the workflow         | -                               | Podcasts from RSS Feed, Archived Podcasts | ## 1. Weekly trigger to fetch RSS feed and load archived GUIDs                              |
| Podcasts from RSS Feed     | RSS Feed Read                  | Fetch latest podcast episodes                 | Trigger Sunday 20:00             | Combine with archived guid(s)        |                                                                                              |
| Archived Podcasts          | Data Table                     | Retrieve previously archived podcast GUIDs   | Trigger Sunday 20:00             | Unique guid(s)                      |                                                                                              |
| Unique guid(s)             | Aggregate                     | Aggregate archived GUIDs into list            | Archived Podcasts                | Combine with archived guid(s)        |                                                                                              |
| Combine with archived guid(s) | Merge                       | Combine current and archived GUIDs            | Podcasts from RSS Feed, Unique guid(s) | Filter out duplicates             | ## 2. Filter out podcasts that have already been sent                                        |
| Filter out duplicates      | Filter                        | Remove episodes that have already been sent  | Combine with archived guid(s)   | Latest Episode                     |                                                                                              |
| Latest Episode             | Limit                         | Select latest unsent episode                   | Filter out duplicates            | Extract Podcast Information         | ## 3. Select latest unsent episode and fetch its page content                                |
| Extract Podcast Information | HTTP Request                  | Fetch episode web page HTML                    | Latest Episode                  | Parse HTML                        |                                                                                              |
| Parse HTML                | Code                          | Parse HTML to extract transcript, vocab, etc.| Extract Podcast Information     | Email Hook, Practice Exercises, Discussion Questions, Split Out Vocabulary |                                                                                              |
| Model: Email Hook          | Langchain OpenAI Chat Model    | Generate engaging email hook message          | Parse HTML                     | Merge All Sections                 | ## 4. Generate hook and exercises from the episode content                                  |
| Model: Practice Exercises  | Langchain OpenAI Chat Model    | Generate fill-in-the-blank practice sentences | Parse HTML                     | Merge All Sections                 |                                                                                              |
| Model: Discussion Questions | Langchain OpenAI Chat Model   | Generate discussion questions                  | Parse HTML                     | Merge All Sections                 |                                                                                              |
| Merge All Sections         | Merge                         | Combine AI-generated sections                  | Email Hook, Practice Exercises, Discussion Questions | Prepare Email                    |                                                                                              |
| Prepare Email              | Code                          | Build full styled HTML email content           | Merge All Sections             | Combine Email + Attachment         | ## 6. Build and send the HTML study email with vocabulary audio attached                     |
| Split Out Vocabulary       | Split Out                     | Split vocabulary items array for aggregation   | Parse HTML                     | Combine Words                     | ## 5. Generate vocabulary audio with ElevenLabs                                              |
| Combine Words              | Aggregate                     | Aggregate vocabulary words into list           | Split Out Vocabulary           | Generate Audio Transcript          |                                                                                              |
| Generate Audio Transcript  | ElevenLabs Text-to-Speech      | Generate audio pronunciation of vocab words   | Combine Words                  | Combine Email + Attachment         |                                                                                              |
| Combine Email + Attachment | Merge                         | Combine email HTML and audio attachment        | Prepare Email, Generate Audio Transcript | Send Email                      |                                                                                              |
| Send Email                 | Gmail                         | Send the final email with attachment           | Combine Email + Attachment      | -                                 |                                                                                              |
| Sticky Note                | Sticky Note                   | Notes on building and sending email            | -                             | -                                 | ## 6. Build and send the HTML study email with vocabulary audio attached                     |
| Sticky Note4               | Sticky Note                   | Overview of AI English lesson generator workflow| -                             | -                                 | ## AI English Lesson Generator from Podcasts ... (full setup and customization instructions)|
| Sticky Note3               | Sticky Note                   | Link to YouTube tutorial                        | -                             | -                                 | ## [Check my Tutorial](https://www.youtube.com/watch?v=IKugAVZWgKM)                         |
| Sticky Note5               | Sticky Note                   | Notes on generating hook and exercises          | -                             | -                                 | ## 4. Generate hook and exercises from the episode content                                  |
| Sticky Note6               | Sticky Note                   | Notes on episode selection and content fetch   | -                             | -                                 | ## 3. Select latest unsent episode and fetch its page content                                |
| Sticky Note7               | Sticky Note                   | Notes on filtering out sent episodes            | -                             | -                                 | ## 2. Filter out podcasts that have already been sent                                        |
| Sticky Note8               | Sticky Note                   | Notes on weekly trigger and RSS feed retrieval | -                             | -                                 | ## 1. Weekly trigger to fetch RSS feed and load archived GUIDs                              |
| Sticky Note9               | Sticky Note                   | Notes on vocabulary audio generation            | -                             | -                                 | ## 5. Generate vocabulary audio with ElevenLabs                                              |

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Node Type: Schedule Trigger  
   - Set to run every Sunday at 20:00 (weekly).  
   - No credentials required.

2. **Add an RSS Feed Read node:**  
   - Node Type: RSS Feed Read  
   - URL: `https://feeds.bbci.co.uk/learningenglish/english/features/6-minute-english/rss`  
   - Connect input from Schedule Trigger node.

3. **Add a Data Table node for Archived Podcasts:**  
   - Node Type: Data Table (get operation)  
   - Configure your Data Table with fields: `guid`, `title`, `link`, `processed_date`.  
   - Connect input from Schedule Trigger node.

4. **Add Aggregate node "Unique guid(s)":**  
   - Aggregate archived podcasts by the `guid` field into an array named `guids`.  
   - Input from "Archived Podcasts".

5. **Add Merge node "Combine with archived guid(s)":**  
   - Mode: Combine by SQL  
   - Inputs: RSS Feed Read (current episodes) and Unique guid(s) (archived GUIDs).  
   - Connect outputs accordingly.

6. **Add Filter node "Filter out duplicates":**  
   - Condition: Filter out items where `guid` is included in archived `guids` array. Expression: `{{$json.guids.includes($json.guid)}}` must be false.  
   - Input from Merge node.

7. **Add Limit node "Latest Episode":**  
   - Limit to 1 item (latest episode)  
   - Input from Filter node.

8. **Add HTTP Request node "Extract Podcast Information":**  
   - HTTP Method: GET  
   - URL: `={{ $json.link }}` (dynamic episode URL)  
   - Input from "Latest Episode".

9. **Add Code node "Parse HTML":**  
   - JavaScript code to parse HTML content from HTTP Request node:  
     - Clean HTML text  
     - Extract episode ID, title, description  
     - Extract vocabulary list (word and definition)  
     - Extract transcript paragraphs  
     - Extract download links (audio, PDF, transcript, podcast)  
   - Input from "Extract Podcast Information".

10. **Add three Langchain OpenAI Chat nodes:**  
    - **Model: Email Hook**: Prompt to generate a 3-paragraph hook message.  
    - **Model: Practice Exercises**: Prompt to generate fill-in-the-blank sentences for each vocabulary word.  
    - **Model: Discussion Questions**: Prompt to generate 5 engaging discussion questions.  
    - Use OpenAI GPT-4.1-mini model with appropriate max tokens and temperature settings.  
    - Input for all: output from "Parse HTML".

11. **Add a Merge node "Merge All Sections":**  
    - Inputs: outputs from the three AI nodes (Email Hook, Practice Exercises, Discussion Questions).  
    - Mode: Number of inputs = 3.

12. **Add Code node "Prepare Email":**  
    - JavaScript code to build full HTML email template embedding episode metadata, hook message, resources, vocabulary list, practice exercises, discussion questions, and footer.  
    - Input: output from "Merge All Sections" and episode data from "Parse HTML".

13. **Add Split Out node "Split Out Vocabulary":**  
    - Field to split: `vocabulary_items` array from "Parse HTML".  
    - Input: from "Parse HTML".

14. **Add Aggregate node "Combine Words":**  
    - Aggregate all split vocabulary items by their `word` field into an array named `words`.  
    - Input: from "Split Out Vocabulary".

15. **Add ElevenLabs Text-to-Speech node "Generate Audio Transcript":**  
    - Text: `={{ $json.words.join(', ') }}`  
    - Voice: select a voice ID (e.g., "5Q0t7uMcjvnagumLfvZi" for Paul).  
    - Configure voice settings for stability, speed (0.8 for slow), and style.  
    - Input: from "Combine Words".

16. **Add Merge node "Combine Email + Attachment":**  
    - Combine mode: Combine by position  
    - Inputs: "Prepare Email" output and "Generate Audio Transcript" output.  
    - Output: combined email JSON and audio attachment.

17. **Add Gmail node "Send Email":**  
    - Configure Gmail credentials with OAuth2.  
    - Destination email address: set your recipient.  
    - Subject: `={{ $('Combine Email + Attachment').item.json.subject }}`  
    - Message: `={{ $json.html }}` (from merged email)  
    - Attach the binary audio file.  
    - Input from "Combine Email + Attachment".

18. **Connect nodes as per described flow:**  
    - Trigger -> RSS Feed and Archived Podcasts  
    - Archived Podcasts -> Unique guid(s) -> Combine with archived guid(s)  
    - RSS Feed -> Combine with archived guid(s)  
    - Combine with archived guid(s) -> Filter out duplicates -> Latest Episode -> Extract Podcast Information -> Parse HTML  
    - Parse HTML -> Email Hook, Practice Exercises, Discussion Questions, Split Out Vocabulary  
    - AI nodes -> Merge All Sections -> Prepare Email  
    - Split Out Vocabulary -> Combine Words -> Generate Audio Transcript  
    - Prepare Email + Generate Audio Transcript -> Combine Email + Attachment -> Send Email

19. **Credentials Setup:**  
    - OpenAI API credentials for AI nodes (GPT-4.1-mini).  
    - ElevenLabs API key for Text-to-Speech node.  
    - Gmail OAuth2 credentials for sending email.

20. **Optional:**  
    - Create and configure the Data Table for archived podcasts.  
    - Customize email HTML template in "Prepare Email" node as needed.  
    - Adjust schedule trigger timing or frequency as desired.

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow inspired by BBC 6 Minute English podcast for English learning automation with AI and audio synthesis.                          |                                                                                               |
| Setup instructions and customization tips are included in the Sticky Note4 node content within the workflow.                             |                                                                                               |
| Video walkthrough available: [Check my Tutorial](https://www.youtube.com/watch?v=IKugAVZWgKM)                                            | Linked in Sticky Note3                                                                          |
| The workflow requires an n8n Data Table configured to store and retrieve archived podcast GUIDs to prevent duplicated sends.             | Important for filtering logic                                                                  |
| AI prompts are designed to target intermediate English learners (B1-B2) for vocabulary practice, discussion, and engagement hooks.       |                                                                                               |
| ElevenLabs voice settings adjust speed and style to generate clear, slow-paced vocabulary pronunciation audio.                           |                                                                                               |
| Gmail node requires OAuth2 credentials and correct recipient email configuration before sending emails.                                   |                                                                                               |
| The HTML email template uses responsive design and inline styling to ensure compatibility with major email clients.                      |                                                                                               |

---

**Disclaimer:** The text provided here is exclusively derived from an automated workflow created with n8n, complying fully with content policies and legal standards. All data processed is publicly available and lawful.