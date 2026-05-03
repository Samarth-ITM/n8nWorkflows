Edit Images with Text Prompts using Gemini AI and Telegram Bot

https://n8nworkflows.xyz/workflows/edit-images-with-text-prompts-using-gemini-ai-and-telegram-bot-8998


# Edit Images with Text Prompts using Gemini AI and Telegram Bot

---

### 1. Workflow Overview

This workflow implements an AI-powered Telegram bot that edits images sent by users based on accompanying text prompts. It listens for Telegram messages containing image documents and captions, downloads the image, uses Google Gemini AI to apply edits according to the prompt, and then returns the modified image back to the user on Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:**  
  Receives Telegram messages, filters them to ensure they contain both an image document and a caption prompt.

- **1.2 Image Downloading:**  
  Downloads the image file from Telegram servers as binary data, preparing it for AI processing.

- **1.3 AI Image Editing:**  
  Sends the downloaded image and the caption prompt to Google Gemini AI for creative image editing.

- **1.4 Sending Edited Image Back:**  
  Sends the AI-generated edited image back to the originating Telegram chat.

- **1.5 Documentation and Notes:**  
  Several sticky notes provide detailed context, instructions, and troubleshooting tips for users and maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  Listens for new Telegram messages with attached documents and captions, filtering out any messages missing either to avoid workflow errors.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Filter: Has Caption and File  
  - Note: Trigger1  
  - Note: Filter

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Entry point; listens for incoming Telegram updates of type "message"  
    - Configuration:  
      - Updates filtered to "message" only  
      - Does not download files at this stage (`download: false`)  
      - Uses Telegram API credential ("LinkedIn Post")  
    - Input: Webhook from Telegram  
    - Output: JSON containing message data  
    - Edge Cases:  
      - No message updates received if bot token invalid or webhook misconfigured  
      - Messages without documents or captions proceed but are filtered later  
    - Sticky Note: "Telegram Trigger" explains purpose and filtering

  - **Filter: Has Caption and File**  
    - Type: Filter node  
    - Role: Validates presence of both a caption and a document file in the Telegram message  
    - Configuration:  
      - Checks that `message.caption` exists and is non-empty  
      - Checks that `message.document.file_id` exists and is non-empty  
      - Logical AND condition requiring both to be true  
    - Input: Output from Telegram Trigger  
    - Output: Passes valid messages to next node; skips invalid ones  
    - Edge Cases:  
      - Messages missing caption or document are ignored, preventing downstream errors  
      - Expression failures if JSON structure unexpected  
    - Sticky Note: "Filter: Has Caption and File" explains logic and purpose

  - **Note: Trigger1** (Sticky Note)  
    - Purpose: Documents the Telegram Trigger node's role and filtering logic  
    - Position: Near Telegram Trigger

  - **Note: Filter** (Sticky Note)  
    - Purpose: Explains the filtering logic and rationale  
    - Position: Near Filter node

#### 2.2 Image Downloading

- **Overview:**  
  Downloads the image file from Telegram servers using the file ID extracted from the message, fetching it as binary data for AI processing.

- **Nodes Involved:**  
  - Download Image  
  - Note: Download Image

- **Node Details:**

  - **Download Image**  
    - Type: Telegram node (file download operation)  
    - Role: Downloads the image binary from Telegram using `file_id` from the message document  
    - Configuration:  
      - `fileId` set to `={{ $json.message.document.file_id }}`  
      - Resource: "file"  
      - Uses Telegram API credential ("LinkedIn Post")  
    - Input: Output of Filter node (valid messages only)  
    - Output: Binary data of the image file  
    - Edge Cases:  
      - Download failures if file expired (Telegram files expire after ~1 hour)  
      - Invalid file ID or permission issues  
      - Network or API timeout errors  
    - Sticky Note: "Download Image" explains purpose and notes file expiration risk

  - **Note: Download Image** (Sticky Note)  
    - Provides context on immediate download necessity due to file expiration  
    - Positioned near Download Image node

#### 2.3 AI Image Editing

- **Overview:**  
  Sends the downloaded image and caption prompt to Google Gemini AI to generate an edited version of the image based on the user's text prompt.

- **Nodes Involved:**  
  - Edit Image with AI  
  - Note: Edit AI

- **Node Details:**

  - **Edit Image with AI**  
    - Type: Google Gemini node (LangChain integration)  
    - Role: Performs image editing operation using Gemini AI, applying the prompt to the downloaded image  
    - Configuration:  
      - Input image passed as binary (from Download Image output)  
      - Prompt set dynamically from `={{ $('Telegram Trigger').item.json.message.caption }}`  
      - Operation: "edit" on resource "image"  
      - Output binary property named "data"  
      - Uses Google Gemini API credentials ("v9")  
    - Input: Binary image from Download Image; prompt from Telegram Trigger  
    - Output: Edited image binary data  
    - Edge Cases:  
      - API quota exceeded or authentication errors  
      - Prompt too long or invalid causing AI errors  
      - Timeout or service unavailability  
    - Sticky Note: "Edit Image with AI" explains creative editing purpose and output

  - **Note: Edit AI** (Sticky Note)  
    - Describes the AI editing functionality and creative scope  
    - Positioned near Edit Image with AI node

#### 2.4 Sending Edited Image Back

- **Overview:**  
  Sends the AI-edited image back to the Telegram chat from which the original message was received.

- **Nodes Involved:**  
  - Send Edited Image

- **Node Details:**

  - **Send Edited Image**  
    - Type: Telegram node (sendPhoto operation)  
    - Role: Sends the edited image binary back to the user chat on Telegram  
    - Configuration:  
      - Chat ID dynamically set to `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
      - Operation: sendPhoto  
      - Binary data enabled (`binaryData: true`)  
      - Uses Telegram API credential ("LinkedIn Post")  
    - Input: Edited image binary from Edit Image with AI  
    - Output: Message sent confirmation  
    - Edge Cases:  
      - Chat ID invalid or user blocked bot  
      - Binary data missing or corrupted  
      - API rate limits or network failures

---

### 3. Summary Table

| Node Name             | Node Type                     | Functional Role                              | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|-----------------------|-------------------------------|----------------------------------------------|-------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger       | Telegram Trigger               | Listens for incoming Telegram messages       | (Webhook)               | Filter: Has Caption and File | 📥 Telegram Trigger: Purpose and filtering explained                                           |
| Filter: Has Caption and File | Filter                    | Validates presence of caption and file       | Telegram Trigger        | Download Image           | 🔍 Filter: Has Caption and File: Logical AND filter explained                                  |
| Download Image        | Telegram (file download)        | Downloads image file from Telegram servers   | Filter: Has Caption and File | Edit Image with AI      | 📥 Download Image: Downloads immediately due to file expiration                                |
| Edit Image with AI     | Google Gemini AI (LangChain)   | Edits image using AI based on caption prompt | Download Image          | Send Edited Image        | 🖼️ Edit Image with AI: Creative AI image editing explained                                    |
| Send Edited Image      | Telegram (sendPhoto)            | Sends edited image back to Telegram chat     | Edit Image with AI      |                         |                                                                                                |
| Note: Trigger1         | Sticky Note                    | Documents Telegram Trigger role               |                         |                         |                                                                                                |
| Note: Filter           | Sticky Note                    | Explains filter logic                         |                         |                         |                                                                                                |
| Note: Download Image   | Sticky Note                    | Context on image download necessity          |                         |                         |                                                                                                |
| Note: Edit AI          | Sticky Note                    | Describes AI editing step                     |                         |                         |                                                                                                |
| Overview Note4         | Sticky Note                    | Full workflow overview, prerequisites & troubleshooting |                         |                         | # 🤖 AI Image Editor using Nano-Banana Telegram Bot<br>Includes setup, use cases, and troubleshooting |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: Select only "message"  
     - Additional Fields: Set `download` to false  
   - Credentials: Select or create Telegram API credential with your bot token  
   - Position: Start of the workflow

2. **Add Filter Node Named "Filter: Has Caption and File"**  
   - Type: Filter  
   - Configure conditions (AND logic):  
     - Check if expression `{{$json.message.caption}}` exists and is not empty  
     - Check if expression `{{$json.message.document.file_id}}` exists and is not empty  
   - Connect Telegram Trigger's output to this filter node

3. **Add Telegram Node Named "Download Image"**  
   - Type: Telegram node  
   - Operation: Download file  
   - Parameters:  
     - `fileId` set to expression: `{{$json.message.document.file_id}}`  
     - Resource: File  
   - Credentials: Same Telegram API credential as trigger  
   - Connect Filter node’s "true" output to this node

4. **Add Google Gemini AI Node Named "Edit Image with AI"**  
   - Type: Google Gemini (LangChain) node  
   - Resource: Image  
   - Operation: Edit  
   - Parameters:  
     - Images: Pass the binary data from "Download Image" node (usually automatic)  
     - Prompt: Set to expression: `{{$node["Telegram Trigger"].json.message.caption}}`  
     - Options: Set output binary property name to "data" (default)  
   - Credentials: Add Google Gemini API key credential  
   - Connect "Download Image" output to this node

5. **Add Telegram Node Named "Send Edited Image"**  
   - Type: Telegram node  
   - Operation: Send Photo  
   - Parameters:  
     - Chat ID: Set to expression `{{$node["Telegram Trigger"].json.message.chat.id}}`  
     - Enable Binary Data toggle  
   - Credentials: Same Telegram API credential  
   - Connect "Edit Image with AI" output to this node

6. **Add Sticky Notes for Documentation**  
   - Add notes near each logical block, documenting purpose and important considerations as described in the workflow overview and node details.

7. **Activate the Workflow**  
   - Ensure all credentials are properly configured and active  
   - Test by sending an image document with a caption to your Telegram bot

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow is branded as "Nano-Banana" Telegram bot for AI image editing. | Insight into project branding included in Overview Note4 |
| Setup instructions and troubleshooting tips are detailed in the large sticky note "Overview Note4". | Sticky note content in workflow JSON |
| Prerequisites include Telegram Bot token and Google AI Studio API key for Gemini. | See Overview Note4 in sticky notes |
| Telegram files expire after approximately 1 hour, so immediate download is necessary. | Note: Download Image sticky note |
| Google Gemini AI integration is via LangChain node, requiring valid API credentials. | Node details on "Edit Image with AI" |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---