LinkedIn Content Automation: AI Post Creation & Images with Sheet Approval Workflow

https://n8nworkflows.xyz/workflows/linkedin-content-automation--ai-post-creation---images-with-sheet-approval-workflow-11355


# LinkedIn Content Automation: AI Post Creation & Images with Sheet Approval Workflow

### 1. Workflow Overview

This workflow automates LinkedIn content creation and publishing, targeted at marketers, content creators, agency owners, and solopreneurs. It transforms new Google Sheets campaign entries into fully developed LinkedIn posts enhanced with AI-generated text and images, incorporating a human approval step.

The workflow is logically divided into three main blocks:

- **1.1 Trigger & Data Flow**: Watches for new rows added to a Google Sheet and filters entries that require content generation, extracting key campaign details.

- **1.2 Content Generation & Approval**: Uses AI (local Ollama or specified LLM) to draft LinkedIn posts based on campaign data and relevant LinkedIn trends retrieved via Tavily search. The draft is saved back to the sheet and sent by email for manual approval. Depending on the approval response, the content is either published or revised.

- **1.3 Image Generation & Publishing**: For approved posts, generates a detailed image prompt using Ollama, creates the image via OpenAI, and publishes the post with the image to LinkedIn.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Data Flow

**Overview:**  
This block initiates the workflow by monitoring a Google Sheet for new campaign entries. It filters out rows that already have LinkedIn posts generated and extracts relevant campaign info for subsequent processing.

**Nodes Involved:**  
- Google Sheets Trigger  
- Filter  
- Code in JavaScript  
- Search

**Node Details:**

- **Google Sheets Trigger**  
  - *Type & Role:* Trigger node; detects new rows added to a specified Google Sheet.  
  - *Configuration:* Polls every minute on the sheet’s specific tab (gid=0). Credentials use OAuth2 for Google Sheets access.  
  - *Expressions:* No complex expressions; triggers on any row addition.  
  - *Connections:* Outputs to Filter node.  
  - *Failures:* Potential auth errors, API limits, or polling delays.  
  - *Sub-workflow:* None.

- **Filter**  
  - *Type & Role:* Filters out rows where the "LinkedIn_Post" field is non-empty, ensuring only new posts proceed.  
  - *Configuration:* Checks if field `LinkedIn_Post` is empty.  
  - *Expressions:* Uses expression `={{ $json.LinkedIn_Post }}` to test emptiness.  
  - *Connections:* Passes filtered rows to "Code in JavaScript".  
  - *Failures:* Expression errors if field missing or misnamed.  
  - *Sub-workflow:* None.

- **Code in JavaScript**  
  - *Type & Role:* Extracts and structures relevant campaign data, including Campaign, Subject, Audience, and row number (ID) for downstream usage.  
  - *Configuration:* Maps each item, returning simplified JSON with key fields.  
  - *Expressions:* Accesses item JSON properties safely, defaulting to "N/A" if missing.  
  - *Connections:* Sends output to "Search" node.  
  - *Failures:* Coding errors or missing input data.  
  - *Sub-workflow:* None.

- **Search (Tavily Node)**  
  - *Type & Role:* Queries Tavily to search LinkedIn for posts relevant to the campaign. Provides context on trending topics or similar content.  
  - *Configuration:* Query format concatenates campaign, subject, audience plus `site:linkedin.com` to limit search to LinkedIn.  
  - *Expressions:* Query built via `={{$json["Campaign"] + " " + $json["Subject"] + " for " + $json["Audience"] + " site:linkedin.com"}}`.  
  - *Connections:* Output goes to "chat input" code node.  
  - *Failures:* API key/auth failures, empty search results, rate limits.  
  - *Sub-workflow:* None.

---

#### 2.2 Content Generation & Approval

**Overview:**  
Takes the campaign data and search results to generate a LinkedIn post draft using AI. The draft is saved back to the sheet and sent via Gmail for manual approval. Based on the approval response, the workflow either proceeds to image generation and publishing or loops back for content revision.

**Nodes Involved:**  
- chat input (Code)  
- Content Generator (Langchain Agent using Ollama or configured LLM)  
- Add the post to the sheet (Google Sheets node)  
- Approval Email (Gmail node)  
- Approved? (IF node)

**Node Details:**

- **chat input (Code in JavaScript)**  
  - *Type & Role:* Aggregates campaign data and Tavily search results into a formatted prompt for the AI content generation model.  
  - *Configuration:* Builds a multi-line string including Campaign, Subject, Audience, and detailed formatted Tavily results as bullet points or fallback message.  
  - *Expressions:* Uses `$json` properties and array mapping to format search results.  
  - *Connections:* Outputs to "Content Generator".  
  - *Failures:* Empty or malformed search results may produce inadequate prompts.  
  - *Sub-workflow:* None.

- **Content Generator (Langchain Agent)**  
  - *Type & Role:* AI node that generates a professional LinkedIn post draft based on input prompt. Uses Ollama or configured LLM to produce polished content including hook, insights, CTA, and emojis.  
  - *Configuration:* System message specifies tone, content requirements, and formatting (no intros, just final post). Input text composes campaign info, Tavily data, and reviewer feedback if any.  
  - *Expressions:* Input text assembled from prior nodes and dynamic feedback.  
  - *Connections:* Sends generated post text to "Add the post to the sheet".  
  - *Failures:* AI failures include model timeouts, API limits, or incoherent output.  
  - *Sub-workflow:* None.

- **Add the post to the sheet (Google Sheets node)**  
  - *Type & Role:* Updates the original Google Sheet row with the newly generated LinkedIn post text.  
  - *Configuration:* Matches row by ID and writes to "LinkedIn_Post" column.  
  - *Expressions:* Uses ID from trigger and AI output for update.  
  - *Connections:* Sends updated row data to "Approval Email".  
  - *Failures:* Google Sheets API errors, mismatched row IDs, write permission issues.  
  - *Sub-workflow:* None.

- **Approval Email (Gmail node)**  
  - *Type & Role:* Sends an email requesting manual approval of the generated post, including a custom form for feedback.  
  - *Configuration:* Sends a templated email with the LinkedIn post content, subject line "Approval for LinkedIn Post Content". Uses Gmail OAuth2 credentials. Waits synchronously for response with checkboxes (Approve Yes/No) and comments.  
  - *Expressions:* Email body includes `{{ $json['LinkedIn_Post '] }}` for dynamic content.  
  - *Connections:* Output goes to "Approved?" IF node.  
  - *Failures:* Email sending errors, response timeout, malformed replies.  
  - *Sub-workflow:* None.

- **Approved? (IF node)**  
  - *Type & Role:* Branches workflow based on email approval response.  
  - *Configuration:* Checks if "Do you approve this text?" response equals "Yes".  
  - *Expressions:* Uses `={{ $json.data['Do you approve this text?'][0] }}`.  
  - *Connections:* If yes, proceeds to "Image Prompt"; if no, loops back to "Content Generator" for revision.  
  - *Failures:* Missing or malformed response data could cause misrouting.  
  - *Sub-workflow:* None.

---

#### 2.3 Image Generation & Publishing

**Overview:**  
For approved content, generates an image prompt using Ollama, creates an image with OpenAI, and publishes the post with the image via LinkedIn API.

**Nodes Involved:**  
- Image Prompt (Ollama)  
- Generate image (OpenAI Image)  
- Create a post (LinkedIn Community Management)

**Node Details:**

- **Image Prompt (Ollama)**  
  - *Type & Role:* AI node that transforms approved LinkedIn post text into a single, detailed, realistic image prompt.  
  - *Configuration:* Uses "llama3.2:latest" model with strict instructions to output only one clean prompt describing visual subject, colors, mood, environment, etc.  
  - *Expressions:* Input is the approved LinkedIn post text from the sheet.  
  - *Connections:* Output prompt sent to "Generate image".  
  - *Failures:* Model errors, incomplete prompt generation.  
  - *Sub-workflow:* None.

- **Generate image (OpenAI Image)**  
  - *Type & Role:* Generates a realistic image based on the prompt received.  
  - *Configuration:* Uses OpenAI image generation API, prompt dynamically populated with image prompt content.  
  - *Expressions:* Prompt constructed as `"Generate an image for a linkedin post this is the description:{{ $json.content }} .The images should be realistic for linkedin."`  
  - *Connections:* Sends generated image to "Create a post".  
  - *Failures:* API rate limits, prompt rejection, image generation latency.  
  - *Sub-workflow:* None.

- **Create a post (LinkedIn Community Management)**  
  - *Type & Role:* Posts the final content and image on LinkedIn under the user’s community management account.  
  - *Configuration:* Authenticates via LinkedIn OAuth2, posts with media category IMAGE. Uses credentials linked to user’s LinkedIn account.  
  - *Expressions:* Receives image and text from previous nodes.  
  - *Connections:* Final node; no output connections.  
  - *Failures:* LinkedIn API errors, token expiration, media upload failures.  
  - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                         | Input Node(s)                 | Output Node(s)               | Sticky Note                                                                                                                         |
|-------------------------|---------------------------------|---------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1            | Sticky Note                     | Documentation overview                 |                              |                             | ## Who’s it for... (Full workflow description and setup instructions)                                                             |
| Google Sheets Trigger   | Google Sheets Trigger           | Start - detects new campaign rows     |                              | Filter                      | ## 1. Trigger & Data Flow                                                                                                           |
| Filter                  | Filter                         | Filters rows without LinkedIn post    | Google Sheets Trigger         | Code in JavaScript           | ## 1. Trigger & Data Flow                                                                                                           |
| Code in JavaScript      | Code                           | Extracts campaign data fields          | Filter                       | Search                      | ## 1. Trigger & Data Flow                                                                                                           |
| Search                  | Tavily Search                  | Searches LinkedIn for relevant posts  | Code in JavaScript            | chat input                  | ## 1. Trigger & Data Flow                                                                                                           |
| chat input              | Code                           | Creates AI prompt from campaign & search | Search                      | Content Generator           | ## 2. Content Generation & Approval                                                                                                |
| Content Generator       | Langchain Agent (AI Model)     | Generates LinkedIn post draft          | chat input, Approved? (on no approval) | Add the post to the sheet, Approved? (on yes) | ## 2. Content Generation & Approval                                                                                                |
| Add the post to the sheet | Google Sheets                 | Updates sheet with LinkedIn post       | Content Generator            | Approval Email              | ## 2. Content Generation & Approval                                                                                                |
| Approval Email          | Gmail                         | Sends email for manual approval        | Add the post to the sheet     | Approved?                   | ## 2. Content Generation & Approval                                                                                                |
| Approved?               | IF                            | Branches based on approval response    | Approval Email                | Image Prompt (yes), Content Generator (no) | ## 2. Content Generation & Approval                                                                                                |
| Image Prompt            | Ollama (AI Tool)              | Generates detailed image prompt        | Approved? (yes)               | Generate image              | ## 3. Image & Publish                                                                                                              |
| Generate image          | OpenAI Image                  | Creates image from prompt               | Image Prompt                 | Create a post               | ## 3. Image & Publish                                                                                                              |
| Create a post           | LinkedIn Community Management  | Publishes post with image to LinkedIn  | Generate image               |                             | ## 3. Image & Publish                                                                                                              |
| Sticky Note             | Sticky Note                   | Documentation of Block 1 description   |                              |                             | ## 1. Trigger & Data Flow                                                                                                           |
| Sticky Note2            | Sticky Note                   | Documentation of Block 2 description   |                              |                             | ## 2. Content Generation & Approval                                                                                                |
| Sticky Note3            | Sticky Note                   | Documentation of Block 3 description   |                              |                             | ## 3. Image & Publish                                                                                                              |
| Ollama Chat Model       | Langchain Ollama LM           | Alternative or auxiliary AI model      |                              | Content Generator (ai_languageModel) |                                                                                                                                    |
| Message a model in Ollama | Langchain Ollama Tool        | Possibly auxiliary or experimental node|                              | Image Prompt (ai_tool)      |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node:**  
   - Set event to "rowAdded" on your target Google Sheet and sheet tab (gid=0).  
   - Configure OAuth2 credentials for Google Sheets.  
   - Set polling interval to every minute.

2. **Add Filter node:**  
   - Condition: proceed only if `LinkedIn_Post` field is empty (string empty check).  
   - Connect Google Sheets Trigger output to Filter input.

3. **Add Code node ("Code in JavaScript"):**  
   - Extract `Campaign`, `Subject`, `Audience`, and row ID (`ID`).  
   - Return simplified JSON with those fields.  
   - Connect Filter output to this node’s input.

4. **Add Tavily Search node:**  
   - Configure with your Tavily API key credentials.  
   - Set query expression: `={{$json["Campaign"] + " " + $json["Subject"] + " for " + $json["Audience"] + " site:linkedin.com"}}`.  
   - Connect Code node output to Search input.

5. **Add Code node ("chat input"):**  
   - Compose multi-line prompt including campaign data and Tavily results formatted as bullet points or fallback text.  
   - Connect Search output to this node.

6. **Add Langchain Agent ("Content Generator"):**  
   - Configure system message outlining post requirements: hook, insights, CTA, emojis, and tone.  
   - Use Ollama or your chosen LLM credentials.  
   - Connect chat input output to Content Generator input.

7. **Add Google Sheets node ("Add the post to the sheet"):**  
   - Set operation to "update" on the same Google Sheet and sheet tab.  
   - Map `ID` (row) and update `LinkedIn_Post` column with AI output post content.  
   - Connect Content Generator output to this node.

8. **Add Gmail node ("Approval Email"):**  
   - Configure OAuth2 credentials for Gmail.  
   - Set operation to "send and wait".  
   - Message body includes the post content dynamically.  
   - Include custom form fields with checkbox ("Do you approve this text?") with options Yes/No and a comment field.  
   - Connect Google Sheets update node output to this node.

9. **Add IF node ("Approved?"):**  
   - Condition: check if email response field "Do you approve this text?" equals "Yes".  
   - Connect Approval Email output to IF node.

10. **On IF node True branch (approved):**  
    - Add Ollama node ("Image Prompt"):  
      - Configure with model "llama3.2:latest" and system instructions to output only a clean image prompt.  
      - Input is the approved LinkedIn post text from the sheet.  
    - Connect IF node True output to this.

11. **Add OpenAI Image node ("Generate image"):**  
    - Configure with OpenAI credentials.  
    - Set image prompt dynamically from Ollama output: `"Generate an image for a linkedin post this is the description:{{ $json.content }} .The images should be realistic for linkedin."`  
    - Connect Ollama output to this node.

12. **Add LinkedIn node ("Create a post"):**  
    - Configure with LinkedIn Community Management OAuth2 credentials.  
    - Set authentication to communityManagement.  
    - Set media category to IMAGE and map generated image and post text.  
    - Connect OpenAI Image output to this node.

13. **On IF node False branch (not approved):**  
    - Connect back to Content Generator node to allow revision based on feedback.

14. **Add optional auxiliary nodes as needed:**  
    - Ollama Chat Model or Message a Model in Ollama for possible alternative AI calls or extensions.

15. **Add Sticky Notes:**  
    - Add clear documentation nodes near each logical block describing purpose and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                                               |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is intended for marketers, content creators, agency owners, and solopreneurs to automate LinkedIn content production using AI, with integration into Google Sheets for campaign input and Gmail for approval. It supports customization of AI model, tone, image generation provider, and approval logic.                                                      | Sticky Note1 content                                                                                                                          |
| Requires Google Sheets account, LinkedIn OAuth app, Tavily API key, and Ollama or OpenAI for AI and image generation.                                                                                                                                                                                                                                            | Sticky Note1 content                                                                                                                          |
| The approval step is manual via Gmail with a custom form, but can be modified for automatic or other forms of approvals.                                                                                                                                                                                                                                         | Sticky Note2 content                                                                                                                          |
| The image prompt generation is strictly instructed to produce concise, realistic prompts suitable for LinkedIn posts, enhancing visual appeal.                                                                                                                                                                                                                   | Sticky Note3 content                                                                                                                          |
| Workflow uses OAuth2 for Google Sheets, Gmail, and LinkedIn to ensure secure and authorized API access.                                                                                                                                                                                                                                                          | Credential configurations                                                                                                                     |
| Tavily is used to extract relevant trends and related content from LinkedIn to enrich the AI-generated post context.                                                                                                                                                                                                                                            | Node "Search"                                                                                                                                |
| The workflow depends on reliable AI model responses and prompt engineering to ensure posts are professional, engaging, and ready for LinkedIn publication without additional editing.                                                                                                                                                                           | Content Generator node and system message                                                                                                   |
| For troubleshooting, monitor API rate limits, auth token expiration, and network latency especially for AI and LinkedIn API calls.                                                                                                                                                                                                                              | General operational consideration                                                                                                           |
| Workflow design includes loopback for revision if approval is denied, enabling iterative content refinement.                                                                                                                                                                                                                                                     | Approved? node logic                                                                                                                          |

---

*Disclaimer:*  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.