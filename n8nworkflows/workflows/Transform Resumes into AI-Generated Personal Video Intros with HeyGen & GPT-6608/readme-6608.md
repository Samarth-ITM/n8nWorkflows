Transform Resumes into AI-Generated Personal Video Intros with HeyGen & GPT

https://n8nworkflows.xyz/workflows/transform-resumes-into-ai-generated-personal-video-intros-with-heygen---gpt-6608


# Transform Resumes into AI-Generated Personal Video Intros with HeyGen & GPT

### 1. Workflow Overview

This workflow automates the transformation of uploaded resumes and user photos into personalized AI-generated video introductions featuring avatars with synthesized voices. It is designed for use cases such as recruitment, personal branding, or digital introductions where users upload their resume and photo, and receive a short, AI-scripted video intro.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Reception and extraction of user-submitted files (resume PDF and image).
- **1.2 Resume Parsing & Data Extraction:** Extract textual information from the resume PDF and parse key personal and professional data.
- **1.3 AI Processing (Script & Gender Detection):** Use OpenAI GPT to generate a personalized video script and predict the user's gender based on extracted resume data.
- **1.4 Avatar Photo Upload:** Upload user photo to HeyGen API to generate a talking photo avatar.
- **1.5 Video Generation & Voice Selection:** Based on gender, generate an AI video with the appropriate voice using HeyGen API.
- **1.6 Video Status Monitoring:** Wait and poll HeyGen API until the video generation is complete.
- **1.7 Result Logging:** Append the generated video URL, script, and avatar photo data into a Google Sheet for record-keeping and further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Accept user inputs via a form trigger node, allowing file uploads of an image and a resume PDF.
- **Nodes Involved:** 
  - Upload Resume & Photo (Form Trigger)
  - Extract Resume (File Extraction)
  - Talk image id generate (HTTP Request, “Avatar upload” node)

- **Node Details:**

  - **Upload Resume & Photo**
    - Type: Form Trigger
    - Role: Entry point to receive user-submitted resume (PDF) and photo (JPEG/PNG).
    - Configuration: Form with two required file fields — one for image (jpeg, jpg, png) and one for resume (pdf, docs).
    - Output: Binary data for resume and image.
    - Edge Cases: File upload failure, unsupported file types, missing files.
  
  - **Extract Resume**
    - Type: Extract From File
    - Role: Extract text contents from uploaded PDF resume.
    - Configuration: Operation set to PDF extraction on binary property named "Resume".
    - Output: Extracted resume text.
    - Edge Cases: Poorly formatted or scanned PDFs may result in extraction errors or incomplete text.
  
  - **Avatar upload (Talk image id generate)**
    - Type: HTTP Request
    - Role: Upload user image to HeyGen API to create talking photo avatar.
    - Configuration: POST to HeyGen `/v1/talking_photo` endpoint with binary image data, headers include `Content-Type: image/jpeg` and `x-api-key` with API key.
    - Input: Binary image from form trigger.
    - Output: JSON with `talking_photo_id` and photo URL.
    - Edge Cases: API key invalid or rate limits, image upload failures, unsupported image formats.

#### 2.2 Resume Parsing & Data Extraction

- **Overview:** Parse the extracted resume text to extract structured personal and professional information.
- **Nodes Involved:** 
  - Resume Information Extractor (Langchain Information Extractor)

- **Node Details:**

  - **Resume Information Extractor**
    - Type: Langchain Information Extractor
    - Role: Extract name, job title, skills, education, and summary from raw resume text.
    - Configuration: Attributes explicitly defined as required fields — name, JobTitle, skill, Education, Summary.
    - Input: Text extracted from resume.
    - Output: JSON with extracted structured fields.
    - Edge Cases: Ambiguous or incomplete resume content, extraction failures due to formatting.

#### 2.3 AI Processing (Script & Gender Detection)

- **Overview:** Use OpenAI GPT-4o-mini to generate a personalized 30-second video script and predict the user's gender based on extracted resume data.
- **Nodes Involved:** 
  - Generate Script & Find Gender (OpenAI Chat Model)

- **Node Details:**

  - **Generate Script & Find Gender**
    - Type: OpenAI Chat Model (Langchain openAi)
    - Role: Generate a JSON response containing predicted gender and a video intro script.
    - Configuration: 
      - Model: gpt-4o-mini
      - Input messages include user data (name, job title, skills, education, summary).
      - System prompt instructs to infer gender, produce a warm, professional 30-second script, and output strictly in JSON format.
    - Input: Extracted information from Resume Information Extractor.
    - Output: JSON with `gender` (Male/Female) and `script` text.
    - Edge Cases: Ambiguous names, unisex names, model misclassification, API rate limits.

#### 2.4 Avatar Photo Upload & Merging

- **Overview:** Combine the avatar talking photo data with the AI-generated script for further processing.
- **Nodes Involved:** 
  - Merge Image & Script (Merge node)

- **Node Details:**

  - **Merge Image & Script**
    - Type: Merge
    - Role: Combine outputs from avatar upload (image data) and script generation nodes into a single item.
    - Configuration: Combine mode by position (1-to-1 combining).
    - Input: Outputs from Talk image id generate and Generate Script & Find Gender.
    - Output: Merged JSON containing both talking_photo_id and script.
    - Edge Cases: Mismatched item counts; missing data from either branch.

#### 2.5 Gender-Based Video Generation

- **Overview:** Branch workflow based on predicted gender to generate video with male or female voice using HeyGen API.
- **Nodes Involved:** 
  - Switch to Male & Female (Switch node)
  - Generate Male Voice Video (HTTP Request)
  - Generate Female Voice Video (HTTP Request)

- **Node Details:**

  - **Switch to Male & Female**
    - Type: Switch
    - Role: Route flow based on gender output from AI script generation.
    - Configuration: Checks `$json.choices[0].message.content.gender` equals "Male" or "Female".
    - Output: Two branches, Male or Female.
    - Edge Cases: Gender field missing or different value; no matching branch.

  - **Generate Male Voice Video**
    - Type: HTTP Request
    - Role: Call HeyGen video generation API with male voice selection.
    - Configuration: 
      - POST to `/v2/video/generate` with JSON body including talking photo ID, script text, and male voice ID.
      - Headers include `X-Api-Key`.
      - Background color set to #FAFAFA.
    - Input: Merged data with talking_photo_id and script.
    - Output: Video generation job response including video_id.
    - Edge Cases: API errors, invalid keys, voice ID deprecated, network timeouts.

  - **Generate Female Voice Video**
    - Type: HTTP Request
    - Role: Same as Male Voice node but uses female voice ID.
    - Configuration: As above but with female voice ID.
    - Edge Cases: Same as male branch.

#### 2.6 Video Status Monitoring

- **Overview:** Wait for a fixed delay, then poll HeyGen API to check if video generation is complete.
- **Nodes Involved:** 
  - Wait (Wait node)
  - Check Status Video (HTTP Request)
  - Video is Complated or Not (If node)

- **Node Details:**

  - **Wait**
    - Type: Wait
    - Role: Delay execution for 3 minutes to allow video generation.
    - Configuration: Wait 3 minutes.
    - Edge Cases: Delays too short causing premature status check; excessive delays impact throughput.

  - **Check Status Video**
    - Type: HTTP Request
    - Role: GET request to HeyGen video status endpoint with video_id.
    - Configuration: Uses dynamic URL with video_id; headers include `X-Api-Key`.
    - Output: Video status JSON.
    - Edge Cases: API downtime, invalid video_id, rate limiting.

  - **Video is Complated or Not**
    - Type: If
    - Role: Check if status equals "completed".
    - True branch proceeds to append video info.
    - False branch loops back to Wait node for another delay cycle.
    - Edge Cases: Status unknown or error states, infinite loops if video never completes.

#### 2.7 Result Logging

- **Overview:** Append the generated video URL, associated avatar photo info, and script to a Google Sheet for record-keeping.
- **Nodes Involved:** 
  - Append video (Google Sheets)

- **Node Details:**

  - **Append video**
    - Type: Google Sheets
    - Role: Append a new row with video metadata.
    - Configuration:
      - Document ID and sheet name predefined.
      - Columns: Talking Photo ID, Photo URL, Script, Video URL.
      - Uses OAuth2 credentials for Google Sheets.
    - Input: Video URL from status check, photo data, and script.
    - Edge Cases: Google Sheets API quota limits, authentication failures, schema mismatch.

---

### 3. Summary Table

| Node Name                  | Node Type                       | Functional Role                        | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                      |
|----------------------------|--------------------------------|-------------------------------------|--------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| Upload Resume & Photo       | Form Trigger                   | Input reception of resume & image   | -                              | Extract Resume, Talk image id generate |                                                                                                 |
| Extract Resume             | Extract From File              | Extract text from resume PDF         | Upload Resume & Photo           | Resume Information Extractor    |                                                                                                 |
| Resume Information Extractor| Langchain Information Extractor| Parse structured info from resume   | Extract Resume                 | Generate Script & Find Gender   |                                                                                                 |
| Generate Script & Find Gender| OpenAI Chat Model             | Generate intro script & predict gender| Resume Information Extractor  | Merge Image & Script            |                                                                                                 |
| Talk image id generate (Avatar upload) | HTTP Request           | Upload user photo to HeyGen          | Upload Resume & Photo           | Merge Image & Script            |                                                                                                 |
| Merge Image & Script        | Merge                         | Combine avatar data & AI script      | Generate Script & Find Gender, Talk image id generate | Switch to Male & Female        |                                                                                                 |
| Switch to Male & Female     | Switch                        | Split flow by predicted gender       | Merge Image & Script            | Generate Male Voice Video, Generate Female Voice Video |                                                                                                 |
| Generate Male Voice Video   | HTTP Request                  | Create male-voice video on HeyGen    | Switch to Male & Female         | Wait                          |                                                                                                 |
| Generate Female Voice Video | HTTP Request                  | Create female-voice video on HeyGen  | Switch to Male & Female         | Wait                          |                                                                                                 |
| Wait                       | Wait                          | Delay to allow video generation      | Generate Male Voice Video, Generate Female Voice Video, Video is Complated or Not (false branch) | Check Status Video            |                                                                                                 |
| Check Status Video          | HTTP Request                  | Poll video generation status         | Wait                         | Video is Complated or Not       |                                                                                                 |
| Video is Complated or Not   | If                            | Check if video status is "completed" | Check Status Video              | Append video (true), Wait (false) |                                                                                                 |
| Append video                | Google Sheets                 | Append video info to Google Sheet    | Video is Complated or Not      | -                             |                                                                                                 |
| OpenAI Chat Model           | Langchain OpenAI Chat Model   | Connected to Resume Information Extractor (Legacy node, unused in main flow) | -                              | Resume Information Extractor    |                                                                                                 |
| Sticky Note                | Sticky Note                   | Displays sample Google Sheet link    | -                              | -                             | Sample Google Sheet: https://docs.google.com/spreadsheets/d/1jQtu4ZREsMge8elB-6xJoOMDMKsJLqof2dDmh0sxqWk/edit?usp=sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("Upload Resume & Photo")**  
   - Type: Form Trigger  
   - Configure form with two required file fields:  
     - "image" (accept `.jpeg, .png, .jpg`)  
     - "Resume" (accept `.pdf, .docs`)  
   - This node acts as the entry point for user uploads.

2. **Add Extract From File Node ("Extract Resume")**  
   - Type: Extract From File  
   - Set operation to `pdf`.  
   - Binary Property Name: `Resume` (matches form field name).  
   - Connect `Upload Resume & Photo` output to this node.

3. **Add Langchain Information Extractor Node ("Resume Information Extractor")**  
   - Type: Langchain Information Extractor  
   - Text input: Use expression `{{$json.text}}` from `Extract Resume` output.  
   - Define required attributes:  
     - name (description: extract name)  
     - JobTitle  
     - skill  
     - Education  
     - Summary  
   - Connect `Extract Resume` output to this node.

4. **Add OpenAI Chat Model Node ("Generate Script & Find Gender")**  
   - Type: OpenAI Chat Model (Langchain openAi)  
   - Model: `gpt-4o-mini`  
   - Messages setup:  
     - User prompt: Inject extracted info using Mustache expressions like `- Name: {{ $json.output.name }}`, etc.  
     - System prompt: Instructions for gender prediction and script generation as described in overview.  
   - Enable JSON output parsing.  
   - Connect `Resume Information Extractor` output to this node.  
   - Configure OpenAI credentials with valid API key.

5. **Add HTTP Request Node ("Talk image id generate")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://upload.heygen.com/v1/talking_photo`  
   - Headers:  
     - `Content-Type`: `image/jpeg`  
     - `x-api-key`: Your HeyGen API key  
   - Send binary data from `image` input field.  
   - Configure binary data field name as `image`.  
   - Connect `Upload Resume & Photo` output to this node.

6. **Add Merge Node ("Merge Image & Script")**  
   - Type: Merge  
   - Mode: Combine by position  
   - Connect outputs from `Generate Script & Find Gender` and `Talk image id generate` to this node.

7. **Add Switch Node ("Switch to Male & Female")**  
   - Type: Switch  
   - Condition on expression: `{{$json.choices[0].message.content.gender}}`  
   - Rule 1: Equals "Male" → Male branch  
   - Rule 2: Equals "Female" → Female branch  
   - Connect `Merge Image & Script` output to this node.

8. **Add HTTP Request Node ("Generate Male Voice Video")**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Headers:  
     - `X-Api-Key`: HeyGen API key  
   - JSON Body:  
     ```json
     {
       "video_inputs": [
         {
           "character": {
             "type": "talking_photo",
             "talking_photo_id": "{{ $json.data.talking_photo_id }}"
           },
           "voice": {
             "type": "text",
             "input_text": "{{ $json.choices[0].message.content.script }}",
             "voice_id": "2cfa7550c4dc41c7bd1b0751e7903332"
           },
           "background": {
             "type": "color",
             "value": "#FAFAFA"
           }
         }
       ]
     }
     ```
   - Connect Male output from Switch node here.

9. **Add HTTP Request Node ("Generate Female Voice Video")**  
   - Type: HTTP Request  
   - Same as Male Voice node except change `voice_id` to `"293ebc69dad345fc9dd1d542bc900673"`.  
   - Connect Female output from Switch node here.

10. **Add Wait Node ("Wait")**  
    - Type: Wait  
    - Configure to wait for 3 minutes.  
    - Connect outputs from both `Generate Male Voice Video` and `Generate Female Voice Video` to this node.

11. **Add HTTP Request Node ("Check Status Video")**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://api.heygen.com/v1/video_status.get?video_id={{ $json.data.video_id }}`  
    - Headers:  
      - `Accept`: `application/json`  
      - `X-Api-Key`: HeyGen API key  
    - Connect `Wait` node output here.

12. **Add If Node ("Video is Complated or Not")**  
    - Type: If  
    - Condition: Check if `{{$json.data.status}}` equals `"completed"`.  
    - True output: Connect to `Append video` node.  
    - False output: Loop back to `Wait` node for re-check.

13. **Add Google Sheets Node ("Append video")**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: Your target Google Sheets document ID.  
    - Sheet name: `gid=0` or your specific sheet.  
    - Columns to append:  
      - Talking Photo ID  
      - Photo URL  
      - Script  
      - Video Url  
    - Map fields using expressions:  
      - Photo: `{{ $('Talk image id generate').item.json.data.talking_photo_url }}`  
      - Script: `{{ $('Generate Script & Find Gender').item.json.choices[0].message.content.script }}`  
      - Video Url: `{{ $json.data.video_url }}`  
      - Talking Photo ID: `{{ $('Talk image id generate').item.json.data.talking_photo_id }}`  
    - Connect `Video is Complated or Not` true output here.  
    - Configure OAuth2 credentials for Google Sheets API.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                                   |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Sample Google Sheet used for storing generated video details.                                 | https://docs.google.com/spreadsheets/d/1jQtu4ZREsMge8elB-6xJoOMDMKsJLqof2dDmh0sxqWk/edit?usp=sharing             |
| HeyGen API documentation and voice IDs should be verified and updated as necessary.          | HeyGen API documentation (external, not included in workflow)                                                    |
| OpenAI GPT-4o-mini model used for script generation and gender prediction.                    | Requires valid OpenAI API key and adherence to usage policies                                                    |
| Workflow requires valid API keys and OAuth2 credentials for HeyGen and Google Sheets services.| Ensure credentials are securely stored and refreshed as needed                                                  |
| Workflow includes retry loops on video generation status to handle async processing delays.   | Potential infinite loop if video never completes; consider max retry count or timeout implementation             |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created using n8n, an integration and automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.