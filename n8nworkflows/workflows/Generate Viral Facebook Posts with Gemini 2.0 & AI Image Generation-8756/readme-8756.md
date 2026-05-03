Generate Viral Facebook Posts with Gemini 2.0 & AI Image Generation

https://n8nworkflows.xyz/workflows/generate-viral-facebook-posts-with-gemini-2-0---ai-image-generation-8756


# Generate Viral Facebook Posts with Gemini 2.0 & AI Image Generation

### 1. Workflow Overview

This workflow automates the generation and publication of viral Facebook posts using Google Gemini 2.0 AI for text generation and Hugging Face for AI image generation. It is designed for content marketers or social media managers who want to streamline content creation, image generation, and Facebook posting with integrated logging and notification.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception and Logging:** Receives user input via a web form, logs the input to Google Sheets for tracking.
- **1.2 AI Processing (Google Gemini 2.0):** Uses Google Gemini AI to analyze and rewrite the input content into a Facebook-ready post with an image prompt.
- **1.3 Image Generation:** Generates an AI image based on the prompt using Hugging Face API, converts it to a file, and uploads it to Facebook.
- **1.4 Facebook Posting:** Publishes the post with or without an attached image to a configured Facebook page.
- **1.5 Content Logging and Notification:** Saves the final post content to Google Sheets and sends a notification email after posting.
- **1.6 Conditional Logic and Error Handling:** Uses IF nodes to branch logic based on the presence of image prompts or post success flags.
- **1.7 Setup and Documentation:** Embedded Sticky Notes provide comprehensive setup, configuration instructions, and testing guidelines.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Logging

**Overview:**  
This block captures post ideas submitted via a web form and logs them to a Google Sheet for record-keeping.

**Nodes Involved:**  
- On form submission1  
- Append row in sheet  
- If  

**Node Details:**

- **On form submission1**  
  - *Type & Role:* Trigger node of type `formTrigger`. Receives user-submitted text input from a web form.  
  - *Config:* Webhook ID set to receive form data with a textarea field labeled "Input Information here" requiring at least 50 words.  
  - *Key Expressions:* User input accessed via `$json["Input Information here"]`.  
  - *Connections:* Output to `Append row in sheet`.  
  - *Edge Cases:* Missing or short input triggers form validation.  

- **Append row in sheet**  
  - *Type & Role:* Google Sheets node appending a new row. Logs submission date, time, and content into the Input Tracking Sheet.  
  - *Config:* Writes columns "Date" (current date), "Time" (current hour), and "Content" (form input). Document and sheet IDs are configurable.  
  - *Connections:* Output to `If`.  
  - *Edge Cases:* Google API errors, permission issues, or quota exceeded.  

- **If**  
  - *Type & Role:* Conditional node to check if the content length is sufficient (>50 characters) either in the "Content" field or fallback to nested JSON fields.  
  - *Config:* Checks `$json["Content"]` length or `$json.message.text.length >= 50`.  
  - *Connections:* If true, proceeds to `Basic LLM Chain`. Else, stops or takes alternative action.  
  - *Edge Cases:* Empty or malformed input may cause false negatives.

---

#### 2.2 AI Processing (Google Gemini 2.0)

**Overview:**  
Processes the input content by analyzing and generating a post text plus an image prompt using Google Gemini 2.0 language model.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Basic LLM Chain  
- code2  
- If2  
- HTTP Request  
- Convert to File code  
- Basic LLM Chain1  
- save content  
- Format Content  
- Merge  

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type & Role:* LangChain Google Gemini chat model node. Generates initial AI analysis and structured JSON output with `prompt_image` and `content`.  
  - *Config:* Model set to "models/gemini-2.0-flash-001", default options.  
  - *Connections:* Output to `Basic LLM Chain`.  
  - *Edge Cases:* API rate limits, invalid API key, timeouts, malformed input.  

- **Basic LLM Chain**  
  - *Type & Role:* LangChain LLM chain node executing a prompt that instructs the AI to analyze content, bullet main points, and return JSON with `prompt_image` and `content`.  
  - *Config:* Uses input content from form or sheet, outputs structured JSON within markdown code block.  
  - *Connections:* Output to `code2`.  
  - *Edge Cases:* AI response parsing failures if output format is incorrect.  

- **code2**  
  - *Type & Role:* `Code` node parsing the JSON embedded inside triple backticks (` ```json ... ``` `) returned by the AI. Extracts `prompt_image` and `content`.  
  - *Config:* JavaScript parsing with error handling; returns empty strings and error message if parsing fails.  
  - *Connections:* Outputs to `If2` and `Basic LLM Chain1`.  
  - *Edge Cases:* Parsing error if AI output format changes or contains invalid JSON.  

- **If2**  
  - *Type & Role:* Checks if `prompt_image` is present and truthy.  
  - *Config:* Condition: `!!$json.prompt_image === true`.  
  - *Connections:* If true, proceeds to `HTTP Request` for image generation; if false, skips image generation.  
  - *Edge Cases:* Missing or empty prompt leads to no image generation.  

- **HTTP Request**  
  - *Type & Role:* Calls Hugging Face API to generate an AI image based on the `prompt_image` text.  
  - *Config:* POST request with JSON body including `response_format: b64_json`, model `"black-forest-labs/FLUX.1-schnell"`, and prompt from previous node. Auth header with Bearer token must be configured.  
  - *Connections:* Output to `Convert to File code`.  
  - *Edge Cases:* API failures, invalid auth, slow response, empty image data.  

- **Convert to File code**  
  - *Type & Role:* Converts base64 image data from the API into a binary file format accepted by Facebook upload node.  
  - *Config:* Extracts `b64_json` property, converts to binary data with `image/png` MIME type and filename `image.png`.  
  - *Connections:* Output to `Facebook Upload Img`.  
  - *Edge Cases:* Missing or malformed base64 data throws error.  

- **Basic LLM Chain1**  
  - *Type & Role:* Second LangChain node rewriting the AI summary into a complete, coherent Facebook post article.  
  - *Config:* Takes summary from `code2` and instructs AI to generate friendly, concise Facebook-ready text (200-350 words, no hashtags or titles).  
  - *Connections:* Output to `save content`.  
  - *Edge Cases:* AI model errors, output formatting issues.

- **save content**  
  - *Type & Role:* Appends the final post content and metadata to a Google Sheets Content Log Sheet.  
  - *Config:* Columns include current date, short content from `code2`, and full content from AI output.  
  - *Connections:* Output to `Format Content`.  
  - *Edge Cases:* Google API failures or permission errors.  

- **Format Content**  
  - *Type & Role:* Cleans and formats the final post content by removing special characters before posting.  
  - *Config:* JavaScript function removing hashtags, asterisks, trimming spaces, and preparing JSON with `content` and platform info.  
  - *Connections:* Output to `Merge` node (second input).  
  - *Edge Cases:* Unexpected input format could cause trimming errors.  

- **Merge**  
  - *Type & Role:* Merges outputs from image upload and content formatting nodes to prepare for Facebook post.  
  - *Config:* Mode set to "combine by position".  
  - *Connections:* Output to `Facebook Graph API` node.  
  - *Edge Cases:* Node synchronization issues if inputs do not arrive simultaneously.

---

#### 2.3 Image Upload and Facebook Posting

**Overview:**  
Uploads generated images to Facebook as unpublished photos, then posts the Facebook feed with or without attached media.

**Nodes Involved:**  
- Facebook Upload Img  
- Facebook Graph API  
- If1  
- Send a message  

**Node Details:**

- **Facebook Upload Img**  
  - *Type & Role:* Facebook Graph API node uploading photo to the configured Facebook page.  
  - *Config:* POST to `photos` edge, with parameter `published` set to false (unpublished photo). Uses binary data from image conversion.  
  - *Connections:* Output to `Merge`.  
  - *Edge Cases:* Facebook API auth errors, upload failures, binary data missing.  

- **Facebook Graph API**  
  - *Type & Role:* Posts to the Facebook page feed, with message content and optional media attachment.  
  - *Config:* POST to `feed` edge, parameters include `message` from formatted content and `attached_media` JSON referencing uploaded photo ID if available.  
  - *Connections:* Output to `If1`.  
  - *Edge Cases:* Posting errors, permission issues, invalid page ID.  

- **If1**  
  - *Type & Role:* Conditional node checking if the post operation returned a client mutation ID indicating success.  
  - *Config:* Condition: `$json.post_supports_client_mutation_id === true`  
  - *Connections:* If true, sends notification email via `Send a message`.  
  - *Edge Cases:* API errors might cause missing mutation ID.  

- **Send a message**  
  - *Type & Role:* Gmail node sending a notification email confirming that the post will be published after 2 minutes, including a link to review the Facebook post.  
  - *Config:* Sends to configured email with subject and message referencing post ID parts.  
  - *Edge Cases:* Email API failures, invalid recipient address.

---

#### 2.4 Setup and Documentation

**Overview:**  
Sticky Note nodes provide detailed instructions for setup, configuration, and testing.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  

**Node Details:**

- **Sticky Note**  
  - Content includes comprehensive Facebook app setup, token generation, and page ID retrieval guide.  
- **Sticky Note1**  
  - Details Google Cloud API enabling, service account creation, key generation, and Google Sheets preparation.  
- **Sticky Note2**  
  - Instructions for obtaining Gemini API key, importing workflow, and credential configuration in n8n.  
- **Sticky Note3**  
  - Testing and validation steps for Google Sheets, Gemini AI node, image generation, Facebook upload, and end-to-end testing.

---

### 3. Summary Table

| Node Name          | Node Type                          | Functional Role                        | Input Node(s)           | Output Node(s)         | Sticky Note                              |
|--------------------|----------------------------------|--------------------------------------|------------------------|-----------------------|-----------------------------------------|
| On form submission1 | formTrigger                      | Receives user input via web form     | -                      | Append row in sheet    |                                         |
| Append row in sheet | googleSheets                    | Logs form input to Google Sheets     | On form submission1    | If                    |                                         |
| If                 | if                              | Checks content length > 50 chars     | Append row in sheet    | Basic LLM Chain       |                                         |
| Google Gemini Chat Model | lmChatGoogleGemini            | AI language model for content analysis | If                    | Basic LLM Chain       |                                         |
| Basic LLM Chain     | chainLlm                        | Analyzes content, outputs JSON format | Google Gemini Chat Model | code2                  |                                         |
| code2              | code                            | Parses AI JSON output                 | Basic LLM Chain        | If2, Basic LLM Chain1 |                                         |
| If2                | if                              | Checks if image prompt exists         | code2                  | HTTP Request          |                                         |
| HTTP Request       | httpRequest                     | Calls Hugging Face for AI image generation | If2                  | Convert to File code  |                                         |
| Convert to File code | code                            | Converts base64 image to binary file | HTTP Request           | Facebook Upload Img   |                                         |
| Facebook Upload Img | facebookGraphApi                | Uploads image as unpublished photo   | Convert to File code   | Merge                 |                                         |
| Basic LLM Chain1    | chainLlm                        | Rewrites summary into full content   | code2                  | save content          |                                         |
| save content       | googleSheets                    | Logs final post content to Google Sheets | Basic LLM Chain1      | Format Content        |                                         |
| Format Content     | function                       | Cleans and formats post content      | save content           | Merge                 |                                         |
| Merge              | merge                          | Combines image upload and content    | Facebook Upload Img, Format Content | Facebook Graph API |                                         |
| Facebook Graph API | facebookGraphApi                | Publishes post to Facebook feed      | Merge                  | If1                   |                                         |
| If1                | if                              | Checks if Facebook post succeeded    | Facebook Graph API     | Send a message        |                                         |
| Send a message     | gmail                          | Sends email notification              | If1                    | -                     |                                         |
| Sticky Note        | stickyNote                     | Setup guide Facebook app              | -                      | -                     | Complete Setup Guide: AI Facebook Post Generator |
| Sticky Note1       | stickyNote                     | Setup guide Google services           | -                      | -                     | Step 2: Google Services Setup           |
| Sticky Note2       | stickyNote                     | Setup guide AI services & n8n config  | -                      | -                     | Step 3 & 4: AI Services & Workflow Setup|
| Sticky Note3       | stickyNote                     | Testing and validation instructions   | -                      | -                     | Step 5: Testing & Validation             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node for Input**  
   - Add a **Form Trigger** node named `On form submission1`.  
   - Configure webhook with a textarea field labeled "Input Information here", required, placeholder "At least 50 words".  
   - Set button label "Gửi thông tin" and response text "Information sent".  

2. **Log Input to Google Sheets**  
   - Add a **Google Sheets** node named `Append row in sheet`.  
   - Set operation to "append" row.  
   - Map columns:  
     - Date: `{{$now.toFormat("dd/MM/yyyy")}}`  
     - Time: `{{$now.toFormat("HH")}}`  
     - Content: `{{$json["Input Information here"]}}`  
   - Connect credentials for Google Sheets (Service Account).  
   - Connect `On form submission1` output to this node's input.  

3. **Content Length Check**  
   - Add an **IF** node named `If`.  
   - Condition: Check if `{{$json["Content"]?.length > 0}}` or fallback to `$json.message?.text?.length >= 50`.  
   - Connect `Append row in sheet` output to `If`.  

4. **Configure Google Gemini Chat Model**  
   - Add a **LangChain Google Gemini Chat Model** node named `Google Gemini Chat Model`.  
   - Select model "models/gemini-2.0-flash-001".  
   - Connect output of `If` (true branch) to this node.  
   - Set Google PaLM API credential with Gemini API key.  

5. **Create Basic LLM Chain for Analysis**  
   - Add a **Basic LLM Chain** node.  
   - Configure with prompt instructing AI to analyze input content, output JSON with fields `prompt_image` and `content`.  
   - Connect `Google Gemini Chat Model` output to this node.  

6. **Parse AI JSON Output**  
   - Add a **Code** node named `code2`.  
   - JavaScript code to extract JSON inside triple backticks, parse it safely, return `prompt_image` and `content`.  
   - Connect `Basic LLM Chain` output to this node.  

7. **Check if Image Prompt Exists**  
   - Add an **IF** node named `If2`.  
   - Condition: `{{$json.prompt_image}}` cast to boolean equals true.  
   - Connect `code2` output to this node.  

8. **Image Generation HTTP Request**  
   - Add an **HTTP Request** node.  
   - Method: POST to `https://router.huggingface.co/together/v1/images/generations`.  
   - JSON Body: `{ "response_format": "b64_json", "prompt": "{{$json.prompt_image}}", "model": "black-forest-labs/FLUX.1-schnell" }`.  
   - Headers: Authorization Bearer token and Content-Type `application/json`.  
   - Connect `If2` true output to this node.  

9. **Convert Base64 to Binary File**  
   - Add a **Code** node named `Convert to File code`.  
   - JavaScript to extract `b64_json` and convert to binary property `data` with MIME type `image/png`.  
   - Connect `HTTP Request` output to this node.  

10. **Facebook Image Upload**  
    - Add a **Facebook Graph API** node named `Facebook Upload Img`.  
    - POST to `photos` edge of your Facebook page ID.  
    - Query param: `published=false`.  
    - Send binary data from previous node (`data`).  
    - Connect `Convert to File code` output to this node.  

11. **Create Basic LLM Chain for Final Post**  
    - Add a **Basic LLM Chain** node named `Basic LLM Chain1`.  
    - Configure prompt to rewrite summary into a Facebook-friendly post (200-350 words, friendly tone, no hashtags).  
    - Connect `code2` output (false branch of `If2`) to this node (also true branch via `code2` to `Basic LLM Chain1`).  

12. **Save Final Content to Google Sheets**  
    - Add a **Google Sheets** node named `save content`.  
    - Append content to Content Log Sheet with columns: Date, Short Content (from `code2`), Full Content (from `Basic LLM Chain1`).  
    - Connect `Basic LLM Chain1` output to this node.  

13. **Format Content for Posting**  
    - Add a **Function** node named `Format Content`.  
    - JavaScript to clean post content by removing special chars and preparing JSON with `content` and platform "facebook".  
    - Connect `save content` output to this node.  

14. **Merge Content and Image Upload**  
    - Add a **Merge** node.  
    - Mode: Combine by position.  
    - Connect `Facebook Upload Img` output to first input.  
    - Connect `Format Content` output to second input.  

15. **Facebook Post Publishing**  
    - Add a **Facebook Graph API** node named `Facebook Graph API`.  
    - POST to `feed` edge of your Facebook page ID.  
    - Parameters:  
      - `message` from merged content JSON.  
      - `attached_media` JSON referencing uploaded image id if available.  
    - Connect `Merge` output to this node.  

16. **Post Success Check**  
    - Add an **IF** node named `If1`.  
    - Condition: Check if `$json.post_supports_client_mutation_id` exists and true.  
    - Connect `Facebook Graph API` output to this node.  

17. **Send Notification Email**  
    - Add a **Gmail** node named `Send a message`.  
    - Configure recipient email, subject, and message with link to Facebook post using `$json.id`.  
    - Connect `If1` true output to this node.  
    - Set Gmail OAuth2 credentials.  

18. **Add Sticky Notes for Setup and Instructions**  
    - Add 4 **Sticky Note** nodes with the detailed setup guide for Facebook, Google, Gemini API, and testing instructions as documented.  

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Complete setup guide for Facebook app creation, token generation, and page ID retrieval. | Refer to Sticky Note titled "Complete Setup Guide: AI Facebook Post Generator" in the workflow. |
| Google Cloud services setup instructions including API enabling, service account creation, and Google Sheets setup. | See Sticky Note1 for detailed steps and links to Google Cloud Console. |
| Gemini API key acquisition and n8n credential configuration. | Described in Sticky Note2, includes link to https://makersuite.google.com/app/apikey |
| Testing and validation procedures for each workflow component including Google Sheets, AI nodes, image generation, and Facebook posting. | Provided in Sticky Note3 with step-by-step verification instructions. |
| Facebook Graph API version used is v22.0; ensure tokens have proper permissions: `pages_manage_posts`, `pages_read_engagement`, `pages_show_list`. | Relevant for Facebook Graph API and Facebook Upload Img nodes. |
| Hugging Face API requires valid Bearer token for image generation; model used is "black-forest-labs/FLUX.1-schnell". | Critical for HTTP Request node for image generation. |
| Gmail node requires OAuth2 credentials with sending permissions configured. | For Send a message node notifications. |

---

This documentation enables advanced users and AI agents to understand, reproduce, and maintain the workflow with clarity on logic, dependencies, and error management.