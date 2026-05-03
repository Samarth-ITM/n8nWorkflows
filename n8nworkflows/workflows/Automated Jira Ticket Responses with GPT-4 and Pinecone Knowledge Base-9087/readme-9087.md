Automated Jira Ticket Responses with GPT-4 and Pinecone Knowledge Base

https://n8nworkflows.xyz/workflows/automated-jira-ticket-responses-with-gpt-4-and-pinecone-knowledge-base-9087


# Automated Jira Ticket Responses with GPT-4 and Pinecone Knowledge Base

### 1. Workflow Overview

This workflow automates Jira ticket responses by leveraging GPT-4 AI and a Pinecone vector-based knowledge base. It targets project teams using Jira to handle assigned issues, aiming to generate intelligent, context-aware comments and update ticket assignees automatically. The workflow is structured into four logical blocks:

- **1.1 Scheduled Trigger & Issue Retrieval:** Periodically fetch Jira issues assigned to a specific user.
- **1.2 Issue Detail Extraction & Validation:** Retrieve detailed issue comments and metadata, then verify assignee identity.
- **1.3 AI Processing with Knowledge Retrieval:** Use GPT-4 combined with Pinecone vector search to generate an informed response based on issue summary, description, and comment history.
- **1.4 Jira Ticket Update:** Post AI-generated comments back to Jira and update the ticket assignee accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Issue Retrieval

**Overview:**  
This block initiates the workflow on a schedule and retrieves all Jira issues assigned to a specified account. It batches the issues for downstream processing.

**Nodes Involved:**  
- Schedule Trigger  
- Jira Issue List  
- Loop Over Items

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow at defined intervals (default unspecified interval).  
  - *Configuration:* Uses default scheduling options (interval empty array implies continuous or default frequency).  
  - *Inputs:* None  
  - *Outputs:* Jira Issue List  
  - *Failures:* Possible misconfiguration of schedule or trigger execution delays.

- **Jira Issue List**  
  - *Type:* Jira Node (Get All Issues)  
  - *Role:* Retrieves all Jira issues assigned to a specific user via JQL query `"assignee = ＜アカウントIDを入力してください＞"`.  
  - *Configuration:* Requires a valid Jira account ID to filter issues.  
  - *Inputs:* Schedule Trigger  
  - *Outputs:* Loop Over Items  
  - *Failures:* Auth errors (invalid Jira credentials), empty or invalid JQL, network timeouts.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes issues in batches for controlled downstream processing.  
  - *Configuration:* No batch size specified (defaults apply).  
  - *Inputs:* Jira Issue List  
  - *Outputs:* Two outputs — (1) empty, (2) to Code1 node.  
  - *Failures:* Potential batch size misconfiguration causing performance issues.

---

#### 1.2 Issue Detail Extraction & Validation

**Overview:**  
Fetches detailed comments for each Jira issue and extracts key fields such as summary, description, comment history, and assignee accountId. It checks if the issue is assigned to the expected user before proceeding.

**Nodes Involved:**  
- Jira Issue Detail  
- Code1  
- Code2  
- check accountId

**Node Details:**

- **Jira Issue Detail**  
  - *Type:* Jira Node (Get All Comments)  
  - *Role:* Retrieves all comments for the Jira issue identified by its ID.  
  - *Configuration:* Uses the issue ID from the Loop Over Items node. Limits to 5 comments, resource set to issueComment.  
  - *Inputs:* Loop Over Items → Code1  
  - *Outputs:* Code2  
  - *Failures:* API rate limits, missing issue keys, network errors.

- **Code1**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Extracts and returns the issue ID, summary, and description from the first item in batch.  
  - *Configuration:* Custom JS code accesses JSON fields `id`, `fields.summary`, and `fields.description`.  
  - *Inputs:* Loop Over Items (2nd output)  
  - *Outputs:* Jira Issue Detail  
  - *Failures:* Data field missing or malformed JSON.

- **Code2**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Aggregates comment history text and extracts the latest comment’s author accountId.  
  - *Configuration:* Iterates over all comment items, concatenates updated timestamps and comment text, extracts `updateAuthor.accountId`.  
  - *Inputs:* Jira Issue Detail  
  - *Outputs:* check accountId  
  - *Failures:* Unexpected comment structure, empty comments array.

- **check accountId**  
  - *Type:* If Node  
  - *Role:* Checks if extracted accountId differs from a placeholder accountId `"＜アカウントIDを入力してください＞"`. If true, proceeds; else loops back.  
  - *Configuration:* String not-equals condition on accountId extracted in Code2.  
  - *Inputs:* Code2  
  - *Outputs:* True → AI Agent, False → Loop Over Items  
  - *Failures:* Incorrect comparison string, false negatives leading to skipping valid issues.

---

#### 1.3 AI Processing with Knowledge Retrieval

**Overview:**  
This core block uses LangChain’s AI Agent with GPT-4 to generate a context-aware comment. It integrates Pinecone vector store to retrieve relevant Jira knowledge and OpenAI embeddings to support the vector search.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Pinecone Vector Store  
- Embeddings OpenAI

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent Node  
  - *Role:* Generates Jira response comments using GPT-4 and Pinecone knowledge base.  
  - *Configuration:*  
    - Prompt includes Jira issue title (`summary`), description, and concatenated comment history.  
    - System message sets AI role as project developer using past knowledge.  
    - Output is formatted to be directly posted as Jira comments, including knowledge base IDs.  
  - *Inputs:* check accountId (true branch)  
  - *Outputs:* Jira Add Comment  
  - *Inner Connections:*  
    - Uses OpenAI Chat Model as language model.  
    - Uses Pinecone Vector Store as tool for knowledge retrieval.  
  - *Failures:* API limits (OpenAI, Pinecone), malformed prompt data, vector store access issues.

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model Node  
  - *Role:* Provides GPT-4.1-mini model for AI Agent language generation.  
  - *Configuration:* Model set to `"gpt-4.1-mini"`. No extra options specified.  
  - *Inputs:* AI Agent (ai_languageModel)  
  - *Outputs:* AI Agent  
  - *Failures:* Invalid API key, model unavailability, rate limits.

- **Pinecone Vector Store**  
  - *Type:* LangChain Vector Store Node  
  - *Role:* Retrieves relevant Jira knowledge documents from Pinecone vector database as a tool for AI Agent.  
  - *Configuration:*  
    - Mode: `"retrieve-as-tool"`  
    - Pinecone Index: placeholder `<Pinecone Indexを入力>` (must be replaced with actual index)  
    - Tool description: `"jira情報"` (Jira Information)  
  - *Inputs:* Embeddings OpenAI (ai_embedding)  
  - *Outputs:* AI Agent (ai_tool)  
  - *Failures:* Pinecone auth errors, missing index, network failures.

- **Embeddings OpenAI**  
  - *Type:* LangChain OpenAI Embeddings Node  
  - *Role:* Generates 1536-dimensional embeddings for text queries to support Pinecone retrieval.  
  - *Configuration:* Dimensions set to 1536 (default for OpenAI).  
  - *Inputs:* None directly (used internally by Vector Store).  
  - *Outputs:* Pinecone Vector Store (ai_embedding)  
  - *Failures:* API key errors, embedding generation failures.

---

#### 1.4 Jira Ticket Update

**Overview:**  
Posts the AI-generated comment back to the Jira issue and updates the assignee field to the accountId extracted from comments.

**Nodes Involved:**  
- Jira Add Comment  
- Jira Assign

**Node Details:**

- **Jira Add Comment**  
  - *Type:* Jira Node (Add Comment)  
  - *Role:* Adds AI Agent’s generated text as a comment on the Jira issue.  
  - *Configuration:*  
    - Comment content comes from AI Agent’s output field `output`.  
    - Issue key from Code1’s `id` field.  
  - *Inputs:* AI Agent  
  - *Outputs:* Jira Assign  
  - *Failures:* Permission denied, invalid issue key, comment format errors.

- **Jira Assign**  
  - *Type:* Jira Node (Update Issue)  
  - *Role:* Updates the assignee of the Jira issue to the latest comment author’s accountId.  
  - *Configuration:*  
    - Issue key from Code1’s `id`.  
    - Assignee field set dynamically from Code2’s extracted `accountId`.  
  - *Inputs:* Jira Add Comment  
  - *Outputs:* Loop Over Items (to process next issue)  
  - *Failures:* Invalid accountId, permission issues, Jira API errors.

---

### 3. Summary Table

| Node Name           | Node Type                                 | Functional Role                         | Input Node(s)           | Output Node(s)          | Sticky Note                                      |
|---------------------|-------------------------------------------|---------------------------------------|-------------------------|-------------------------|-------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                          | Starts workflow on schedule            | None                    | Jira Issue List         | Step1 - Scheduled Execution                      |
| Jira Issue List      | Jira Node (Get All Issues)                | Retrieves assigned Jira issues         | Schedule Trigger        | Loop Over Items         | Step2 - Jira Trigger (Issue Assigned)            |
| Loop Over Items      | Split In Batches                          | Batch processes Jira issues            | Jira Issue List         | Code1 (2nd output), empty| Step2 - Jira Trigger (Issue Assigned)           |
| Code1               | Code Node (JS)                            | Extracts issue id, summary, description| Loop Over Items (2nd out)| Jira Issue Detail       | Step2 - Jira Trigger (Issue Assigned)           |
| Jira Issue Detail    | Jira Node (Get All Comments)              | Retrieves comments for issue           | Code1                   | Code2                   | Step2 - Jira Trigger (Issue Assigned)           |
| Code2               | Code Node (JS)                            | Aggregates comments, extracts accountId| Jira Issue Detail        | check accountId         | Step2 - Jira Trigger (Issue Assigned)           |
| check accountId     | If Node                                  | Validates assignee accountId           | Code2                   | True → AI Agent, False → Loop Over Items | Step2 - Jira Trigger (Issue Assigned)           |
| AI Agent            | LangChain Agent Node                      | Generates Jira response comment        | check accountId (true)  | Jira Add Comment        | Step3 - AI Assistant                             |
| OpenAI Chat Model    | LangChain OpenAI Chat Model               | Provides GPT-4.1-mini model            | AI Agent (ai_languageModel) | AI Agent            | Step3 - AI Assistant                             |
| Pinecone Vector Store| LangChain Vector Store (Pinecone)         | Retrieves knowledge base vectors       | Embeddings OpenAI       | AI Agent (ai_tool)      | Step3 - AI Assistant                             |
| Embeddings OpenAI   | LangChain OpenAI Embeddings                | Generates text embeddings               | None (internal use)     | Pinecone Vector Store   | Step3 - AI Assistant                             |
| Jira Add Comment    | Jira Node (Add Comment)                    | Posts AI-generated comment to Jira     | AI Agent                | Jira Assign             | Step4 - Response Generation / Ticket Update     |
| Jira Assign        | Jira Node (Update Issue - Assignee)       | Updates ticket assignee                 | Jira Add Comment        | Loop Over Items         | Step4 - Response Generation / Ticket Update     |
| Sticky Note         | Sticky Note                               | Visual annotation                       | None                    | None                    | Step1 - Scheduled Execution                      |
| Sticky Note1        | Sticky Note                               | Visual annotation                       | None                    | None                    | Step2 - Jira Trigger (Issue Assigned)            |
| Sticky Note2        | Sticky Note                               | Visual annotation                       | None                    | None                    | Step3 - AI Assistant                             |
| Sticky Note3        | Sticky Note                               | Visual annotation                       | None                    | None                    | Step4 - Response Generation / Ticket Update     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure interval as desired (e.g., every 1 hour).  
   - This node starts the workflow periodically.

2. **Add Jira Issue List node:**  
   - Type: Jira (Get All Issues)  
   - Connect input from Schedule Trigger.  
   - Configure operation: `getAll`  
   - Set JQL query: `assignee = ＜アカウントIDを入力してください＞` (replace placeholder with actual Jira account ID).  
   - Configure Jira credentials with OAuth2 or API token.

3. **Add Loop Over Items node:**  
   - Type: Split In Batches  
   - Connect input from Jira Issue List.  
   - No batch size specified (default).  
   - This enables processing of multiple issues sequentially.

4. **Add Code1 node:**  
   - Type: Code (JavaScript)  
   - Connect from Loop Over Items (2nd output).  
   - Paste code to extract issue ID, summary, and description from first item:  
     ```js
     return {
       'id': $input.first().json.id,
       'summary': $input.first().json.fields.summary,
       'description': $input.first().json.fields.description,
     }
     ```
   - Set `executeOnce` to true.

5. **Add Jira Issue Detail node:**  
   - Type: Jira (Get All Comments)  
   - Connect input from Code1.  
   - Configure operation: `getAll`  
   - Resource: `issueComment`  
   - Set Issue Key: `={{ $json.id }}` (from Code1 output).  
   - Limit: 5 comments.

6. **Add Code2 node:**  
   - Type: Code (JavaScript)  
   - Connect input from Jira Issue Detail.  
   - Paste code to concatenate comments and extract last comment author accountId:  
     ```js
     let comments = '';
     let accountId = '';
     for (const item of $input.all()) {
       comments += '## ' + item.json.updated + '\n';
       comments += item.json.body.content[0].content[0].text + '\n\n';
       accountId = item.json.updateAuthor.accountId;
     }
     return {
       'comments': comments,
       'accountId': accountId
     };
     ```

7. **Add If node (check accountId):**  
   - Type: If  
   - Connect input from Code2.  
   - Condition:  
     - Left Value: `={{ $json.accountId }}`  
     - Operator: not equals  
     - Right Value: `＜アカウントIDを入力してください＞` (replace with actual accountId).  
   - True output leads to AI Agent; False loops back to Loop Over Items (to process next issue).

8. **Add LangChain AI Agent node:**  
   - Type: LangChain Agent Node  
   - Connect input from check accountId (true output).  
   - Configure prompt (text) using expressions:  
     ```
     =# タイトル
     {{ $('Code1').first().json.summary }}

     # 説明
     {{ $('Code1').first().json.description }}

     # コメント履歴
     {{ $json.comments }}
     ```  
   - System message:  
     ```
     あなたはプロジェクトの開発者です。タスクがアサインされました。過去の知見を利用して対応をしてください。outputの内容をそのままJiraのコメント欄に記載します。また関連する過去の知見はIDも合わせて回答してください。
     ```  
   - Prompt type: Define  
   - Set AI language model node and AI tool node connections as below.

9. **Add LangChain OpenAI Chat Model node:**  
   - Type: LangChain OpenAI Chat Model  
   - Connect input to AI Agent (ai_languageModel).  
   - Configure model as `"gpt-4.1-mini"`.  
   - Provide OpenAI credentials.

10. **Add LangChain OpenAI Embeddings node:**  
    - Type: LangChain OpenAI Embeddings  
    - Set dimensions: 1536.  
    - Connect output to Pinecone Vector Store (ai_embedding).  
    - Provide OpenAI credentials.

11. **Add LangChain Pinecone Vector Store node:**  
    - Type: LangChain Vector Store (Pinecone)  
    - Connect input from Embeddings OpenAI (ai_embedding).  
    - Configure mode: `retrieve-as-tool`.  
    - Set Pinecone Index: Replace `<Pinecone Indexを入力>` with actual Pinecone index name.  
    - Set tool description: `"jira情報"`.  
    - Connect output to AI Agent (ai_tool).  
    - Provide Pinecone API credentials.

12. **Add Jira Add Comment node:**  
    - Type: Jira (Add Comment)  
    - Connect input from AI Agent.  
    - Configure:  
      - Comment: `={{ $json.output }}` (AI Agent output).  
      - Issue Key: `={{ $('Code1').first().json.id }}`.  
    - Jira credentials required.

13. **Add Jira Assign node:**  
    - Type: Jira (Update Issue)  
    - Connect input from Jira Add Comment.  
    - Configure:  
      - Issue Key: `={{ $('Code1').first().json.id }}`.  
      - Operation: Update  
      - Update Fields: Assignee set dynamically to `={{ $('Code2').item.json.accountId }}`.  
    - Jira credentials required.

14. **Connect Jira Assign output to Loop Over Items:**  
    - This loops the workflow back to process the next issue in batch.

15. **Add Sticky Notes (optional):**  
    - Add visual sticky notes for each Step as per workflow structure:
      - Step1: Scheduled Execution  
      - Step2: Jira Trigger (Issue Assigned)  
      - Step3: AI Assistant  
      - Step4: Response Generation / Ticket Update

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                         |
|------------------------------------------------------------------------------|--------------------------------------------------------|
| Replace placeholders `<Pinecone Indexを入力>` and `＜アカウントIDを入力してください＞` with actual Pinecone index and Jira account IDs to ensure correct operation. | Critical configuration before workflow activation.     |
| The workflow uses GPT-4.1-mini via OpenAI; ensure your OpenAI API plan supports this model. | OpenAI API documentation: https://platform.openai.com/docs/models |
| The AI Agent prompt is in Japanese, tailored for project developers handling Jira tasks. Adjust language as needed. | Localization note for prompt customization.             |
| Pinecone vector retrieval enhances AI responses by including relevant past Jira knowledge documents. | Pinecone docs: https://www.pinecone.io/docs/            |
| Jira API permissions must include reading issues, adding comments, and updating assignees. | Jira API documentation: https://developer.atlassian.com/cloud/jira/platform/rest/v3/ |
| Potential failure points include API rate limits, incorrect credentials, and data structure changes in Jira comments or issue fields. | Monitor n8n execution logs for troubleshooting.         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.