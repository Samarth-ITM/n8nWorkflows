Summarize YouTube videos and generate thumbnails using Anthropic and deAPI

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-and-generate-thumbnails-using-anthropic-and-deapi-13889


# Summarize YouTube videos and generate thumbnails using Anthropic and deAPI

# 1. Workflow Overview

This workflow takes a video URL, transcribes the video content with deAPI, summarizes the transcript with Anthropic via an n8n AI Agent, generates an optimized YouTube thumbnail prompt using deAPI’s prompt booster tool, creates a thumbnail image, and uploads that image to Google Drive.

Primary use cases:
- Summarizing long-form YouTube videos
- Creating AI-generated thumbnails for content repurposing
- Building a lightweight content-processing pipeline for creators or media teams
- Extending to other supported video platforms such as Twitch VODs, X, and Kick

Logical blocks:

## 1.1 Trigger and Input Setup
Starts the workflow manually and defines the source video URL.

## 1.2 Video Transcription
Sends the video URL to deAPI for transcription and returns the transcript text.

## 1.3 AI Summary and Prompt Engineering
Uses an Anthropic chat model inside an AI Agent to:
- summarize the transcript,
- generate a thumbnail concept,
- call the deAPI prompt booster tool,
- enforce structured JSON output.

## 1.4 Thumbnail Generation
Uses the boosted prompt to generate a 1280x720 thumbnail image via deAPI.

## 1.5 File Upload
Uploads the generated thumbnail to Google Drive with a dynamic filename.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and Input Setup

**Overview:**  
This block initializes the workflow and provides the video URL to process. It is currently manual, making it ideal for testing, but it can be replaced by a form or webhook-based trigger.

**Nodes Involved:**
- When clicking 'Execute workflow'
- Set YouTube URL

### Node: When clicking 'Execute workflow'
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; entry-point node for manual execution.
- **Configuration choices:** No parameters are configured; execution starts when the user clicks the test/execute button.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Input: none  
  - Output: `Set YouTube URL`
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**  
  - No runtime failure expected from the node itself.  
  - Not suitable for unattended production automation.
- **Sub-workflow reference:** None.

### Node: Set YouTube URL
- **Type and technical role:** `n8n-nodes-base.set`; creates a field containing the video URL.
- **Configuration choices:**  
  - Assigns a string field named `youtubeUrl`
  - Default value is a placeholder: `https://www.youtube.com/watch?v=YOUR_VIDEO_ID`
- **Key expressions or variables used:**  
  - Output field: `{{$json.youtubeUrl}}` becomes available downstream
- **Input and output connections:**  
  - Input: `When clicking 'Execute workflow'`  
  - Output: `deAPI Transcribe Video`
- **Version-specific requirements:** Type version `3.4`.
- **Edge cases or potential failure types:**  
  - If the placeholder is not replaced, transcription will likely fail or deAPI will reject the request.
  - Invalid or private video URLs may cause downstream API errors.
  - If switched to another trigger type, ensure `youtubeUrl` is still present in the incoming data.
- **Sub-workflow reference:** None.

---

## 2.2 Video Transcription

**Overview:**  
This block sends the supplied video URL to deAPI for speech-to-text transcription. The returned transcript is the core input for the AI Agent.

**Nodes Involved:**
- deAPI Transcribe Video

### Node: deAPI Transcribe Video
- **Type and technical role:** `n8n-nodes-deapi.deapi`; deAPI application node configured for video transcription.
- **Configuration choices:**  
  - Resource: `video`
  - Operation: `transcribe`
  - Video URL is read from the input field
  - `includeTimestamps` is set to `false`
  - Wait timeout is increased to `600` seconds
- **Key expressions or variables used:**  
  - `={{ $json.youtubeUrl }}`
  - Downstream transcript field referenced later as `{{$json.data.transcription}}`
- **Input and output connections:**  
  - Input: `Set YouTube URL`  
  - Output: `AI Agent`
- **Version-specific requirements:** Type version `1`; requires the deAPI community/integration node to be installed in the n8n instance.
- **Edge cases or potential failure types:**  
  - Invalid deAPI credentials
  - Unsupported or inaccessible video URL
  - Long media processing time exceeding timeout
  - API rate limits or quota exhaustion
  - Unexpected response shape if deAPI changes its schema
  - Transcription quality issues for noisy audio, accents, or music-heavy videos
- **Sub-workflow reference:** None.

---

## 2.3 AI Summary and Prompt Engineering

**Overview:**  
This block uses an n8n AI Agent with Anthropic as the language model. It receives the transcript, produces a summary, generates a thumbnail concept, calls the deAPI image prompt booster tool to refine that concept, and returns structured JSON parsed by the output parser.

**Nodes Involved:**
- AI Agent
- Anthropic Chat Model
- Image prompt booster in deAPI
- Structured Output Parser

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; orchestrates LLM reasoning, tool usage, and structured output.
- **Configuration choices:**  
  - Prompt type: defined directly in the node
  - User text includes the transcript from the deAPI transcription result
  - System message instructs the agent to:
    1. write a concise summary with 3–5 bullet points,
    2. create a compelling thumbnail prompt,
    3. use the `imagePromptBooster` tool,
    4. return results in structured JSON
  - Output parser is enabled
- **Key expressions or variables used:**  
  - Input transcript: `{{ $json.data.transcription }}`
  - Downstream parsed output is later used as `{{$json.output.boosted_prompt}}`
- **Input and output connections:**  
  - Main input: `deAPI Transcribe Video`  
  - Main output: `deAPI Generate Thumbnail`  
  - AI language model input: `Anthropic Chat Model`  
  - AI tool input: `Image prompt booster in deAPI`  
  - AI output parser input: `Structured Output Parser`
- **Version-specific requirements:** Type version `1.7`; requires n8n AI/langchain nodes available in the instance.
- **Edge cases or potential failure types:**  
  - Missing transcript field if deAPI response format differs
  - Model/tool call failures
  - Output parser mismatch if the model does not follow the required schema
  - Token/context issues for very long transcripts
  - Hallucinated or weak summaries if transcript quality is poor
- **Sub-workflow reference:** None.

### Node: Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`; provides the LLM used by the AI Agent.
- **Configuration choices:**  
  - Model selected: `claude-opus-4-6`
  - No custom options configured
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - Output via AI connector to `AI Agent`
- **Version-specific requirements:** Type version `1.3`; requires valid Anthropic API credentials and model access in the Anthropic account.
- **Edge cases or potential failure types:**  
  - Invalid credentials
  - Model access not enabled on the Anthropic account
  - Rate limiting
  - Context-length constraints for large transcripts
- **Sub-workflow reference:** None.

### Node: Image prompt booster in deAPI
- **Type and technical role:** `n8n-nodes-deapi.deapiTool`; exposes deAPI prompt enhancement as an AI tool callable by the agent.
- **Configuration choices:**  
  - Resource: `prompt`
  - Prompt input is dynamically supplied by the AI Agent using `$fromAI(...)`
- **Key expressions or variables used:**  
  - `={{ $fromAI('Prompt', \`\`, 'string') }}`
- **Input and output connections:**  
  - Output via AI tool connector to `AI Agent`
- **Version-specific requirements:** Type version `1`; requires deAPI credentials and support for the tool node in the installed package.
- **Edge cases or potential failure types:**  
  - Tool invocation may fail if the model does not provide a proper prompt argument
  - Credential or deAPI service errors
  - Agent may skip or misuse the tool depending on model behavior
- **Sub-workflow reference:** None.

### Node: Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`; enforces a predictable JSON schema for the AI Agent output.
- **Configuration choices:**  
  - Example schema defines:
    - `summary`
    - `boosted_prompt`
- **Key expressions or variables used:** None directly; parser shapes the final `output` object.
- **Input and output connections:**  
  - Output via AI output parser connector to `AI Agent`
- **Version-specific requirements:** Type version `1.2`.
- **Edge cases or potential failure types:**  
  - If the model response cannot be parsed into the expected schema, the AI Agent execution can fail.
  - Field names must match downstream expectations exactly, especially `boosted_prompt`.
- **Sub-workflow reference:** None.

---

## 2.4 Thumbnail Generation

**Overview:**  
This block creates a YouTube thumbnail image from the AI-generated boosted prompt. The output is expected to be an image asset suitable for upload.

**Nodes Involved:**
- deAPI Generate Thumbnail

### Node: deAPI Generate Thumbnail
- **Type and technical role:** `n8n-nodes-deapi.deapi`; deAPI image-generation node.
- **Configuration choices:**  
  - Ratio: `landscape`
  - Prompt is taken from the AI Agent structured output
  - Wait timeout: `120` seconds
  - Explicit landscape size: `1280x720`
- **Key expressions or variables used:**  
  - `={{ $json.output.boosted_prompt }}`
- **Input and output connections:**  
  - Input: `AI Agent`  
  - Output: `Google Drive Upload`
- **Version-specific requirements:** Type version `1`; requires the deAPI integration node and valid credentials.
- **Edge cases or potential failure types:**  
  - Missing `output.boosted_prompt` if AI parsing failed or schema changed
  - API timeout for image generation
  - Content moderation rejections by the image provider
  - Response format mismatch affecting downstream binary/file upload behavior
- **Sub-workflow reference:** None.

---

## 2.5 File Upload

**Overview:**  
This block uploads the generated thumbnail into a specified Google Drive folder. The filename is dynamically created from the deAPI job request ID.

**Nodes Involved:**
- Google Drive Upload

### Node: Google Drive Upload
- **Type and technical role:** `n8n-nodes-base.googleDrive`; uploads generated content to Google Drive.
- **Configuration choices:**  
  - File name pattern: `thumbnail_{{ $json.job_request_id }}.png`
  - Drive: `My Drive`
  - Folder ID is manually specified
- **Key expressions or variables used:**  
  - `=thumbnail_{{ $json.job_request_id }}.png`
- **Input and output connections:**  
  - Input: `deAPI Generate Thumbnail`  
  - Output: none
- **Version-specific requirements:** Type version `3`; requires Google Drive OAuth2 credentials.
- **Edge cases or potential failure types:**  
  - Invalid or expired OAuth token
  - Incorrect folder ID
  - Missing binary data or unsupported upload mapping if the deAPI node output is not in the expected format
  - Filename collisions depending on Google Drive behavior and node settings
  - Permission issues if uploading to a shared or restricted folder
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note - Overview | n8n-nodes-base.stickyNote | Visual documentation block describing the workflow |  |  | ## Try It Out!<br>### Automatically summarize any YouTube video and generate a custom thumbnail using AI.<br><br>This workflow transcribes a YouTube video, uses AI to create a summary and an optimized thumbnail prompt, generates a professional thumbnail image, and uploads everything to Google Drive.<br><br>### How it works<br>1. **Manual Trigger** starts the workflow<br>2. **Set YouTube URL** defines the video to process<br>3. **deAPI Transcribe** extracts the full transcript using Whisper<br>4. **AI Agent** summarizes the transcript and uses the deAPI Prompt Booster tool to craft an optimized thumbnail prompt<br>5. **deAPI Generate Image** creates a professional thumbnail in landscape (1280x720)<br>6. **Google Drive** uploads the thumbnail image<br><br>### Requirements<br>- [deAPI](https://deapi.ai) account for transcription and image generation<br>- Anthropic account for AI Agent<br>- Google Drive API access for file uploads<br>- n8n instance must be on **HTTPS**<br><br>### Need Help?<br>Join the [n8n Discord](https://discord.gg/n8n) or ask in the [Forum](https://community.n8n.io/)!<br><br>Happy Automating! |
| Sticky Note - Trigger | n8n-nodes-base.stickyNote | Visual note for the trigger stage |  |  | ## 1. Manual Trigger<br>Click **Test workflow** to start.<br><br>You can swap this for a **Form Trigger** to let users submit YouTube URLs through a web form. |
| Sticky Note - Set URL | n8n-nodes-base.stickyNote | Visual note for the URL definition stage |  |  | ## 2. Set YouTube URL<br>Replace the placeholder URL with any YouTube video link.<br><br>Supported platforms:<br>- YouTube<br>- Twitch VODs<br>- X (Twitter)<br>- Kick |
| Sticky Note - Transcribe | n8n-nodes-base.stickyNote | Visual note for transcription stage |  |  | ## 3. Transcribe Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Uses **Whisper Large V3** to transcribe the video with timestamps.<br><br>The transcript is passed to the AI Agent for summarization. |
| Sticky Note - AI Agent | n8n-nodes-base.stickyNote | Visual note for AI summarization/prompting stage |  |  | ## 4. AI-Powered Summary & Prompt<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the transcript and:<br>- Writes a concise summary with key points<br>- Crafts a thumbnail prompt using the **deAPI Image Prompt Booster** tool<br><br>The **Structured Output Parser** ensures consistent JSON output. |
| Sticky Note - Generate | n8n-nodes-base.stickyNote | Visual note for image generation stage |  |  | ## 5. Generate Thumbnail<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Generates a **1280x720** landscape image — the standard YouTube thumbnail size.<br><br>Uses the AI-boosted prompt for best results. |
| Sticky Note - Upload | n8n-nodes-base.stickyNote | Visual note for Google Drive upload stage |  |  | ## 6. Upload to Google Drive<br>[Read more about Google Drive node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive)<br><br>Uploads the generated thumbnail image to your Google Drive.<br><br>You can also extend this to save the summary to a Google Doc or send it via Slack/email. |
| When clicking 'Execute workflow' | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Set YouTube URL | ## 1. Manual Trigger<br>Click **Test workflow** to start.<br><br>You can swap this for a **Form Trigger** to let users submit YouTube URLs through a web form. |
| Set YouTube URL | n8n-nodes-base.set | Defines the source video URL as `youtubeUrl` | When clicking 'Execute workflow' | deAPI Transcribe Video | ## 2. Set YouTube URL<br>Replace the placeholder URL with any YouTube video link.<br><br>Supported platforms:<br>- YouTube<br>- Twitch VODs<br>- X (Twitter)<br>- Kick |
| deAPI Transcribe Video | n8n-nodes-deapi.deapi | Transcribes the source video into text | Set YouTube URL | AI Agent | ## 3. Transcribe Video<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Uses **Whisper Large V3** to transcribe the video with timestamps.<br><br>The transcript is passed to the AI Agent for summarization. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Summarizes transcript and orchestrates prompt boosting with structured output | deAPI Transcribe Video | deAPI Generate Thumbnail | ## 4. AI-Powered Summary & Prompt<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the transcript and:<br>- Writes a concise summary with key points<br>- Crafts a thumbnail prompt using the **deAPI Image Prompt Booster** tool<br><br>The **Structured Output Parser** ensures consistent JSON output. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforces JSON schema for AI Agent output |  | AI Agent | ## 4. AI-Powered Summary & Prompt<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the transcript and:<br>- Writes a concise summary with key points<br>- Crafts a thumbnail prompt using the **deAPI Image Prompt Booster** tool<br><br>The **Structured Output Parser** ensures consistent JSON output. |
| Image prompt booster in deAPI | n8n-nodes-deapi.deapiTool | AI tool that enhances thumbnail prompts |  | AI Agent | ## 4. AI-Powered Summary & Prompt<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the transcript and:<br>- Writes a concise summary with key points<br>- Crafts a thumbnail prompt using the **deAPI Image Prompt Booster** tool<br><br>The **Structured Output Parser** ensures consistent JSON output. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | LLM backend for the AI Agent |  | AI Agent | ## 4. AI-Powered Summary & Prompt<br>[Read more about AI Agents](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)<br><br>The AI Agent analyzes the transcript and:<br>- Writes a concise summary with key points<br>- Crafts a thumbnail prompt using the **deAPI Image Prompt Booster** tool<br><br>The **Structured Output Parser** ensures consistent JSON output. |
| deAPI Generate Thumbnail | n8n-nodes-deapi.deapi | Generates a 1280x720 thumbnail image from the boosted prompt | AI Agent | Google Drive Upload | ## 5. Generate Thumbnail<br>[deAPI Documentation](https://docs.deapi.ai)<br><br>Generates a **1280x720** landscape image — the standard YouTube thumbnail size.<br><br>Uses the AI-boosted prompt for best results. |
| Google Drive Upload | n8n-nodes-base.googleDrive | Uploads the generated image to Google Drive | deAPI Generate Thumbnail |  | ## 6. Upload to Google Drive<br>[Read more about Google Drive node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive)<br><br>Uploads the generated thumbnail image to your Google Drive.<br><br>You can also extend this to save the summary to a Google Doc or send it via Slack/email. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `YouTube Video Summarizer & Thumbnail Generator`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Keep default settings.
   - This will be the workflow entry point.

3. **Add a Set node**
   - Node type: `Set`
   - Connect `Manual Trigger -> Set`
   - Configure one field:
     - `youtubeUrl` = string
     - Example value: `https://www.youtube.com/watch?v=YOUR_VIDEO_ID`
   - This field must exist because downstream transcription reads from it.

4. **Add the deAPI transcription node**
   - Node type: `deAPI`
   - Connect `Set -> deAPI`
   - Configure:
     - Resource: `video`
     - Operation: `transcribe`
     - Video URL: `{{ $json.youtubeUrl }}`
     - Include timestamps: `false`
     - Wait timeout: `600`
   - Credentials:
     - Create/select a `deAPI` credential
     - Use your deAPI API key or required auth method supported by the installed node
   - Expected output:
     - A JSON payload containing transcription data, including `data.transcription`

5. **Add an AI Agent node**
   - Node type: `AI Agent`
   - Connect `deAPI Transcribe Video -> AI Agent`
   - Configure prompt mode as a directly defined prompt.
   - User prompt / text:
     - `Here is the transcript of a YouTube video. Please summarize it and create a thumbnail prompt.`
     - Include transcript expression:
       `{{ $json.data.transcription }}`
   - Enable structured output / output parser.
   - Set system message to instruct the agent to:
     - summarize the transcript,
     - produce 3–5 key bullet points,
     - create a compelling thumbnail prompt,
     - use the deAPI image prompt booster tool,
     - return JSON.

6. **Add an Anthropic Chat Model node**
   - Node type: `Anthropic Chat Model`
   - Connect it to the AI Agent using the **AI language model** connection.
   - Configure:
     - Model: `claude-opus-4-6`
   - Credentials:
     - Create/select `Anthropic API` credentials
     - Ensure your Anthropic account has access to the selected model

7. **Add a Structured Output Parser node**
   - Node type: `Structured Output Parser`
   - Connect it to the AI Agent using the **AI output parser** connection.
   - Configure the schema example with these fields:
     - `summary`
     - `boosted_prompt`
   - Example structure:
     - `summary`: string containing bullet points
     - `boosted_prompt`: string containing the enhanced image prompt
   - Make sure the field name is exactly `boosted_prompt`, since the next node depends on it.

8. **Add the deAPI Tool node for prompt boosting**
   - Node type: `deAPI Tool`
   - Connect it to the AI Agent using the **AI tool** connection.
   - Configure:
     - Resource: `prompt`
     - Prompt input: `{{ $fromAI('Prompt', '', 'string') }}`
   - Credentials:
     - Reuse the same `deAPI` credentials
   - This allows the agent to call deAPI during reasoning to improve the thumbnail prompt.

9. **Add the deAPI image-generation node**
   - Node type: `deAPI`
   - Connect `AI Agent -> deAPI Generate Thumbnail`
   - Configure:
     - Prompt: `{{ $json.output.boosted_prompt }}`
     - Ratio: `landscape`
     - Landscape size: `1280x720`
     - Wait timeout: `120`
   - Credentials:
     - Reuse the same `deAPI` credentials
   - Expected output:
     - Image generation response including the generated asset and job metadata such as `job_request_id`

10. **Add a Google Drive node**
    - Node type: `Google Drive`
    - Connect `deAPI Generate Thumbnail -> Google Drive`
    - Configure upload behavior for a file
    - Set filename:
      - `thumbnail_{{ $json.job_request_id }}.png`
    - Select drive:
      - `My Drive`
    - Set destination folder ID:
      - your target Google Drive folder ID
    - Credentials:
      - Create/select `Google Drive OAuth2 API` credentials
      - Authenticate with an account that has write access to the target folder

11. **Verify binary/file handling in the Google Drive node**
    - Confirm the deAPI image-generation node outputs file/binary data in a format the Google Drive node can upload.
    - If your installed deAPI node version returns only a URL instead of binary data, add an intermediate HTTP Request or file-download node before Google Drive.
    - This is the most likely implementation detail to vary by node version.

12. **Add optional sticky notes**
    - Add notes for:
      - trigger usage,
      - URL replacement,
      - deAPI requirements,
      - Anthropic requirements,
      - Google Drive upload notes.

13. **Credential setup checklist**
    - **deAPI**
      - Required for transcription and image generation
      - Also required for the prompt booster tool
    - **Anthropic**
      - Required for the AI Agent’s LLM
      - Confirm access to `claude-opus-4-6`
    - **Google Drive OAuth2**
      - Required for upload
      - n8n should be accessible over HTTPS for OAuth flows, as noted in the workflow comments

14. **Test the workflow**
    - Replace the placeholder YouTube URL with a real public video.
    - Execute the workflow manually.
    - Validate each stage:
      - transcription exists,
      - AI output contains `output.summary` and `output.boosted_prompt`,
      - image generation completes,
      - file appears in Google Drive.

15. **Production-hardening recommendations**
    - Replace the Manual Trigger with:
      - Form Trigger,
      - Webhook,
      - Schedule Trigger,
      - or another event source.
    - Add validation for `youtubeUrl`.
    - Add error handling branches or an Error Trigger workflow.
    - Store the AI summary as a separate file or send it to Slack/email/Docs.

**Sub-workflow setup:**  
This workflow does not contain any Execute Workflow or sub-workflow nodes. No sub-workflows are required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| deAPI account is required for both transcription and image generation. | https://deapi.ai |
| deAPI documentation is referenced in the transcription and generation stages. | https://docs.deapi.ai |
| Anthropic account is required for the AI Agent language model. | Anthropic credentials in n8n |
| n8n AI Agent documentation is referenced in the workflow notes. | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent |
| Google Drive node documentation is referenced in the upload stage. | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive |
| The workflow note states that the n8n instance must be on HTTPS. This is especially relevant for OAuth-based integrations such as Google Drive. | General deployment requirement |
| Community help links provided in the workflow notes. | n8n Discord: https://discord.gg/n8n |
| Community help links provided in the workflow notes. | n8n Forum: https://community.n8n.io/ |