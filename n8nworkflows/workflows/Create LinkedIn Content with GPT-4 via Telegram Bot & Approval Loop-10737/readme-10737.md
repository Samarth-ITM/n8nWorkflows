Create LinkedIn Content with GPT-4 via Telegram Bot & Approval Loop

https://n8nworkflows.xyz/workflows/create-linkedin-content-with-gpt-4-via-telegram-bot---approval-loop-10737


# Create LinkedIn Content with GPT-4 via Telegram Bot & Approval Loop

---

### 1. Workflow Overview

This workflow is designed to transform Telegram messages (text or voice) into polished LinkedIn posts using GPT-4, with an interactive approval loop and direct publishing via Blotato. It targets users who want a streamlined, AI-assisted content creation and publishing pipeline integrated with Telegram and LinkedIn.

The workflow is logically divided into four key blocks:

- **1.1 Input Reception:** Captures Telegram messages, including voice notes, and converts voice inputs to text.
- **1.2 AI Content Creation:** Uses GPT-4 to draft LinkedIn posts from user input, allowing iterative refinements based on user feedback.
- **1.3 Approval & Publishing:** Detects approval signals from the user, extracts the final post, and publishes it to LinkedIn through Blotato.
- **1.4 Status Monitoring & Confirmation:** Polls Blotato to verify post publication status and sends confirmation with the live post link back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Handles incoming Telegram messages via webhook. Differentiates between text and voice messages; voice files are downloaded and transcribed into text using OpenAI Whisper.

**Nodes Involved:**  
- Start: Telegram Message  
- Prepare Input  
- Check: ist it a Voice?  
- Get Voice File  
- Speech to Text

**Node Details:**

- **Start: Telegram Message**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point receiving all Telegram messages.  
  - *Config:* Listens to "message" updates, uses Telegram API credentials.  
  - *Connections:* Outputs to Prepare Input node.  
  - *Failures:* Telegram API errors, webhook misconfiguration.

- **Prepare Input**  
  - *Type:* Set  
  - *Role:* Extracts text field from the Telegram message JSON or sets empty string if none.  
  - *Config:* Sets `text` to either `message.text` or empty string.  
  - *Connections:* Outputs to Check: ist it a Voice? node.

- **Check: ist it a Voice?**  
  - *Type:* If  
  - *Role:* Determines if the message is a voice note by checking if text is empty.  
  - *Config:* Condition: text is empty → voice message; else text message.  
  - *Connections:*  
    - If true (voice): to Get Voice File node.  
    - If false (text): directly to AI: Draft & Revise Post node.  
  - *Edge Cases:* Messages with empty text but no voice file may cause issues.

- **Get Voice File**  
  - *Type:* Telegram Node (File)  
  - *Role:* Downloads the voice file from Telegram using voice `file_id`.  
  - *Config:* `fileId` dynamically extracted from the original Telegram message.  
  - *Connections:* Outputs file binary data to Speech to Text node.  
  - *Failures:* Expired or invalid file ID, Telegram API limits.

- **Speech to Text**  
  - *Type:* OpenAI Audio Transcription  
  - *Role:* Converts voice audio into transcribed text using OpenAI Whisper.  
  - *Config:* Standard transcription operation, uses dedicated OpenAI API credentials.  
  - *Connections:* Outputs transcript text to AI: Draft & Revise Post node.  
  - *Failures:* Audio file format issues, API rate limits, transcription errors.

---

#### 2.2 AI Content Creation

**Overview:**  
This block uses LangChain's agent pattern with OpenAI GPT-4 to generate an optimized LinkedIn post draft from user input, leveraging a conversational memory buffer for iterative refinement based on user feedback.

**Nodes Involved:**  
- OpenAI Chat Model  
- Window Buffer Memory  
- AI: Draft & Revise Post

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model node  
  - *Role:* Provides GPT-4.1-mini model for text generation.  
  - *Config:* Model set to "gpt-4.1-mini", uses OpenAI credentials.  
  - *Connections:* Feeds AI: Draft & Revise Post as language model.  
  - *Failures:* API key invalid, rate limiting, model availability.

- **Window Buffer Memory**  
  - *Type:* LangChain Memory Buffer (Window)  
  - *Role:* Maintains last 10 user interactions for context in AI conversations, keyed by Telegram user ID.  
  - *Config:* Uses custom session key based on Telegram user ID from incoming messages.  
  - *Connections:* Feeds AI: Draft & Revise Post as memory input.  
  - *Failures:* Mis-keyed sessions can cause context loss.

- **AI: Draft & Revise Post**  
  - *Type:* LangChain Agent  
  - *Role:* Core AI agent that analyzes input text, drafts LinkedIn post with hashtags, and manages iterative refinement prompts.  
  - *Config:*  
    - System prompt specifies the LinkedIn post structure (hook, value, CTA, hashtags).  
    - Handles user feedback requests and only outputs JSON with final post on approval keywords.  
  - *Connections:* Outputs to Check if Approved node.  
  - *Variables:* Uses `$json.text` as input text, session memory, and GPT-4 model.  
  - *Failures:* Prompt misconfiguration, unexpected user input format, conversational memory desync.

---

#### 2.3 Approval & Publishing

**Overview:**  
Determines if user input signals approval to post. Upon approval, extracts the final post from AI output JSON, then creates the post on LinkedIn via Blotato API.

**Nodes Involved:**  
- Check if Approved  
- Approval: Extract Final Post Text  
- Create post with Blotato  
- Post Suggestion Or Ask For Approval

**Node Details:**

- **Check if Approved**  
  - *Type:* If  
  - *Role:* Checks if AI output contains approval keywords or JSON (indicating final approval).  
  - *Config:* Checks if AI output contains "videoPrompt", "socialMediaText", or starts with "{" (JSON).  
  - *Connections:*  
    - If true: to Approval: Extract Final Post Text node.  
    - If false: to Post Suggestion Or Ask For Approval (send draft back to user).  
  - *Failures:* False negatives if user approval phrasing differs.

- **Approval: Extract Final Post Text**  
  - *Type:* Code  
  - *Role:* Parses AI output JSON strictly to extract the final LinkedIn post text.  
  - *Config:*  
    - Removes markdown fences.  
    - Extracts JSON object and validates presence of "Post" field.  
    - Fallback regex extraction if JSON parsing fails.  
  - *Connections:* Outputs normalized post text to Create post with Blotato.  
  - *Failures:* Malformed AI output, missing "Post" field.

- **Create post with Blotato**  
  - *Type:* Blotato LinkedIn Node  
  - *Role:* Publishes the final LinkedIn post via Blotato API.  
  - *Config:*  
    - Uses LinkedIn platform, account ID selected from list.  
    - Post content taken from extracted JSON "Post" text.  
    - Requires Blotato API credentials.  
  - *Connections:* Outputs to Give Blotat 5s :) node (wait).  
  - *Failures:* API authorization errors, content validation errors.

- **Post Suggestion Or Ask For Approval**  
  - *Type:* Telegram  
  - *Role:* Sends AI-generated draft post back to user in Telegram, requesting feedback or approval.  
  - *Config:* Sends AI output text to Telegram chat ID from original message.  
  - *Connections:* Terminal node for non-approved outputs.  
  - *Failures:* Telegram API errors, message formatting issues.

---

#### 2.4 Status Monitoring & Confirmation

**Overview:**  
After posting, this block polls Blotato to monitor the post’s publication status. It waits and retries if the post is still "in-progress". Upon publication, it sends a confirmation message with the live post URL to the user on Telegram. If an error occurs, it notifies the user.

**Nodes Involved:**  
- Give Blotat 5s :)  
- Check post status  
- Published?  
- In Progress?  
- Give Blotat other 5s :)  
- Send a confirmation message  
- Send an error message

**Node Details:**

- **Give Blotat 5s :)**  
  - *Type:* Wait  
  - *Role:* Pauses execution for 5 seconds to allow Blotato processing time.  
  - *Connections:* Outputs to Check post status node.  
  - *Failures:* None typical, except workflow timeout if stuck.

- **Check post status**  
  - *Type:* Blotato API Node  
  - *Role:* Requests the status of the post submission from Blotato using `postSubmissionId`.  
  - *Config:* Operation "get" with dynamic postSubmissionId from previous node.  
  - *Connections:* Outputs to Published? node.  
  - *Failures:* API errors, invalid postSubmissionId.

- **Published?**  
  - *Type:* If  
  - *Role:* Checks if the post status is "published".  
  - *Connections:*  
    - If true: to Send a confirmation message node.  
    - If false: to In Progress? node.  
  - *Failures:* Status field missing or unexpected status values.

- **In Progress?**  
  - *Type:* If  
  - *Role:* Checks if the post status is "in-progress".  
  - *Connections:*  
    - If true: to Give Blotat other 5s :) (wait and recheck).  
    - If false: to Send an error message node (assumes failure).  
  - *Failures:* Unexpected status values.

- **Give Blotat other 5s :)**  
  - *Type:* Wait  
  - *Role:* Additional 5-second pause before rechecking status.  
  - *Connections:* Loops back to Check post status node.  
  - *Failures:* Potential infinite loop if status never resolves.

- **Send a confirmation message**  
  - *Type:* Telegram  
  - *Role:* Sends a message to the user confirming that the LinkedIn post is live, including a clickable public URL.  
  - *Config:* Uses original Telegram chat ID, message text includes dynamic post URL.  
  - *Connections:* Terminal node.  
  - *Failures:* Telegram API errors.

- **Send an error message**  
  - *Type:* Telegram  
  - *Role:* Notifies the user that an error occurred during posting.  
  - *Config:* Fixed error text sent to original Telegram chat ID.  
  - *Connections:* Terminal node.  
  - *Failures:* Telegram API errors.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                | Input Node(s)                    | Output Node(s)                          | Sticky Note                                                                                                       |
|-------------------------------|----------------------------------|------------------------------------------------|---------------------------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Start: Telegram Message        | Telegram Trigger                 | Captures incoming Telegram messages             | -                               | Prepare Input                         | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper          |
| Prepare Input                 | Set                              | Extracts text or sets empty string               | Start: Telegram Message          | Check: ist it a Voice?                 | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper          |
| Check: ist it a Voice?         | If                               | Checks if message is voice or text               | Prepare Input                   | Get Voice File / AI: Draft & Revise Post | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper          |
| Get Voice File                | Telegram Node (File)             | Downloads voice file from Telegram                | Check: ist it a Voice?           | Speech to Text                       | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper          |
| Speech to Text                | OpenAI Audio Transcription       | Transcribes voice to text                          | Get Voice File                  | AI: Draft & Revise Post              | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper          |
| OpenAI Chat Model             | LangChain OpenAI Chat Model      | Provides GPT-4 model for AI content generation   | - (connected via ai_languageModel) | AI: Draft & Revise Post              | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory |
| Window Buffer Memory          | LangChain Memory Buffer Window   | Maintains conversation context per user          | - (connected via ai_memory)      | AI: Draft & Revise Post              | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory |
| AI: Draft & Revise Post       | LangChain Agent                 | Drafts LinkedIn post, handles iterative feedback | Speech to Text / Check: ist it a Voice? / OpenAI Chat Model / Window Buffer Memory | Check if Approved                   | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory |
| Check if Approved             | If                               | Determines if user approved final post            | AI: Draft & Revise Post          | Approval: Extract Final Post Text / Post Suggestion Or Ask For Approval | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, and posts to LinkedIn via Blotato |
| Approval: Extract Final Post Text | Code                         | Parses AI output JSON to extract final post text | Check if Approved               | Create post with Blotato             | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, and posts to LinkedIn via Blotato |
| Create post with Blotato       | Blotato Node                   | Publishes post on LinkedIn via Blotato           | Approval: Extract Final Post Text | Give Blotat 5s :)                   | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, and posts to LinkedIn via Blotato |
| Post Suggestion Or Ask For Approval | Telegram                     | Sends AI draft post back to user for feedback     | Check if Approved               | (Terminal)                         | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, and posts to LinkedIn via Blotato |
| Give Blotat 5s :)             | Wait                             | Waits 5 seconds before checking post status       | Create post with Blotato         | Check post status                   | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| Check post status             | Blotato Node                    | Checks LinkedIn post publication status           | Give Blotat 5s :)               | Published?                         | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| Published?                   | If                               | Checks if post status is "published"              | Check post status               | Send a confirmation message / In Progress? | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| In Progress?                 | If                               | Checks if post status is "in-progress"             | Published?                     | Give Blotat other 5s :) / Send an error message | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| Give Blotat other 5s :)       | Wait                             | Additional wait before re-checking post status    | In Progress?                   | Check post status                   | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| Send a confirmation message   | Telegram                        | Sends confirmation and live post link to Telegram | Published?                     | (Terminal)                         | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| Send an error message         | Telegram                        | Notifies user about posting error                   | In Progress?                   | (Terminal)                         | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |
| Sticky Note                  | Sticky Note                     | Overview and instructions                          | -                               | -                                  | See next section for details                                                                                     |
| Sticky Note1                 | Sticky Note                     | Describes Input Reception block                    | -                               | -                                  | ## 1. Capture Input<br>Receives Telegram messages and converts voice notes to text using OpenAI Whisper          |
| Sticky Note2                 | Sticky Note                     | Describes AI Content Creation block                 | -                               | -                                  | ## 2. AI Content Creation<br>Generates LinkedIn posts and handles revision requests through conversational memory |
| Sticky Note3                 | Sticky Note                     | Describes Approval & Publishing block               | -                               | -                                  | ## 3. Approval & Publishing<br>Detects approval keywords, extracts final content, and posts to LinkedIn via Blotato |
| Sticky Note4                 | Sticky Note                     | Describes Status Monitoring block                    | -                               | -                                  | ## 4. Status Monitoring<br>Polls Blotato API to verify publication and sends confirmation with post link to Telegram |

---

### 4. Reproducing the Workflow from Scratch

**Step 1:** Create a Telegram Trigger node named "Start: Telegram Message"  
- Set to listen for "message" updates only.  
- Configure Telegram API credentials (your Telegram bot token).  

**Step 2:** Add a Set node named "Prepare Input"  
- Extract the text content from the Telegram message JSON using expression `{{$json?.message?.text || ""}}`.  
- Output field named "text".  
- Connect "Start: Telegram Message" → "Prepare Input".

**Step 3:** Add an If node named "Check: ist it a Voice?"  
- Condition: Check if `text` field is empty (true if voice message).  
- Connect "Prepare Input" → "Check: ist it a Voice?".

**Step 4:** Add a Telegram node named "Get Voice File"  
- Operation: Download file.  
- Set "fileId" to `={{ $('Start: Telegram Message').item.json.message.voice.file_id }}`.  
- Use Telegram API credentials.  
- Connect "Check: ist it a Voice?" (true) → "Get Voice File".

**Step 5:** Add an OpenAI node named "Speech to Text"  
- Operation: Audio transcription.  
- Use OpenAI API credentials.  
- Connect "Get Voice File" → "Speech to Text".

**Step 6:** Add LangChain OpenAI Chat Model node named "OpenAI Chat Model"  
- Set model to "gpt-4.1-mini".  
- Use OpenAI API credentials.

**Step 7:** Add LangChain Memory Buffer Window node named "Window Buffer Memory"  
- Set session key to Telegram user ID: `={{ $('Start: Telegram Message').first().json.message.from.id }}`.  
- Context window length: 10.

**Step 8:** Add LangChain Agent node named "AI: Draft & Revise Post"  
- Input text: `={{ $json.text }}`.  
- Assign "OpenAI Chat Model" as language model input.  
- Assign "Window Buffer Memory" as memory input.  
- Provide the system prompt that instructs the model to create LinkedIn posts, handle feedback, and output JSON only upon approval.  
- Connect outputs from "Speech to Text" (voice path) and "Check: ist it a Voice?" (text path) to "AI: Draft & Revise Post".

**Step 9:** Add an If node named "Check if Approved"  
- Condition: Check if AI output contains approval keywords or JSON (starts with "{").  
- Connect "AI: Draft & Revise Post" → "Check if Approved".

**Step 10:** Add a Code node named "Approval: Extract Final Post Text"  
- JavaScript code to parse and extract the "Post" field from AI output JSON, with fallback regex parsing.  
- Connect "Check if Approved" (true) → "Approval: Extract Final Post Text".

**Step 11:** Add a Blotato node named "Create post with Blotato"  
- Operation: Create post.  
- Platform: LinkedIn.  
- Account: select appropriate Blotato LinkedIn account ID.  
- Post content text: `={{ $json.Post }}` from previous step.  
- Use Blotato API credentials.  
- Connect "Approval: Extract Final Post Text" → "Create post with Blotato".

**Step 12:** Add a Wait node named "Give Blotat 5s :)"  
- Duration: 5 seconds.  
- Connect "Create post with Blotato" → "Give Blotat 5s :)".

**Step 13:** Add a Blotato node named "Check post status"  
- Operation: Get post status.  
- Post submission ID: `={{ $json.postSubmissionId }}` from Blotato create post output.  
- Use Blotato credentials.  
- Connect "Give Blotat 5s :)" → "Check post status".

**Step 14:** Add an If node named "Published?"  
- Condition: Check if status equals "published".  
- Connect "Check post status" → "Published?".

**Step 15:** Add an If node named "In Progress?"  
- Condition: Check if status equals "in-progress".  
- Connect "Published?" (false) → "In Progress?".

**Step 16:** Add a Wait node named "Give Blotat other 5s :)"  
- Duration: 5 seconds.  
- Connect "In Progress?" (true) → "Give Blotat other 5s :)" → loop back to "Check post status".

**Step 17:** Add a Telegram node named "Send a confirmation message"  
- Text: `"Your post is online!\n\nClick here to go to the post:\n{{ $json.publicUrl }}"`.  
- Chat ID: `={{ $('Start: Telegram Message').item.json.message.chat.id }}`.  
- Connect "Published?" (true) → "Send a confirmation message".

**Step 18:** Add a Telegram node named "Send an error message"  
- Text: `"There was an error uploading your post."`.  
- Chat ID: same as above.  
- Connect "In Progress?" (false) → "Send an error message".

**Step 19:** Add a Telegram node named "Post Suggestion Or Ask For Approval"  
- Text: `={{ $('AI: Draft & Revise Post').item.json.output }}` (draft output).  
- Chat ID: same as above.  
- Connect "Check if Approved" (false) → "Post Suggestion Or Ask For Approval".

**Step 20:** Add Sticky Notes (optional) for documentation at appropriate workflow positions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                 |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Transform your Telegram into a LinkedIn content assistant with iterative AI drafting and approval.                                                                                                                                               | Overview from Sticky Note content                                                             |
| Create your Telegram bot with @BotFather and add API token to n8n credentials.                                                                                                                                                                | Setup instruction in Sticky Note                                                              |
| Connect OpenAI API keys for Chat and Audio transcription in n8n credentials.                                                                                                                                                                  | Required for AI generation and Whisper transcription                                          |
| Connect your LinkedIn profile to Blotato and add Blotato API credentials in n8n.                                                                                                                                                                | [Blotato](https://blotato.com/?ref=feras)                                                    |
| Customize the AI system prompt in "AI: Draft & Revise Post" node to match your brand voice, style, tone, and hashtag preferences.                                                                                                           | Prompt text inside AI node                                                                    |
| For scheduling posts instead of immediate publishing, modify Blotato operation from "create" to "schedule".                                                                                                                               | Customization tip from Sticky Note                                                           |
| The workflow includes robust parsing of AI JSON output to handle mixed text and JSON scenarios, ensuring only final approved content is posted.                                                                                              | Node: Approval: Extract Final Post Text                                                      |
| Edge cases include API rate limits, malformed AI outputs, Telegram voice message edge cases, and potential infinite loops in status polling (recommended to set workflow timeouts).                                                            | General operational considerations                                                          |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow designed with n8n, a workflow automation tool. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All processed data are legal and publicly available.

---