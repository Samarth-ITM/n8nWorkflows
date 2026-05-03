Automated Image Editing with GPT-4 Vision & Gemini for Instagram Posts from Airtable

https://n8nworkflows.xyz/workflows/automated-image-editing-with-gpt-4-vision---gemini-for-instagram-posts-from-airtable-8546


# Automated Image Editing with GPT-4 Vision & Gemini for Instagram Posts from Airtable

### 1. Workflow Overview

This workflow automates the generation and social media posting of creative Instagram images based on Airtable data inputs. It is designed for marketing teams or content creators who want to streamline visual content creation combined with AI-powered image analysis and editing. The workflow integrates Airtable for data management, OpenAI GPT-4 vision models for image analysis, Google Gemini for image generation based on prompts, and the Facebook Graph API to post on Instagram Business accounts.

The core logical blocks are:

- **1.1 Input Reception:** Receives trigger from an external webhook with a record ID.
- **1.2 Airtable Data Retrieval:** Fetches image and metadata records from Airtable based on the webhook input.
- **1.3 Image Analysis via GPT-4 Vision:** Analyzes the base image to extract detailed descriptive and stylistic information.
- **1.4 Prompt Optimization:** Combines user editing specifications with the analyzed image description to create a refined AI prompt.
- **1.5 AI Image Generation (Gemini):** Uses the optimized prompt and original image to create an edited image.
- **1.6 Airtable Upload & Status Update:** Uploads the generated image back to Airtable and updates the record status to “Completed” or “Error.”
- **1.7 Instagram Posting:** Creates a media container and publishes the new image post on Instagram Business via Facebook Graph API.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives webhook calls with query parameters, specifically a record ID, which triggers the workflow.

**Nodes Involved:**  
- Webhook

**Node Details:**  
- **Webhook**  
  - Type: Webhook  
  - Purpose: Trigger the workflow on incoming HTTP requests; expects a query parameter `recordId` to identify which image record to process.  
  - Configuration: Path is a unique UUID string; no authentication or additional options enabled.  
  - Inputs: External HTTP request  
  - Outputs: Passes HTTP query parameters downstream.  
  - Edge Cases: Missing or invalid recordId parameter may cause errors downstream. No explicit error handling here.

---

#### 2.2 Airtable Data Retrieval

**Overview:**  
Fetches specific image data and related metadata from Airtable based on the record ID received from the webhook. Also searches for rows with status “Create” for processing.

**Nodes Involved:**  
- Get Image  
- Get Rows

**Node Details:**  
- **Get Image**  
  - Type: Airtable node (read)  
  - Purpose: Retrieve the image record details by record ID from a specified Airtable base and table.  
  - Configuration: Uses the recordId from webhook query parameters.  
  - Inputs: Receives webhook output  
  - Outputs: Passes the full image record JSON downstream.  
  - Credential: Airtable personal access token  
  - Edge Cases: Record not found or API errors can cause failures here.

- **Get Rows**  
  - Type: Airtable node (search)  
  - Purpose: Searches for all records in the same Airtable table where `Status` equals “Create,” indicating images awaiting processing.  
  - Configuration: Retrieves specified fields relevant to image editing prompts.  
  - Inputs: Receives output of Analyze Image node (see below)  
  - Outputs: Passes matching rows downstream for prompt optimization.  
  - Credential: Airtable personal access token  
  - Edge Cases: Empty search results if no records with “Create” status exist.

---

#### 2.3 Image Analysis via GPT-4 Vision

**Overview:**  
Analyzes the base image to generate a detailed textual description and stylistic breakdown using OpenAI’s GPT-4 model with vision capabilities.

**Nodes Involved:**  
- Analyze image

**Node Details:**  
- **Analyze image**  
  - Type: OpenAI image analysis (Langchain integration)  
  - Purpose: Generate a structured text report describing the image’s content, style, lighting, mood, composition, colors, and any text overlays.  
  - Configuration: Uses the URL of the image from Airtable; prompts the model to produce a detailed, formatted report without extraneous text.  
  - Inputs: Receives Airtable “Get Image” output (image URL)  
  - Outputs: Provides the descriptive text downstream for prompt crafting.  
  - Credential: OpenAI API key  
  - Edge Cases: Network/API timeouts, image URL invalidity, or model errors could cause failures.

---

#### 2.4 Prompt Optimization

**Overview:**  
Combines the base image analysis output with user-specified editing instructions from Airtable to produce a refined AI prompt for image generation.

**Nodes Involved:**  
- Prompt Optimization  
- OpenAI Chat Model (auxiliary)

**Node Details:**  
- **Prompt Optimization**  
  - Type: Langchain LLM chain node  
  - Purpose: Constructs a detailed image editing prompt that incorporates the original image description and requested edits such as composition, lighting, style, atmosphere, color palette, and text overlay.  
  - Configuration: Uses expressions to pull user input fields and analyzed image content; formats prompt strictly to avoid conversational filler.  
  - Inputs: Receives “Get Rows” output (user edits) and “Analyze image” content  
  - Outputs: Text prompt for Gemini image generation  
  - Edge Cases: Expression errors if fields are missing or malformed.

- **OpenAI Chat Model**  
  - Type: AI language model node  
  - Purpose: Invokes GPT-4 model variant for refining or supporting prompt generation logic.  
  - Configuration: Uses “gpt-4.1-mini” model.  
  - Inputs: Connected to “Prompt Optimization” node as ai_languageModel input (optional/auxiliary)  
  - Outputs: Supports prompt creation.

---

#### 2.5 AI Image Generation (Gemini)

**Overview:**  
Generates a new edited image from the optimized prompt and the original image using Google’s Gemini model API.

**Nodes Involved:**  
- Nano Bannana Image Generation

**Node Details:**  
- **Nano Bannana Image Generation**  
  - Type: HTTP Request  
  - Purpose: Sends a POST request to Gemini’s image generation API with the editing prompt and original image data to produce a modified image.  
  - Configuration:  
    - Sends JSON body with combined text prompt and inline image data (base64 or URL).  
    - Uses temperature 0.7, max tokens 1000 to control creativity and output size.  
    - Uses batching with batch size 1 and 5-second interval.  
    - On error: configured to continue with error output to trigger error status update.  
  - Inputs: Receives prompt text from “Prompt Optimization” and image URL from “Get Image.”  
  - Outputs: On success routes to Airtable upload and Instagram posting; on error routes to status update node.  
  - Credential: HTTP header authentication with Gemini API key  
  - Edge Cases: API rate limits, network errors, invalid prompt format, image data issues.

---

#### 2.6 Airtable Upload & Status Update

**Overview:**  
Uploads the generated image back to Airtable attachment field and updates the record status either to “Completed” or “Error.”

**Nodes Involved:**  
- Airtable Upload  
- Status Complete  
- Status Error

**Node Details:**  
- **Airtable Upload**  
  - Type: HTTP Request (Airtable content API)  
  - Purpose: Uploads the generated image file as an attachment to the Airtable record’s “Nano Image” field.  
  - Configuration: POST request to Airtable content endpoint; uses dynamic URL and file metadata; sends JSON body including image URL and filename.  
  - Inputs: Receives generated image URL from Gemini node and record ID from “Get Rows.”  
  - Credential: Airtable personal access token  
  - Edge Cases: Upload failures due to API limits, invalid URLs, or permissions.

- **Status Complete**  
  - Type: Airtable node (update)  
  - Purpose: Updates the record’s “Status” field to “Completed” after successful image generation and upload.  
  - Configuration: Matches record by ID, updates “Status” and keeps other fields intact.  
  - Inputs: Triggered from Instagram post publish success.  
  - Credential: Airtable personal access token  
  - Edge Cases: Airtable update errors or conflicts.

- **Status Error**  
  - Type: Airtable node (update)  
  - Purpose: Updates the record’s status to “Error” and logs the error code/status from the Gemini API response.  
  - Configuration: Matches record by ID.  
  - Inputs: Triggered on Gemini API failure output.  
  - Credential: Airtable personal access token  
  - Edge Cases: Airtable update errors.

---

#### 2.7 Instagram Posting

**Overview:**  
Creates a media container for Instagram with the generated image and publishes it as a post on an Instagram Business account via Facebook Graph API.

**Nodes Involved:**  
- Create Media Container  
- Publish Media

**Node Details:**  
- **Create Media Container**  
  - Type: HTTP Request  
  - Purpose: Sends a POST request to Facebook Graph API to create a media container with the new image URL and caption.  
  - Configuration: Uses Instagram Business account ID placeholder; sends JSON with image_url, caption, and access token.  
  - Inputs: Receives generated image URL and caption text (currently placeholder).  
  - Credential: None in workflow JSON but requires valid Facebook API access token (hardcoded or environment).  
  - Edge Cases: Token expiration, invalid account ID, API errors.

- **Publish Media**  
  - Type: HTTP Request  
  - Purpose: Publishes the media container to Instagram by referencing the container ID.  
  - Configuration: POST request with creation_id from “Create Media Container” and access token.  
  - Inputs: Receives media container ID  
  - Credential: Same as above  
  - Outputs: Triggers “Status Complete” on success.  
  - Edge Cases: API permission errors, rate limits.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                                  | Input Node(s)             | Output Node(s)                           | Sticky Note                                      |
|-----------------------------|--------------------------------|-------------------------------------------------|---------------------------|-----------------------------------------|-------------------------------------------------|
| Webhook                     | Webhook                        | Receive external trigger with record ID          | External HTTP             | Get Image                               |                                                 |
| Get Image                   | Airtable                       | Retrieve image record by record ID                | Webhook                   | Analyze image                           |                                                 |
| Analyze image               | OpenAI Image Analysis          | Generate descriptive report of base image        | Get Image                 | Get Rows                               |                                                 |
| Get Rows                    | Airtable                       | Search records with status “Create” for editing  | Analyze image             | Prompt Optimization                    |                                                 |
| Prompt Optimization         | Langchain LLM Chain            | Create editing prompt combining base and edits   | Get Rows                  | Nano Bannana Image Generation          |                                                 |
| OpenAI Chat Model           | Langchain Chat Model           | Auxiliary LLM support for prompt optimization    | Prompt Optimization (ai)  | Prompt Optimization                    |                                                 |
| Nano Bannana Image Generation | HTTP Request (Gemini API)      | Generate edited image from prompt and base image | Prompt Optimization       | Airtable Upload, Create Media Container, Status Error |                                                 |
| Airtable Upload             | HTTP Request (Airtable content)| Upload generated image attachment to Airtable    | Nano Bannana Image Generation | (none)                               |                                                 |
| Create Media Container      | HTTP Request (Facebook Graph)  | Create Instagram media container for posting     | Nano Bannana Image Generation | Publish Media                         |                                                 |
| Publish Media               | HTTP Request (Facebook Graph)  | Publish media container as Instagram post         | Create Media Container    | Status Complete                        |                                                 |
| Status Complete             | Airtable                       | Update record status to “Completed”                | Publish Media             | (none)                                |                                                 |
| Status Error                | Airtable                       | Update record status to “Error” and log error     | Nano Bannana Image Generation (on error) | (none)                    |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook node:**  
   - Set type to Webhook (version 2.1).  
   - Configure unique webhook path (e.g., UUID).  
   - No authentication.  
   - This node triggers the workflow and expects a query parameter `recordId`.

2. **Create Airtable 'Get Image' node:**  
   - Use Airtable node (version 2.1).  
   - Configure with Airtable Personal Access Token credentials.  
   - Set base and table to your Airtable base/table with images.  
   - Use expression for record ID: `={{ $json.query.recordId }}`.  
   - Connect output of Webhook to this node.

3. **Create OpenAI 'Analyze image' node:**  
   - Use the Langchain OpenAI node (version 1.8).  
   - Set resource to Image, operation to Analyze.  
   - Set model to GPT-4o-mini.  
   - Provide detailed prompt requesting structured analysis of the image (see above for prompt text).  
   - Set image URL to the first attachment URL from Airtable record.  
   - Connect output of 'Get Image' to this node.

4. **Create Airtable 'Get Rows' node:**  
   - Airtable node (version 2.1), same base/table as above.  
   - Use operation 'search' with filter formula `{Status} = "Create"`.  
   - Select fields needed for prompt optimization (e.g., Style, Lighting, etc.).  
   - Connect output of 'Analyze image' to this node.

5. **Create Prompt Optimization node:**  
   - Langchain chainLlm node (version 1.7).  
   - Use expressions to build prompt combining analyzed image content and user edits from 'Get Rows'.  
   - Configure prompt to produce only the image editing prompt text, no extra explanation.  
   - Connect output of 'Get Rows' to this node.

6. **Optionally add OpenAI Chat Model node:**  
   - Langchain lmChatOpenAi node (version 1.2) with GPT-4.1-mini model.  
   - Connect as ai_languageModel input to 'Prompt Optimization' node.

7. **Create HTTP Request node for Gemini Image Generation:**  
   - HTTP Request node (version 4.2).  
   - Method: POST  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
   - JSON body: includes prompt text and inline image data from 'Get Image' node.  
   - Use HTTP Header Auth credentials with Gemini API key.  
   - Configure batching (batch size 1, interval 5000 ms).  
   - Error workflow set to continue with error output.  
   - Connect output of 'Prompt Optimization' to this node.

8. **Create Airtable Upload node:**  
   - HTTP Request node (version 4.2).  
   - POST to Airtable content URL to upload image attachment.  
   - Use Airtable Personal Access Token credentials.  
   - Use dynamic URL and JSON body referencing Gemini output image URL.  
   - Connect success output from Gemini node.

9. **Create HTTP Request nodes for Instagram Posting:**  
   - **Create Media Container:**  
     - POST to `https://graph.facebook.com/v18.0/{instagram-business-account-id}/media`  
     - JSON body: includes image_url (generated image), caption, access_token (Facebook).  
     - Connect success output of Gemini node as well (parallel to Airtable upload).  
   - **Publish Media:**  
     - POST to `https://graph.facebook.com/v18.0/{instagram-business-account-id}/media_publish`  
     - JSON body: includes creation_id from previous node and access_token.  
     - Connect from 'Create Media Container' node.

10. **Create Airtable 'Status Complete' node:**  
    - Airtable node (version 2.1).  
    - Update record’s `Status` field to “Completed.”  
    - Match record by ID from 'Get Rows.'  
    - Connect output of 'Publish Media' node.

11. **Create Airtable 'Status Error' node:**  
    - Airtable node (version 2.1).  
    - Update record’s `Status` field to “Error” with error details from Gemini output.  
    - Match record by ID.  
    - Connect error output of Gemini node to this node.

12. **Connect all nodes as described:**  
    - Webhook → Get Image → Analyze Image → Get Rows → Prompt Optimization → Nano Bannana Image Generation  
    - Nano Bannana Image Generation success → Airtable Upload + Create Media Container  
    - Create Media Container → Publish Media → Status Complete  
    - Nano Bannana Image Generation error → Status Error

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow uses OpenAI GPT-4o-mini model for image analysis and GPT-4.1-mini for prompt refinement. | OpenAI API documentation: https://platform.openai.com/docs/models/gpt-4                            |
| Gemini API requires HTTP Header authentication with a specific API key.                           | Gemini generative language API docs (internal Google API, access restricted)                       |
| Instagram Business posting requires valid Facebook Graph API tokens and Instagram Business account ID. | Facebook Graph API for Instagram: https://developers.facebook.com/docs/instagram-api               |
| Airtable content upload uses the Airtable content API endpoint for attachments.                   | Airtable API documentation: https://airtable.com/api/content-api                                   |
| The workflow assumes consistent field naming and structure in Airtable table “NannoB.”           | Ensure Airtable schema matches fields referenced in nodes (e.g., Status, Nano Image, etc.)         |
| Error handling is implemented only for Gemini API failures; other nodes lack explicit error capture. | Consider adding error capture for Airtable and API nodes to improve robustness.                    |

---

**Disclaimer:** The provided text is derived exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.