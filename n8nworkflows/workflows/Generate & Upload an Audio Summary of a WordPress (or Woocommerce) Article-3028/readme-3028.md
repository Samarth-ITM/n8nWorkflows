Generate & Upload an Audio Summary of a WordPress (or Woocommerce) Article

https://n8nworkflows.xyz/workflows/generate---upload-an-audio-summary-of-a-wordpress--or-woocommerce--article-3028


# Generate & Upload an Audio Summary of a WordPress (or Woocommerce) Article

### 1. Workflow Overview

This workflow automates the generation and upload of an audio summary or transcription of a WordPress (or WooCommerce) article. It is designed to improve content accessibility and engagement by converting article text into speech and embedding the resulting audio directly into the WordPress post.

**Target Use Cases:**  
- Content creators who want to provide audio versions of their articles.  
- Website owners aiming to enhance accessibility for users who prefer listening over reading.  
- E-commerce or affiliate marketers who want to highlight product benefits via audio summaries.

**Logical Blocks:**  
- **1.1 Trigger & Initialization:** Manual start and setting up essential variables.  
- **1.2 Article Retrieval:** Fetching the WordPress article content by post ID.  
- **1.3 Text Processing (Summary/Transcription):** Using an LLM (GPT-4o-mini) to generate a summary or transcription based on the article content.  
- **1.4 Text-to-Speech Generation:** Converting the processed text into an MP3 audio file via Eleven Labs API.  
- **1.5 Audio Upload:** Uploading the generated MP3 file to WordPress media library.  
- **1.6 Post Update:** Embedding the audio player into the original WordPress post content.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

- **Overview:**  
  Starts the workflow manually and sets the base URL for the WordPress site.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - settings

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Configuration: No parameters; triggers on user action.  
    - Inputs: None  
    - Outputs: Triggers the next node (`settings`).  
    - Edge Cases: None typical; manual trigger ensures controlled execution.

  - **settings**  
    - Type: Set  
    - Role: Defines workflow-level variables, specifically the WordPress site URL.  
    - Configuration: Sets `site_url` to `"https://mydomain.com/"` (replace with actual domain).  
    - Inputs: From manual trigger  
    - Outputs: Passes data to `Retrieve WordPress Article`.  
    - Edge Cases: Incorrect URL will cause failures in subsequent HTTP requests.

---

#### 1.2 Article Retrieval

- **Overview:**  
  Retrieves the WordPress article content using the WordPress API and a specified post ID.

- **Nodes Involved:**  
  - Retrieve WordPress Article

- **Node Details:**

  - **Retrieve WordPress Article**  
    - Type: WordPress  
    - Role: Fetches article data by post ID (1032 in this example).  
    - Configuration:  
      - Operation: `get`  
      - Post ID: `1032` (hardcoded; can be parameterized)  
      - Credentials: WordPress API credentials configured in n8n.  
    - Inputs: From `settings` node  
    - Outputs: Article JSON including content, slug, and ID.  
    - Edge Cases:  
      - Invalid post ID or permissions errors.  
      - Network or authentication failures.  
      - Missing content field may cause downstream issues.

---

#### 1.3 Text Processing (Summary/Transcription)

- **Overview:**  
  Processes the article content using GPT-4o-mini to generate either a summary or full transcription, depending on the prompt.

- **Nodes Involved:**  
  - Generate Summary or Transcription  
  - OpenAI Chat Model  
  - Sticky Note (prompt customization guidance)

- **Node Details:**

  - **Generate Summary or Transcription**  
    - Type: LangChain LLM Chain  
    - Role: Sends article content to the LLM for summarization or transcription.  
    - Configuration:  
      - Input text: `{{$json.content}}` (article content from previous node)  
      - Prompt: `"Summarize or transcribe this article, depending on the workflow setting."`  
      - Output parser enabled to handle LLM response cleanly.  
    - Inputs: Article content JSON from `Retrieve WordPress Article`  
    - Outputs: Processed text (summary or transcription) in `$json.text`.  
    - Edge Cases:  
      - LLM API failures or rate limits.  
      - Unexpected output formats (HTML or malformed text).  
      - Empty or very short article content.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the GPT-4o-mini model for text generation.  
    - Configuration:  
      - Model: `gpt-4o-mini`  
      - Credentials: OpenAI API key configured in n8n.  
    - Inputs: Receives prompt from `Generate Summary or Transcription` node.  
    - Outputs: LLM-generated text to `Generate Summary or Transcription`.  
    - Edge Cases: API key invalid, quota exceeded, or network issues.

  - **Sticky Note (Prompt Customization)**  
    - Content: Guidance on modifying the prompt to switch between summary and transcription, or customize for e-commerce benefits.  
    - Role: Documentation aid for users to tailor AI output.

---

#### 1.4 Text-to-Speech Generation

- **Overview:**  
  Converts the processed text into an MP3 audio file using Eleven Labs Text-to-Speech API.

- **Nodes Involved:**  
  - Generate Speech  
  - Sticky Note1 (Eleven Labs API usage instructions)

- **Node Details:**

  - **Generate Speech**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Eleven Labs API to generate speech audio.  
    - Configuration:  
      - URL: `https://api.elevenlabs.io/v1/text-to-speech/voice_id` (replace `voice_id` with actual voice identifier)  
      - Method: POST  
      - Authentication: Custom HTTP header with Eleven Labs API key (`xi-api-key`)  
      - Body parameters:  
        - `text`: `{{$json.text}}` (text from previous node)  
        - `model_id`: `"eleven_multilingual_v2"` (multilingual voice model)  
        - `output_format`: `"mp3_44100_128"` (MP3 format, 44.1kHz, 128kbps)  
    - Inputs: Text from `Generate Summary or Transcription`  
    - Outputs: Binary MP3 data for upload.  
    - Edge Cases:  
      - Invalid API key or voice_id.  
      - API rate limits or service downtime.  
      - Large text input causing timeouts or truncation.

  - **Sticky Note1 (Eleven Labs API)**  
    - Content: Detailed instructions on setting up Eleven Labs API credentials, choosing voices, model IDs, and output formats.  
    - Includes links:  
      - [Eleven Labs Text-to-Speech Demo](https://try.elevenlabs.io/text-audio)  
      - [Eleven Labs API Reference](https://try.elevenlabs.io/api-reference-text-to-speech)  
    - Role: User guidance for configuring Eleven Labs integration.

---

#### 1.5 Audio Upload

- **Overview:**  
  Uploads the generated MP3 audio file to the WordPress media library via REST API.

- **Nodes Involved:**  
  - Upload MP3

- **Node Details:**

  - **Upload MP3**  
    - Type: HTTP Request  
    - Role: Sends the MP3 binary data as a media upload to WordPress.  
    - Configuration:  
      - URL: `={{ $('settings').item.json['site_url'] }}wp-json/wp/v2/media` (dynamic site URL)  
      - Method: POST  
      - Content-Type: Binary data  
      - Headers:  
        - `Content-Disposition`: `attachment; filename="{{ $('Retrieve WordPress Article').item.json.slug }}.mp3"` (filename based on article slug)  
      - Authentication: WordPress API credentials  
      - Retry on failure enabled for robustness.  
      - Input data field: `data` (binary MP3 from previous node)  
    - Inputs: Binary MP3 from `Generate Speech`  
    - Outputs: JSON response with media ID and URL for the uploaded audio.  
    - Edge Cases:  
      - Authentication errors or permission issues.  
      - Network failures or timeouts.  
      - Filename conflicts or invalid characters.

---

#### 1.6 Post Update

- **Overview:**  
  Updates the original WordPress post to embed an audio player referencing the uploaded MP3 file.

- **Nodes Involved:**  
  - Update WordPress Post

- **Node Details:**

  - **Update WordPress Post**  
    - Type: WordPress  
    - Role: Updates the post content to include an audio block with the uploaded MP3.  
    - Configuration:  
      - Operation: `update`  
      - Post ID: Dynamic, from `Retrieve WordPress Article` (`{{$json.id}}`)  
      - Content:  
        - Embeds WordPress audio block referencing the uploaded media ID and URL.  
        - Includes a caption: "🗣️ Listen to the summary or transcription. 👆"  
        - Appends original article content after the audio block.  
      - Credentials: WordPress API credentials  
    - Inputs: Media upload response from `Upload MP3` and original article data  
    - Outputs: Updated post JSON response  
    - Edge Cases:  
      - Permission denied errors.  
      - Content formatting issues causing rendering problems.  
      - Post ID mismatch or update conflicts.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                          |
|-----------------------------|--------------------------------|----------------------------------------|------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                 | Manual start of workflow                | None                         | settings                       |                                                                                                    |
| settings                    | Set                            | Defines site URL variable               | When clicking ‘Test workflow’ | Retrieve WordPress Article     |                                                                                                    |
| Retrieve WordPress Article  | WordPress                      | Fetches article content by post ID     | settings                     | Generate Summary or Transcription |                                                                                                    |
| OpenAI Chat Model           | LangChain OpenAI Chat Model    | Provides GPT-4o-mini model for LLM     | Generate Summary or Transcription (ai_languageModel) | Generate Summary or Transcription (ai_languageModel) |                                                                                                    |
| Generate Summary or Transcription | LangChain LLM Chain          | Generates summary or transcription text | Retrieve WordPress Article, OpenAI Chat Model | Generate Speech               | # Modify This Prompt: Guidance on customizing AI prompt for summary or transcription.              |
| Generate Speech             | HTTP Request                   | Converts text to MP3 via Eleven Labs   | Generate Summary or Transcription | Upload MP3                    | 🎙️ Generate Text-to-Speech Using Eleven Labs via API with setup instructions and links.           |
| Upload MP3                  | HTTP Request                   | Uploads MP3 audio to WordPress media   | Generate Speech              | Update WordPress Post          |                                                                                                    |
| Update WordPress Post       | WordPress                      | Embeds audio player and updates post   | Upload MP3                   | None                         |                                                                                                    |
| Sticky Note                 | Sticky Note                    | Prompt customization guidance          | None                         | None                         | # Modify This Prompt: Guidance on prompt customization for summary or transcription.               |
| Sticky Note1                | Sticky Note                    | Eleven Labs API usage instructions     | None                         | None                         | 🎙️ Generate Text-to-Speech Using Eleven Labs via API with setup instructions and links.           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create Set Node (settings)**  
   - Type: Set  
   - Add string variable `site_url` with value `"https://mydomain.com/"` (replace with your WordPress site URL).  
   - Connect Manual Trigger → Set.

3. **Create WordPress Node (Retrieve WordPress Article)**  
   - Type: WordPress  
   - Operation: `get`  
   - Post ID: `1032` (or parameterize as needed)  
   - Credentials: Configure WordPress API credentials (OAuth2 or Application Password).  
   - Connect Set → Retrieve WordPress Article.

4. **Create LangChain OpenAI Chat Model Node (OpenAI Chat Model)**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Credentials: Configure OpenAI API key.  
   - No direct input connection; will be linked via AI language model input.

5. **Create LangChain LLM Chain Node (Generate Summary or Transcription)**  
   - Type: LangChain LLM Chain  
   - Parameters:  
     - Text input: `={{ $json.content }}` (article content)  
     - Prompt: `"Summarize or transcribe this article, depending on the workflow setting."`  
     - Enable output parser.  
   - Connect Retrieve WordPress Article → Generate Summary or Transcription (main input).  
   - Connect OpenAI Chat Model → Generate Summary or Transcription (ai_languageModel input).

6. **Create HTTP Request Node (Generate Speech)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.elevenlabs.io/v1/text-to-speech/voice_id` (replace `voice_id` with actual voice ID from Eleven Labs)  
   - Authentication: Custom HTTP header authentication with key `xi-api-key` set to your Eleven Labs API key.  
   - Body parameters (JSON):  
     - `text`: `={{ $json.text }}` (text from previous node)  
     - `model_id`: `"eleven_multilingual_v2"`  
     - `output_format`: `"mp3_44100_128"`  
   - Connect Generate Summary or Transcription → Generate Speech.

7. **Create HTTP Request Node (Upload MP3)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `={{ $('settings').item.json['site_url'] }}wp-json/wp/v2/media`  
   - Authentication: WordPress API credentials (same as Retrieve WordPress Article).  
   - Content-Type: Binary data  
   - Header: `Content-Disposition` set to `attachment; filename="{{ $('Retrieve WordPress Article').item.json.slug }}.mp3"`  
   - Input Data Field Name: `data` (binary data from Generate Speech)  
   - Enable retry on failure.  
   - Connect Generate Speech → Upload MP3.

8. **Create WordPress Node (Update WordPress Post)**  
   - Type: WordPress  
   - Operation: `update`  
   - Post ID: `={{ $('Retrieve WordPress Article').item.json.id }}`  
   - Update Fields → Content:  
     ```
     <!-- wp:audio {"id":{{ $json.id }}} -->
     <figure class="wp-block-audio"><audio controls src="{{ $json.guid.rendered }}"></audio><figcaption class="wp-element-caption">🗣️ Listen to the summary or transcription. 👆</figcaption></figure>
     <!-- /wp:audio --><br>{{ $('Retrieve WordPress Article').item.json.content.rendered }}
     ```  
   - Credentials: WordPress API credentials  
   - Connect Upload MP3 → Update WordPress Post.

9. **Optional: Add Sticky Notes**  
   - Add notes for prompt customization and Eleven Labs API instructions for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Obtain Eleven Labs API key and test voices at: [Eleven Labs Text-to-Speech Demo](https://try.elevenlabs.io/text-audio) | Eleven Labs API key and voice selection guide                                                      |
| Eleven Labs API documentation: [API Reference - Eleven Labs](https://try.elevenlabs.io/api-reference-text-to-speech) | Full API reference for text-to-speech integration                                                  |
| Modify AI prompt to switch between summary and transcription or customize for e-commerce use cases.           | Prompt customization guidance in Sticky Note                                                       |
| Configure WordPress API credentials in n8n for both reading and writing posts/media.                          | WordPress REST API authentication setup                                                           |
| Use retry on failure for media upload to handle transient network or server issues.                            | Robustness improvement for media upload node                                                      |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and customizing the workflow to generate and upload audio summaries or transcriptions of WordPress articles using n8n, OpenAI, and Eleven Labs APIs.