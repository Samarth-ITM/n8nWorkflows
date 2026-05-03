Automated End-to-End Fine-Tuning of OpenAI Models with Google Drive Integration

https://n8nworkflows.xyz/workflows/automated-end-to-end-fine-tuning-of-openai-models-with-google-drive-integration-2781


# Automated End-to-End Fine-Tuning of OpenAI Models with Google Drive Integration

### 1. Workflow Overview

This workflow automates the end-to-end fine-tuning process of OpenAI language models using training data stored on Google Drive. It is designed for users who want to streamline the preparation, upload, training, and testing of custom fine-tuned models without manual API calls.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger & Data Retrieval**: Starts the workflow manually and downloads the training `.jsonl` file from Google Drive.
- **1.2 File Upload to OpenAI**: Uploads the downloaded `.jsonl` file to OpenAI’s file storage with the purpose set to `"fine-tune"`.
- **1.3 Fine-tuning Job Creation**: Initiates a fine-tuning job on OpenAI by sending a POST request with the uploaded file ID and base model.
- **1.4 Chat Interaction with Fine-tuned Model**: Listens for chat messages and routes them to the fine-tuned OpenAI model via an AI Agent for responses.
- **1.5 Documentation & Guidance**: Provides integrated sticky notes with instructions on file formatting and monitoring training progress.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger & Data Retrieval

- **Overview:**  
  This block initiates the workflow manually and downloads the training dataset from Google Drive as a `.jsonl` file.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Google Drive

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - *Type & Role:* Manual trigger node; starts the workflow on user command.  
    - *Configuration:* No parameters; triggers on manual execution.  
    - *Expressions/Variables:* None.  
    - *Connections:* Output → Google Drive node.  
    - *Version:* v1 (standard manual trigger).  
    - *Edge Cases:* User forgets to trigger; no automatic start.  
    - *Sub-workflow:* None.

  - **Google Drive**  
    - *Type & Role:* Google Drive node; downloads a file by ID.  
    - *Configuration:*  
      - Operation: `download`  
      - File ID: `1wvlEcbxFIENvqL-bACzlLEfy5gA6uF9J` (modifiable)  
      - Binary Property Name: `data.jsonl` (stores downloaded file content)  
      - Google Docs conversion set to PDF (not used here, but configured)  
    - *Expressions/Variables:* File ID is static but can be parameterized.  
    - *Connections:* Input ← Manual Trigger; Output → Upload File node.  
    - *Version:* v3 (latest Google Drive node version).  
    - *Edge Cases:*  
      - Invalid or expired Google Drive credentials (OAuth2).  
      - File not found or access denied.  
      - Large file size causing timeout.  
    - *Sub-workflow:* None.

#### 2.2 File Upload to OpenAI

- **Overview:**  
  Uploads the `.jsonl` training file to OpenAI’s file storage for fine-tuning purposes.

- **Nodes Involved:**  
  - Upload File

- **Node Details:**

  - **Upload File**  
    - *Type & Role:* OpenAI file upload node (LangChain integration).  
    - *Configuration:*  
      - Resource: `file`  
      - Purpose: `"fine-tune"` (required by OpenAI API to mark file for training)  
      - Binary Property: `data.jsonl` (from Google Drive node)  
    - *Expressions/Variables:* Uses binary data from Google Drive node.  
    - *Connections:* Input ← Google Drive; Output → Create Fine-tuning Job.  
    - *Version:* v1.8 (LangChain OpenAI node).  
    - *Edge Cases:*  
      - Invalid OpenAI API key or quota exceeded.  
      - File format errors (non-JSONL or malformed data).  
      - Network timeouts or API rate limits.  
    - *Sub-workflow:* None.

#### 2.3 Fine-tuning Job Creation

- **Overview:**  
  Sends an HTTP POST request to OpenAI’s fine-tuning jobs endpoint to start training the model using the uploaded file.

- **Nodes Involved:**  
  - Create Fine-tuning Job

- **Node Details:**

  - **Create Fine-tuning Job**  
    - *Type & Role:* HTTP Request node; calls OpenAI fine-tuning API.  
    - *Configuration:*  
      - URL: `https://api.openai.com/v1/fine_tuning/jobs`  
      - Method: POST  
      - Headers: `Content-Type: application/json` and Authorization via HTTP Header Auth credential  
      - Body (JSON):  
        ```json
        {
          "training_file": "{{ $json.id }}",
          "model": "gpt-4o-mini-2024-07-18"
        }
        ```  
      - `{{ $json.id }}` references the file ID returned by the Upload File node.  
    - *Expressions/Variables:* Uses dynamic file ID from previous node output.  
    - *Connections:* Input ← Upload File; Output: none (end of this chain).  
    - *Version:* v4.2 (latest HTTP Request node).  
    - *Edge Cases:*  
      - Invalid or expired OpenAI API key.  
      - Incorrect model name or unsupported base model.  
      - API rate limits or server errors.  
      - Missing or invalid file ID.  
    - *Sub-workflow:* None.

#### 2.4 Chat Interaction with Fine-tuned Model

- **Overview:**  
  Listens for incoming chat messages and uses the fine-tuned OpenAI model to generate responses via an AI Agent.

- **Nodes Involved:**  
  - When chat message received  
  - OpenAI Chat Model  
  - AI Agent

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* Chat trigger node; webhook that activates on incoming chat messages.  
    - *Configuration:* Default options; webhook ID `88151d03-e7f5-4c9a-8190-7cff8e849ca2`.  
    - *Expressions/Variables:* None.  
    - *Connections:* Output → AI Agent.  
    - *Version:* v1.1.  
    - *Edge Cases:*  
      - Webhook not reachable or misconfigured.  
      - Message format errors.  
    - *Sub-workflow:* None.

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node using OpenAI fine-tuned chat model.  
    - *Configuration:*  
      - Model: `ft:gpt-4o-mini-2024-07-18:n3w-italia::AsVfsl7B` (custom fine-tuned model)  
      - No additional options set.  
      - Credentials: OpenAI API key.  
    - *Expressions/Variables:* Static model name; can be updated to reflect new fine-tuned model.  
    - *Connections:* Output → AI Agent (as language model input).  
    - *Version:* v1.1.  
    - *Edge Cases:*  
      - Model not yet available if fine-tuning incomplete.  
      - Invalid or revoked API key.  
    - *Sub-workflow:* None.

  - **AI Agent**  
    - *Type & Role:* LangChain AI Agent node; orchestrates chat interaction using the specified language model.  
    - *Configuration:* Default options; uses the OpenAI Chat Model node as language model.  
    - *Expressions/Variables:* None.  
    - *Connections:* Input ← Chat Trigger; Language Model input ← OpenAI Chat Model; Output: chat responses.  
    - *Version:* v1.7.  
    - *Edge Cases:*  
      - Language model errors or timeouts.  
      - Unexpected input formats.  
    - *Sub-workflow:* None.

#### 2.5 Documentation & Guidance

- **Overview:**  
  Provides embedded instructions and references to help users prepare training data and monitor fine-tuning progress.

- **Nodes Involved:**  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**

  - **Sticky Note1**  
    - *Type & Role:* Visual note with instructions on monitoring fine-tuning progress and model availability.  
    - *Content Highlights:*  
      - Link to OpenAI fine-tuning dashboard: https://platform.openai.com/finetune/  
      - Explanation that a new model will be created and accessible via API after successful training.  
      - Example model name format.  
    - *Connections:* None (informational only).  
    - *Version:* v1.

  - **Sticky Note2**  
    - *Type & Role:* Visual note with instructions on formatting the `.jsonl` training file and uploading it to Google Drive.  
    - *Content Highlights:*  
      - Example JSON structure for training messages (system, user, assistant roles).  
      - Link to OpenAI file upload page: https://platform.openai.com/storage/files  
    - *Connections:* None (informational only).  
    - *Version:* v1.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                          | Input Node(s)              | Output Node(s)            | Sticky Note                                                                                              |
|---------------------------|----------------------------------|----------------------------------------|---------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts workflow manually                | —                         | Google Drive              |                                                                                                        |
| Google Drive              | Google Drive                     | Downloads training `.jsonl` file        | When clicking ‘Test workflow’ | Upload File               |                                                                                                        |
| Upload File               | OpenAI File Upload (LangChain)  | Uploads `.jsonl` to OpenAI for fine-tune | Google Drive              | Create Fine-tuning Job    |                                                                                                        |
| Create Fine-tuning Job    | HTTP Request                    | Creates fine-tuning job on OpenAI       | Upload File               | —                         |                                                                                                        |
| When chat message received | Chat Trigger (LangChain)         | Listens for incoming chat messages      | —                         | AI Agent                  |                                                                                                        |
| OpenAI Chat Model         | OpenAI Chat Model (LangChain)    | Uses fine-tuned model for chat responses | —                         | AI Agent (language model) |                                                                                                        |
| AI Agent                  | LangChain AI Agent               | Orchestrates chat interaction            | When chat message received, OpenAI Chat Model | —                         |                                                                                                        |
| Sticky Note1              | Sticky Note                     | Instructions on monitoring fine-tuning | —                         | —                         | Once the .jsonl file is uploaded, a new model will be created and accessible via API. See: https://platform.openai.com/finetune/ |
| Sticky Note2              | Sticky Note                     | Instructions on preparing `.jsonl` file | —                         | —                         | Create the training file `.jsonl` with the specified syntax and upload it to Drive. See: https://platform.openai.com/storage/files |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.  
   - No parameters needed.

2. **Add Google Drive Node**  
   - Add a **Google Drive** node named `Google Drive`.  
   - Set operation to `download`.  
   - Enter the file ID of your `.jsonl` training file (e.g., `1wvlEcbxFIENvqL-bACzlLEfy5gA6uF9J`).  
   - Set binary property name to `data.jsonl`.  
   - Configure Google Drive OAuth2 credentials with your Google account.  
   - Connect output of Manual Trigger → input of Google Drive node.

3. **Add OpenAI File Upload Node**  
   - Add an **OpenAI File Upload** node (LangChain OpenAI node) named `Upload File`.  
   - Set resource to `file`.  
   - Set purpose to `"fine-tune"`.  
   - Set binary property name to `data.jsonl` (to receive file content from Google Drive).  
   - Configure OpenAI API credentials with your API key.  
   - Connect output of Google Drive → input of Upload File node.

4. **Add HTTP Request Node to Create Fine-tuning Job**  
   - Add an **HTTP Request** node named `Create Fine-tuning Job`.  
   - Set method to `POST`.  
   - Set URL to `https://api.openai.com/v1/fine_tuning/jobs`.  
   - Set authentication to HTTP Header Auth with OpenAI API key.  
   - Add header: `Content-Type: application/json`.  
   - Set body type to JSON and enter:  
     ```json
     {
       "training_file": "{{ $json.id }}",
       "model": "gpt-4o-mini-2024-07-18"
     }
     ```  
   - Connect output of Upload File → input of Create Fine-tuning Job node.

5. **Add Chat Trigger Node**  
   - Add a **Chat Trigger** node named `When chat message received`.  
   - Configure webhook (auto-generated or custom).  
   - No additional parameters needed.

6. **Add OpenAI Chat Model Node**  
   - Add an **OpenAI Chat Model** node (LangChain) named `OpenAI Chat Model`.  
   - Set model to your fine-tuned model name (e.g., `ft:gpt-4o-mini-2024-07-18:n3w-italia::AsVfsl7B`).  
   - Configure OpenAI API credentials.  
   - No extra options needed.

7. **Add AI Agent Node**  
   - Add an **AI Agent** node (LangChain) named `AI Agent`.  
   - Connect the `When chat message received` node output to AI Agent input.  
   - Connect the `OpenAI Chat Model` node output to AI Agent’s language model input.

8. **Add Sticky Notes (Optional but Recommended)**  
   - Add two **Sticky Note** nodes with the following content:  
     - Sticky Note 1: Instructions on `.jsonl` file creation and upload to Google Drive.  
     - Sticky Note 2: Instructions on monitoring fine-tuning progress on OpenAI platform with link: https://platform.openai.com/finetune/.

9. **Test the Workflow**  
   - Trigger the workflow manually via `When clicking ‘Test workflow’`.  
   - Confirm the `.jsonl` file downloads, uploads, and fine-tuning job creation succeed.  
   - Use the chat webhook to send messages and receive responses from the fine-tuned model.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The `.jsonl` training file must follow OpenAI’s required format with `messages` array containing roles.   | See Sticky Note2 content and https://platform.openai.com/storage/files |
| Monitor fine-tuning job status and model availability at OpenAI’s dashboard.                              | https://platform.openai.com/finetune/                |
| The fine-tuned model name format is typically: `ft:<base-model>:<organization>::<suffix>`                 | Example: `ft:gpt-4o-mini-2024-07-18:n3w-italia::XXXXX7B` |
| Ensure valid OAuth2 credentials for Google Drive and valid API keys for OpenAI before running the workflow.| Credential setup section in workflow description.    |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the fine-tuning workflow integrating Google Drive and OpenAI services.