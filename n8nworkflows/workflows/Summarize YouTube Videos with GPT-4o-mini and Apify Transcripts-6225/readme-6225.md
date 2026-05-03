Summarize YouTube Videos with GPT-4o-mini and Apify Transcripts

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-with-gpt-4o-mini-and-apify-transcripts-6225


# Summarize YouTube Videos with GPT-4o-mini and Apify Transcripts

### 1. Workflow Overview

This n8n workflow automates the summarization of YouTube videos by leveraging Apify’s video transcription capabilities and OpenAI’s GPT-4o-mini language model. It is designed for users who want concise summaries of YouTube content by simply submitting a YouTube URL through a web form.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user input (YouTube URL) via a web form.
- **1.2 Payload Preparation:** Formats the input into a structured JSON payload suitable for the Apify actor.
- **1.3 Transcript Retrieval:** Runs an Apify actor to fetch YouTube video transcripts (captions).
- **1.4 Caption Extraction:** Extracts and prepares caption data from the Apify response.
- **1.5 Summarization:** Uses the LangChain Summarization Chain with OpenAI’s GPT-4o-mini model to generate a concise summary of the transcript.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the YouTube URL from the user through a custom form submission, initiating the workflow.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - *Type:* `formTrigger` (Webhook form trigger)  
    - *Role:* Entry point capturing user input via a web form titled "Youtube Summary" with a single required field "Youtube URL".  
    - *Configuration:*  
      - Form title: "Youtube Summary"  
      - Form field: "Youtube URL" (required, placeholder "Enter youtube url")  
      - Form description: "Give youtube url and get summary"  
    - *Input:* HTTP request with form data  
    - *Output:* JSON containing the submitted YouTube URL under the key `Youtube URL`  
    - *Potential failures:*  
      - Missing or invalid URL input (no validation beyond required field)  
      - Webhook connectivity or access issues  
    - *Version:* 2.2  
    - *Connections:* Outputs to node `Payload`

#### 2.2 Payload Preparation

- **Overview:**  
  Converts the raw user input into a detailed JSON payload structure required by the Apify YouTube transcript actor.

- **Nodes Involved:**  
  - Payload

- **Node Details:**

  - **Payload**  
    - *Type:* `set` node  
    - *Role:* Constructs the JSON payload for the Apify actor input.  
    - *Configuration:*  
      - Uses a raw mode JSON output with embedded expression to insert the submitted YouTube URL into the `urls` array.  
      - Sets various options for the Apify actor, including max retries (8), proxy usage via Apify proxy group "BUYPROXIES94952", and output format as "captions".  
      - Enables extraction of channel name, channel ID, and date published metadata; disables other metadata fields.  
    - *Key expressions:*  
      - `{{ $json['Youtube URL'] }}` used to dynamically fill the URLs array.  
    - *Input:* JSON from `On form submission`  
    - *Output:* JSON with the full payload under the key `youtube`  
    - *Potential failures:*  
      - Expression failure if `Youtube URL` is missing or malformed  
      - Incorrect proxy or actor configuration causing Apify actor failure  
    - *Version:* 3.4  
    - *Connections:* Outputs to node `Apify`

#### 2.3 Transcript Retrieval

- **Overview:**  
  Executes the Apify actor to retrieve the transcript (captions) of the provided YouTube video URL.

- **Nodes Involved:**  
  - Apify

- **Node Details:**

  - **Apify**  
    - *Type:* Apify node (custom n8n node for Apify integration)  
    - *Role:* Runs the Apify actor specified by `actorId` to scrape YouTube video transcripts.  
    - *Configuration:*  
      - Operation: "Run actor"  
      - Actor ID: dynamically linked to an external actor named "Youtube Transcripts" (value managed via n8n UI, not hardcoded)  
      - Custom body: passes the JSON payload from the previous node as the actor input  
      - Wait for finish: true (waits for actor completion before proceeding)  
      - Proxy enabled (via payload configuration)  
    - *Credentials:* Requires Apify API credentials configured in n8n.  
    - *Input:* Payload JSON with YouTube URLs and options  
    - *Output:* JSON containing the actor run results, including captions.  
    - *Potential failures:*  
      - Apify API authentication errors  
      - Actor execution failure or timeout  
      - Invalid or empty transcript data returned  
    - *Version:* 1  
    - *Connections:* Outputs to node `Caption`

#### 2.4 Caption Extraction

- **Overview:**  
  Extracts the caption data from the Apify actor’s response to prepare for summarization.

- **Nodes Involved:**  
  - Caption

- **Node Details:**

  - **Caption**  
    - *Type:* `set` node  
    - *Role:* Formats the output to isolate the `captions` property from the Apify response JSON.  
    - *Configuration:*  
      - Raw mode with JSON output: `{"captions": {{ $json.captions }}}`  
    - *Input:* JSON from Apify node containing full transcript data  
    - *Output:* JSON containing only the `captions` array or object  
    - *Potential failures:*  
      - Missing or undefined `captions` property in Apify response  
      - JSON expression errors if data structure changes  
    - *Version:* 3.4  
    - *Connections:* Outputs to node `Summarization Chain`

#### 2.5 Summarization

- **Overview:**  
  Uses LangChain’s Summarization Chain powered by OpenAI GPT-4o-mini to generate a concise summary of the extracted captions.

- **Nodes Involved:**  
  - Summarization Chain  
  - OpenAI Chat Model

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type:* `lmChatOpenAi` (LangChain OpenAI chat model node)  
    - *Role:* Provides GPT-4o-mini as the language model backing the summarization chain.  
    - *Configuration:*  
      - Model: `gpt-4o-mini`  
      - No additional options set  
    - *Credentials:* Requires OpenAI API credentials configured in n8n.  
    - *Input:* Connected as AI language model input for Summarization Chain  
    - *Output:* Model completions to Summarization Chain  
    - *Potential failures:*  
      - OpenAI API key missing or invalid  
      - Rate limits or request timeouts  
    - *Version:* 1.2  
    - *Connections:* Outputs to `Summarization Chain` via `ai_languageModel` input

  - **Summarization Chain**  
    - *Type:* `chainSummarization` (LangChain summarization chain)  
    - *Role:* Orchestrates the summarization of the video transcript captions using the OpenAI model.  
    - *Configuration:* Default options (presumably uses default prompt and chain steps)  
    - *Input:* Captions JSON from `Caption` node and language model from `OpenAI Chat Model`  
    - *Output:* Final summarized text of the YouTube video transcript  
    - *Potential failures:*  
      - Input captions missing or malformed  
      - Language model API errors or timeouts  
    - *Version:* 2.1  
    - *Connections:* Terminal output of the workflow (no next node)

---

### 3. Summary Table

| Node Name           | Node Type                    | Functional Role                         | Input Node(s)         | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|------------------------------|---------------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------|
| On form submission   | formTrigger                  | Receives YouTube URL input via form   | —                     | Payload                 |                                                                                              |
| Payload             | set                          | Formats user input into Apify payload | On form submission    | Apify                   |                                                                                              |
| Apify               | Apify actor runner           | Runs Apify actor to retrieve transcript | Payload               | Caption                 |                                                                                              |
| Caption             | set                          | Extracts captions from Apify response | Apify                 | Summarization Chain     |                                                                                              |
| OpenAI Chat Model   | lmChatOpenAi (LangChain)     | Provides GPT-4o-mini language model   | —                     | Summarization Chain (ai_languageModel) |                                                                                              |
| Summarization Chain | chainSummarization (LangChain) | Generates summary from captions       | Caption, OpenAI Chat Model | —                       |                                                                                              |
| Sticky Note         | stickyNote                   | Workflow title                        | —                     | —                       | ## Summarize Youtube Video using OpenAI and APify                                           |
| Sticky Note1        | stickyNote                   | Workflow summary explanation          | —                     | —                       | ## Summary\nUser submits a YouTube URL via a form → the URL is formatted into a payload → Apify actor fetches the video transcript → captions are extracted → OpenAI GPT-4o-mini generates the summary. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node ("On form submission")**  
   - Type: `formTrigger`  
   - Set form title to "Youtube Summary"  
   - Add a required field labeled "Youtube URL" with placeholder "Enter youtube url"  
   - Set form description to "Give youtube url and get summary"  
   - Ensure the webhook is active for receiving submissions.

2. **Create a Set Node ("Payload")**  
   - Type: `set`  
   - Mode: Raw JSON  
   - Enter JSON output with the following structure, embedding the form input dynamically:  
     ```json
     {
       "youtube": {
         "urls": ["{{ $json['Youtube URL'] }}"],
         "maxRetries": 8,
         "proxyOptions": {
           "useApifyProxy": true,
           "apifyProxyGroups": ["BUYPROXIES94952"]
         },
         "outputFormat": "captions",
         "channelNameBoolean": true,
         "channelIDBoolean": true,
         "dateTextBoolean": false,
         "relativeDateTextBoolean": false,
         "datePublishedBoolean": true,
         "viewCountBoolean": false,
         "likesBoolean": false,
         "commentsBoolean": false,
         "keywordsBoolean": false,
         "thumbnailBoolean": false,
         "descriptionBoolean": false
       }
     }
     ```
   - Connect the output of the Form Trigger node to this node.

3. **Create the Apify Node ("Apify")**  
   - Type: Apify integration node  
   - Operation: Run actor  
   - Actor ID: Select or enter the actor ID for the "Youtube Transcripts" actor (you must have this actor in your Apify account)  
   - Set "Wait for finish" option enabled to wait for actor execution completion  
   - Pass the payload from the previous node as the custom body (should be JSON)  
   - Configure Apify API credentials (create in n8n with your Apify API token)  
   - Connect the output of the Payload node to this node.

4. **Create Set Node ("Caption")**  
   - Type: `set`  
   - Mode: Raw JSON  
   - Output: `{"captions": {{ $json.captions }}}` to isolate the captions property.  
   - Connect the output of the Apify node here.

5. **Create OpenAI Chat Model Node ("OpenAI Chat Model")**  
   - Type: `lmChatOpenAi` (LangChain OpenAI node)  
   - Model: Select `gpt-4o-mini` from the model list  
   - Leave other options as default unless customization is needed  
   - Configure OpenAI API credentials with your OpenAI API key  
   - This node will connect as the AI language model input to the Summarization Chain.

6. **Create LangChain Summarization Chain Node ("Summarization Chain")**  
   - Type: `chainSummarization`  
   - Use default options or customize the summarization prompt if desired  
   - Connect the `Caption` node output to its main input  
   - Connect the `OpenAI Chat Model` node output to its `ai_languageModel` input  
   - This node produces the final summary output.

7. **(Optional) Add Sticky Notes**  
   - Add a sticky note with content: "## Summarize Youtube Video using OpenAI and APify" near the start of the workflow.  
   - Add a second sticky note with the summary explanation near the bottom:  
     ```
     ## Summary
     User submits a YouTube URL via a form → the URL is formatted into a payload → Apify actor fetches the video transcript → captions are extracted → OpenAI GPT-4o-mini generates the summary.
     ```

8. **Configure Execution Settings**  
   - Set execution order to “v1” (default).  
   - Activate all nodes in sequence.  
   - Test by submitting a valid YouTube URL in the form.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The Apify actor used is expected to be a YouTube transcript fetcher capable of returning captions in JSON format.          | Requires a valid Apify actor ID and account credentials.                                        |
| OpenAI GPT-4o-mini model is a smaller, faster variant suitable for summarization tasks with reasonable cost and latency.    | Model selection in OpenAI node allows balancing quality vs. cost.                              |
| Proxy usage via Apify proxy group "BUYPROXIES94952" is configured to avoid rate limits or IP bans during transcript fetch.| Proxy configuration is crucial for large-scale or repeated requests to YouTube.                |
| The workflow assumes the user inputs a valid YouTube URL; no URL validation is performed beyond mandatory input.            | Consider adding URL validation or error handling for malformed inputs.                          |
| For detailed info on LangChain summarization chain usage in n8n, visit: https://n8n.io/integrations/builtin/langchain      | Useful for customizing summarization behavior or chaining additional NLP operations.           |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and publicly accessible.