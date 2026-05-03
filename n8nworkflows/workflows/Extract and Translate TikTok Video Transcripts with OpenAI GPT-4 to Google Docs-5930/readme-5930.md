Extract and Translate TikTok Video Transcripts with OpenAI GPT-4 to Google Docs

https://n8nworkflows.xyz/workflows/extract-and-translate-tiktok-video-transcripts-with-openai-gpt-4-to-google-docs-5930


# Extract and Translate TikTok Video Transcripts with OpenAI GPT-4 to Google Docs

### 1. Workflow Overview

This n8n workflow automates the extraction, AI-driven processing, and documentation of TikTok video transcripts. It is designed for users who want to convert TikTok video audio into text, interpret or translate that text using OpenAI GPT-4, and save the results in Google Docs for easy access and sharing.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user input through a form, accepting a TikTok video URL and the desired output language.
- **1.2 Transcript Retrieval:** Uses the TikTok Transcript API to fetch the video's subtitles.
- **1.3 Synchronization Delay:** Introduces a wait/pause to ensure the transcript data is fully available before proceeding.
- **1.4 AI Processing:** Sends the transcript to OpenAI GPT-4 via an API for language-sensitive interpretation, summarization, or translation.
- **1.5 Documentation:** Updates a Google Docs document with the processed transcript output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block captures user input through a web form, which triggers the workflow. It collects the TikTok video URL and the target language for AI processing.

**Nodes Involved:**  
- On form submission

**Node Details:**

- **On form submission**  
  - Type: Form Trigger node (event-trigger)  
  - Configuration:  
    - Form titled "Tiktok Transcript"  
    - Two mandatory fields: "url" (TikTok video URL) and "language" (target language for AI response)  
  - Key Expressions/Variables: User inputs accessible as `$json.url` and `$json.language`  
  - Input: External form submission (HTTP webhook)  
  - Output: Passes user input JSON downstream  
  - Version-specific: Uses webhookId for unique webhook endpoint  
  - Potential Failures: Form validation failure if required fields are empty; webhook unavailability or network issues  
  - Sub-workflows: None

#### 1.2 Transcript Retrieval

**Overview:**  
Fetches the TikTok video transcript using a third-party RapidAPI service specialized for TikTok subtitles.

**Nodes Involved:**  
- Tiktok Transcript

**Node Details:**

- **Tiktok Transcript**  
  - Type: HTTP Request node (API call)  
  - Configuration:  
    - POST request to `https://tiktok-transcript-ai.p.rapidapi.com/tiktok/index.php`  
    - Body parameter: `url` extracted from the form input (`={{ $json.url }}`)  
    - Headers: RapidAPI host and API key for authentication  
  - Input: JSON with video URL from the form submission node  
  - Output: JSON containing video transcript data (key: `subtitles`)  
  - Version-specific: Requires valid RapidAPI key and endpoint  
  - Potential Failures: API key invalid or quota exceeded, network timeout, invalid video URL, empty or missing transcript data  
  - Sub-workflows: None

#### 1.3 Synchronization Delay

**Overview:**  
Adds a wait time to ensure the transcript retrieval completes and data is stable before the next API call.

**Nodes Involved:**  
- Wait

**Node Details:**

- **Wait**  
  - Type: Wait node (delay)  
  - Configuration: Default wait time (not explicitly set, presumably short)  
  - Input: Transcript data from TikTok Transcript node  
  - Output: Passes data downstream after delay  
  - Version-specific: None  
  - Potential Failures: None typical; excessively long wait may delay workflow unnecessarily  
  - Sub-workflows: None

#### 1.4 AI Processing

**Overview:**  
Sends the TikTok transcript to OpenAI GPT-4 API via RapidAPI for processing. The AI uses the selected language to generate a meaningful response such as summarization, translation, or interpretation.

**Nodes Involved:**  
- Open AI

**Node Details:**

- **Open AI**  
  - Type: HTTP Request node (API call)  
  - Configuration:  
    - POST request to `https://openai-gpt-4o-mini.p.rapidapi.com/chatAI/gpt-4-1-mini.php`  
    - Request body: JSON containing a chat conversation with system and user roles  
      - System content instructs AI to be helpful and respond in the language specified by the form input (`{{ $('On form submission').item.json.language }}`)  
      - User content includes the transcript text (`{{ $json.subtitles.replace(/\"/g, '\\\"').replace(/\n/g, '\\n') }}`)  
    - Headers: RapidAPI host and API key for OpenAI GPT-4 service  
  - Input: JSON with subtitles from Wait node  
  - Output: AI-generated text response  
  - Version-specific: Requires valid RapidAPI key for OpenAI GPT-4 API  
  - Potential Failures: API quota exceeded, invalid API key, malformed request due to expression errors, network issues, or AI service downtime  
  - Sub-workflows: None

#### 1.5 Documentation

**Overview:**  
Updates a Google Docs document by inserting the processed text from OpenAI, enabling easy sharing and persistent storage.

**Nodes Involved:**  
- Google Docs

**Node Details:**

- **Google Docs**  
  - Type: Google Docs node (Google API integration)  
  - Configuration:  
    - Operation: Update document by inserting content  
    - Credentials: Uses OAuth2 credentials for Google Docs (`Google Docs account 2`)  
    - No explicit document ID or insertion text in provided data; presumably configured in node parameters with dynamic content from Open AI output  
  - Input: AI-generated processed text from Open AI node  
  - Output: Confirmation of document update  
  - Version-specific: Requires OAuth2 credentials with Google Docs API enabled  
  - Potential Failures: OAuth token expiry, insufficient permissions, document ID errors, API rate limits, network errors  
  - Sub-workflows: None

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                              | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                                 |
|---------------------|---------------------|----------------------------------------------|-----------------------|-----------------------|-------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger        | Capture TikTok URL and language from user form | External trigger      | Tiktok Transcript     | Triggers on user form submission with URL and language inputs                                               |
| Tiktok Transcript    | HTTP Request        | Fetch TikTok video transcript from RapidAPI   | On form submission    | Wait                  | Fetches transcript via TikTok Transcript API                                                                |
| Wait                | Wait                | Delay to ensure transcript retrieval completes | Tiktok Transcript     | Open AI               | Adds delay to prevent race conditions                                                                        |
| Open AI             | HTTP Request        | Process transcript with OpenAI GPT-4 API       | Wait                  | Google Docs           | Sends transcript to OpenAI GPT-4 API for language-aware processing                                           |
| Google Docs         | Google Docs         | Update Google Doc with processed AI output     | Open AI               | None                  | Saves AI-generated content into Google Docs                                                                  |
| Sticky Note         | Sticky Note         | Workflow overview and explanation               | None                  | None                  | # TikTok Transcriptor overview and benefits                                                                 |
| Sticky Note1        | Sticky Note         | Explains On form submission node                | None                  | None                  | Details about form submission node                                                                           |
| Sticky Note2        | Sticky Note         | Explains Tiktok Transcript node                 | None                  | None                  | Details about transcript retrieval node                                                                      |
| Sticky Note3        | Sticky Note         | Explains Wait node                               | None                  | None                  | Details about wait/delay node                                                                                 |
| Sticky Note4        | Sticky Note         | Explains Open AI node                            | None                  | None                  | Details about OpenAI GPT-4 processing node                                                                   |
| Sticky Note5        | Sticky Note         | Explains Google Docs node                        | None                  | None                  | Details about Google Docs update node                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Set form title: "Tiktok Transcript"  
   - Add two required fields:  
     - `url` (text)  
     - `language` (text)  
   - Save and note the webhook URL generated for later testing.

2. **Add HTTP Request Node ("Tiktok Transcript"):**  
   - Connect input from "On form submission" node.  
   - Method: POST  
   - URL: `https://tiktok-transcript-ai.p.rapidapi.com/tiktok/index.php`  
   - Body parameters:  
     - `url`: Use expression `{{$json["url"]}}`  
   - Headers:  
     - `x-rapidapi-host`: `tiktok-transcript-ai.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key for TikTok Transcript API  
   - Ensure "Send Body" and "Send Headers" options are enabled.  
   - Set response format to JSON.

3. **Add Wait Node ("Wait"):**  
   - Connect input from "Tiktok Transcript" node.  
   - Use default wait duration (or specify a short delay, e.g., 1-3 seconds) to ensure API data readiness.

4. **Add HTTP Request Node ("Open AI"):**  
   - Connect input from "Wait" node.  
   - Method: POST  
   - URL: `https://openai-gpt-4o-mini.p.rapidapi.com/chatAI/gpt-4-1-mini.php`  
   - Headers:  
     - `x-rapidapi-host`: `openai-gpt-4o-mini.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key for OpenAI GPT-4 API  
   - Body (JSON, with expressions):  
     ```json
     {
       "temperature": 0.7,
       "messages": [
         {
           "role": "system",
           "content": "You are a helpful assistant. Your task is to process the subtitles provided to you and return a meaningful response based on the content. The subtitles may be in various languages. Please handle them accordingly and respond in the {{ $('On form submission').item.json.language }} language."
         },
         {
           "role": "user",
           "content": "{{ $json.subtitles.replace(/\"/g, '\\\"').replace(/\\n/g, '\\n') }}"
         }
       ]
     }
     ```  
   - Enable "Send Body" and "Send Headers" with JSON specification.

5. **Add Google Docs Node ("Google Docs"):**  
   - Connect input from "Open AI" node.  
   - Credentials: Set up or select OAuth2 credentials with Google Docs API access.  
   - Operation: "Update" document by inserting text.  
   - Configure document ID (static or dynamic) where the processed text will be inserted.  
   - Insert content: Use the AI response text from the previous node. Expression example: `{{$json.choices[0].message.content}}` (adjust path based on actual response structure).  
   - Save node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow leverages the TikTok Transcript API and OpenAI GPT-4 API via RapidAPI for seamless integration and scalability. | https://rapidapi.com/skdeveloper/api/tiktok-transcript-ai and https://rapidapi.com/skdeveloper/api/openai-gpt-4o-mini |
| Google Docs integration requires OAuth2 credentials with permission to edit documents. Ensure the Google project has Google Docs API enabled. | https://developers.google.com/docs/api |
| The workflow supports multiple languages by passing the user-selected language to the AI system prompt, enabling translation or localization. | Workflow design notes |
| Delay node ("Wait") avoids race conditions ensuring transcript data is ready before AI processing. Adjust wait time based on API response speed. | Workflow design notes |
| Replace placeholder API keys (`your key`) in HTTP Request nodes with valid RapidAPI keys before running the workflow. | Security best practice |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow. It complies fully with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and publicly accessible.