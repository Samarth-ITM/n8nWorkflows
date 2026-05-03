Generate & Deploy Landing Pages with GPT, Gemini and Vercel

https://n8nworkflows.xyz/workflows/generate---deploy-landing-pages-with-gpt--gemini-and-vercel-10447


# Generate & Deploy Landing Pages with GPT, Gemini and Vercel

---
### 1. Workflow Overview

This workflow, titled **"AI landing page generator"**, automates the creation and deployment of AI-generated landing pages. It transforms a user’s text prompt into a fully designed web page with a hero image, styled HTML content, and a live deployment on Vercel. The workflow is designed for interactive sessions, allowing iterative improvements to the landing page based on user input during each session.

**Target use cases:**  
- Marketers or developers who want to quickly generate and deploy landing pages from simple textual ideas.  
- Teams leveraging AI for rapid prototyping of landing page designs.  
- Use cases requiring automated updates and versioning of landing pages with session-based memory.

**Logical blocks:**  
- **1.1 Input Reception & Memory Retrieval:** Receive user chat input and fetch or clear session memory to retain or reset landing page state.  
- **1.2 Image Prompt Generation & Creation:** Generate a descriptive prompt for a hero image and create the image using Google Gemini (PaLM) API.  
- **1.3 Image Processing & Hosting:** Convert the generated image to Base64, format as a data URI, upload it to Cloudinary, and retrieve the hosted image URL.  
- **1.4 Landing Page HTML Generation:** Use OpenAI GPT-4o-mini to generate or update landing page HTML using the user prompt and hero image URL.  
- **1.5 Memory Save & Deployment Preparation:** Save generated HTML to a data table keyed by session, prepare a deployment payload for Vercel.  
- **1.6 Deployment & Response:** Deploy the landing page to Vercel, make the deployment publicly accessible, retrieve and format the deployment URL, and respond to the user with the live page link.  
- **1.7 Optional Deployment Cleanup:** (Optional) Clean up older Vercel deployments, keeping only the two most recent to manage resources.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Memory Retrieval

**Overview:**  
This block receives the user’s chat input, fetches any previously saved landing page HTML for the session, and prepares a prompt that either incorporates previous HTML or clears it when requested.

**Nodes Involved:**  
- Chat input  
- Get memory for current session  
- Prepare Chat Prompt with Memory  
- Sticky Note (instructions)

**Node Details:**

- **Chat input**  
  - Type: LangChain Chat Trigger (webhook)  
  - Role: Entry point capturing user messages publicly via webhook.  
  - Configuration: Public enabled, responseMode set to "responseNodes" for asynchronous flow.  
  - Inputs: External webhook call  
  - Outputs: To "Get memory for current session"  
  - Edge cases: Missing or malformed chat input; webhook availability.

- **Get memory for current session**  
  - Type: Data Table (n8n native)  
  - Role: Retrieves stored HTML content by matching sessionID from chat input.  
  - Configuration: Filters on sessionID column equal to current session’s ID.  
  - Inputs: Chat input node output  
  - Outputs: To "Prepare Chat Prompt with Memory"  
  - Edge cases: No existing memory found returns empty; data table must be correctly configured with sessionID and html columns.

- **Prepare Chat Prompt with Memory**  
  - Type: Code (JavaScript)  
  - Role: Combines user chat input with existing HTML memory or clears HTML if the user requests "start new".  
  - Key expressions: Reads chat input and memory JSON; conditional clearing on "start new" phrase.  
  - Inputs: Chat input and memory nodes  
  - Outputs: To "Generate Image Prompt"  
  - Edge cases: Case-insensitive matching for "start new"; missing or malformed memory data.

- **Sticky Note1**  
  - Purpose: Explains the need to update data table names and structure for session memory.

---

#### 1.2 Image Prompt Generation & Creation

**Overview:**  
Generates a vivid, concise textual prompt for the hero image based on user input using OpenAI GPT-4o-mini, then creates the image using Google Gemini (PaLM) Nano Banana model.

**Nodes Involved:**  
- Generate Image Prompt (OpenAI)  
- Generate an image1 (Google Gemini)  
- Sticky Note3 (instructions)

**Node Details:**

- **Generate Image Prompt**  
  - Type: OpenAI (chat)  
  - Role: Converts user landing page idea into a detailed, vivid scene description for image generation.  
  - Configuration: GPT-4o-mini model; max tokens 120; temperature 0.6; system prompt instructs to exclude UI text and output only description.  
  - Inputs: Output from "Prepare Chat Prompt with Memory" (user prompt)  
  - Outputs: To "Generate an image1"  
  - Edge cases: Rate limits, API key validity, prompt truncation.

- **Generate an image1**  
  - Type: LangChain Google Gemini (image generation)  
  - Role: Creates hero image based on the prompt from OpenAI node.  
  - Configuration: Model set to "models/gemini-2.5-flash-image"; prompt directly from last node’s output; resource set to image generation.  
  - Credentials: Google Palm API key (Google Cloud Platform required)  
  - Inputs: Image prompt string  
  - Outputs: To "Convert Image to Base64"  
  - Edge cases: API quota, authentication failures, image generation errors.

- **Sticky Note3**  
  - Highlights the need to replace API keys and notes alternative image generation options like OpenAI DALL·E 2.

---

#### 1.3 Image Processing & Hosting

**Overview:**  
Processes the generated image by converting it to Base64, formats it as a valid data URI, then uploads the image to Cloudinary and returns the hosted URL.

**Nodes Involved:**  
- Convert Image to Base64  
- Format Image as Data URI  
- Upload Image to Cloudinary  
- Return Hero Image URL  
- Sticky Note5, Sticky Note7 (instructions)

**Node Details:**

- **Convert Image to Base64**  
  - Type: Extract from File  
  - Role: Converts binary image output into Base64 string inside JSON property.  
  - Inputs: Image binary from Gemini node  
  - Outputs: Formatted Base64 string to next node  
  - Edge cases: Binary data missing or corrupt.

- **Format Image as Data URI**  
  - Type: Set  
  - Role: Prepends "data:image/png;base64," to Base64 string to create a valid image URI.  
  - Inputs: Base64 string from previous node  
  - Outputs: JSON with `file` property for upload  
  - Edge cases: Incorrect Base64 string formatting.

- **Upload Image to Cloudinary**  
  - Type: HTTP Request  
  - Role: Uploads image URI to Cloudinary using multipart/form-data POST request.  
  - Configuration: URL includes Cloudinary cloud name placeholder; uses unsigned upload preset and targets "ai_websites/landing_pages" folder.  
  - Inputs: Data URI string  
  - Outputs: Cloudinary JSON response including public URL  
  - Credentials: Cloudinary API credentials (replace `<your_cloud_key>`)  
  - Edge cases: API errors, authentication failure, invalid request format.

- **Return Hero Image URL**  
  - Type: Code  
  - Role: Extracts and returns the `url` property from Cloudinary response for use in HTML generation.  
  - Inputs: Cloudinary upload response  
  - Outputs: JSON with `hero_image` URL field  
  - Edge cases: Missing or malformed response URL.

- **Sticky Note5 & Sticky Note7**  
  - Provide instructions on converting and uploading images, and remind users to configure Cloudinary credentials.

---

#### 1.4 Landing Page HTML Generation

**Overview:**  
Generates the landing page HTML using GPT-4o-mini, incorporating the user prompt and the hosted hero image URL, then saves the HTML content for session memory.

**Nodes Involved:**  
- Generate HTML (OpenAI)  
- Save HTML to Memory Table  
- Sticky Note9, Sticky Note10 (instructions)

**Node Details:**

- **Generate HTML (OpenAI)**  
  - Type: OpenAI (chat)  
  - Role: Generates full HTML of the landing page based on user prompt and hero image URL.  
  - Configuration: Model gpt-4o-mini; system prompt instructs expert Apple-style web design; user prompt includes dynamic insertion of stored prompt and hero image URL; CSS inline only, no external assets; output HTML only.  
  - Inputs: Hero image URL from "Return Hero Image URL"; user prompt and memory from earlier nodes.  
  - Outputs: To "Save HTML to Memory Table"  
  - Edge cases: API limits, prompt complexity, output formatting issues.

- **Save HTML to Memory Table**  
  - Type: Data Table (n8n native)  
  - Role: Upserts the generated HTML into the session memory table keyed by sessionID.  
  - Configuration: Maps HTML content from OpenAI output to `html` field; matches by sessionID from chat input.  
  - Inputs: HTML from OpenAI node  
  - Outputs: To "Prepare Vercel Deployment Payload"  
  - Edge cases: Data table write errors, missing sessionID, concurrency issues.

- **Sticky Note9 & Sticky Note10**  
  - Emphasize editing OpenAI credentials and data table names for memory persistence.

---

#### 1.5 Memory Save & Deployment Preparation

**Overview:**  
Prepares the Vercel deployment payload by cleaning the generated HTML, then triggers deployment.

**Nodes Involved:**  
- Prepare Vercel Deployment Payload  
- Deploy to Vercel  
- Sticky Note12 (instructions)

**Node Details:**

- **Prepare Vercel Deployment Payload**  
  - Type: Function (JavaScript)  
  - Role: Cleans markdown fences from HTML content, then formats JSON payload with deployment name, files array containing index.html, and deployment target.  
  - Inputs: HTML from "Generate HTML (OpenAI)" or fallback JSON property  
  - Outputs: To "Deploy to Vercel"  
  - Edge cases: Missing or malformed HTML; markdown fences not removed properly.

- **Deploy to Vercel**  
  - Type: HTTP Request  
  - Role: Sends POST request to Vercel API to initiate deployment with prepared payload.  
  - Configuration: URL is `https://api.vercel.com/v13/deployments`; authentication via HTTP header token.  
  - Inputs: Deployment payload JSON  
  - Outputs: To "Fetch Latest Deployment URL1"  
  - Credentials: Vercel API header auth token  
  - Edge cases: Auth failure, API quota, network issues.

- **Sticky Note12**  
  - Details the deployment step and reminds updating Vercel credentials.

---

#### 1.6 Deployment & Response

**Overview:**  
Fetches the latest deployment, makes it publicly accessible, formats the URL, and sends it back to the user.

**Nodes Involved:**  
- Fetch Latest Deployment URL1  
- Make Deployment URL Public  
- Format Deployment URL  
- Respond to Chat

**Node Details:**

- **Fetch Latest Deployment URL1**  
  - Type: HTTP Request  
  - Role: Retrieves the most recent deployment info with limit=1 from Vercel API.  
  - Configuration: Authenticated GET request.  
  - Inputs: Triggered after deployment  
  - Outputs: To "Make Deployment URL Public"  
  - Edge cases: Empty deployments list, auth failures.

- **Make Deployment URL Public**  
  - Type: HTTP Request  
  - Role: PATCH request to remove SSO or password protection, ensuring deployment is public.  
  - Configuration: URL dynamically constructed from projectId in previous node’s response; sends JSON body disabling protection.  
  - Inputs: Deployment info JSON  
  - Outputs: To "Format Deployment URL"  
  - Edge cases: API errors, invalid projectId.

- **Format Deployment URL**  
  - Type: Set  
  - Role: Constructs the full HTTPS URL for the deployed page by prefixing Vercel’s returned URL.  
  - Inputs: Deployment response JSON  
  - Outputs: To "Respond to Chat"  
  - Edge cases: Missing URL field.

- **Respond to Chat**  
  - Type: LangChain Chat  
  - Role: Sends final live landing page URL back to user with a friendly message.  
  - Inputs: Formatted deployment URL  
  - Outputs: Ends workflow  
  - Edge cases: Chat delivery errors, user disconnection.

---

#### 1.7 Optional Deployment Cleanup

**Overview:**  
Deletes older Vercel deployments, keeping only the two newest to conserve resources.

**Nodes Involved:**  
- list deployments  
- Code in JavaScript1  
- HTTP Request (delete deployment)  
- Sticky Note

**Node Details:**

- **list deployments**  
  - Type: HTTP Request  
  - Role: Retrieves up to 1000 deployments from Vercel API.  
  - Inputs: Manual or scheduled trigger (not connected in main flow)  
  - Outputs: To "Code in JavaScript1"  
  - Edge cases: Large data volume, auth errors.

- **Code in JavaScript1**  
  - Type: Code (JavaScript)  
  - Role: Sorts deployments by creation date descending, slices to keep only the two newest, returns UIDs for older deployments.  
  - Inputs: Deployment list JSON  
  - Outputs: To "HTTP Request" (delete node)  
  - Edge cases: Empty list, sorting errors.

- **HTTP Request** (delete deployment)  
  - Type: HTTP Request  
  - Role: Deletes deployments by UID via Vercel API.  
  - Inputs: Deployment UIDs from code node  
  - Outputs: None (cleanup only)  
  - Edge cases: Deletion failures, API rate limits.

- **Sticky Note**  
  - Warns about permanent deletion risks, credential updates, and usage caution.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                                   | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                          |
|--------------------------------|--------------------------------------|--------------------------------------------------|--------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Chat input                    | LangChain Chat Trigger                | Receives user chat input                          | -                              | Get memory for current session  |                                                                                                    |
| Get memory for current session | Data Table                           | Retrieves stored HTML by session                  | Chat input                    | Prepare Chat Prompt with Memory | See Sticky Note1: Update data table name and ensure columns `sessionID` and `html` exist            |
| Prepare Chat Prompt with Memory| Code (JavaScript)                    | Combines chat input with memory or resets HTML   | Get memory for current session | Generate Image Prompt          |                                                                                                    |
| Generate Image Prompt          | OpenAI (chat)                       | Creates detailed prompt for hero image generation| Prepare Chat Prompt with Memory| Generate an image1             | See Sticky Note3: Replace OpenAI and Gemini API keys; alternative image generation options          |
| Generate an image1             | LangChain Google Gemini (image)      | Generates hero image from prompt                  | Generate Image Prompt          | Convert Image to Base64        |                                                                                                    |
| Convert Image to Base64        | Extract From File                   | Converts image binary to Base64 string            | Generate an image1             | Format Image as Data URI       | See Sticky Note5: Image conversion and preparation                                                |
| Format Image as Data URI       | Set                                | Formats Base64 string as valid data URI           | Convert Image to Base64        | Upload Image to Cloudinary     |                                                                                                    |
| Upload Image to Cloudinary     | HTTP Request                      | Uploads image to Cloudinary and returns URL       | Format Image as Data URI       | Return Hero Image URL          | See Sticky Note7: Replace `<your_cloud_key>` with Cloudinary cloud name                            |
| Return Hero Image URL          | Code (JavaScript)                   | Extracts public image URL                          | Upload Image to Cloudinary     | Generate HTML (OpenAI)         |                                                                                                    |
| Generate HTML (OpenAI)         | OpenAI (chat)                      | Generates landing page HTML                        | Return Hero Image URL          | Save HTML to Memory Table      | See Sticky Note9: Update OpenAI credentials                                                       |
| Save HTML to Memory Table      | Data Table                        | Saves generated HTML by session                    | Generate HTML (OpenAI)         | Prepare Vercel Deployment Payload| See Sticky Note10: Update data table name                                                        |
| Prepare Vercel Deployment Payload | Function (JavaScript)             | Cleans HTML, prepares deployment JSON payload     | Save HTML to Memory Table      | Deploy to Vercel               |                                                                                                    |
| Deploy to Vercel              | HTTP Request                      | Deploys landing page to Vercel                      | Prepare Vercel Deployment Payload| Fetch Latest Deployment URL1 | See Sticky Note12: Update Vercel API token                                                       |
| Fetch Latest Deployment URL1   | HTTP Request                      | Retrieves latest deployment info                    | Deploy to Vercel               | Make Deployment URL Public     |                                                                                                    |
| Make Deployment URL Public     | HTTP Request                      | Makes deployment publicly accessible                | Fetch Latest Deployment URL1   | Format Deployment URL          |                                                                                                    |
| Format Deployment URL          | Set                                | Formats full deployment URL                         | Make Deployment URL Public     | Respond to Chat                |                                                                                                    |
| Respond to Chat               | LangChain Chat                    | Sends live landing page URL to user                 | Format Deployment URL          | -                              |                                                                                                    |
| list deployments              | HTTP Request                      | Lists all Vercel deployments                        | -                              | Code in JavaScript1            | See Sticky Note: Optional cleanup flow; use with caution; update credentials                       |
| Code in JavaScript1            | Code (JavaScript)                   | Selects older deployments to delete                 | list deployments               | HTTP Request (delete deployments)|                                                                                                    |
| HTTP Request (delete deployment)| HTTP Request                    | Deletes old deployments by UID                       | Code in JavaScript1            | -                              |                                                                                                    |
| Sticky Note1                  | Sticky Note                      | Instructions on memory data table setup             | -                              | -                              |                                                                                                    |
| Sticky Note3                  | Sticky Note                      | Instructions for image generation API keys          | -                              | -                              |                                                                                                    |
| Sticky Note5                  | Sticky Note                      | Instructions on image conversion step                | -                              | -                              |                                                                                                    |
| Sticky Note7                  | Sticky Note                      | Instructions on Cloudinary upload                     | -                              | -                              |                                                                                                    |
| Sticky Note9                  | Sticky Note                      | Instructions on HTML generation                       | -                              | -                              |                                                                                                    |
| Sticky Note10                 | Sticky Note                      | Instructions on saving HTML to memory                 | -                              | -                              |                                                                                                    |
| Sticky Note12                 | Sticky Note                      | Instructions on Vercel deployment                      | -                              | -                              |                                                                                                    |
| Sticky Note                   | Sticky Note                      | Optional cleanup flow warning                          | -                              | -                              |                                                                                                    |
| Sticky Note2                  | Sticky Note                      | Workflow overview and setup notes                      | -                              | -                              |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Input Node**  
   - Type: LangChain Chat Trigger  
   - Set webhook to public with responseMode: "responseNodes"  
   - This node serves as the entry point for user messages.

2. **Add Data Table Node to Get Memory**  
   - Type: Data Table  
   - Operation: Get  
   - Filter: sessionID equals `{{$json.sessionId}}` from Chat input  
   - Data table must have columns: sessionID (string), html (string)  
   - Retrieves stored HTML for the current session.

3. **Add Code Node to Prepare Prompt with Memory**  
   - Type: Code (JavaScript)  
   - Logic:  
     - Read user chat message from Chat input.  
     - Retrieve existing HTML memory.  
     - If user says "start new" (case-insensitive), clear HTML memory.  
     - Output JSON with keys: prompt, current_html, sessionId.  
   - Connect input from Get memory node.

4. **Add OpenAI Node to Generate Image Prompt**  
   - Type: OpenAI Chat  
   - Model: gpt-4o-mini  
   - System Prompt: Prompt generator for DALL·E style image descriptions (exclude logos/UI text).  
   - User Prompt: Use the `prompt` field from previous node.  
   - Max tokens: 120, Temperature: 0.6.

5. **Add Google Gemini Node to Generate Image**  
   - Type: LangChain Google Gemini (PaLM)  
   - Model ID: models/gemini-2.5-flash-image (Nano Banana)  
   - Prompt: Output from OpenAI image prompt node  
   - Credentials: Google Palm API key from Google Cloud Platform.

6. **Add Extract From File Node to Convert Image to Base64**  
   - Operation: binaryToProperty  
   - Input: Image binary from Gemini node.

7. **Add Set Node to Format Image as Data URI**  
   - Assign new property `file`: prepend "data:image/png;base64," to Base64 string.

8. **Add HTTP Request Node to Upload Image to Cloudinary**  
   - URL: `https://api.cloudinary.com/v1_1/<your_cloud_key>/image/upload` (replace `<your_cloud_key>`)  
   - Method: POST  
   - Content Type: multipart/form-data  
   - Body Parameters:  
     - file = formatted data URI from previous node  
     - upload_preset = "unsigned_upload"  
     - folder = "ai_websites/landing_pages"  
   - Credentials: Cloudinary API key.

9. **Add Code Node to Extract Hero Image URL**  
   - Return JSON with property `hero_image` set to Cloudinary response `url`.

10. **Add OpenAI Node to Generate Landing Page HTML**  
    - Model: gpt-4o-mini  
    - System prompt: Expert Apple-style web designer generating HTML only  
    - User prompt: includes prompt from memory and `hero_image` URL  
    - No external CSS/JS, all CSS inline.

11. **Add Data Table Node to Save HTML to Memory**  
    - Operation: Upsert  
    - Columns: sessionID (from Chat input), html (from OpenAI HTML output)  
    - Filter: sessionID equals current session ID.

12. **Add Function Node to Prepare Vercel Deployment Payload**  
    - Clean HTML string by removing markdown fences (` ```html ` and ` ``` `)  
    - Create JSON payload with:  
      - name: `ai-site-${timestamp}`  
      - files: [{ file: "index.html", data: cleaned HTML }]  
      - projectSettings: { framework: null }  
      - target: "production"

13. **Add HTTP Request Node to Deploy to Vercel**  
    - URL: `https://api.vercel.com/v13/deployments`  
    - Method: POST  
    - Body: JSON from previous node  
    - Authentication: HTTP Header Auth with Vercel token.

14. **Add HTTP Request Node to Fetch Latest Deployment**  
    - URL: `https://api.vercel.com/v6/deployments?limit=1`  
    - Method: GET  
    - Authentication: Same as deploy node.

15. **Add HTTP Request Node to Make Deployment Public**  
    - URL: `https://api.vercel.com/v9/projects/{{deployment.projectId}}`  
    - Method: PATCH  
    - Body: JSON disabling SSO and password protection  
    - Authentication: Same as above.

16. **Add Set Node to Format Deployment URL**  
    - Assign `deployment_url`: prepend "https://" to deployment URL from previous node.

17. **Add LangChain Chat Node to Respond to User**  
    - Message: `"Here’s your live page:\n{{ $json.deployment_url }}"`  
    - WaitUserReply: false

18. **Optional: Add Cleanup Nodes for Vercel Deployments**  
    - HTTP Request node to list deployments (limit 1000)  
    - Code node to sort and keep only two newest  
    - HTTP Request node(s) to delete older deployments by UID  
    - Use with caution; update credentials accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow requires API keys for OpenAI, Google Gemini (PaLM), Cloudinary, and Vercel.            | Setup instructions are inline in Sticky Notes throughout the workflow.                              |
| Google Gemini node can be replaced with OpenAI DALL·E 2 for simpler image generation at lower cost.  | Tradeoff between image quality and cost.                                                           |
| The n8n Data Table must include `sessionID` and `html` columns to enable session memory persistence. | Critical for iterative landing page editing per user session.                                      |
| Vercel deployment requires a personal access token with appropriate scopes; use HTTP Header Auth.    | Tokens should be securely stored in n8n credentials.                                               |
| See Sticky Note2 for a detailed project overview and usage notes.                                    | Provides a comprehensive workflow summary and setup checklist.                                     |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It fully complies with legal and content policies, contains no illegal or offensive elements, and handles only public and legal data.