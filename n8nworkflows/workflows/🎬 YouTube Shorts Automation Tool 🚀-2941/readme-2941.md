🎬 YouTube Shorts Automation Tool 🚀

https://n8nworkflows.xyz/workflows/---youtube-shorts-automation-tool----2941


# 🎬 YouTube Shorts Automation Tool 🚀

### 1. Workflow Overview

The **YouTube Shorts Automation Tool** workflow automates the end-to-end creation of engaging YouTube Shorts videos from a simple text prompt. It targets content creators, marketing agencies, and business owners who want to produce viral short-form videos quickly without video editing skills.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the initial video idea via chat interface.
- **1.2 Script Generation & Organization:** Uses AI to generate and organize a video script optimized for SEO and engagement.
- **1.3 Visual Content Creation:** Generates image prompts, requests images, and retrieves them.
- **1.4 Video Assembly & Processing:** Requests video generation, waits for completion, retrieves video assets, and uploads media to Cloudinary.
- **1.5 Video Editing & Rendering:** Merges media, creates editor JSON, sends to video editor API, waits for rendering, and retrieves the final video.
- **1.6 Output Delivery:** Provides the final video link for sharing.

Each block consists of nodes connected sequentially or in parallel to handle specific tasks, integrating multiple APIs such as OpenAI, ElevenLabs, Cloudinary, Replicate, and Creatomate.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming chat messages containing video ideas and triggers the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**  
  - **When chat message received**  
    - Type: Chat Trigger (Langchain)  
    - Role: Entry point webhook that activates the workflow when a chat message is received.  
    - Configuration: Default webhook with no parameters; listens for chat input.  
    - Inputs: External chat message (e.g., "top 10 größten brücken")  
    - Outputs: Passes chat input to the next node.  
    - Edge Cases: Webhook failures, malformed chat input, or missing session ID.  
    - Version: 1.1  

---

#### 2.2 Script Generation & Organization

- **Overview:**  
  Generates a detailed video script from the input idea using AI, then organizes the script for further processing.

- **Nodes Involved:**  
  - Script Prompter  
  - Script Organizer  
  - Script Generator  
  - Outliner  
  - Upload to Cloudinary

- **Node Details:**  
  - **Script Prompter**  
    - Type: OpenAI (Langchain)  
    - Role: Sends the chat input to OpenAI to generate a script prompt.  
    - Configuration: Uses OpenAI credentials, prompt templates optimized for SEO and engagement.  
    - Inputs: Chat message from "When chat message received"  
    - Outputs: Script prompt JSON to "Script Organizer"  
    - Edge Cases: API rate limits, prompt failures, invalid responses.  
    - Version: 1.8  

  - **Script Organizer**  
    - Type: Set  
    - Role: Structures and formats the AI-generated script for downstream nodes.  
    - Configuration: Sets variables and formats JSON for the next HTTP request.  
    - Inputs: Script prompt from "Script Prompter"  
    - Outputs: Organized script data to "Script Generator"  
    - Edge Cases: Expression errors, missing fields.  
    - Version: 3.4  

  - **Script Generator**  
    - Type: HTTP Request  
    - Role: Calls an external API (likely OpenAI or ElevenLabs) to generate the full script text.  
    - Configuration: HTTP POST with script data, includes authentication headers.  
    - Inputs: Organized script data from "Script Organizer"  
    - Outputs: Script text to "Outliner" and triggers "Upload to Cloudinary"  
    - Edge Cases: HTTP errors, timeouts, auth failures.  
    - Version: 4.2  

  - **Outliner**  
    - Type: HTTP Request  
    - Role: Processes the script to create an outline or split it into segments.  
    - Configuration: Calls an API endpoint with the script text.  
    - Inputs: Script text from "Script Generator"  
    - Outputs: Outline data to "Split Out" node  
    - Edge Cases: API errors, malformed script input.  
    - Version: 4.2  

  - **Upload to Cloudinary**  
    - Type: HTTP Request  
    - Role: Uploads generated script or related assets to Cloudinary for storage.  
    - Configuration: Uses Cloudinary API with authentication.  
    - Inputs: Script data from "Script Generator"  
    - Outputs: Upload confirmation to "Merge" node  
    - Edge Cases: Upload failures, network issues, auth errors.  
    - Version: 4.2  

---

#### 2.3 Visual Content Creation

- **Overview:**  
  Generates image prompts from script segments, requests images from an image generation API, waits for image processing, and retrieves images.

- **Nodes Involved:**  
  - Split Out  
  - Image Prompter  
  - Request Image  
  - Wait For Image  
  - Get Image

- **Node Details:**  
  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the outline or script segments into individual items for parallel processing.  
    - Configuration: Default splitting of array items.  
    - Inputs: Outline data from "Outliner"  
    - Outputs: Individual script segments to "Image Prompter"  
    - Edge Cases: Empty arrays, malformed data.  
    - Version: 1  

  - **Image Prompter**  
    - Type: OpenAI (Langchain)  
    - Role: Generates descriptive image prompts based on script segments.  
    - Configuration: Uses OpenAI with prompt templates for image description.  
    - Inputs: Script segment from "Split Out"  
    - Outputs: Image prompt text to "Request Image"  
    - Edge Cases: API limits, prompt failures.  
    - Version: 1.8  

  - **Request Image**  
    - Type: HTTP Request  
    - Role: Sends image prompt to an image generation API (e.g., Replicate) to create images.  
    - Configuration: HTTP POST with prompt, includes API key.  
    - Inputs: Image prompt from "Image Prompter"  
    - Outputs: Image generation job ID to "Wait For Image"  
    - Edge Cases: API errors, rate limits, invalid prompts.  
    - Version: 4.2  

  - **Wait For Image**  
    - Type: Wait  
    - Role: Pauses workflow until image generation is complete.  
    - Configuration: Waits for webhook callback or fixed delay.  
    - Inputs: Job ID from "Request Image"  
    - Outputs: Triggers "Get Image" after wait  
    - Edge Cases: Timeout, webhook failures.  
    - Version: 1.1  

  - **Get Image**  
    - Type: HTTP Request  
    - Role: Retrieves the generated image URL or file from the image API.  
    - Configuration: HTTP GET with job ID or token.  
    - Inputs: Trigger from "Wait For Image"  
    - Outputs: Image data to "Request Video"  
    - Edge Cases: Retrieval failures, expired job IDs.  
    - Version: 4.2  

---

#### 2.4 Video Assembly & Processing

- **Overview:**  
  Requests video generation using images and script, waits for video processing, retrieves video assets, and aggregates media for editing.

- **Nodes Involved:**  
  - Request Video  
  - Wait For Video  
  - Get Video  
  - Aggregate  
  - Merge

- **Node Details:**  
  - **Request Video**  
    - Type: HTTP Request  
    - Role: Sends assembled media and script data to a video generation API (e.g., Creatomate).  
    - Configuration: HTTP POST with media URLs and script segments.  
    - Inputs: Image data from "Get Image"  
    - Outputs: Video generation job ID to "Wait For Video"  
    - Edge Cases: API errors, invalid media URLs.  
    - Version: 4.2  

  - **Wait For Video**  
    - Type: Wait  
    - Role: Waits for video rendering completion.  
    - Configuration: Waits for webhook callback or fixed delay.  
    - Inputs: Job ID from "Request Video"  
    - Outputs: Triggers "Get Video"  
    - Edge Cases: Timeout, webhook failures.  
    - Version: 1.1  

  - **Get Video**  
    - Type: HTTP Request  
    - Role: Retrieves the rendered video asset from the video API.  
    - Configuration: HTTP GET with job ID or token.  
    - Inputs: Trigger from "Wait For Video"  
    - Outputs: Video data to "Aggregate"  
    - Edge Cases: Retrieval failures, expired job IDs.  
    - Version: 4.2  

  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines multiple video or media assets into a single data structure.  
    - Configuration: Aggregates all video segments or images.  
    - Inputs: Video data from "Get Video"  
    - Outputs: Aggregated media to "Merge"  
    - Edge Cases: Empty inputs, aggregation errors.  
    - Version: 1  

  - **Merge**  
    - Type: Merge  
    - Role: Merges multiple data streams (e.g., uploaded assets and aggregated media) for final processing.  
    - Configuration: Merges inputs from "Upload to Cloudinary" and "Aggregate"  
    - Inputs: Upload confirmation and aggregated media  
    - Outputs: To "Create Editor JSON"  
    - Edge Cases: Data mismatch, merge conflicts.  
    - Version: 3  

---

#### 2.5 Video Editing & Rendering

- **Overview:**  
  Prepares editor JSON, sends it to the video editor API, waits for rendering, and retrieves the final video URL.

- **Nodes Involved:**  
  - Create Editor JSON  
  - Set JSON Variable  
  - Editor  
  - Rendering  
  - Get Final Video

- **Node Details:**  
  - **Create Editor JSON**  
    - Type: HTTP Request  
    - Role: Generates the JSON configuration for the video editor API based on merged media.  
    - Configuration: HTTP POST with merged data.  
    - Inputs: Merged media from "Merge"  
    - Outputs: Editor JSON to "Set JSON Variable"  
    - Edge Cases: API errors, malformed JSON.  
    - Version: 4.2  

  - **Set JSON Variable**  
    - Type: Set  
    - Role: Sets or modifies variables in the editor JSON before sending to the editor.  
    - Configuration: Adjusts JSON fields as needed.  
    - Inputs: Editor JSON from "Create Editor JSON"  
    - Outputs: Prepared JSON to "Editor"  
    - Edge Cases: Expression errors, invalid JSON structure.  
    - Version: 3.4  

  - **Editor**  
    - Type: HTTP Request  
    - Role: Sends the JSON to the video editor API to start editing.  
    - Configuration: HTTP POST with editor JSON.  
    - Inputs: Prepared JSON from "Set JSON Variable"  
    - Outputs: Editor job ID or confirmation to "Rendering"  
    - Edge Cases: API errors, auth failures.  
    - Version: 4.2  

  - **Rendering**  
    - Type: Wait  
    - Role: Waits for the video rendering process to complete.  
    - Configuration: Waits for webhook callback or fixed delay.  
    - Inputs: Editor job ID from "Editor"  
    - Outputs: Triggers "Get Final Video"  
    - Edge Cases: Timeout, webhook failures.  
    - Version: 1.1  

  - **Get Final Video**  
    - Type: HTTP Request  
    - Role: Retrieves the final rendered video URL or file.  
    - Configuration: HTTP GET with job ID or token.  
    - Inputs: Trigger from "Rendering"  
    - Outputs: Final video link for delivery  
    - Edge Cases: Retrieval failures, expired job IDs.  
    - Version: 4.2  

---

#### 2.6 Output Delivery

- **Overview:**  
  The final video URL is made available for sharing or further use. This block is implicit as the last node outputs the video link.

- **Nodes Involved:**  
  - Get Final Video

- **Node Details:**  
  - See above in 2.5.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                      | Input Node(s)               | Output Node(s)              | Sticky Note                         |
|-------------------------|----------------------------------|------------------------------------|-----------------------------|-----------------------------|-----------------------------------|
| When chat message received | Chat Trigger (Langchain)          | Entry point, receives chat input   | -                           | Script Prompter             |                                   |
| Script Prompter          | OpenAI (Langchain)                | Generates script prompt             | When chat message received  | Script Organizer            |                                   |
| Script Organizer         | Set                              | Organizes script data               | Script Prompter             | Script Generator            |                                   |
| Script Generator         | HTTP Request                     | Generates full script text          | Script Organizer            | Outliner, Upload to Cloudinary |                                   |
| Outliner                 | HTTP Request                     | Creates script outline              | Script Generator            | Split Out                   |                                   |
| Split Out                | Split Out                        | Splits outline into segments        | Outliner                    | Image Prompter              |                                   |
| Image Prompter           | OpenAI (Langchain)               | Generates image prompts             | Split Out                   | Request Image               |                                   |
| Request Image            | HTTP Request                    | Requests image generation           | Image Prompter              | Wait For Image              |                                   |
| Wait For Image           | Wait                            | Waits for image generation          | Request Image               | Get Image                   |                                   |
| Get Image                | HTTP Request                    | Retrieves generated image           | Wait For Image              | Request Video               |                                   |
| Request Video            | HTTP Request                    | Requests video generation           | Get Image                   | Wait For Video              |                                   |
| Wait For Video           | Wait                            | Waits for video rendering           | Request Video               | Get Video                   |                                   |
| Get Video                | HTTP Request                    | Retrieves generated video           | Wait For Video              | Aggregate                   |                                   |
| Aggregate                | Aggregate                       | Aggregates media assets             | Get Video                   | Merge                       |                                   |
| Upload to Cloudinary     | HTTP Request                    | Uploads assets to Cloudinary        | Script Generator            | Merge                       |                                   |
| Merge                   | Merge                           | Merges media and upload data        | Upload to Cloudinary, Aggregate | Create Editor JSON          |                                   |
| Create Editor JSON       | HTTP Request                    | Creates editor JSON config          | Merge                       | Set JSON Variable           |                                   |
| Set JSON Variable        | Set                             | Adjusts editor JSON                 | Create Editor JSON          | Editor                      |                                   |
| Editor                  | HTTP Request                    | Sends JSON to video editor API      | Set JSON Variable           | Rendering                   |                                   |
| Rendering                | Wait                            | Waits for video rendering           | Editor                      | Get Final Video             |                                   |
| Get Final Video          | HTTP Request                    | Retrieves final video URL            | Rendering                   | -                           |                                   |
| Sticky Note              | Sticky Note                     | -                                  | -                           | -                           |                                   |
| Sticky Note1             | Sticky Note                     | -                                  | -                           | -                           |                                   |
| Sticky Note2             | Sticky Note                     | -                                  | -                           | -                           |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Entry Node:**  
   - Add **When chat message received** (Langchain Chat Trigger) node.  
   - Configure webhook to listen for chat input (no special parameters).  

2. **Add Script Prompting:**  
   - Add **Script Prompter** (OpenAI Langchain) node.  
   - Connect from "When chat message received".  
   - Configure with OpenAI credentials and prompt template to generate a script prompt from chat input.  

3. **Organize Script Data:**  
   - Add **Script Organizer** (Set) node.  
   - Connect from "Script Prompter".  
   - Configure to format and set variables for script generation API.  

4. **Generate Full Script:**  
   - Add **Script Generator** (HTTP Request) node.  
   - Connect from "Script Organizer".  
   - Configure HTTP POST to script generation API (OpenAI or ElevenLabs).  
   - Set authentication headers and body with organized script data.  

5. **Outline Script:**  
   - Add **Outliner** (HTTP Request) node.  
   - Connect from "Script Generator".  
   - Configure to call an API that creates an outline or segments from the script text.  

6. **Upload Script Assets:**  
   - Add **Upload to Cloudinary** (HTTP Request) node.  
   - Connect from "Script Generator" (parallel to Outliner).  
   - Configure Cloudinary API credentials and upload parameters.  

7. **Split Outline:**  
   - Add **Split Out** node.  
   - Connect from "Outliner".  
   - Configure to split outline array into individual segments.  

8. **Generate Image Prompts:**  
   - Add **Image Prompter** (OpenAI Langchain) node.  
   - Connect from "Split Out".  
   - Configure with OpenAI credentials and prompt templates for image description.  

9. **Request Images:**  
   - Add **Request Image** (HTTP Request) node.  
   - Connect from "Image Prompter".  
   - Configure HTTP POST to image generation API (e.g., Replicate) with prompt and API key.  

10. **Wait for Image Completion:**  
    - Add **Wait For Image** (Wait) node.  
    - Connect from "Request Image".  
    - Configure wait for webhook callback or fixed delay.  

11. **Retrieve Images:**  
    - Add **Get Image** (HTTP Request) node.  
    - Connect from "Wait For Image".  
    - Configure HTTP GET to retrieve generated images using job ID.  

12. **Request Video Generation:**  
    - Add **Request Video** (HTTP Request) node.  
    - Connect from "Get Image".  
    - Configure HTTP POST to video generation API with media URLs and script segments.  

13. **Wait for Video Rendering:**  
    - Add **Wait For Video** (Wait) node.  
    - Connect from "Request Video".  
    - Configure wait for webhook callback or fixed delay.  

14. **Retrieve Video:**  
    - Add **Get Video** (HTTP Request) node.  
    - Connect from "Wait For Video".  
    - Configure HTTP GET to retrieve rendered video using job ID.  

15. **Aggregate Media:**  
    - Add **Aggregate** node.  
    - Connect from "Get Video".  
    - Configure to aggregate video segments or media assets.  

16. **Merge Upload and Aggregated Media:**  
    - Add **Merge** node.  
    - Connect inputs from "Upload to Cloudinary" and "Aggregate".  
    - Configure merge mode to combine both inputs.  

17. **Create Editor JSON:**  
    - Add **Create Editor JSON** (HTTP Request) node.  
    - Connect from "Merge".  
    - Configure HTTP POST to video editor API to generate editing JSON config.  

18. **Set JSON Variables:**  
    - Add **Set JSON Variable** (Set) node.  
    - Connect from "Create Editor JSON".  
    - Configure to adjust or add variables in the editor JSON.  

19. **Send to Video Editor:**  
    - Add **Editor** (HTTP Request) node.  
    - Connect from "Set JSON Variable".  
    - Configure HTTP POST to video editor API to start editing.  

20. **Wait for Rendering:**  
    - Add **Rendering** (Wait) node.  
    - Connect from "Editor".  
    - Configure wait for webhook callback or fixed delay.  

21. **Get Final Video:**  
    - Add **Get Final Video** (HTTP Request) node.  
    - Connect from "Rendering".  
    - Configure HTTP GET to retrieve final video URL.  

22. **Credentials Setup:**  
    - Configure OpenAI credentials for "Script Prompter" and "Image Prompter".  
    - Configure ElevenLabs or equivalent for voiceover if applicable (not explicitly shown).  
    - Configure Cloudinary credentials for "Upload to Cloudinary".  
    - Configure Replicate or image generation API credentials for "Request Image".  
    - Configure Creatomate or video generation API credentials for "Request Video", "Editor", and related nodes.  

23. **Testing:**  
    - Test each block independently with sample inputs.  
    - Monitor webhook URLs and ensure they are publicly accessible for wait nodes.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates YouTube Shorts creation with AI-generated scripts, voiceovers, images, and video | Workflow description and use cases                                                             |
| Requires API keys for OpenAI, ElevenLabs, Cloudinary, Replicate, Creatomate                          | API integration setup                                                                           |
| Video tutorial available for guided setup                                                           | Mentioned in workflow description (not linked here)                                            |
| Designed for content creators, marketing agencies, and business owners                              | Target audience                                                                                |
| Workflow uses webhook-based wait nodes to handle asynchronous media generation                       | Important for ensuring correct webhook URLs and timing                                         |

---

This document provides a detailed, structured reference to understand, reproduce, and modify the YouTube Shorts Automation Tool workflow efficiently.