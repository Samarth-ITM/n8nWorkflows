Turn YouTube Videos into Summaries, Transcripts, and Visual Insights

https://n8nworkflows.xyz/workflows/turn-youtube-videos-into-summaries--transcripts--and-visual-insights-3188


# Turn YouTube Videos into Summaries, Transcripts, and Visual Insights

### 1. Workflow Overview

This workflow automates the extraction and analysis of YouTube video content using Google’s Gemini 1.5 Flash AI model. It is designed for users such as learners, content creators, YouTube managers, and social media strategists who want to quickly generate transcripts, summaries, visual scene descriptions, and discover shareable clips from videos without manual viewing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Initialization:** Manual trigger and initial variable setup including API key, YouTube URL, and prompt type.
- **1.2 Prompt Selection Logic:** A switch node routes the flow based on the type of analysis requested (transcript, timestamped transcript, summary, scene description, clips, or fallback).
- **1.3 Prompt Configuration:** Nodes that set the specific prompt text and model parameters for the Google Gemini API call.
- **1.4 API Request and Response Handling:** HTTP request node sends the prompt and video URL to Google’s generative language API, followed by merging and formatting the response.
- **1.5 Output Preparation:** Final set node formats the AI-generated content and metadata for downstream use.
- **1.6 Optional Error Handling:** Disabled node for detecting errors in the API response.
- **1.7 Documentation and Guidance:** Sticky notes provide detailed instructions, prompt inspirations, and usage tips.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

- **Overview:** This block starts the workflow manually and defines initial variables such as API key, YouTube URL, and the type of prompt to use.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set: Define Initial Variables

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution  
    - Config: No parameters; triggers workflow on demand  
    - Inputs: None  
    - Outputs: Connects to "Set: Define Initial Variables"  
    - Edge Cases: None (manual trigger)

  - **Set: Define Initial Variables**  
    - Type: Set  
    - Role: Defines key variables for the workflow execution  
    - Config:  
      - `automationID`: Optional reference string for tracking  
      - `apiKey`: Placeholder for Google API key (must be replaced)  
      - `youtubeURL`: Default YouTube video URL for testing  
      - `promptType`: Defines the type of analysis (default "transcript")  
    - Inputs: From manual trigger  
    - Outputs: Connects to "Switch: What kind of question do we want to ask?"  
    - Edge Cases: Missing or invalid API key or URL will cause downstream failures

#### 2.2 Prompt Selection Logic

- **Overview:** Routes the workflow based on the `promptType` variable to select the appropriate prompt and model configuration.
- **Nodes Involved:**  
  - Switch: What kind of question do we want to ask?

- **Node Details:**

  - **Switch: What kind of question do we want to ask?**  
    - Type: Switch  
    - Role: Branches workflow into different prompt preparation paths  
    - Config: Case-insensitive matching on `promptType` to outputs:  
      - Transcript  
      - Transcript with Timestamps  
      - Summary  
      - Scene Description  
      - Clips  
      - Fallback (default)  
    - Inputs: From "Set: Define Initial Variables"  
    - Outputs: Connects to respective "Set:" nodes for prompt configuration  
    - Edge Cases: Unknown or misspelled `promptType` routes to fallback prompt

#### 2.3 Prompt Configuration

- **Overview:** Sets the prompt text and model name for the Google API call according to the selected analysis type.
- **Nodes Involved:**  
  - Set: Transcript Prompt  
  - Set: Transcript with Timestamps Prompt  
  - Set: Summarize Prompt  
  - Set: Scene Prompt  
  - Set: Scene Clips  
  - Set: Fallback

- **Node Details:**

  Each "Set:" node:

  - Type: Set  
  - Role: Defines the prompt text and model ("gemini-1.5-flash")  
  - Config:  
    - `prompt`: Custom prompt text tailored to the analysis type (e.g., transcription, summary, scene description)  
    - `model`: Fixed to "gemini-1.5-flash"  
  - Inputs: From Switch node  
  - Outputs: Connects to "Set: Merged Values"  
  - Edge Cases: Prompt text must be well-formed; malformed prompts may cause API errors

  Specific prompt highlights:

  - **Transcript Prompt:** Requests verbatim spoken dialogue only.  
  - **Transcript with Timestamps Prompt:** Requests timestamped transcript lines in `[hh:mm:ss] Dialogue` format.  
  - **Summarize Prompt:** Requests concise nested bullet summary with minimal quotes.  
  - **Scene Prompt:** Requests detailed scene description including setting, objects, people, lighting, colors, and camera angles.  
  - **Scene Clips:** Requests shareable clips with timestamps, transcript, and rationale for social media appeal.  
  - **Fallback:** Requests actionable insights summary focusing on tools, strategies, and resources.

#### 2.4 API Request and Response Handling

- **Overview:** Sends the prompt and YouTube video URL to Google’s generative language API and merges the response with prior data.
- **Nodes Involved:**  
  - Set: Merged Values  
  - HTTP Request to Google  
  - Code: Merge data from prior nodes with HTTP Output

- **Node Details:**

  - **Set: Merged Values**  
    - Type: Set  
    - Role: Passes through all existing data unchanged to prepare for HTTP request  
    - Config: Includes all fields from input  
    - Inputs: From any "Set:" prompt node  
    - Outputs: Connects to "HTTP Request to Google"  
    - Edge Cases: None

  - **HTTP Request to Google**  
    - Type: HTTP Request  
    - Role: Calls Google Gemini API to generate content based on prompt and video URL  
    - Config:  
      - URL: `https://generativelanguage.googleapis.com/v1beta/models/{{ $json.model }}:generateContent?key={{ $json.apiKey }}`  
      - Method: POST  
      - Headers: Content-Type: application/json  
      - Body (JSON):  
        ```json
        {
          "contents": [
            {
              "parts": [
                { "text": "<prompt text>" },
                { "file_data": { "file_uri": "<YouTube URL>" } }
              ]
            }
          ]
        }
        ```  
      - Sends prompt and YouTube URL as file_uri for analysis  
    - Inputs: From "Set: Merged Values"  
    - Outputs: Connects to "Code: Merge data from prior nodes with HTTP Output" and optionally to error check node  
    - Edge Cases:  
      - API key invalid or quota exceeded → error response  
      - Network timeout or API downtime  
      - Malformed prompt or unsupported model version

  - **Code: Merge data from prior nodes with HTTP Output**  
    - Type: Code (JavaScript)  
    - Role: Combines HTTP response JSON with data from previous nodes to preserve context  
    - Config: Merges `$json` from HTTP response with `$json` from "Set: Merged Values"  
    - Inputs: From HTTP Request node  
    - Outputs: Connects to "Set Fields: Define Outcome"  
    - Edge Cases: Script errors if input data missing or malformed

#### 2.5 Output Preparation

- **Overview:** Extracts and renames key fields from the API response for easier consumption downstream.
- **Nodes Involved:**  
  - Set Fields: Define Outcome

- **Node Details:**

  - **Set Fields: Define Outcome**  
    - Type: Set  
    - Role: Simplifies and renames response data fields for clarity  
    - Config:  
      - `answerAIGenerated`: Extracts AI-generated text from nested JSON or returns error message  
      - `promptTokenCount`, `candidatesTokenCount`, `totalTokenCount`: Extracts token usage metadata or error  
      - `modelVersionUsed`: Extracts model version or error  
      - Excludes raw `candidates` and `usageMetadata` fields to reduce clutter  
    - Inputs: From "Code: Merge data from prior nodes with HTTP Output"  
    - Outputs: None (end of workflow)  
    - Edge Cases: Missing or unexpected response structure may cause empty or error messages

#### 2.6 Optional Error Handling (Disabled)

- **Overview:** Intended to detect errors in API response and branch accordingly; currently disabled.
- **Nodes Involved:**  
  - If: Was an error detected?

- **Node Details:**

  - **If: Was an error detected?**  
    - Type: If  
    - Role: Checks if `error` field exists in JSON response  
    - Config: Condition tests existence of `$json.error`  
    - Inputs: From "HTTP Request to Google"  
    - Outputs: None (disabled)  
    - Edge Cases: Could be enabled for advanced error handling or retries

#### 2.7 Documentation and Guidance

- **Overview:** Provides extensive usage instructions, prompt inspirations, and setup notes via sticky notes.
- **Nodes Involved:**  
  - Sticky Note6 (How to Use This Workflow)  
  - Sticky Note1 (Prompt Inspiration Ideas)  
  - Sticky Note5 (Set Values For Use in Flow)  
  - Sticky Note7 (Prompt Type Explanation)  
  - Sticky Note8 (Prompt & Model Setup)  
  - Sticky Note9 (Reference Values for HTTP Node)  
  - Sticky Note11 (Google API Call Explanation)  
  - Sticky Note13 (Data Merge Explanation)  
  - Sticky Note14 (Output Naming Explanation)  
  - Sticky Note10 (Optional Error Handling)  
  - Sticky Note (Trigger Ideas)  
  - Sticky Note2 (Next Step Ideas)

- **Node Details:**

  - All sticky notes are informational only, no inputs or outputs, positioned for user reference.

---

### 3. Summary Table

| Node Name                             | Node Type           | Functional Role                          | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                  |
|-------------------------------------|---------------------|----------------------------------------|--------------------------------------|---------------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’       | Manual Trigger      | Entry point for manual execution       | None                                 | Set: Define Initial Variables          | 💡 Trigger Ideas: Can replace trigger with webhook, YouTube node, form, etc.                                |
| Set: Define Initial Variables        | Set                 | Defines API key, URL, prompt type      | When clicking ‘Test workflow’         | Switch: What kind of question do we want to ask? | ✏️ Set Values For Use in Flow: automationID, apiKey, youtubeURL, promptType                                  |
| Switch: What kind of question do we want to ask? | Switch              | Routes flow based on promptType        | Set: Define Initial Variables         | Set: Transcript Prompt, Set: Transcript with Timestamps Prompt, Set: Summarize Prompt, Set: Scene Prompt, Set: Scene Clips, Set: Fallback | ℹ️ Determine which prompt will be passed in API call based on promptType value                               |
| Set: Transcript Prompt               | Set                 | Sets transcript prompt and model       | Switch: What kind of question do we want to ask? (Transcript) | Set: Merged Values                     | ✏️ Set Values For Prompts & Model                                                                           |
| Set: Transcript with Timestamps Prompt | Set                 | Sets timestamped transcript prompt     | Switch: What kind of question do we want to ask? (Timestamps) | Set: Merged Values                     | ✏️ Set Values For Prompts & Model                                                                           |
| Set: Summarize Prompt               | Set                 | Sets summary prompt and model          | Switch: What kind of question do we want to ask? (Summary) | Set: Merged Values                     | ✏️ Set Values For Prompts & Model                                                                           |
| Set: Scene Prompt                  | Set                 | Sets scene description prompt and model | Switch: What kind of question do we want to ask? (Scene) | Set: Merged Values                     | ✏️ Set Values For Prompts & Model                                                                           |
| Set: Scene Clips                   | Set                 | Sets social media clips prompt and model | Switch: What kind of question do we want to ask? (Clips) | Set: Merged Values                     | ✏️ Set Values For Prompts & Model                                                                           |
| Set: Fallback                     | Set                 | Sets fallback prompt and model         | Switch: What kind of question do we want to ask? (Fallback) | Set: Merged Values                     | ✏️ Set Values For Prompts & Model                                                                           |
| Set: Merged Values                 | Set                 | Passes all data forward unchanged      | Any Set: Prompt node                  | HTTP Request to Google                 | ℹ️ Making it easier to reference values in the http node                                                    |
| HTTP Request to Google             | HTTP Request        | Calls Google Gemini API with prompt and video URL | Set: Merged Values                   | Code: Merge data from prior nodes with HTTP Output, If: Was an error detected? (disabled) | ℹ️ Makes call to Google endpoint using values set in earlier nodes                                          |
| Code: Merge data from prior nodes with HTTP Output | Code                | Merges HTTP response with prior data   | HTTP Request to Google                | Set Fields: Define Outcome             | ℹ️ Merges data from returned by Google with values set in prior nodes so that earlier data isn't lost       |
| Set Fields: Define Outcome         | Set                 | Extracts and renames key response fields | Code: Merge data from prior nodes with HTTP Output | None                                  | ℹ️ Gives returned data meaningful names; Simplifies amount of data available to follow-on nodes             |
| If: Was an error detected?         | If                  | Checks for error in API response (disabled) | HTTP Request to Google                | None                                  | ℹ️ If you want to add special processing when errors occur (Optional)                                       |
| Sticky Note6                      | Sticky Note         | Usage instructions and overview        | None                                 | None                                  | How to Use This Workflow with video overview and setup instructions                                        |
| Sticky Note1                      | Sticky Note         | Prompt inspiration ideas                | None                                 | None                                  | Prompt Inspiration Ideas with detailed example prompts                                                    |
| Sticky Note5                      | Sticky Note         | Explanation of initial variable usage  | None                                 | None                                  | ✏️ Set Values For Use in Flow                                                                               |
| Sticky Note7                      | Sticky Note         | Explanation of promptType switch logic | None                                 | None                                  | ℹ️ Determine which prompt will be passed in API call based on promptType value                              |
| Sticky Note8                      | Sticky Note         | Explanation of prompt and model setting | None                                 | None                                  | ✏️ Set Values For Prompts & Model                                                                           |
| Sticky Note9                      | Sticky Note         | Explanation of data referencing in HTTP node | None                                 | None                                  | ℹ️ Making it easier to reference values in the http node                                                   |
| Sticky Note11                     | Sticky Note         | Explanation of Google API call          | None                                 | None                                  | ℹ️ Makes call to Google endpoint using values set in earlier nodes                                         |
| Sticky Note13                     | Sticky Note         | Explanation of data merging             | None                                 | None                                  | ℹ️ Merges data from returned by Google with values set in prior nodes so that earlier data isn't lost      |
| Sticky Note14                     | Sticky Note         | Explanation of output field naming      | None                                 | None                                  | ℹ️ Gives returned data meaningful names; Simplifies amount of data available to follow-on nodes            |
| Sticky Note10                     | Sticky Note         | Optional error handling notes           | None                                 | None                                  | ℹ️ If you want to add special processing when errors occur (Optional)                                      |
| Sticky Note                      | Sticky Note         | Trigger ideas for workflow execution    | None                                 | None                                  | 💡 Trigger Ideas                                                                                             |
| Sticky Note2                     | Sticky Note         | Next step ideas for output usage        | None                                 | None                                  | 💡 Next Step Ideas                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: "When clicking ‘Test workflow’"  
   - Purpose: Manual start of the workflow.

2. **Add a Set Node for Initial Variables**  
   - Name: "Set: Define Initial Variables"  
   - Assign the following variables:  
     - `automationID`: Optional string for tracking (e.g., "optional-enter-reference-tracking-identifier")  
     - `apiKey`: Your Google API key (replace placeholder)  
     - `youtubeURL`: YouTube video URL to analyze (default example provided)  
     - `promptType`: Type of analysis ("transcript", "timestamps", "summary", "scene", "clips", or fallback)

3. **Connect Manual Trigger → Set: Define Initial Variables**

4. **Add a Switch Node to Route Based on `promptType`**  
   - Name: "Switch: What kind of question do we want to ask?"  
   - Configure rules (case-insensitive) on `promptType.toLowerCase()` with outputs:  
     - "transcript" → Output "Transcript"  
     - "timestamps" → Output "Transcript with Timestamps"  
     - "summary" → Output "Summary"  
     - "scene" → Output "Scene Description"  
     - "clips" → Output "Clips"  
     - Default → Output "extra" (Fallback)

5. **Connect Set: Define Initial Variables → Switch Node**

6. **Create Set Nodes for Each Prompt Type** (six nodes total):

   - **Set: Transcript Prompt**  
     - `prompt`: "Transcribe the video. Return only the spoken dialogue, verbatim. Omit any additional text or descriptions."  
     - `model`: "gemini-1.5-flash"

   - **Set: Transcript with Timestamps Prompt**  
     - `prompt`: "Generate a timestamped transcript of the video. Each line must follow this format precisely:  [hh:mm:ss] Dialogue. Return only the timestamp and spoken content; omit any other text or formatting."  
     - `model`: "gemini-1.5-flash"

   - **Set: Summarize Prompt**  
     - `prompt`: "Provide a concise summary of the main points in nested bullets, using quotes only when absolutely essential for clarity. Start output directly with the response."  
     - `model`: "gemini-1.5-flash"

   - **Set: Scene Prompt**  
     - `prompt`: Detailed scene description instructions covering setting, objects, people, lighting, colors, camera angle/movement. Start output directly with response.  
     - `model`: "gemini-1.5-flash"

   - **Set: Scene Clips**  
     - `prompt`: Instructions to extract shareable clips with timestamps, transcript, and rationale for social media appeal. Start output directly with response.  
     - `model`: "gemini-1.5-flash"

   - **Set: Fallback**  
     - `prompt`: "Summarize this YouTube video with a focus on actionable insights. Use nested bullets and include relevant quotes. Highlight recommended tools, strategies, or resources."  
     - `model`: "gemini-1.5-flash"

7. **Connect Switch Outputs to Corresponding Set Prompt Nodes**

8. **Add a Set Node "Set: Merged Values"**  
   - Purpose: Pass all incoming data unchanged to the HTTP request node.

9. **Connect Each Set Prompt Node → Set: Merged Values**

10. **Add an HTTP Request Node**  
    - Name: "HTTP Request to Google"  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/{{ $json.model }}:generateContent?key={{ $json.apiKey }}`  
    - Headers: Content-Type: application/json  
    - Body (JSON):  
      ```json
      {
        "contents": [
          {
            "parts": [
              { "text": {{ JSON.stringify($json.prompt) }} },
              { "file_data": { "file_uri": "{{ $json.youtubeURL }}" } }
            ]
          }
        ]
      }
      ```  
    - Send body as JSON  
    - Connect input from "Set: Merged Values"

11. **Add a Code Node "Code: Merge data from prior nodes with HTTP Output"**  
    - JavaScript code:  
      ```js
      return {
        json: {
          ...$json, // HTTP response data
          ...$('Set: Merged Values').item.json, // Prior data
        }
      };
      ```  
    - Connect input from "HTTP Request to Google"

12. **Add a Set Node "Set Fields: Define Outcome"**  
    - Purpose: Extract and rename key fields for output  
    - Assignments:  
      - `answerAIGenerated`: Extract text from nested response or error message  
      - `promptTokenCount`, `candidatesTokenCount`, `totalTokenCount`: Extract token usage or error  
      - `modelVersionUsed`: Extract model version or error  
    - Exclude raw `candidates` and `usageMetadata` fields  
    - Connect input from "Code: Merge data from prior nodes with HTTP Output"

13. **Optional: Add an If Node "If: Was an error detected?"**  
    - Condition: Check if `$json.error` exists  
    - Connect input from "HTTP Request to Google"  
    - Currently disabled; can be enabled for error handling

14. **Add Sticky Notes for Documentation**  
    - Add notes with usage instructions, prompt inspirations, setup tips, and trigger ideas as per original workflow for user reference.

15. **Test the workflow manually** by triggering the manual node and verifying output.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Video Overview and Usage Instructions: This workflow is designed to analyze YouTube videos using Google Gemini AI. Start with summary or transcript prompts, then customize for detailed scene analysis or clip extraction. Flexible triggers and outputs allow integration with various platforms.                                                                                                                                                                                                                                                                                                                                            | https://www.youtube.com/watch?v=Ovg_KfKxnC8                                                           |
| Google API Key Requirement: You must provide a valid Google API key with access to the Gemini generative language API. Obtain via Google Cloud Console or AI Studio.                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | https://console.developers.google.com/ and https://aistudio.google.com/apikey                          |
| Gemini API Documentation and Pricing: Explore the Gemini API vision capabilities and pricing details to understand usage limits and features.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | https://ai.google.dev/gemini-api/docs/vision?lang=python and https://ai.google.dev/gemini-api/docs/pricing |
| Prompt Inspiration Ideas: Extensive examples of prompts for various use cases including summarization, SEO package generation, structured content mapping, actionable insights, and rhetorical analysis. These can be adapted to fit specific user needs.                                                                                                                                                                                                                                                                                                                                                                                    | See Sticky Note1 content in workflow                                                                  |
| Trigger Flexibility: The manual trigger can be replaced with webhook, Airtable event, YouTube node, or other n8n triggers to automate or schedule runs.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | See Sticky Note content on trigger ideas                                                             |
| Output Integration Suggestions: Results can be sent to Notion, Airtable, CMS platforms, or stored within n8n for further processing or archival.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | See Sticky Note2 content on next step ideas                                                          |
| Workflow Tested on Videos Ranging from Shorts to One Hour Length: Designed for robustness across different video lengths and content types.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | See Sticky Note6                                                                                      |
| Future-proofing: Currently configured for "gemini-1.5-flash" model; prompt nodes can be updated to support newer models or API endpoints as they become available.                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | See Sticky Note6                                                                                      |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, enabling users and automation agents to reproduce, modify, and troubleshoot it effectively.