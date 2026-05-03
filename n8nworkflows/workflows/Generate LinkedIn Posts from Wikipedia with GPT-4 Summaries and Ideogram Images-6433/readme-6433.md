Generate LinkedIn Posts from Wikipedia with GPT-4 Summaries and Ideogram Images

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-from-wikipedia-with-gpt-4-summaries-and-ideogram-images-6433


# Generate LinkedIn Posts from Wikipedia with GPT-4 Summaries and Ideogram Images

### 1. Workflow Overview

This workflow automates the creation and posting of LinkedIn content derived from Wikipedia articles. It is designed for users who want to generate professional LinkedIn posts summarizing Wikipedia content, enriched with AI-generated images for enhanced engagement. The workflow integrates web scraping, AI summarization, image generation, and social media posting through the following logical blocks:

- **1.1 Input Reception & Data Extraction:** Receives user input (article name), triggers a web scraping operation on Bright Data to extract Wikipedia content, and polls until data is ready.
- **1.2 AI Summarization:** Feeds the scraped Wikipedia text to an AI agent powered primarily by GPT-4 to produce a concise, professional LinkedIn-ready summary.
- **1.3 Image Generation:** Uses the Ideogram API to create an illustrative image based on the AI-generated summary.
- **1.4 LinkedIn Posting:** Publishes the summarized text with the generated image to a specified LinkedIn profile and constructs a shareable URL to the post.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Extraction

**Overview:**  
This block captures the article name submitted via a form and initiates a scraping request to Bright Data’s dataset API. It then polls the API to check the scraping progress and finally retrieves the scraped Wikipedia content.

**Nodes Involved:**  
- 📝 On form submission  
- 🌐 HTTP Request  
- Wait for status  
- Check Final Status  
- Wait (1 minute)  
- Wikipedia Scrap Post  

**Node Details:**

- **📝 On form submission**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input for the Wikipedia article name via a form titled "Wikipedia Search".  
  - Configuration: Single form field labeled "Article Name".  
  - Inputs: User form data  
  - Outputs: Passes form data to HTTP Request node.  
  - Edge cases: Missing or malformed input; no retries configured.  

- **🌐 HTTP Request**  
  - Type: HTTP Request (POST)  
  - Role: Triggers a Bright Data dataset crawl using the article name as a keyword.  
  - Configuration: Calls Bright Data API endpoint `/datasets/v3/trigger` with dataset_id fixed; sends JSON body with keyword from form input and other dataset parameters. Authorization via Bearer token (BRIGHT_DATA_API_KEY) required.  
  - Inputs: Article name from form submission  
  - Outputs: JSON response containing snapshot_id for later status checks.  
  - Edge cases: API authentication failure, invalid dataset ID, network timeouts.  

- **Wait for status**  
  - Type: HTTP Request (GET)  
  - Role: Queries Bright Data API for the progress status of the scraping job using snapshot_id.  
  - Configuration: GET request to `/datasets/v3/progress/{{snapshot_id}}` with Authorization header.  
  - Inputs: snapshot_id from previous node  
  - Outputs: Status JSON with field `status`.  
  - Edge cases: Snapshot ID invalid or expired, API rate limiting, network issues.  

- **Check Final Status**  
  - Type: IF node  
  - Role: Checks if the scraping job status equals "ready".  
  - Configuration: Condition compares `$json.status` to `"ready"`.  
  - Inputs: Status JSON from "Wait for status"  
  - Outputs:  
    - True branch: proceeds to "Wikipedia Scrap Post"  
    - False branch: triggers "Wait" node to pause before rechecking.  
  - Edge cases: Unexpected status values, JSON parsing errors.  

- **Wait (1 minute)**  
  - Type: Wait  
  - Role: Pauses execution for 1 minute before rechecking the status.  
  - Inputs: From "Check Final Status" false branch  
  - Outputs: Loops back to "Wait for status"  
  - Edge cases: Workflow timeout if scraping takes too long.  

- **Wikipedia Scrap Post**  
  - Type: HTTP Request (GET)  
  - Role: Fetches the scraped Wikipedia data (title and article text) once scraping is complete.  
  - Configuration: GET request to `/datasets/v3/snapshot/{{snapshot_id}}` with Authorization header.  
  - Inputs: snapshot_id from previous nodes  
  - Outputs: JSON containing `cataloged_text` array with fields `title` and `text`.  
  - Edge cases: Data missing or incomplete, API errors, malformed JSON.  

---

#### 2.2 AI Summarization

**Overview:**  
Uses AI language models to summarize the scraped Wikipedia article into a concise, professional LinkedIn post. The primary model is GPT-4. A fallback Anthropic Claude model and output parsers ensure structured, high-quality output.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Anthropic Chat Model  
- Auto-fixing Output Parser  
- Structured Output Parser  

**Node Details:**

- **AI Agent**  
  - Type: LangChain AI Agent  
  - Role: Coordinates the summarization task. Receives Wikipedia article text and title, sends prompt to language models.  
  - Configuration: System message defines summarization task (max 2000 characters, bullet points allowed, professional LinkedIn tone).  
  - Inputs: Article text from "Wikipedia Scrap Post"  
  - Outputs: AI-generated summary text.  
  - Key expressions: Embeds article text in prompt dynamically.  
  - Edge cases: Model timeout, incomplete summaries, prompt injection risks.  

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI GPT-4 Chat Model  
  - Role: Primary language model for AI Agent.  
  - Configuration: Uses GPT-4.1-mini model. Requires OpenAI API credentials.  
  - Inputs: Prompt from AI Agent  
  - Outputs: Raw AI text response.  
  - Edge cases: API rate limits, authentication errors, latency.  

- **Anthropic Chat Model**  
  - Type: LangChain Anthropic Claude Model  
  - Role: Alternative language model, used downstream for output fixing.  
  - Configuration: Uses Claude Sonnet 4 model with Anthropic API credentials.  
  - Inputs: To Auto-fixing Output Parser  
  - Outputs: Refined text response.  
  - Edge cases: API errors, rate limits.  

- **Auto-fixing Output Parser**  
  - Type: LangChain Auto-fixing Output Parser  
  - Role: Automatically corrects or improves AI-generated output format and content.  
  - Inputs: Output from Anthropic Chat Model  
  - Outputs: Cleaned and fixed AI text for further parsing.  
  - Edge cases: Parsing failures, incomplete fixes.  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Ensures final AI output conforms to a specified JSON schema (example: `{ "text": "California" }`).  
  - Inputs: Output from Auto-fixing Output Parser  
  - Outputs: Final structured JSON summary text.  
  - Edge cases: Schema validation errors, malformed JSON.  

---

#### 2.3 Image Generation

**Overview:**  
Generates a relevant illustrative image based on the AI summary text using the Ideogram API, then downloads the image for LinkedIn posting.

**Nodes Involved:**  
- Image Generate  
- HTTP Request1  

**Node Details:**

- **Image Generate**  
  - Type: HTTP Request (POST)  
  - Role: Sends summarized text as prompt to Ideogram API to generate an image.  
  - Configuration: POST to `https://api.ideogram.ai/v1/ideogram-v3/generate` with multipart form-data including prompt from AI Agent output, rendering speed TURBO, resolution 1280x704. Authentication via `Api-Key` header with IDEOGRAM_API_KEY.  
  - Inputs: AI summary text from "AI Agent"  
  - Outputs: JSON response containing image URL.  
  - Edge cases: API key invalid, generation failures, rate limits.  

- **HTTP Request1**  
  - Type: HTTP Request (GET)  
  - Role: Downloads the generated image file from the URL provided by Ideogram API.  
  - Configuration: URL dynamically obtained from Ideogram response data. Response type configured to download file.  
  - Inputs: Image URL from "Image Generate"  
  - Outputs: Binary image data for posting.  
  - Edge cases: URL invalid, download failures, timeouts.  

---

#### 2.4 LinkedIn Posting

**Overview:**  
Posts the AI-generated summary and image to LinkedIn on behalf of the user and constructs a publicly accessible LinkedIn post URL.

**Nodes Involved:**  
- Create a post  
- LinkedIn URL  

**Node Details:**

- **Create a post**  
  - Type: LinkedIn node  
  - Role: Creates a LinkedIn post with text and associated image for a specified profile.  
  - Configuration: Text set to AI summary output; `shareMediaCategory` set to IMAGE to attach media. Requires LinkedIn OAuth2 credentials. Target person/profile ID specified.  
  - Inputs: Summary text from "AI Agent"; image data from "HTTP Request1"  
  - Outputs: JSON including LinkedIn post URN.  
  - Edge cases: OAuth token expiration, permissions error, API limits.  

- **LinkedIn URL**  
  - Type: Code (JavaScript)  
  - Role: Constructs a public LinkedIn post URL from the post URN returned by LinkedIn API.  
  - Configuration: Checks if URN starts with `urn:li:share:`, then builds URL `https://www.linkedin.com/feed/update/{URN}`. Otherwise returns "Invalid LinkedIn URN".  
  - Inputs: Post URN JSON from "Create a post"  
  - Outputs: JSON with `linkedinUrl` property for easy access.  
  - Edge cases: Missing or malformed URN, string parsing errors.  

---

### 3. Summary Table

| Node Name            | Node Type                          | Functional Role                             | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                                                 |
|----------------------|----------------------------------|---------------------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------------------------------------|
| 📝 On form submission | Form Trigger                     | Captures article name input                  | —                     | 🌐 HTTP Request       | 📝+🌐 Start & Submit Article: User starts flow by entering article name; triggers scraping request to Bright Data.           |
| 🌐 HTTP Request       | HTTP Request (POST)              | Initiates Bright Data scraping               | 📝 On form submission  | Wait for status       | See above                                                                                                                  |
| Wait for status       | HTTP Request (GET)               | Polls Bright Data for scraping job status   | 🌐 HTTP Request       | Check Final Status    | 🔷 Bright Data Extraction Section: Polls status using snapshot_id                                                         |
| Check Final Status    | IF                              | Checks if scraping status is 'ready'         | Wait for status       | Wikipedia Scrap Post, Wait | See above                                                                                                                  |
| Wait                 | Wait                            | Waits 1 minute before rechecking status       | Check Final Status    | Wait for status       | See above                                                                                                                  |
| Wikipedia Scrap Post  | HTTP Request (GET)               | Fetches scraped Wikipedia content             | Check Final Status    | AI Agent              | See above                                                                                                                  |
| AI Agent             | LangChain AI Agent               | Summarizes Wikipedia text for LinkedIn post  | Wikipedia Scrap Post  | Image Generate        | 🤖 AI Summarization Section: Produces LinkedIn-ready summary using GPT-4                                                    |
| OpenAI Chat Model    | LangChain OpenAI GPT-4 Model    | Primary AI language model                      | AI Agent              | AI Agent (as LM)      | See above                                                                                                                  |
| Anthropic Chat Model | LangChain Anthropic Claude Model| Alternative AI model for output refinement    | Auto-fixing Output Parser | Auto-fixing Output Parser | See above                                                                                                                  |
| Auto-fixing Output Parser | LangChain Output Parser Autofixing | Fixes and cleans AI output                  | Anthropic Chat Model  | AI Agent (as output parser) | See above                                                                                                                  |
| Structured Output Parser | LangChain Output Parser Structured | Validates AI output format                    | Auto-fixing Output Parser | Auto-fixing Output Parser | See above                                                                                                                  |
| Image Generate       | HTTP Request (POST)              | Generates image from summary text via Ideogram API | AI Agent              | HTTP Request1         | 🖼️ Image Generation Section: Generates relevant image based on summarized text                                             |
| HTTP Request1        | HTTP Request (GET)               | Downloads generated image                      | Image Generate        | Create a post         | See above                                                                                                                  |
| Create a post        | LinkedIn Node                   | Posts summary and image to LinkedIn profile   | HTTP Request1         | LinkedIn URL          | 📤 Publishing Section: Posts content and image to LinkedIn                                                                 |
| LinkedIn URL         | Code (JavaScript)                | Constructs public LinkedIn post URL            | Create a post         | —                     | See above                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a form trigger node ("📝 On form submission"):**  
   - Type: Form Trigger  
   - Form title: "Wikipedia Search"  
   - Add one field: "Article Name" (text)  

2. **Add HTTP Request node ("🌐 HTTP Request"):**  
   - Method: POST  
   - URL: https://api.brightdata.com/datasets/v3/trigger  
   - Query parameters:  
     - dataset_id: gd_lr9978962kkjr3nx49  
     - include_errors: true  
     - type: discover_new  
     - discover_by: keyword  
     - limit_per_input: 1  
   - Request body (JSON): Array with object:  
     `{ "keyword": "{{ $json['Article Name'] }}", "pages_load": 1 }`  
   - Headers: Authorization: Bearer `BRIGHT_DATA_API_KEY` (set as credential/environment variable)  
   - Connect output of form trigger to this node.  

3. **Add HTTP Request node ("Wait for status"):**  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Header: Authorization: Bearer `BRIGHT_DATA_API_KEY`  
   - Connect output of HTTP Request to this node.  

4. **Add IF node ("Check Final Status"):**  
   - Condition: `$json.status` equals `"ready"` (case sensitive)  
   - Connect "Wait for status" output to this IF node.  

5. **Add Wait node ("Wait"):**  
   - Wait for 1 minute  
   - Connect IF node’s false output to this Wait node.  
   - Connect Wait node output back to "Wait for status" node (loop).  

6. **Add HTTP Request node ("Wikipedia Scrap Post"):**  
   - Method: GET  
   - URL: `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}`  
   - Header: Authorization: Bearer `BRIGHT_DATA_API_KEY`  
   - Connect IF node’s true output to this node.  

7. **Add AI Agent node ("AI Agent"):**  
   - Type: LangChain Agent  
   - Text/Input:  
     ```
     here is the title:-  {{ $json.cataloged_text[0].title }}
     here is the article of my title :- {{ $json.cataloged_text[0].text }}
     ```  
   - System message:  
     ```
     Task:- 
     Summarize the following article in under 2000 characters, keeping it professional, informative, and engaging enough for a LinkedIn audience.
     Use bullet points if helpful. Avoid repetition. Remove any unnecessary fluff.
     Tone should be confident, insightful, and thought-leadership oriented — ideal for busy professionals who want quick understanding.
     Here's the content:
     ---
     {{ $json.cataloged_text[0].text }}
     ---
     ```  
   - Prompt type: Define  
   - Connect output of "Wikipedia Scrap Post" to this node.  

8. **Add OpenAI Chat Model node ("OpenAI Chat Model"):**  
   - Model: GPT-4.1-mini  
   - Credentials: OpenAI API key (configured in n8n credentials)  
   - Connect this node as the AI language model for "AI Agent".  

9. **Add Anthropic Chat Model node ("Anthropic Chat Model"):**  
   - Model: Claude Sonnet 4  
   - Credentials: Anthropic API key (configured in n8n credentials)  
   - Connect as AI language model for "Auto-fixing Output Parser".  

10. **Add Auto-fixing Output Parser node ("Auto-fixing Output Parser"):**  
    - Connect AI output parser input from "Anthropic Chat Model".  
    - Connect output back to AI Agent output parser input.  

11. **Add Structured Output Parser node ("Structured Output Parser"):**  
    - JSON schema example: `{ "text": "California" }`  
    - Connect output parser input from "Auto-fixing Output Parser".  

12. **Add HTTP Request node ("Image Generate"):**  
    - Method: POST  
    - URL: https://api.ideogram.ai/v1/ideogram-v3/generate  
    - Content type: multipart-form-data  
    - Body parameters:  
      - `prompt`: `{{ $json.output.text }}` (output text from AI Agent)  
      - `rendering_speed`: TURBO  
      - `resolution`: 1280x704  
    - Headers: Api-Key: `IDEOGRAM_API_KEY` (credential)  
    - Connect AI Agent main output to this node.  

13. **Add HTTP Request node ("HTTP Request1"):**  
    - Method: GET  
    - URL: `{{ $json.data[0].url }}` (image URL from previous node)  
    - Response: file download (binary response)  
    - Connect output of "Image Generate" to this node.  

14. **Add LinkedIn node ("Create a post"):**  
    - Text: `{{ $('AI Agent').item.json.output.text }}`  
    - Person: LinkedIn profile ID (set as parameter)  
    - Share media category: IMAGE  
    - Credentials: LinkedIn OAuth2 credentials configured in n8n  
    - Connect output of "HTTP Request1" to this node.  

15. **Add Code node ("LinkedIn URL"):**  
    - JavaScript to construct LinkedIn post URL:  
      ```javascript
      const item = $input.item;
      const shareUrn = item.json.urn || '';
      if (shareUrn.startsWith('urn:li:share:')) {
        item.json.linkedinUrl = `https://www.linkedin.com/feed/update/${shareUrn}/`;
      } else {
        item.json.linkedinUrl = 'Invalid LinkedIn URN';
      }
      return item;
      ```  
    - Connect output of "Create a post" to this node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow starts with user input via form and uses Bright Data API for Wikipedia scraping.                                         | Sticky Note near "📝 On form submission" and "🌐 HTTP Request".                                   |
| AI summarization leverages GPT-4 (OpenAI) primarily, with fallback Anthropic Claude for output refinement and parsing.           | Sticky Note near AI Agent and related nodes.                                                     |
| Image generation uses Ideogram API to create relevant visuals based on AI summary text.                                           | Sticky Note near "Image Generate" node.                                                          |
| Final LinkedIn post includes generated image and summary, with URL constructed for easy access.                                  | Sticky Note near "Create a post" and "LinkedIn URL".                                            |
| Requires API credentials setup for Bright Data, OpenAI, Anthropic, Ideogram, and LinkedIn OAuth2.                                | Credential nodes must be configured with valid API keys for each service.                        |
| Bright Data dataset ID is fixed as `gd_lr9978962kkjr3nx49` — this ID must be valid and accessible in your Bright Data account. | Workflow depends on Bright Data dataset and associated API access.                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.