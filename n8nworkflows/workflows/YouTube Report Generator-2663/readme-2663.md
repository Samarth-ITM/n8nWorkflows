YouTube Report Generator

https://n8nworkflows.xyz/workflows/youtube-report-generator-2663


# YouTube Report Generator

### 1. Workflow Overview

The **YouTube Subtitles Report Generator** workflow enables automated extraction and thematic analysis of YouTube video subtitles, producing concise analytical reports about the video's main content theme. It is designed for users who want to analyze video content without relying on YouTube API keys, by parsing publicly accessible HTML and subtitle files.

The workflow can be logically divided into the following functional blocks:

- **1.1 Input Reception:** Receives YouTube video IDs via a webhook.
- **1.2 Data Retrieval:** Fetches the YouTube video HTML page and extracts subtitle URLs.
- **1.3 Subtitle Acquisition:** Downloads subtitle content in XML format from extracted URLs.
- **1.4 AI-Powered Analysis:** Processes subtitle XML to generate a thematic report using an AI language model.
- **1.5 Output Delivery:** Returns the generated analytical report as a webhook response.

Each block connects sequentially, ensuring a streamlined flow from input to output with error handling for missing subtitle data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow by accepting incoming HTTP requests containing YouTube video IDs through a webhook.
- **Nodes Involved:**  
  - *Trigger Webhook*

- **Node Details:**

  - **Trigger Webhook**  
    - *Type:* Webhook (HTTP trigger)  
    - *Role:* Listens for incoming POST/GET requests with the video ID payload.  
    - *Configuration:*  
      - Webhook path set to a unique identifier (`c18956b9-f9b7-4fc8-b01c-67d5c9eeddd9`).  
      - Response mode set to "responseNode" to allow downstream nodes to determine the output sent back.  
    - *Input/Output:* No input connections; outputs the received data as JSON, expected to include a query parameter `id` representing the YouTube video ID.  
    - *Edge Cases:*  
      - Missing or malformed video ID input will cause downstream HTTP request to fail.  
      - No direct validation of video ID format here; errors surface later.  

#### 2.2 Data Retrieval

- **Overview:** Fetches the HTML content of the YouTube video page based on the video ID and extracts the subtitle URLs embedded within the HTML.
- **Nodes Involved:**  
  - *Fetch Video HTML*  
  - *Extract Subtitles URLs*

- **Node Details:**

  - **Fetch Video HTML**  
    - *Type:* HTTP Request  
    - *Role:* Downloads YouTube video page HTML using the video ID as a query parameter.  
    - *Configuration:*  
      - URL: `https://www.youtube.com/watch` with query parameter `v` set dynamically from `{{$json.query.id}}`.  
      - Headers mimic a modern browser user-agent to avoid blocking.  
      - Accept headers set to fetch HTML content.  
    - *Input:* Connected from *Trigger Webhook*.  
    - *Output:* HTML content of the video page in the response body (`data` field).  
    - *Edge Cases:*  
      - Network errors, rate limits, or video unavailability may cause request failure.  
      - Videos without subtitles or live streams may not contain subtitle data in HTML.  

  - **Extract Subtitles URLs**  
    - *Type:* Code (JavaScript)  
    - *Role:* Parses the raw HTML string to locate subtitle metadata and extract all available subtitle base URLs.  
    - *Configuration:*  
      - Uses regex to find JSON-like "captions" block in HTML.  
      - Extracts all `baseUrl` fields for subtitles, decodes them, and cleans escape sequences.  
      - Throws errors if no captions or URLs are found.  
    - *Input:* Connected from *Fetch Video HTML*. Receives video HTML as a string.  
    - *Output:* Array of JSON objects each containing a `baseUrl` property with a subtitle URL.  
    - *Edge Cases:*  
      - No captions block in HTML → error thrown, workflow execution halts.  
      - Malformed or changed YouTube HTML structure may break regex extraction.  

#### 2.3 Subtitle Acquisition

- **Overview:** Downloads the subtitles XML content from the extracted subtitle URLs.
- **Nodes Involved:**  
  - *Fetch Subtitles Content*

- **Node Details:**

  - **Fetch Subtitles Content**  
    - *Type:* HTTP Request  
    - *Role:* Fetches subtitle XML content from each provided subtitle URL.  
    - *Configuration:*  
      - URL is set dynamically from `{{$json.baseUrl}}` (output of previous node).  
      - No special headers or authentication needed since URLs are public.  
    - *Input:* Receives multiple subtitle URL entries from *Extract Subtitles URLs*.  
    - *Output:* Subtitles content in XML format as text in the response body (`data` field).  
    - *Edge Cases:*  
      - Subtitle URL may expire or be invalid → HTTP errors.  
      - Downloaded XML may be empty or malformed.  
      - Multiple subtitle tracks may exist; the workflow does not filter by language.  

#### 2.4 AI-Powered Analysis

- **Overview:** Processes the fetched subtitles XML to generate an analytical thematic report using a language model.
- **Nodes Involved:**  
  - *Generate Analytical Report*  
  - *AI Model Configuration*

- **Node Details:**

  - **Generate Analytical Report**  
    - *Type:* LangChain LLM Chain node  
    - *Role:* Receives subtitle XML content and applies a customized AI prompt to extract and summarize the main theme analytically.  
    - *Configuration:*  
      - Input text set to the subtitle XML content (`{{$json.data}}`).  
      - Prompt defined with detailed instructions to ignore XML tags, extract text, identify themes, and produce a formal analytical report.  
      - Output format includes a concise Title and a Body report of up to three paragraphs.  
    - *Input:* Receives subtitle XML from *Fetch Subtitles Content*.  
    - *Output:* JSON containing the AI-generated report text.  
    - *Edge Cases:*  
      - AI model failures or timeouts.  
      - Input text too long or malformed affecting model performance.  
      - Misinterpretation of subtitles if XML tags vary significantly.  

  - **AI Model Configuration**  
    - *Type:* LangChain Google Gemini Chat Model node  
    - *Role:* Specifies the AI model and parameters used by the *Generate Analytical Report* node.  
    - *Configuration:*  
      - Model: `models/gemini-1.5-flash-002` (Google Gemini)  
      - Temperature: 0.4 (moderate creativity)  
      - Credentials: Google PaLM API key stored securely in n8n.  
    - *Input:* Configures the language model for *Generate Analytical Report*.  
    - *Output:* Connected internally to the chain node.  
    - *Edge Cases:*  
      - API auth failures or quota limits.  
      - Model unavailability or version changes.  

#### 2.5 Output Delivery

- **Overview:** Sends the AI-generated analytical report back as the response to the original webhook request.
- **Nodes Involved:**  
  - *Return Analytical Report*

- **Node Details:**

  - **Return Analytical Report**  
    - *Type:* Respond to Webhook  
    - *Role:* Returns the AI-generated thematic report text as the HTTP response.  
    - *Configuration:*  
      - Response mode set to send plain text (`respondWith: "text"`).  
      - Response body dynamically set to the generated report (`{{$json.text}}`).  
    - *Input:* Receives final report from *Generate Analytical Report*.  
    - *Output:* HTTP response to the webhook caller.  
    - *Edge Cases:*  
      - Missing or empty report data leads to empty response.  
      - Timing out if AI processing is slow.  

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                      | Input Node(s)          | Output Node(s)               | Sticky Note                                                                                                            |
|-------------------------|--------------------------------|------------------------------------|------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Trigger Webhook         | Webhook                        | Entry point, receives video ID     | —                      | Fetch Video HTML             |                                                                                                                        |
| Fetch Video HTML        | HTTP Request                  | Retrieves YouTube video HTML       | Trigger Webhook        | Extract Subtitles URLs       |                                                                                                                        |
| Extract Subtitles URLs  | Code (JavaScript)             | Extracts subtitle URLs from HTML   | Fetch Video HTML       | Fetch Subtitles Content      |                                                                                                                        |
| Fetch Subtitles Content | HTTP Request                  | Downloads subtitles XML content    | Extract Subtitles URLs | Generate Analytical Report   |                                                                                                                        |
| AI Model Configuration | LangChain LM Chat Model (Google Gemini) | Configures AI model for analysis   | —                      | Generate Analytical Report   |                                                                                                                        |
| Generate Analytical Report | LangChain LLM Chain           | Processes subtitles and generates report | Fetch Subtitles Content, AI Model Configuration | Return Analytical Report |                                                                                                                        |
| Return Analytical Report | Respond to Webhook            | Sends AI report as webhook response | Generate Analytical Report | —                           |                                                                                                                        |
| Workflow Overview      | Sticky Note                   | Workflow summary and notes         | —                      | —                           | This workflow processes YouTube video subtitles to generate a thematic analytical report. Steps: 1. Webhook trigger 2. Fetch HTML 3. Extract subtitles 4. AI report generation 5. Return report. No API key required. Cost depends on model & tokens. Use free tier of Google Gemini to minimize costs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node:**  
   - Type: Webhook  
   - Set the webhook path to a unique string (e.g., `c18956b9-f9b7-4fc8-b01c-67d5c9eeddd9`)  
   - Set response mode to `responseNode` for deferred response handling.  
   - This node receives YouTube video IDs (expect query parameter `id`).

2. **Create an HTTP Request Node (Fetch Video HTML):**  
   - Connect from the webhook node.  
   - Set method to GET.  
   - URL: `https://www.youtube.com/watch`  
   - Add query parameter `v` with expression `{{$json.query.id}}` to dynamically set video ID.  
   - Set headers to mimic a browser User-Agent and accept HTML content.

3. **Create a Code Node (Extract Subtitles URLs):**  
   - Connect from the HTTP Request node.  
   - Use JavaScript code to parse the HTML string to find the `"captions"` block and extract all `baseUrl` subtitle URLs.  
   - Decode URLs and return as multiple JSON items with `baseUrl` fields.  
   - Implement error handling for missing captions or URLs.

4. **Create an HTTP Request Node (Fetch Subtitles Content):**  
   - Connect from the Code node.  
   - Set method GET.  
   - Set URL dynamically to `{{$json.baseUrl}}` to fetch each subtitle XML.  
   - No authentication or special headers needed.

5. **Create an AI Model Configuration Node:**  
   - Type: LangChain LM Chat Model (Google Gemini)  
   - Configure model name as `models/gemini-1.5-flash-002`  
   - Set temperature to 0.4 for balanced creativity.  
   - Add Google PaLM API credentials stored in n8n.

6. **Create a LangChain LLM Chain Node (Generate Analytical Report):**  
   - Connect from the Fetch Subtitles Content node.  
   - Set the input text to `{{$json.data}}` (subtitle XML content).  
   - Define a detailed prompt instructing the model to ignore XML tags, extract text, analyze themes, and produce a formal analytical report with a title and max three paragraphs.  
   - Link this node to the AI Model Configuration node for model specification.

7. **Create a Respond to Webhook Node (Return Analytical Report):**  
   - Connect from the Generate Analytical Report node.  
   - Configure to respond with plain text.  
   - Set response body to the generated report text `{{$json.text}}`.

8. **Activate the Workflow:**  
   - Test with a valid YouTube video ID that has subtitles.  
   - Verify the response contains a clear thematic report.  
   - Adjust prompt or model settings as needed for quality.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow extracts subtitles directly from YouTube's public HTML, avoiding the need for API keys. Execution cost depends on the AI model chosen and subtitle length (token count). Google Gemini free tier is recommended for cost control. | Workflow Overview Sticky Note                                                                             |
| The AI prompt is designed to ignore XML tags and focus on thematic content, excluding references to the source or promotional content, ensuring a clean analytical report.                                                                | Prompt Description Section                                                                                 |
| Users may customize the prompt to tailor analysis to specific needs or use alternative AI models compatible with LangChain integration.                                                                                                | Key Features and Setup Instructions                                                                       |
| Example output demonstrates a formal, concise thematic summary without direct mention of the video source.                                                                                                                               | Example Output Section                                                                                      |

---

This documentation fully captures the structure, logic, key configurations, and integration points of the YouTube Subtitles Report Generator workflow, enabling reproduction, modification, and error anticipation.