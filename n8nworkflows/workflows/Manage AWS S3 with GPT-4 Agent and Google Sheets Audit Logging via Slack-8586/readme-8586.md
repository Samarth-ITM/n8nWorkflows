Manage AWS S3 with GPT-4 Agent and Google Sheets Audit Logging via Slack

https://n8nworkflows.xyz/workflows/manage-aws-s3-with-gpt-4-agent-and-google-sheets-audit-logging-via-slack-8586


# Manage AWS S3 with GPT-4 Agent and Google Sheets Audit Logging via Slack

### 1. Workflow Overview

This workflow enables natural language management of AWS S3 resources via chat platforms such as Slack or Telegram. Users send chat messages to trigger AWS S3 operations like listing buckets, managing files and folders, copying or deleting files, and creating folders, without direct AWS console or CLI access. Each operation is automatically logged to a Google Sheets audit log for compliance and traceability.

The workflow is logically divided into these main blocks:

- **1.1 Input Reception:** Listens for incoming chat messages from Slack, Telegram, or similar chat platforms.
- **1.2 AI Processing:** Uses an OpenAI GPT-4 powered AI Agent with memory to interpret user messages, determine intents, and orchestrate corresponding AWS S3 operations.
- **1.3 AWS S3 Operations:** Executes specific AWS S3 actions such as listing buckets, listing files, copying files, deleting files, listing folders, and creating folders.
- **1.4 Audit Logging:** After every AWS S3 operation, logs detailed information about the action to a Google Sheets document for audit purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for chat messages from integrated platforms and triggers the workflow.

- **Nodes Involved:**  
  - When chat message received

- **Node Details:**

  - **When chat message received**  
    - *Type & Role:* `chatTrigger` node; webhook trigger for incoming chat messages.  
    - *Configuration:* Default options; exposes a webhook endpoint for Slack, Telegram, or any chat integration to post messages.  
    - *Key Expressions:* None.  
    - *Connections:* Output connected to `AWS S3 Manager Agent`.  
    - *Version Requirements:* n8n version supporting Langchain chat triggers (≥1.3).  
    - *Edge Cases:* Webhook misconfiguration, message format errors, unauthorized sources.  
    - *Sub-Workflow:* None.

---

#### 2.2 AI Processing

- **Overview:**  
  Uses OpenAI GPT-4 based language model and memory buffer to parse natural language, select appropriate AWS S3 tool calls, and manage conversation context.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Simple Memory  
  - AWS S3 Manager Agent

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type & Role:* Language model node using OpenAI GPT-4 mini variant.  
    - *Configuration:* Model set to `gpt-4.1-mini`, no additional options.  
    - *Credentials:* Uses configured OpenAI API key.  
    - *Connections:* Output connected to `AWS S3 Manager Agent` as language model input.  
    - *Version Requirements:* n8n Langchain integration version 1.2+.  
    - *Edge Cases:* API rate limits, network issues, invalid prompts.

  - **Simple Memory**  
    - *Type & Role:* Memory buffer to maintain conversation context (last 10 interactions).  
    - *Configuration:* Context window length of 10 messages.  
    - *Connections:* Memory linked into the `AWS S3 Manager Agent`.  
    - *Version Requirements:* n8n Langchain Memory node version 1.3+.  
    - *Edge Cases:* Memory overflow, data consistency.

  - **AWS S3 Manager Agent**  
    - *Type & Role:* Langchain agent node orchestrating AI-driven tool calls for AWS S3 operations.  
    - *Configuration:*  
      - System message instructs it to interpret user intent and call one of 7 tools: ListBuckets, ListObjects, CopyObject, DeleteObject, ListFolders, CreateFolder, AddAuditLog.  
      - Enforces calling `AddAuditLog` immediately after any operational tool call (1-6), except when the tool itself is `AddAuditLog` to avoid recursion.  
      - Passes reasoning, user info, timestamps, and original chat prompts for logging.  
    - *Connections:*  
      - Input: receives chat trigger, OpenAI model, and memory input.  
      - Output: calls specific AWS S3 nodes and Google Sheets node as AI tools.  
    - *Version Requirements:* Langchain agent node version 2.2+.  
    - *Edge Cases:* Misinterpretation of user intent, invalid parameters, infinite loops if AddAuditLog called incorrectly.

---

#### 2.3 AWS S3 Operations

- **Overview:**  
  Executes specific AWS S3 API actions based on agent decisions, including managing buckets, files, and folders.

- **Nodes Involved:**  
  - Get many buckets in AWS S3  
  - Get many files in AWS S3  
  - Copy a file in AWS S3  
  - Delete a file in AWS S3  
  - Get many folders in AWS S3  
  - Create a folder in AWS S3

- **Node Details:**

  - **Get many buckets in AWS S3**  
    - *Type & Role:* AWS S3 node to list all S3 buckets.  
    - *Configuration:* Resource: bucket; Operation: getAll; ReturnAll toggled via AI expression.  
    - *Credentials:* AWS account with S3 read access.  
    - *Connections:* Output returns data to agent as tool result.  
    - *Edge Cases:* AWS permission errors, API limits.

  - **Get many files in AWS S3**  
    - *Type & Role:* AWS S3 node to list files in a bucket or prefix.  
    - *Configuration:* Operation: getAll; Bucket name dynamically set from AI parameters; ReturnAll dynamic.  
    - *Connections:* Output to agent.  
    - *Edge Cases:* Invalid bucket names, empty buckets.

  - **Copy a file in AWS S3**  
    - *Type & Role:* AWS S3 node to copy a file from source to destination path.  
    - *Configuration:* Operation: copy; SourcePath and DestinationPath dynamically set via AI expressions.  
    - *Connections:* Output to agent.  
    - *Edge Cases:* Missing source file, permission denied, invalid paths.

  - **Delete a file in AWS S3**  
    - *Type & Role:* AWS S3 node to delete a specified file.  
    - *Configuration:* Operation: delete; BucketName and FileKey dynamically set via AI expressions.  
    - *Connections:* Output to agent.  
    - *Edge Cases:* File not found, permission denied.

  - **Get many folders in AWS S3**  
    - *Type & Role:* AWS S3 node to list folders using prefixes.  
    - *Configuration:* Resource: folder; Operation: getAll; BucketName dynamic.  
    - *Connections:* Output to agent.  
    - *Edge Cases:* No folders, incorrect bucket.

  - **Create a folder in AWS S3**  
    - *Type & Role:* AWS S3 node to create a zero-byte object with trailing slash to represent a folder.  
    - *Configuration:* Resource: folder; BucketName and FolderName dynamic.  
    - *Connections:* Output to agent.  
    - *Edge Cases:* Folder already exists, invalid folder names.

---

#### 2.4 Audit Logging

- **Overview:**  
  Logs every S3 operation call (except the logging action itself) with full context into a Google Sheets document for audit and compliance purposes.

- **Nodes Involved:**  
  - Append or update row in sheet in Google Sheets

- **Node Details:**

  - **Append or update row in sheet in Google Sheets**  
    - *Type & Role:* Google Sheets node to append or update audit log rows.  
    - *Configuration:*  
      - Operation: appendOrUpdate  
      - Document ID and Sheet Name statically configured to a specific Google Sheet (`AWS S3 Audit Logs`).  
      - Columns auto-mapped from input data, expecting keys like timestamp, tool, status, chat_prompt, parameters, user_name, tool_call_reasoning.  
    - *Credentials:* Google OAuth2 credentials with write access to the target Sheet.  
    - *Connections:* Called by the AI agent as `AddAuditLog` tool.  
    - *Edge Cases:* Permission errors, sheet locked, rate limits.

---

### 3. Summary Table

| Node Name                          | Node Type                           | Functional Role                    | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                   |
|-----------------------------------|-----------------------------------|----------------------------------|------------------------------|-----------------------------|-----------------------------------------------------------------------------------------------|
| When chat message received         | chatTrigger                       | Input trigger for incoming chat  |                              | AWS S3 Manager Agent         | ### **Webhook Trigger** Slack, Telegram, or custom chat platform → connects to n8n.          |
| OpenAI Chat Model                  | lmChatOpenAi                     | Language model for NLP processing|                              | AWS S3 Manager Agent         | ### **OpenAI Agent** - Model: `gpt-4` or `gpt-3.5-turbo` - Memory: Simple Memory Node - Prompt: instructs agent to log actions. |
| Simple Memory                     | memoryBufferWindow                | Maintains conversation context   |                              | AWS S3 Manager Agent         |                                                                                               |
| AWS S3 Manager Agent              | agent                            | AI orchestrator and tool caller  | When chat message received, OpenAI Chat Model, Simple Memory | AWS S3 nodes, Google Sheets node |                                                                                               |
| Get many buckets in AWS S3        | awsS3Tool                       | Lists all S3 buckets             | AWS S3 Manager Agent (ai_tool)| AWS S3 Manager Agent          | ### **AWS S3 Nodes** Configure each tool with AWS credentials. Tools: getAll bucket, file, copy, delete, folder, create folder |
| Get many files in AWS S3           | awsS3Tool                       | Lists files in bucket            | AWS S3 Manager Agent (ai_tool)| AWS S3 Manager Agent          |                                                                                               |
| Copy a file in AWS S3              | awsS3Tool                       | Copies file within S3            | AWS S3 Manager Agent (ai_tool)| AWS S3 Manager Agent          |                                                                                               |
| Delete a file in AWS S3            | awsS3Tool                       | Deletes file from S3             | AWS S3 Manager Agent (ai_tool)| AWS S3 Manager Agent          |                                                                                               |
| Get many folders in AWS S3         | awsS3Tool                       | Lists folders in bucket          | AWS S3 Manager Agent (ai_tool)| AWS S3 Manager Agent          |                                                                                               |
| Create a folder in AWS S3          | awsS3Tool                       | Creates new folder in bucket     | AWS S3 Manager Agent (ai_tool)| AWS S3 Manager Agent          |                                                                                               |
| Append or update row in sheet in Google Sheets | googleSheetsTool               | Logs audit info into Google Sheets| AWS S3 Manager Agent (ai_tool)|                             | ### **Google Sheets Node** Sheet: `AWS S3 Audit Logs` Operation: `Append or Update Row`. Columns must match input keys.          |
| Sticky Note                      | stickyNote                      | Documentation and instructions   |                              |                             | # AI-Powered AWS S3 Manager with Audit Logging in n8n (Slack/ChatOps Workflow) - Detailed overview and setup instructions.        |
| Sticky Note1                     | stickyNote                      | Documentation summary            |                              |                             | ### **Webhook Trigger** Slack, Telegram, or custom chat platform → connects to n8n.          |
| Sticky Note2                     | stickyNote                      | Documentation summary            |                              |                             | ### **OpenAI Agent** - Model: `gpt-4` or `gpt-3.5-turbo` - Memory: Simple Memory Node - Prompt: instructs agent to log actions.     |
| Sticky Note3                     | stickyNote                      | Documentation summary            |                              |                             | ### **AWS S3 Nodes** Configure each tool with AWS credentials. Tools: getAll bucket, file, copy, delete, folder, create folder     |
| Sticky Note4                     | stickyNote                      | Documentation summary            |                              |                             | ### **Google Sheets Node** Sheet: `AWS S3 Audit Logs` Operation: `Append or Update Row`. Columns must match input keys.            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger Node:**  
   - Add a `chatTrigger` node named `When chat message received`.  
   - Set it up to receive chat messages from Slack, Telegram, or your chat platform via webhook.  
   - Save webhook URL for integration.

2. **Add OpenAI Chat Model Node:**  
   - Add an `lmChatOpenAi` node named `OpenAI Chat Model`.  
   - Configure with OpenAI API credentials.  
   - Set model to `gpt-4.1-mini` (or `gpt-4`/`gpt-3.5-turbo`).  
   - No special options needed.

3. **Add Simple Memory Node:**  
   - Add a `memoryBufferWindow` node named `Simple Memory`.  
   - Set context window length to 10 messages.

4. **Create the AWS S3 Manager Agent Node:**  
   - Add an `agent` node named `AWS S3 Manager Agent`.  
   - Configure with the following system message instructions for the agent:  
     - Manage AWS S3 with 7 tools: ListBuckets, ListObjects, CopyObject, DeleteObject, ListFolders, CreateFolder, AddAuditLog.  
     - Always call `AddAuditLog` immediately after any S3 tool call except `AddAuditLog` itself.  
     - Log timestamp, tool, parameters, status, user info, chat prompt, and reasoning.  
   - Connect `When chat message received` main output to this node’s main input.  
   - Connect `OpenAI Chat Model` output to this node’s language model input.  
   - Connect `Simple Memory` output to this node’s memory input.

5. **Configure AWS S3 Operation Nodes:**  
   For all following nodes, use AWS credentials with S3 permissions:

   - **Get many buckets in AWS S3:**  
     - Type: `awsS3Tool`  
     - Resource: bucket  
     - Operation: getAll  
     - ReturnAll: dynamic boolean (allow AI override)  
     - Connect as AI tool to `AWS S3 Manager Agent`.

   - **Get many files in AWS S3:**  
     - Type: `awsS3Tool`  
     - Operation: getAll  
     - BucketName: dynamic string from AI input  
     - ReturnAll: dynamic boolean  
     - Connect as AI tool to `AWS S3 Manager Agent`.

   - **Copy a file in AWS S3:**  
     - Type: `awsS3Tool`  
     - Operation: copy  
     - SourcePath: dynamic string from AI input  
     - DestinationPath: dynamic string from AI input  
     - Connect as AI tool to `AWS S3 Manager Agent`.

   - **Delete a file in AWS S3:**  
     - Type: `awsS3Tool`  
     - Operation: delete  
     - BucketName & FileKey: dynamic strings from AI input  
     - Connect as AI tool to `AWS S3 Manager Agent`.

   - **Get many folders in AWS S3:**  
     - Type: `awsS3Tool`  
     - Resource: folder  
     - Operation: getAll  
     - BucketName: dynamic from AI input  
     - Connect as AI tool to `AWS S3 Manager Agent`.

   - **Create a folder in AWS S3:**  
     - Type: `awsS3Tool`  
     - Resource: folder  
     - BucketName & FolderName: dynamic from AI input  
     - Connect as AI tool to `AWS S3 Manager Agent`.

6. **Configure Google Sheets Audit Log Node:**  
   - Add a `googleSheetsTool` node named `Append or update row in sheet in Google Sheets`.  
   - Operation: `appendOrUpdate`  
   - Document ID: set to your Google Sheet ID for audit logs.  
   - Sheet Name: set to the specific sheet, e.g., `gid=0` or `Sheet1`.  
   - Map columns to match expected input keys: `timestamp`, `tool`, `status`, `chat_prompt`, `parameters`, `user_name`, `tool_call_reasoning`.  
   - Connect as AI tool to `AWS S3 Manager Agent`.

7. **Connect AI Tool Outputs:**  
   - Ensure each AWS S3 node and the Google Sheets node is connected as an AI tool for the `AWS S3 Manager Agent` node.  
   - This allows the agent to call these tools dynamically based on user input.

8. **Credential Setup:**  
   - Configure AWS credentials with IAM permissions allowing S3 bucket and object operations.  
   - Configure OpenAI API credentials with valid key and permissions.  
   - Configure Google OAuth2 credentials with access to the target Google Sheet.

9. **Test the Workflow:**  
   - Send chat messages like "List all buckets" or "Copy file X from bucket A to bucket B" via your chat platform.  
   - Verify AWS operations execute and audit logs append correctly.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow enables ChatOps for AWS S3 via natural language, ideal for DevOps and support teams requiring auditability.                 | Workflow overview in Sticky Note.                                                                   |
| Requires n8n instance with Langchain AI Agent support and proper API credentials for OpenAI, AWS, and Google Sheets.                     | Setup prerequisites in Sticky Note.                                                                |
| Audit logs include timestamp, user identity, tool used, parameters, and reasoning for compliance and traceability.                      | Detailed in AWS S3 Manager Agent system message instructions.                                       |
| Customize by adding IP logging, latency tracking, or Slack notifications on sensitive actions like file deletion.                        | Suggestions in main Sticky Note under "How to customize the workflow."                              |
| Google Sheet used for logging is named `AWS S3 Audit Logs`, ensure columns match expected keys for correct logging.                      | Google Sheets Node configuration details.                                                           |
| Official n8n documentation for AWS S3 node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.awsS3Tool/                | Useful for understanding AWS S3 node configurations.                                               |
| OpenAI API docs: https://platform.openai.com/docs/models/gpt-4                                                                           | Reference for language model capabilities and limits.                                              |
| Google Sheets API documentation: https://developers.google.com/sheets/api                                                               | Useful for advanced Google Sheets operations and troubleshooting.                                  |

---

**Disclaimer:** The provided information is extracted exclusively from the analyzed n8n workflow JSON. It complies with current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.