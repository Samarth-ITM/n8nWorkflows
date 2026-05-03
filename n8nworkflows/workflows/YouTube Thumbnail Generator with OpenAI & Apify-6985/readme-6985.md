YouTube Thumbnail Generator with OpenAI & Apify

https://n8nworkflows.xyz/workflows/youtube-thumbnail-generator-with-openai---apify-6985


# YouTube Thumbnail Generator with OpenAI & Apify

---

### 1. Workflow Overview

This workflow automates the generation of a YouTube thumbnail image using a combination of input URL capture, data extraction from YouTube via Apify actors, AI-powered prompt generation with OpenAI GPT-4o, and image creation with DALL·E, followed by image resizing for proper YouTube thumbnail dimensions. It targets creators, marketers, or automation engineers who want to streamline thumbnail creation based on video metadata and transcripts.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception & Parameter Preparation**: Receives a YouTube URL via a web form, transforms it into a format compatible with Apify actors.
- **1.2 Data Extraction via Apify**: Queries two Apify actors — one to scrape video metadata and another to extract transcripts.
- **1.3 AI Prompt & Image Generation**: Uses OpenAI GPT-4o to generate a DALL·E prompt from the transcript and title, then creates the thumbnail image.
- **1.4 Image Post-processing**: Resizes the generated image to 1280x720 pixels, the recommended YouTube thumbnail size.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Parameter Preparation

- **Overview:**  
  This block collects the YouTube URL via a user-facing form and structures it into an array format required by downstream Apify API calls.

- **Nodes Involved:**  
  - Get URL (Form Trigger)  
  - Assign parameters (Set)

- **Node Details:**

  - **Get URL**  
    - *Type & Role:* Form Trigger; collects user input via a web form.  
    - *Configuration:* Single required field labeled “Youtube URL” with placeholder text for a typical YouTube video link.  
    - *Key Expressions/Variables:* The submitted URL is accessible as `$json['Youtube URL']`.  
    - *Connections:* Outputs to `Assign parameters`.  
    - *Failure Modes:* User submits invalid URL; form submission issues; webhook connectivity failures.

  - **Assign parameters**  
    - *Type & Role:* Set node; converts the single URL string into an array of objects for Apify.  
    - *Configuration:* Sets `startUrls` to an array containing the submitted YouTube URL as an object `{ url: <URL> }`.  
    - *Key Expressions:* `=[ "{{ $json['Youtube URL'] }}" ]` converted to `[{ url: "<URL>" }]`.  
    - *Connections:* Outputs to `Query Metadata`.  
    - *Failure Modes:* Expression evaluation errors if input missing or malformed.

#### 2.2 Data Extraction via Apify

- **Overview:**  
  This block calls two Apify actors sequentially to gather detailed metadata and transcripts for the given YouTube video URL.

- **Nodes Involved:**  
  - Query Metadata (HTTP Request)  
  - Query Transcript (HTTP Request)

- **Node Details:**

  - **Query Metadata**  
    - *Type & Role:* HTTP Request node; POST request to Apify’s YouTube Scraper actor.  
    - *Configuration:*  
      - URL: `https://api.apify.com/v2/acts/streamers~youtube-scraper/run-sync-get-dataset-items`  
      - JSON body includes multiple filters (e.g., `isHD`, `hasCC`, etc.) all set to false or zero limits except `maxResults`=1.  
      - Sends a single `startUrls` object with the YouTube URL from the form.  
      - Authenticates using HTTP Query Auth with Apify API token credential.  
    - *Key Expressions:* URL and body parameters include `{{ $('Get URL').item.json['Youtube URL'] }}` to dynamically insert the user input.  
    - *Connections:* Outputs to `Query Transcript`.  
    - *Failure Modes:* Authentication failure, network timeouts, API rate limits, invalid or unavailable video URL.

  - **Query Transcript**  
    - *Type & Role:* HTTP Request node; POST request to Apify’s YouTube Transcript Scraper actor.  
    - *Configuration:*  
      - URL: `https://api.apify.com/v2/acts/topaz_sharingan~Youtube-Transcript-Scraper/run-sync-get-dataset-items`  
      - Body parameter `startUrls` is fed from the `Assign parameters` node’s `startUrls` array.  
      - Uses the same Apify HTTP Query Auth credential.  
    - *Key Expressions:* `={{ $('Assign parameters').item.json.startUrls }}` passes the URL array.  
    - *Connections:* Outputs to `Image Prompt Generator`.  
    - *Failure Modes:* Transcript unavailable, API errors, auth issues.

#### 2.3 AI Prompt & Image Generation

- **Overview:**  
  This block uses OpenAI GPT-4o to analyze the video transcript and title, generating a detailed prompt for DALL·E to create a minimalist YouTube thumbnail illustration. It then calls DALL·E to generate the image.

- **Nodes Involved:**  
  - Image Prompt Generator (OpenAI)  
  - Create Image (OpenAI)

- **Node Details:**

  - **Image Prompt Generator**  
    - *Type & Role:* OpenAI node specialized for chat completion (GPT-4o) to create an image prompt.  
    - *Configuration:*  
      - Model: GPT-4o (an extended GPT-4 variant supporting image generation prompt crafting).  
      - Message template includes instructions to generate a minimalist illustration prompt without text, referencing the transcript and video title from previous nodes.  
      - Outputs JSON with the generated prompt.  
    - *Key Expressions:*  
      - Transcript: `{{ $json.max_transcript }}` (from the transcript scraping node output)  
      - Video Title: `{{ $json.videoTitle }}` (from metadata scraping output)  
    - *Connections:* Outputs prompt JSON to `Create Image`.  
    - *Failure Modes:* API key invalid or rate limited, malformed prompt, missing input data.

  - **Create Image**  
    - *Type & Role:* OpenAI node for image generation using DALL·E.  
    - *Configuration:*  
      - Resource: `image`  
      - Model: DALL·E (default with GPT-4o API key)  
      - Prompt: dynamically taken from `Image Prompt Generator` output.  
      - Image size: 1792x1024 pixels (larger than final YouTube thumbnail size for quality).  
    - *Connections:* Outputs generated image to `Resize Image`.  
    - *Failure Modes:* API limits, invalid prompt, generation timeout.

#### 2.4 Image Post-processing

- **Overview:**  
  This block resizes the generated image to the official YouTube thumbnail dimensions (1280x720) without maintaining aspect ratio, ensuring the image fits the platform’s requirements.

- **Nodes Involved:**  
  - Resize Image (Edit Image)

- **Node Details:**

  - **Resize Image**  
    - *Type & Role:* Edit Image node; performs image resizing locally within n8n.  
    - *Configuration:*  
      - Dimensions: width = 1280 px, height = 720 px  
      - Resize option: `ignoreAspectRatio` set to true (forces exact size regardless of original proportions)  
      - Output format: JPEG  
      - Filename: dynamically set to the video title from transcript metadata.  
    - *Connections:* Terminal node (no outputs).  
    - *Failure Modes:* Image data corruption, unsupported format, filename expression failure.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                        |
|---------------------|----------------------------|------------------------------------|-----------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Get URL             | Form Trigger               | Collect YouTube URL via web form   | —                     | Assign parameters        | ### 🔧 Step-by-Step Setup 1️⃣ Form & Parameter Assignment: collects URL via form and converts for Apify                         |
| Assign parameters    | Set                        | Convert URL string to array format | Get URL                | Query Metadata           | ### 🔧 Step-by-Step Setup 1️⃣ Form & Parameter Assignment: converts input URL into array                                          |
| Query Metadata      | HTTP Request                | Call Apify YouTube metadata actor  | Assign parameters       | Query Transcript         | #### 2️⃣ Apify Actors for Data Extraction: calls YouTube-Scraper actor with filters                                               |
| Query Transcript     | HTTP Request                | Call Apify transcript scraper actor| Query Metadata          | Image Prompt Generator   | #### 2️⃣ Apify Actors for Data Extraction: calls transcript scraper actor                                                        |
| Image Prompt Generator | OpenAI (GPT-4o)           | Generate DALL·E prompt from text   | Query Transcript        | Create Image             | #### 3️⃣ OpenAI GPT-4o & DALL·E Generation: creates image prompt from transcript and title                                        |
| Create Image         | OpenAI (DALL·E)            | Generate thumbnail image           | Image Prompt Generator  | Resize Image             | #### 3️⃣ OpenAI GPT-4o & DALL·E Generation: generates image using DALL·E                                                          |
| Resize Image         | Edit Image                 | Resize image to YouTube thumbnail  | Create Image            | —                       | #### 4️⃣ Resize for YouTube Format: resizes image to 1280x720 ignoring aspect ratio                                                |
| Sticky Note          | Sticky Note                 | Documentation and guidance         | —                       | —                        | ### 🔧 Step-by-Step Setup 1️⃣ Form & Parameter Assignment (covers Get URL, Assign parameters)                                      |
| Sticky Note1         | Sticky Note                 | Documentation for Apify actors     | —                       | —                        | #### 2️⃣ Apify Actors for Data Extraction (covers Query Metadata, Query Transcript)                                               |
| Sticky Note2         | Sticky Note                 | Documentation for OpenAI nodes     | —                       | —                        | #### 3️⃣ OpenAI GPT-4o & DALL·E Generation (covers Image Prompt Generator, Create Image)                                          |
| Sticky Note3         | Sticky Note                 | Documentation for image resizing   | —                       | —                        | #### 4️⃣ Resize for YouTube Format (covers Resize Image)                                                                           |
| Sticky Note4         | Sticky Note                 | Author contact and help info       | —                       | —                        | ### 👤 Need more help? Contact Robert Breen, Automation Consultant                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named `Get URL`:**  
   - Use n8n’s Form Trigger node.  
   - Configure with a form titled “Get URL”.  
   - Add one required text field labeled “Youtube URL” with placeholder `https://www.youtube.com/watch?v=lW5xEm7iSXk`.  
   - This node acts as the entry point, collecting user input.

2. **Add a Set node named `Assign parameters`:**  
   - Connect input from `Get URL`.  
   - Add a new field `startUrls` of type Array.  
   - Set its value to:  
     ```json
     [
       { "url": "{{ $json['Youtube URL'] }}" }
     ]
     ```  
   - This formats the input URL into the array structure required by Apify.

3. **Add an HTTP Request node named `Query Metadata`:**  
   - Connect input from `Assign parameters`.  
   - Configure as POST request to:  
     `https://api.apify.com/v2/acts/streamers~youtube-scraper/run-sync-get-dataset-items`  
   - Set Body to JSON with keys:  
     - `downloadSubtitles`: false  
     - `hasCC`: false  
     - `hasLocation`: false  
     - `hasSubtitles`: false  
     - `is360`: false  
     - `is3D`: false  
     - `is4K`: false  
     - `isBought`: false  
     - `isHD`: false  
     - `isHDR`: false  
     - `isLive`: false  
     - `isVR180`: false  
     - `maxResultStreams`: 0  
     - `maxResults`: 1  
     - `maxResultsShorts`: 0  
     - `preferAutoGeneratedSubtitles`: false  
     - `saveSubsToKVS`: false  
     - `startUrls`:  
       ```json
       [
         { "url": "{{ $json['Youtube URL'] }}", "method": "GET" }
       ]
       ```  
   - Authenticate using HTTP Query Auth credentials with your Apify API token.

4. **Add another HTTP Request node named `Query Transcript`:**  
   - Connect input from `Query Metadata`.  
   - Configure as POST request to:  
     `https://api.apify.com/v2/acts/topaz_sharingan~Youtube-Transcript-Scraper/run-sync-get-dataset-items`  
   - Set body parameter `startUrls` to:  
     `={{ $('Assign parameters').item.json.startUrls }}`  
   - Use the same HTTP Query Auth credentials for Apify.

5. **Add an OpenAI node named `Image Prompt Generator`:**  
   - Connect input from `Query Transcript`.  
   - Configure with model ID `gpt-4o`.  
   - Set message content to:  
     ```
     Analyze the transcript and video title below and generate a single, detailed prompt for DALL-E to create a YouTube thumbnail image. The prompt must include:

     Create a minimalist YouTube thumbnail in an illustration style. The background should be a very simple, uncluttered setting with soft, ambient lighting that subtly reflects the essence of the transcript. The overall mood should be professional and non-cluttered, ensuring that the text overlay stands out without distraction. Do not include any text.

     Transcript: {{ $json.max_transcript }}
     Video Title: {{ $json.videoTitle }}
     ```  
   - Enable JSON output to capture the prompt.  
   - Use OpenAI API credentials.

6. **Add an OpenAI node named `Create Image`:**  
   - Connect input from `Image Prompt Generator`.  
   - Configure to generate an image resource.  
   - Use the prompt from the previous node’s JSON output: `={{ $json.message.content.prompt }}`.  
   - Set image size to 1792x1024 pixels.  
   - Use the same OpenAI API credentials.

7. **Add an Edit Image node named `Resize Image`:**  
   - Connect input from `Create Image`.  
   - Configure resize operation with:  
     - Width: 1280 px  
     - Height: 720 px  
     - Resize option: `ignoreAspectRatio` = true  
     - Format: JPEG  
     - Filename: `={{ $('Query Transcript').item.json.videoTitle }}`  
   - This finalizes the image to YouTube thumbnail specs.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                           |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow designed by Robert Breen, Automation Consultant and n8n Expert                       | Contact: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/             |
| Apify actors require account and credits; ensure API token is configured in HTTP Query Auth   | Apify actors used: streamers~youtube-scraper and topaz_sharingan~Youtube-Transcript-Scraper               |
| OpenAI GPT-4o used for prompt crafting; requires OpenAI API key configured                    | OpenAI usage details: GPT-4o for text prompt, DALL·E for image generation                                |
| Image resizing done entirely inside n8n, no external API needed                               | Edit Image's `ignoreAspectRatio` forces exact thumbnail dimensions (1280x720)                            |
| For best results, use valid, publicly accessible YouTube URLs                                | Invalid URLs or private videos may cause API failures or empty results                                   |

---

**Disclaimer:** The text provided is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.

---