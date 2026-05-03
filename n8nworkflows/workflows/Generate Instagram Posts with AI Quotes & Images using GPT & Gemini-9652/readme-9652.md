Generate Instagram Posts with AI Quotes & Images using GPT & Gemini

https://n8nworkflows.xyz/workflows/generate-instagram-posts-with-ai-quotes---images-using-gpt---gemini-9652


# Generate Instagram Posts with AI Quotes & Images using GPT & Gemini

### 1. Workflow Overview

This workflow automates the generation of daily Instagram posts by creating inspirational quotes, captions, hashtags, and corresponding images using AI models. It then sends the compiled post via email for easy sharing. The primary use case is for social media creators or marketers who want to automate content creation and distribution on Instagram.

The workflow consists of these main logical blocks:

- **1.1 Initial Setup & Input Reading:** Reads previously generated posts from a local text file to avoid duplicate content.
- **1.2 AI Quote and Caption Generation:** Uses OpenAI's GPT model to generate a new inspirational quote, caption, and hashtags based on the history.
- **1.3 Post-Processing and Saving:** Splits the AI output into structured fields and saves the new quote to the local storage.
- **1.4 Image Generation:** Generates a custom image for the quote using Google Gemini (PaLM) image generation model.
- **1.5 Email Dispatch:** Sends an email containing the quote, caption, hashtags, and the generated image as an attachment.

---

### 2. Block-by-Block Analysis

#### 1.1 Initial Setup & Input Reading

- **Overview:**  
  This block reads the existing Instagram posts stored locally to ensure new generated quotes are unique and not repeated.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Read Text Files from Disk  
  - Extract from Text File  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: `scheduleTrigger`  
    - Role: Initiates the workflow periodically based on user-defined schedule settings.  
    - Configuration: Default interval settings (user customizable).  
    - Connections: Output → Read Text Files from Disk  
    - Edge Cases: Misconfiguration of schedule can cause no runs or too frequent runs.  
    - Version: 1.2

  - **Read Text Files from Disk**  
    - Type: `readWriteFile`  
    - Role: Reads the local file `/home/node/instagram_posts.txt` containing previously generated posts.  
    - Configuration: Read operation, no additional options.  
    - Connections: Output → Extract from Text File  
    - Edge Cases: File missing or inaccessible permissions may cause read failure.  
    - Version: 1

  - **Extract from Text File**  
    - Type: `extractFromFile`  
    - Role: Converts the text file content into JSON format under the key `post_history`.  
    - Configuration: Keeps original source, extracts text.  
    - Connections: Output → Basic LLM Chain  
    - Edge Cases: Malformed text content may cause extraction errors.  
    - Version: 1

---

#### 1.2 AI Quote and Caption Generation

- **Overview:**  
  Generates a new inspirational quote, caption, and hashtags using OpenAI GPT, ensuring no duplicates from previous posts.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenAI Chat Model  
  - Code-Split-LangChain  

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: `lmChatOpenAi`  
    - Role: Processes the prompt to generate creative text content.  
    - Configuration: Model set to `gpt-4.1-mini`. No specific options enabled.  
    - Credentials: OpenAI API required.  
    - Connections: Output (ai_languageModel) → Basic LLM Chain  
    - Edge Cases: API quota limits, network issues, or invalid credentials may block generation.  
    - Version: 1.2

  - **Basic LLM Chain**  
    - Type: `chainLlm`  
    - Role: Defines the prompt logic and processes the AI response to generate a formatted Instagram post content.  
    - Configuration:  
      - Prompt instructs generation of a short inspirational sentence (quote), caption, and hashtags, avoiding duplicates from `post_history`.  
      - Includes a system message positioning the AI as a self-growth coach.  
    - Connections: Output → Code-Split-LangChain  
    - Edge Cases: Prompt syntax issues may result in improper outputs.  
    - Version: 1.7

  - **Code-Split-LangChain**  
    - Type: `code`  
    - Role: Parses the multi-line AI output into separate JSON fields: `quote`, `caption`, `hashtags`.  
    - Configuration: Custom JavaScript code to split text by lines and assign to fields.  
    - Connections: Output → Convert Quote to File  
    - Edge Cases: Unexpected AI response formatting can cause parsing errors or empty fields.  
    - Version: 2

---

#### 1.3 Post-Processing and Saving

- **Overview:**  
  Converts the generated quote to a file-compatible format and appends it to the local storage file, preserving history for future runs.

- **Nodes Involved:**  
  - Convert Quote to File  
  - Write Text Files from Disk  

- **Node Details:**

  - **Convert Quote to File**  
    - Type: `convertToFile`  
    - Role: Converts the quote text into a binary file object for storage.  
    - Configuration: Converts text from the `quote` property, outputs as binary with property name `quote_for_file`.  
    - Connections: Output → Write Text Files from Disk  
    - Edge Cases: Large text or encoding issues might affect file conversion.  
    - Version: 1.1

  - **Write Text Files from Disk**  
    - Type: `readWriteFile`  
    - Role: Appends the new quote file content to `/home/node/instagram_posts.txt`.  
    - Configuration: Append mode enabled to preserve existing data.  
    - Connections: Output → Generate an image  
    - Edge Cases: File permission errors or disk full might prevent writing.  
    - Version: 1

---

#### 1.4 Image Generation

- **Overview:**  
  Creates a visual image incorporating the quote text using Google Gemini’s image generation model.

- **Nodes Involved:**  
  - Generate an image  

- **Node Details:**

  - **Generate an image**  
    - Type: `@n8n/n8n-nodes-langchain.googleGemini`  
    - Role: Generates a simple image with the quote text embedded in bold, clear font and contrasting colors (red).  
    - Configuration:  
      - Model: `models/gemini-2.0-flash-preview-image-generation` (Google Gemini)  
      - Prompt dynamically includes the generated quote.  
    - Credentials: Google Palm API account required.  
    - Connections: Output → Send email  
    - Edge Cases: API limits, network failures, or malformed prompt can cause image generation failure.  
    - Version: 1

---

#### 1.5 Email Dispatch

- **Overview:**  
  Sends an email containing the daily Instagram post’s quote, caption, hashtags, and the generated image attached.

- **Nodes Involved:**  
  - Send email  

- **Node Details:**

  - **Send email**  
    - Type: `emailSend`  
    - Role: Sends the composed Instagram post content via SMTP email.  
    - Configuration:  
      - Email format: Plain text  
      - Subject: "Daily Instagram Post"  
      - To email: User must specify (currently empty placeholder)  
      - From email: `noreply@aialchemysolutions.com`  
      - Email body dynamically includes quote, caption, hashtags from `Code-Split-LangChain` node.  
      - Attaches the generated image as a file attachment.  
    - Credentials: SMTP account required.  
    - Connections: Terminal node.  
    - Edge Cases: Authentication errors, incorrect recipient emails, or attachment issues may cause email failures.  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                 | Node Type                                 | Functional Role                         | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                          |
|---------------------------|-------------------------------------------|---------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger          | scheduleTrigger                          | Starts workflow periodically          |                                  | Read Text Files from Disk       | # 1. Initial Stage: Schedule controls periodic runs; reads previous quotes to avoid duplicates.    |
| Read Text Files from Disk | readWriteFile                           | Reads stored post history file        | Schedule Trigger                 | Extract from Text File          | # 1. Initial Stage: Reads all previous generated quotes and convert to JSON for next step.         |
| Extract from Text File     | extractFromFile                         | Parses file content to JSON            | Read Text Files from Disk        | Basic LLM Chain                |                                                                                                    |
| OpenAI Chat Model          | lmChatOpenAi                           | AI Model to generate quote/caption    |                                | Basic LLM Chain (ai_languageModel) |                                                                                                    |
| Basic LLM Chain            | chainLlm                              | Defines prompt & processes AI output  | Extract from Text File / OpenAI Chat Model | Code-Split-LangChain          | # 2. Quote Generation: Uses AI model to generate new quote, caption, hashtags avoiding duplicates.  |
| Code-Split-LangChain       | code                                  | Parses AI output into structured data | Basic LLM Chain                 | Convert Quote to File           |                                                                                                    |
| Convert Quote to File      | convertToFile                         | Converts quote text to file format    | Code-Split-LangChain             | Write Text Files from Disk      | # 3. Save new Quote: Saves new quote to file for next runs.                                        |
| Write Text Files from Disk | readWriteFile                        | Appends new quote to local storage    | Convert Quote to File            | Generate an image              |                                                                                                    |
| Generate an image          | googleGemini (image generation)       | Generates image with quote text       | Write Text Files from Disk       | Send email                    | # 4. Image Generation and Send Email: Generate image and send email with post content and image.   |
| Send email                 | emailSend                            | Sends email with post content & image | Generate an image               |                                | Update receiver email address field                                                                |
| Sticky Note                | stickyNote                          | Documentation and instructions        |                                |                                | See detailed notes below                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: `Schedule Trigger`  
   - Set your desired run interval (e.g., daily at a specific time).

2. **Add a Read Text Files from Disk node:**  
   - Type: `Read/Write File`  
   - Operation: Read  
   - File path: `/home/node/instagram_posts.txt`  
   - Connect Schedule Trigger output to this node.

3. **Add an Extract from Text File node:**  
   - Type: `Extract from File`  
   - Operation: Text extraction  
   - Destination key: `post_history`  
   - Connect Read Text Files output to this node.

4. **Add an OpenAI Chat Model node:**  
   - Type: `LangChain AI Chat OpenAI`  
   - Model: `gpt-4.1-mini` (or your preferred GPT model)  
   - Credentials: Configure OpenAI API credentials.  
   - Connect no direct input here; connect Extract from Text File output to Basic LLM Chain (see next step).

5. **Add a Basic LLM Chain node:**  
   - Type: `LangChain Chain LLM`  
   - Prompt:  
     ```
     Create an Instagram post about self-growth and partnership, inspired by Rumi, Khayam, or similar poets. Make sure don't generate these again as previously generated: "{{ $json.post_history }}"

     OUTPUT:
     No additional text, explanations or title for sections and only provide below information:
     1. One short inspirational sentence between 5 to 25 words about self-growth or partnership, including quoting the poet’s name.
     2. A suggested Instagram caption to accompany the post.
     3. A list of suitable hashtags.
     ```
   - System message: "You are a self growth and partnership mentor and coach and by giving inspirational quotes you will give insight and guideline to people for better life."  
   - Connect Extract from Text File output as input for `post_history` context.  
   - Connect OpenAI Chat Model output as AI language model input.

6. **Add a Code node (Code-Split-LangChain):**  
   - Type: `Code` (JavaScript)  
   - Script:  
     ```js
     const input = items[0].json.text.replace(/[\"“]/g, ''); 
     const lines = input.split('\n').filter(line => line.trim() !== ''); 

     return [
       {
         json: {
           quote: lines[0] || '',
           caption: lines[1] || '',
           hashtags: lines[2] || ''
         }
       }
     ];
     ```
   - Connect Basic LLM Chain output to this node.

7. **Add a Convert Quote to File node:**  
   - Type: `Convert To File`  
   - Operation: `toText`  
   - Source Property: `quote`  
   - Binary Property Name: `quote_for_file`  
   - Connect Code node output to this node.

8. **Add a Write Text Files from Disk node:**  
   - Type: `Read/Write File`  
   - Operation: Write  
   - File Name: `/home/node/instagram_posts.txt`  
   - Options: Append enabled  
   - Data Property Name: `quote_for_file`  
   - Connect Convert Quote to File output to this node.

9. **Add a Generate an image node:**  
   - Type: `Google Gemini (PaLM) Image Generation`  
   - Model ID: `models/gemini-2.0-flash-preview-image-generation`  
   - Prompt:  
     ```
     Draw a simple image for below quote and put the text in the image with bold and clear font and contrast color like red. This is the quote: {{ $('Code-Split-LangChain').item.json.quote }}
     ```
   - Credentials: Configure Google Palm API credentials.  
   - Connect Write Text Files from Disk output to this node.

10. **Add a Send Email node:**  
    - Type: `Email Send`  
    - To Email: Set recipient email address.  
    - From Email: `noreply@aialchemysolutions.com` (or your preferred sender)  
    - Subject: "Daily Instagram Post"  
    - Text:  
      ```
      Hello dear,

      Daily tips and guide has attached to this email.

      {{ $('Code-Split-LangChain').item.json.quote }}

      {{ $('Code-Split-LangChain').item.json.caption }}

      {{ $('Code-Split-LangChain').item.json.hashtags }}

      Best
      AI
      ```
    - Attachments: Attach the output file from the image generation node.  
    - Credentials: Configure SMTP credentials.  
    - Connect Generate an image output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow automates Instagram post creation including quote, caption, hashtags, and image generation, then emails the full content.                                                                                                                                                       | Workflow purpose summary.                                                                        |
| To enable full automation, connect the workflow’s output to Facebook API or Instagram API for direct posting.                                                                                                                                                                                 | Extension idea for automation beyond email.                                                     |
| Ensure you have a local file `/home/node/instagram_posts.txt` with appropriate permissions to store generated quotes and avoid duplicates.                                                                                                                                                     | Local file storage requirement.                                                                 |
| Update the system and user prompt in the Basic LLM Chain node to customize the style or topic of generated posts.                                                                                                                                                                            | Customization tip (Sticky Note1).                                                                |
| Modify the image generation prompt in the Google Gemini node to change image style or text formatting.                                                                                                                                                                                       | Customization tip (Sticky Note6).                                                                |
| Update recipient email address in the Send Email node to your desired inbox.                                                                                                                                                                                                                   | Customization tip (Sticky Note7).                                                                |
| Workflow uses OpenAI GPT-4.1-mini and Google Gemini PaLM API; ensure you have valid credentials and API quota.                                                                                                                                                                                | Credential requirements.                                                                         |
| SMTP account configured is required for email delivery; test connection before workflow execution to avoid delivery failures.                                                                                                                                                                | Credential requirements.                                                                         |
| Workflow is designed for periodic execution; adjust Schedule Trigger node interval according to your posting frequency preference.                                                                                                                                                           | Scheduling advice.                                                                              |
| For troubleshooting, monitor API call limits, file permission errors, and email server logs for failures.                                                                                                                                                                                    | Troubleshooting guidance.                                                                        |
| Email format is plain text; if HTML emails are desired, modify the Send Email node settings accordingly.                                                                                                                                                                                      | Possible enhancement.                                                                           |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.