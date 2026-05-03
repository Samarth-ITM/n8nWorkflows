LinkedIn Content Factory with OpenAI Research & Replicate Branded Images

https://n8nworkflows.xyz/workflows/linkedin-content-factory-with-openai-research---replicate-branded-images-8686


# LinkedIn Content Factory with OpenAI Research & Replicate Branded Images

### 1. Workflow Overview

This workflow automates the creation and publication of branded LinkedIn posts based on topics listed in a Google Sheet. It uses AI to research trending, high-engagement angles on given topics, generates professional post content, creates a branded image using AI image generation, and publishes the final post on LinkedIn—all while updating the Google Sheet with status and metadata.

The workflow is organized into four main logical blocks:

- **1.1 Input Reception:** Retrieves pending topics from Google Sheets to start the process.
- **1.2 AI Research & Content Generation:** Uses AI agents and external search tools to research, analyze, and generate LinkedIn post content including title, text, and hashtags.
- **1.3 AI Branded Image Generation:** Constructs a detailed, brand-guided prompt for image generation and manages the generation process via Replicate API.
- **1.4 Post Publishing & Data Update:** Publishes the LinkedIn post with the generated image and updates Google Sheets with the post’s status, content, hashtags, and image URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow, triggered manually or scheduled, and fetches pending topics from a specified Google Sheet for processing.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - 1. Get Pending Topic from Google Sheets

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - Type: Manual Trigger  
     - Role: Entry point to start the workflow manually.  
     - Configuration: No parameters; initiates the workflow immediately on user command.  
     - Inputs: None  
     - Outputs: Connects to "1. Get Pending Topic from Google Sheets"  
     - Edge Cases: None; manual trigger.

  2. **1. Get Pending Topic from Google Sheets**  
     - Type: Google Sheets  
     - Role: Retrieves all rows where the "Status" column equals "Pending" from the specified sheet.  
     - Configuration: Reads from Google Sheet with columns “Topic” and “Status.” Filters rows where Status = "Pending."  
     - Credentials: Google Sheets OAuth2 account configured.  
     - Inputs: Trigger from manual node  
     - Outputs: Sends topic data downstream to AI Research node.  
     - Edge Cases: Google API auth errors, empty results if no pending topics, rate limits.  
     - Notes: Requires correct access to the Google Sheet and proper sheet names.

---

#### 1.2 AI Research & Content Generation

- **Overview:**  
  Performs research on the pending topic using SerpAPI and OpenAI GPT-4.1-mini to find trending angles and then generates a complete LinkedIn post including a catchy title, main text, and hashtags.

- **Nodes Involved:**  
  - SerpAPI1 (SerpAPI call for research)  
  - OpenAI Chat Model1 (GPT-4.1-mini for research)  
  - Structured Output Parser (for research output)  
  - 2. Research Topic & Find Viral Angle with AI (AI Agent using Langchain)  
  - SerpAPI (SerpAPI call for content generation)  
  - OpenAI Chat Model (GPT-4.1-mini for content generation)  
  - Structured Output Parser1 (for content output)  
  - 3. Generate LinkedIn Post Content with AI (AI Agent for post creation)

- **Node Details:**

  1. **SerpAPI1**  
     - Type: SerpAPI Tool Node  
     - Role: Performs web search to gather recent news/trends related to the topic to feed AI research.  
     - Credentials: SerpAPI API key required.  
     - Inputs: Topic from Google Sheets  
     - Outputs: Data to AI research agent.  
     - Edge Cases: API quota exceeded, no results found.

  2. **OpenAI Chat Model1**  
     - Type: Langchain OpenAI Chat Model  
     - Role: Language model used by the research AI agent for understanding and summarizing search results.  
     - Credentials: OpenAI API key.  
     - Configuration: Model set to GPT-4.1-mini.  
     - Inputs: Search data and prompt text.  
     - Outputs: Research JSON output.  
     - Edge Cases: API errors, rate limits, invalid prompt.  
     - Version: v1.2.

  3. **Structured Output Parser**  
     - Type: Langchain Structured Output Parser  
     - Role: Parses AI research output into JSON with keys: selected_topic, research_data, engagement_score.  
     - Inputs: AI research raw output.  
     - Outputs: Structured JSON for next node.  
     - Edge Cases: Parsing failure, unexpected output format.

  4. **2. Research Topic & Find Viral Angle with AI**  
     - Type: Langchain AI Agent  
     - Role: Coordinates research using Langchain tools (SerpAPI, OpenAI) to find a viral angle.  
     - Configuration: System message restricts focus to AI, ML, automation trends.  
     - Inputs: Topic data from Google Sheets.  
     - Outputs: JSON research summary.  
     - Edge Cases: Agent failure, tool integration failures.

  5. **SerpAPI**  
     - Type: SerpAPI Tool Node  
     - Role: Supports content generation by providing real-time search data.  
     - Credentials: SerpAPI key.  
     - Inputs: From AI agent.  
     - Outputs: To content generation AI.  
     - Edge Cases: Similar to SerpAPI1.

  6. **OpenAI Chat Model**  
     - Type: Langchain OpenAI Chat Model  
     - Role: GPT-4.1-mini model used to generate the LinkedIn post content.  
     - Inputs: Research data JSON.  
     - Outputs: Raw AI post JSON.  
     - Edge Cases: Same as OpenAI Chat Model1.

  7. **Structured Output Parser1**  
     - Type: Langchain Structured Output Parser  
     - Role: Parses LinkedIn post JSON output with title, text, hashtags, engagement_elements.  
     - Inputs: Raw AI post output.  
     - Outputs: Structured post data.  
     - Edge Cases: Parsing errors.

  8. **3. Generate LinkedIn Post Content with AI**  
     - Type: Langchain AI Agent  
     - Role: Uses AI to generate a professional LinkedIn post with hooks, CTAs, hashtags.  
     - Configuration: System message emphasizes professional tone and JSON strictness.  
     - Inputs: Research output JSON.  
     - Outputs: Post content JSON.  
     - Edge Cases: Agent errors, malformed JSON.

---

#### 1.3 AI Branded Image Generation

- **Overview:**  
  Takes the generated LinkedIn post content to build a detailed brand-style image prompt and requests image generation from Replicate API, polling until the image is ready.

- **Nodes Involved:**  
  - 4. Generate Branded Image Prompt (Code node)  
  - 4.1. Generate Branded Image Prompt (Code node)  
  - 5a. Start Image Generation (Replicate) (HTTP Request)  
  - 5b. Check Image Status (HTTP Request)  
  - If (Conditional node)  
  - Wait (Delay node)  
  - 6. Download Generated Image (HTTP Request)

- **Node Details:**

  1. **4. Generate Branded Image Prompt**  
     - Type: Code node (JavaScript)  
     - Role: Combines post title and text with a fixed brand style guide (using RAL color codes, lighting, composition) to create a detailed image prompt.  
     - Inputs: Parsed LinkedIn post JSON.  
     - Outputs: JSON containing title, text, hashtags, and imagePrompt string.  
     - Edge Cases: Missing or malformed input JSON.

  2. **4.1. Generate Branded Image Prompt**  
     - Type: Code node (JavaScript)  
     - Role: Refines the image prompt by extracting keywords from post text and appending them to the style guide description for optimized image generation input.  
     - Inputs: Output from node 4.  
     - Outputs: Final imagePrompt string.  
     - Edge Cases: Empty or very short post text.

  3. **5a. Start Image Generation (Replicate)**  
     - Type: HTTP Request  
     - Role: Sends a POST request to Replicate API to start image generation with the constructed prompt and fixed parameters (aspect ratio 3:2, jpg format, safety tolerance, etc.).  
     - Credentials: Replicate API key via HTTP Header Auth.  
     - Inputs: imagePrompt JSON.  
     - Outputs: Prediction ID for status polling.  
     - Edge Cases: API authentication failure, invalid prompt errors, network timeouts.

  4. **5b. Check Image Status**  
     - Type: HTTP Request  
     - Role: Polls Replicate API with prediction ID to check if image generation succeeded or is still processing.  
     - Credentials: Replicate API key.  
     - Inputs: Prediction ID from previous node or Wait node.  
     - Outputs: Status JSON passed to "If" node.  
     - Edge Cases: Rate limiting, timeouts, failed generation.

  5. **If**  
     - Type: Conditional  
     - Role: Checks if the image generation status equals "succeeded".  
     - Inputs: Status JSON from 5b.  
     - Outputs:  
       - If succeeded: proceeds to download image.  
       - Else: triggers Wait node to delay before next polling.  
     - Edge Cases: Unexpected status values.

  6. **Wait**  
     - Type: Wait/Delay node  
     - Role: Pauses workflow for 2 seconds before next status check to avoid excessive polling.  
     - Inputs: From If node when image is not ready.  
     - Outputs: Back to 5b.  
     - Edge Cases: Long delays if image generation is slow.

  7. **6. Download Generated Image**  
     - Type: HTTP Request  
     - Role: Downloads the generated image file from the URL provided by Replicate API after successful generation.  
     - Inputs: image URL from status response.  
     - Outputs: Image file data passed to LinkedIn publishing.  
     - Edge Cases: Download failures, invalid URLs.

---

#### 1.4 Post Publishing & Data Update

- **Overview:**  
  Publishes the LinkedIn post with the generated image and updates the Google Sheet to mark the topic as “done” with the post’s content, hashtags, and image URL.

- **Nodes Involved:**  
  - 7. Publish Post to LinkedIn  
  - Edit Fields (Set node)  
  - 8. Update Google Sheets

- **Node Details:**

  1. **7. Publish Post to LinkedIn**  
     - Type: LinkedIn Node  
     - Role: Publishes the post text together with the generated image on LinkedIn.  
     - Configuration: Text is constructed from title, main text, and hashtags concatenated with line breaks; media category set as IMAGE.  
     - Credentials: LinkedIn OAuth2 API configured.  
     - Inputs: Image file from “6. Download Generated Image” and post text from “4. Generate Branded Image Prompt”.  
     - Outputs: Post publishing confirmation data.  
     - Edge Cases: LinkedIn API rate limits, authentication errors, media upload failures.

  2. **Edit Fields**  
     - Type: Set node  
     - Role: Prepares data fields for Google Sheets update, including topic, text, hashtags, and image URL.  
     - Inputs: Post content and image URL from previous nodes.  
     - Outputs: Data to update Google Sheets.  
     - Edge Cases: Expression evaluation errors, missing data.

  3. **8. Update Google Sheets**  
     - Type: Google Sheets  
     - Role: Updates or appends the row corresponding to the topic, marking status as "done" and storing post data and image URL.  
     - Configuration: Matches rows by "Topic" column; updates columns: text, Status, hashtags, imageUrl.  
     - Credentials: Google Sheets OAuth2 API.  
     - Inputs: Data from “Edit Fields.”  
     - Outputs: Workflow end.  
     - Edge Cases: Google Sheets API errors, mismatched rows, permission issues.

---

### 3. Summary Table

| Node Name                          | Node Type                          | Functional Role                      | Input Node(s)                        | Output Node(s)                     | Sticky Note                                                                                                                                                                                                                             |
|-----------------------------------|----------------------------------|------------------------------------|------------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                   | Workflow entry point               | None                               | 1. Get Pending Topic from Google Sheets | # AI LinkedIn Content Factory: Quick Setup Guide. Configure credentials for Google Sheets, AI Agents, Replicate, LinkedIn. Prepare sheet with "Topic" and "Status" columns set to "Pending". Customize image style if needed.          |
| 1. Get Pending Topic from Google Sheets | Google Sheets                   | Fetch pending topics               | When clicking ‘Execute workflow’   | 2. Research Topic & Find Viral Angle with AI |                                                                                                                                                                                                                                       |
| SerpAPI1                          | SerpAPI Tool                    | Search for recent news/trends     | 2. Research Topic & Find Viral Angle with AI | 2. Research Topic & Find Viral Angle with AI | ### Phase 1: AI Research & Content Generation. Research Topic finds viral angles using SerpAPI and AI.                                                                                                                                 |
| OpenAI Chat Model1                | Langchain OpenAI Chat Model     | GPT-4.1-mini for research         | 2. Research Topic & Find Viral Angle with AI | 2. Research Topic & Find Viral Angle with AI |                                                                                                                                                                                                                                       |
| Structured Output Parser          | Langchain Output Parser         | Parse research AI output JSON     | 2. Research Topic & Find Viral Angle with AI | 2. Research Topic & Find Viral Angle with AI |                                                                                                                                                                                                                                       |
| 2. Research Topic & Find Viral Angle with AI | Langchain AI Agent             | Research trending topic           | 1. Get Pending Topic from Google Sheets | 3. Generate LinkedIn Post Content with AI |                                                                                                                                                                                                                                       |
| SerpAPI                          | SerpAPI Tool                    | Search support for content gen    | 3. Generate LinkedIn Post Content with AI | 3. Generate LinkedIn Post Content with AI |                                                                                                                                                                                                                                       |
| OpenAI Chat Model                | Langchain OpenAI Chat Model     | GPT-4.1-mini for post content     | 3. Generate LinkedIn Post Content with AI | 3. Generate LinkedIn Post Content with AI |                                                                                                                                                                                                                                       |
| Structured Output Parser1        | Langchain Output Parser         | Parse LinkedIn post JSON output   | 3. Generate LinkedIn Post Content with AI | 3. Generate LinkedIn Post Content with AI |                                                                                                                                                                                                                                       |
| 3. Generate LinkedIn Post Content with AI | Langchain AI Agent             | Generate LinkedIn post content    | 2. Research Topic & Find Viral Angle with AI | 4. Generate Branded Image Prompt |                                                                                                                                                                                                                                       |
| 4. Generate Branded Image Prompt | Code                            | Create initial image prompt       | 3. Generate LinkedIn Post Content with AI | 4.1. Generate Branded Image Prompt | ### Phase 2: AI Branded Image Generation. Combines post content with fixed brand style guide for image prompt.                                                                                                                        |
| 4.1. Generate Branded Image Prompt | Code                            | Refine image prompt with keywords | 4. Generate Branded Image Prompt    | 5a. Start Image Generation (Replicate) |                                                                                                                                                                                                                                       |
| 5a. Start Image Generation (Replicate) | HTTP Request                   | Send prompt to Replicate API      | 4.1. Generate Branded Image Prompt | 5b. Check Image Status           |                                                                                                                                                                                                                                       |
| 5b. Check Image Status           | HTTP Request                   | Poll Replicate for image status   | 5a. Start Image Generation (Replicate), Wait | If                             |                                                                                                                                                                                                                                       |
| If                              | Conditional                    | Check if image generation succeeded | 5b. Check Image Status             | 6. Download Generated Image (on success), Wait (on pending) |                                                                                                                                                                                                                                       |
| Wait                            | Delay                         | Wait before next status check     | If (on not succeeded)              | 5b. Check Image Status           |                                                                                                                                                                                                                                       |
| 6. Download Generated Image      | HTTP Request                   | Download final generated image    | If (on succeeded)                  | 7. Publish Post to LinkedIn       |                                                                                                                                                                                                                                       |
| 7. Publish Post to LinkedIn      | LinkedIn                      | Publish post with image           | 6. Download Generated Image        | Edit Fields                      |                                                                                                                                                                                                                                       |
| Edit Fields                     | Set                           | Prepare data fields for sheet update | 7. Publish Post to LinkedIn        | 8. Update Google Sheets           |                                                                                                                                                                                                                                       |
| 8. Update Google Sheets          | Google Sheets                 | Mark post as done and save data   | Edit Fields                       | None                            |                                                                                                                                                                                                                                       |
| Sticky Note                     | Sticky Note                   | Workflow overview and setup guide | None                             | None                            | # AI LinkedIn Content Factory Purpose and Quick Setup Guide: Credential setup, sheet prep, image style customization.                                                                                                                 |
| Sticky Note1                    | Sticky Note                   | Phase 1 explanation               | None                             | None                            | ### Phase 1: AI Research & Content Generation block acts as AI content strategist.                                                                                                                                                |
| Sticky Note2                    | Sticky Note                   | Phase 2 explanation               | None                             | None                            | ### Phase 2: AI Branded Image Generation block creates a unique, branded image prompt and manages generation.                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - Purpose: Initiate workflow manually.

2. **Create Google Sheets Node to Get Pending Topics**  
   - Type: Google Sheets  
   - Name: "1. Get Pending Topic from Google Sheets"  
   - Configuration:  
     - Operation: Read rows  
     - Sheet Name: gid=0 (or your target sheet)  
     - Document ID: Your Google Sheets document ID  
     - Filter: Where column "Status" equals "Pending"  
   - Credentials: Set your Google OAuth2 credentials.  
   - Connect output of Manual Trigger to this node.

3. **Create Langchain AI Agent for Research**  
   - Type: Langchain AI Agent  
   - Name: "2. Research Topic & Find Viral Angle with AI"  
   - Configuration:  
     - Text prompt asking to find 3 recent AI automation trends with high engagement potential, returning JSON with keys: selected_topic, research_data, engagement_score.  
     - Set system message restricting domain to AI/ML/automation.  
     - Enable Output Parser with JSON schema accordingly.  
   - Connect output of Google Sheets node here.

4. **Add SerpAPI Tool Node for Research**  
   - Type: SerpAPI Tool  
   - Name: "SerpAPI1"  
   - Credentials: Set SerpAPI key.  
   - Connect to AI Agent node as part of tools.

5. **Add Langchain OpenAI Chat Model for Research**  
   - Type: Langchain OpenAI Chat Model  
   - Name: "OpenAI Chat Model1"  
   - Model: GPT-4.1-mini  
   - Credentials: Set OpenAI API key.  
   - Connect to AI Agent node.

6. **Add Langchain Structured Output Parser for Research**  
   - Type: Langchain Output Parser Structured  
   - Name: "Structured Output Parser"  
   - JSON Schema example matching research output.  
   - Connect input from AI Agent research output.

7. **Create Langchain AI Agent for LinkedIn Post Generation**  
   - Type: Langchain AI Agent  
   - Name: "3. Generate LinkedIn Post Content with AI"  
   - Configuration:  
     - Prompt to generate LinkedIn post (150–250 words), including title, text, hashtags, hooks, CTAs, professional tone, strict JSON output with keys: title, text, hashtags, engagement_elements.  
     - System message emphasizing professional content creation.  
     - Enable Output Parser with matching JSON schema.  
   - Connect output of research AI Agent here.

8. **Add SerpAPI Tool Node for Content Generation**  
   - Name: "SerpAPI"  
   - Configure as above.  
   - Connect to AI Agent for content generation.

9. **Add Langchain OpenAI Chat Model for Content Generation**  
   - Name: "OpenAI Chat Model"  
   - Configure as above.  
   - Connect to AI Agent for content generation.

10. **Add Langchain Structured Output Parser for Content**  
    - Name: "Structured Output Parser1"  
    - JSON Schema example matching LinkedIn post output.  
    - Connect input from content AI Agent output.

11. **Create Code Node for Initial Image Prompt Generation**  
    - Name: "4. Generate Branded Image Prompt"  
    - JavaScript code:  
      - Accepts LinkedIn post JSON input.  
      - Combines title and text with fixed style guide (RAL color palette, lighting, composition).  
      - Outputs JSON with title, text, hashtags, imagePrompt string.

12. **Create Code Node for Refined Image Prompt**  
    - Name: "4.1. Generate Branded Image Prompt"  
    - JavaScript code:  
      - Extracts keywords from post text.  
      - Constructs final prompt appending keywords to style guide.  
      - Outputs imagePrompt string.

13. **Create HTTP Request Node to Start Image Generation (Replicate)**  
    - Name: "5a. Start Image Generation (Replicate)"  
    - Method: POST  
    - URL: https://api.replicate.com/v1/models/black-forest-labs/flux-1.1-pro-ultra/predictions  
    - Body (JSON): Contains input with imagePrompt, aspect ratio 3:2, output format jpg, safety tolerance 2, prompt strength 0.1  
    - Headers: Include "Prefer: wait"  
    - Credentials: HTTP Header Auth with Replicate API key.

14. **Create HTTP Request Node to Check Image Status**  
    - Name: "5b. Check Image Status"  
    - Method: GET  
    - URL: https://api.replicate.com/v1/predictions/{{ $json.id }}  
    - Credentials: Replicate API key.

15. **Create Conditional Node**  
    - Name: "If"  
    - Condition: Check if status equals "succeeded" in JSON response.  
    - True path: Proceed to download image.  
    - False path: Proceed to Wait node.

16. **Create Wait Node**  
    - Name: "Wait"  
    - Duration: 2 seconds delay.  
    - Connect False path of "If" back to "5b. Check Image Status" for polling loop.

17. **Create HTTP Request Node to Download Generated Image**  
    - Name: "6. Download Generated Image"  
    - Method: GET  
    - URL: Dynamic from successful Replicate output URL.  
    - Response format: File download.

18. **Create LinkedIn Node to Publish Post**  
    - Name: "7. Publish Post to LinkedIn"  
    - Text: Combine title, main text, hashtags with line breaks.  
    - Media: Upload downloaded image.  
    - Credentials: LinkedIn OAuth2 API.

19. **Create Set Node for Editing Fields**  
    - Name: "Edit Fields"  
    - Assign fields: topic, text, hashtags, imageUrl from previous nodes for Google Sheets update.

20. **Create Google Sheets Node to Update Post Status and Data**  
    - Name: "8. Update Google Sheets"  
    - Operation: Append or update row matching "Topic" column.  
    - Update columns: text, Status ("done"), hashtags, imageUrl.  
    - Credentials: Google Sheets OAuth2 API.

21. **Connect nodes in order:**  
    - Manual Trigger → Get Pending Topic → Research AI Agent → Content AI Agent → Generate Image Prompt (2 code nodes) → Start Image Generation (Replicate) → Check Image Status → If (succeeded → Download image → Publish LinkedIn → Edit Fields → Update Sheets; not succeeded → Wait → Check Image Status loop).

22. **Test all credentials and API connections before execution.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                             | Context or Link                                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is branded as **AI LinkedIn Content Factory** designed for automatic LinkedIn post creation with research, content writing, image branding. | Overview sticky note at start of workflow.                                                                                          |
| Fixed image style guide uses RAL color codes: 7016 (Anthracite Grey), 7035 (Light Grey), 3020 (Traffic Red) for consistent brand aesthetics.              | Defined in code nodes 4 and 4.1 for image prompt generation.                                                                         |
| AI Agents leverage OpenAI GPT-4.1-mini and SerpAPI for real-time, relevant data retrieval and content creation.                                         | Nodes 2, 3 and related Langchain nodes.                                                                                            |
| Image generation uses Replicate API’s "black-forest-labs/flux-1.1-pro-ultra" model with specific prompt strength and safety tolerance parameters.         | Nodes 5a and 5b handle generation and status polling.                                                                               |
| LinkedIn publishing requires appropriate OAuth2 credentials and media upload support.                                                                    | Node 7.                                                                                                                             |
| Google Sheets integration requires correct column naming ("Topic", "Status", etc.) and permissions.                                                     | Nodes 1 and 8.                                                                                                                      |
| Polling approach with Wait node ensures efficient image generation without excessive API calls.                                                         | Nodes 5b, If, and Wait.                                                                                                            |
| For detailed setup and credential configuration, refer to the sticky notes visible in the workflow canvas.                                              | Sticky Note nodes at positions near workflow start and AI blocks.                                                                  |
| Workflow designed for English language content with professional tone and strict JSON outputs for parsing.                                              | Enforced in AI Agent system messages and parser schemas.                                                                           |

---

This structured documentation enables advanced users or automation systems to understand, reproduce, and modify the LinkedIn Content Factory workflow efficiently and predict potential error points.