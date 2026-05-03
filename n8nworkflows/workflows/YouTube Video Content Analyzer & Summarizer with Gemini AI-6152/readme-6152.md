YouTube Video Content Analyzer & Summarizer with Gemini AI

https://n8nworkflows.xyz/workflows/youtube-video-content-analyzer---summarizer-with-gemini-ai-6152


# YouTube Video Content Analyzer & Summarizer with Gemini AI

### 1. Workflow Overview

This workflow automates the analysis and summarization of YouTube video content using Google’s Gemini AI model. It accepts a YouTube video URL and an optional user prompt describing what kind of analysis or summary is desired. If the prompt is left empty, a default detailed summary is generated. The workflow then sends this input to the Gemini AI API, retrieves the analysis, and presents it back to the user in a form. It also supports repeating the process for multiple analyses.

The workflow’s logic is divided into the following functional blocks:

- **1.1 Input Reception:** Collects YouTube video URL and optional analysis description from the user via a web form.
- **1.2 Input Validation & Defaulting:** Checks if the user’s description is empty and assigns a default prompt if necessary.
- **1.3 AI Content Generation:** Sends the user input (video URL and prompt) to Google Gemini’s language model API to generate the analysis.
- **1.4 Result Presentation & Loop:** Displays the analysis result to the user and provides an option to analyze another video by redirecting back to the input form.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects input from the user: the YouTube video URL and a textual description of what kind of analysis or summary they want.

**Nodes Involved:**  
- Get Input

**Node Details:**

- **Node Name:** Get Input  
- **Type:** Form Trigger  
- **Technical Role:** Entry point; triggers workflow execution upon form submission.  
- **Configuration Choices:**  
  - Form title: "Analyze YouTube Videos and Generate Summaries with Gemini AI"  
  - Two form fields:  
    - "Video URL" (required, placeholder: "www.youtube")  
    - "What do you want?" (optional, placeholder: "Summarize in 3 sentences")  
  - Form description: "I'll analyze the contents of your YouTube video!"  
- **Key Expressions / Variables:**  
  - Captures input fields as: `Video URL` and `What do you want?`  
- **Input Connections:** None (trigger node)  
- **Output Connections:** Passes data to "Check if description is empty" node  
- **Version Requirements:** n8n Form Trigger node version 2.2 or higher to support webhook and form field definitions  
- **Edge Cases / Failure Types:**  
  - Missing required `Video URL` field will prevent submission (handled by form UI)  
  - Invalid YouTube URL format is not validated here—may cause downstream errors  
- **Sub-workflow:** None  

---

#### 2.2 Input Validation & Defaulting

**Overview:**  
Ensures that the "What do you want?" description field is not empty. If empty, assigns a default prompt instructing Gemini AI to generate a detailed summary.

**Nodes Involved:**  
- Check if description is empty

**Node Details:**

- **Node Name:** Check if description is empty  
- **Type:** Code (JavaScript)  
- **Technical Role:** Input preprocessing and validation  
- **Configuration Choices:**  
  - Loops over input items (although only one expected)  
  - Checks if the field "What u want?" is empty or whitespace-only  
  - If empty, replaces it with default prompt:  
    `"Please be as descriptive as possible about the contents being spoken of in this video after giving a detailed summary."`  
- **Key Expressions / Variables:**  
  - Uses `$input.first().json['What u want?']` to access the description field  
  - Modifies the same field in-place  
- **Input Connections:** Receives data from "Get Input"  
- **Output Connections:** Passes data to "Get analysis from Google Gemini"  
- **Version Requirements:** n8n Code node version 2 or higher for JavaScript support  
- **Edge Cases / Failure Types:**  
  - If input JSON keys change or are missing, code may fail  
  - No validation on URL or prompt content beyond emptiness  
- **Sub-workflow:** None  

---

#### 2.3 AI Content Generation

**Overview:**  
Sends the YouTube video URL and user prompt to Google Gemini AI via HTTP POST request to generate the desired analysis or summary content.

**Nodes Involved:**  
- Get analysis from Google Gemini

**Node Details:**

- **Node Name:** Get analysis from Google Gemini  
- **Type:** HTTP Request  
- **Technical Role:** External API call to Google Gemini generative language endpoint  
- **Configuration Choices:**  
  - HTTP Method: POST  
  - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
  - Request Body (JSON):  
    - `contents` array with two parts:  
      1. Text part: user prompt from field `"What u want?"`  
      2. File data part: includes `file_uri` set to the YouTube video URL field `"Video"`  
  - Sends headers including required `x-goog-api-key` (Google Cloud API key credential must be configured)  
  - Sends JSON body  
- **Key Expressions / Variables:**  
  - `"{{ $json['What u want?'] }}"` for prompt text  
  - `"{{ $json.Video }}"` for video URL  
- **Input Connections:** Receives data from "Check if description is empty"  
- **Output Connections:** Passes response data to "Analysis result" node  
- **Version Requirements:** HTTP Request node version 4.2 or higher for JSON body and header support  
- **Edge Cases / Failure Types:**  
  - Authentication errors if API key missing/invalid  
  - API rate limits or timeouts  
  - Invalid or inaccessible YouTube URLs may cause poor or no results  
  - JSON expression resolution errors if fields missing or malformed  
- **Sub-workflow:** None  

---

#### 2.4 Result Presentation & Loop

**Overview:**  
Displays the AI-generated analysis result in a form, and provides a button to trigger another analysis by redirecting back to the input form.

**Nodes Involved:**  
- Analysis result  
- Redirect for another analysis

**Node Details:**

- **Node Name:** Analysis result  
- **Type:** Form  
- **Technical Role:** Output presentation of the AI analysis result to the user  
- **Configuration Choices:**  
  - Form title: "Analysis"  
  - Button label: "Another Analysis?"  
  - Form description: populated dynamically with the first part of the AI response text (`={{ $json.candidates[0].content.parts[0].text }}`)  
- **Key Expressions / Variables:**  
  - `$json.candidates[0].content.parts[0].text` extracts Gemini AI response content  
- **Input Connections:** Receives data from "Get analysis from Google Gemini"  
- **Output Connections:** Passes flow to "Redirect for another analysis"  
- **Version Requirements:** Form node version 1  
- **Edge Cases / Failure Types:**  
  - If Gemini response structure changes, the expression may fail or show empty content  
- **Sub-workflow:** None  

- **Node Name:** Redirect for another analysis  
- **Type:** Form  
- **Technical Role:** Redirects user back to initial input form for new analysis  
- **Configuration Choices:**  
  - Operation: completion (post form submission)  
  - Redirect URL: static URL of the initial "Get Input" form webhook  
  - Responds with HTTP redirect  
- **Key Expressions / Variables:**  
  - Redirect URL: `https://n8n.srv885623.hstgr.cloud/form/00f1dd2d-2e75-4f8e-b707-388308be0cfb`  
- **Input Connections:** Receives from "Analysis result"  
- **Output Connections:** None (end of flow)  
- **Version Requirements:** Form node version 1  
- **Edge Cases / Failure Types:**  
  - Redirect URL must remain valid and accessible  
  - If user repeatedly submits too quickly, API limits may be hit  
- **Sub-workflow:** None  

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                     | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                                           |
|-------------------------------|--------------------|-----------------------------------|---------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Sticky Note                   | Sticky Note        | Informational (workflow purpose)  | -                         | -                             | This workflow takes as input a YouTube link (required) and a description of what you want to analyze from the video. If the description field is left empty, the default prompt will generate a detailed summary and description of the video's contents. However, you can ask for something more specific using this field/input. |
| Sticky Note1                  | Sticky Note        | Informational (workflow title)    | -                         | -                             | ## Analyze YouTube Videos and Generate Summaries with Gemini AI                                                      |
| Get Input                    | Form Trigger       | Collects user input (video URL and description) | -                         | Check if description is empty  |                                                                                                                       |
| Check if description is empty | Code               | Validates and defaults prompt     | Get Input                 | Get analysis from Google Gemini |                                                                                                                       |
| Get analysis from Google Gemini | HTTP Request       | Calls Gemini AI API for analysis  | Check if description is empty | Analysis result               |                                                                                                                       |
| Analysis result              | Form               | Displays AI analysis result       | Get analysis from Google Gemini | Redirect for another analysis |                                                                                                                       |
| Redirect for another analysis | Form               | Redirects to input form for repeat analysis | Analysis result           | -                             |                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node named "Get Input":**  
   - Set form title to: "Analyze YouTube Videos and Generate Summaries with Gemini AI"  
   - Add two form fields:  
     - "Video URL" (required, placeholder: "www.youtube")  
     - "What do you want?" (optional, placeholder: "Summarize in 3 sentences")  
   - Set form description: "I'll analyze the contents of your YouTube video!"  
   - Save the node and note the webhook URL.

3. **Add a Code node named "Check if description is empty":**  
   - Use JavaScript code to check if `"What u want?"` field is empty or whitespace only. If so, assign the default prompt:  
     `"Please be as descriptive as possible about the contents being spoken of in this video after giving a detailed summary."`  
   - Sample code snippet:  
     ```javascript
     for (const item of $input.all()) {
       if (item.json['What u want?'].trim() === "") {
         item.json['What u want?'] = "Please be as descriptive as possible about the contents being spoken of in this video after giving a detailed summary.";
       }
     }
     return $input.all();
     ```  
   - Connect "Get Input" node output to this node.

4. **Add an HTTP Request node named "Get analysis from Google Gemini":**  
   - Set HTTP Method to POST  
   - Set URL to: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent`  
   - Set Authentication to None (authentication via API key header)  
   - Under Headers, add:  
     - Name: `x-goog-api-key`  
     - Value: Your Google Cloud API key credential (set up in n8n credentials)  
   - Set Body Content Type to JSON  
   - Set Body to the following JSON (using expressions):  
     ```json
     {
       "contents": [
         {
           "parts": [
             { "text": "{{ $json['What u want?'] }}" },
             { "file_data": { "file_uri": "{{ $json.Video }}" } }
           ]
         }
       ]
     }
     ```  
   - Connect output of "Check if description is empty" to this node.

5. **Add a Form node named "Analysis result":**  
   - Set form title to "Analysis"  
   - Set button label to "Another Analysis?"  
   - Set form description to: `={{ $json.candidates[0].content.parts[0].text }}` (expression to show Gemini AI response)  
   - Connect output of "Get analysis from Google Gemini" to this node.

6. **Add a Form node named "Redirect for another analysis":**  
   - Set operation to "completion"  
   - Set redirect URL to the webhook URL of the "Get Input" node (e.g., `https://your-n8n-instance/form/<webhook-id>`)  
   - Set response mode to "redirect"  
   - Connect output of "Analysis result" node to this node.

7. **Test the workflow:**  
   - Activate the workflow  
   - Open the "Get Input" form URL in a browser  
   - Enter a YouTube video URL and optionally a description or leave empty  
   - Submit and verify that analysis result appears  
   - Click "Another Analysis?" to verify redirection back to input form

8. **Credential Setup:**  
   - Add Google Cloud API key in n8n credentials (for `x-goog-api-key` header)  
   - No OAuth credentials required for this workflow

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                                         |
|----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| This workflow takes as input a YouTube link and optional description to analyze the video contents using Gemini AI.         | Sticky Note node content                                                                                               |
| The default prompt ensures the AI generates a detailed summary if no specific instructions are provided.                    | Sticky Note node content                                                                                               |
| Google Gemini API requires a valid Google Cloud API key passed in the `x-goog-api-key` header for authentication.           | API reference: https://cloud.google.com/generative-ai                                                                               |
| The redirect URL in the "Redirect for another analysis" node must match the webhook URL of the form trigger input node.     | Workflow configuration detail                                                                                          |
| Ensure that the YouTube URLs are publicly accessible and valid to allow the Gemini AI to fetch and analyze video content.    | Potential edge case                                                                                                     |

---

**Disclaimer:** The provided documentation is based on an n8n automated workflow and adheres strictly to content policies. It contains no illegal, offensive, or protected content. All data processed is legal and publicly accessible.