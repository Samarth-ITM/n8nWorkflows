Generate AI LinkedIn Posts with Human Approval via Telegram and GPT

https://n8nworkflows.xyz/workflows/generate-ai-linkedin-posts-with-human-approval-via-telegram-and-gpt-9472


# Generate AI LinkedIn Posts with Human Approval via Telegram and GPT

### 1. Workflow Overview

This workflow automates the creation of LinkedIn posts by combining AI-generated content with human-in-the-loop approval via Telegram. Its purpose is to produce valuable, web-enriched LinkedIn posts that are reviewed and edited interactively by a user through Telegram messages before final approval and archival.

**Target Use Cases:**  
- Social media managers or content creators who want to automate LinkedIn post generation enriched with fresh web insights.  
- Teams that require human validation and iterative refinement of AI-generated content.  
- Workflows integrating Telegram as a conversational interface for feedback and approval.  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving user input from Telegram to trigger LinkedIn post creation.  
- **1.2 AI Content Generation:** Querying the web for relevant insights and generating a first draft post via an AI agent.  
- **1.3 Human Review & Feedback:** Sending the draft post to the user on Telegram and collecting approval or revision requests.  
- **1.4 Text Classification:** Automatically categorizing user feedback as approval or denial to control the flow.  
- **1.5 Post Revision:** Using AI to refine the post based on user feedback if not approved.  
- **1.6 Finalization & Archiving:** Appending approved posts to a Google Sheet for record-keeping and future publishing.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives Telegram messages from the user containing topics or instructions for LinkedIn post generation.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**  
  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Listens to incoming Telegram messages to start the workflow.  
    - Configuration: Watches for "message" updates; uses a Telegram Bot credential named `n8nyasser_bot`.  
    - Key Expressions: The entire Telegram message text is passed downstream for processing.  
    - Inputs: None (webhook trigger)  
    - Outputs: Starts the workflow with the message text.  
    - Edge Cases: Telegram API connectivity issues, invalid messages, or unexpected formats may cause failures.

#### 1.2 AI Content Generation

- **Overview:**  
Searches the web for relevant content to enrich the input topic and drafts an initial LinkedIn post using an AI agent.

- **Nodes Involved:**  
  - AI Agent  
  - Tavily Tool  
  - Edit Fields

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent node  
    - Role: Generates LinkedIn post drafts based on the user’s Telegram message.  
    - Configuration:  
      - Uses the Telegram message text as input.  
      - System message instructs it to search the web with the Tavily tool and create a valuable LinkedIn post with at least 5 hashtags.  
      - Uses a prompt type "define" for fixed instructions.  
    - Connections: Receives input from Telegram Trigger; sends output to Edit Fields.  
    - Edge Cases: Failures in calling Tavily tool or LLM timeouts/errors.  
  - **Tavily Tool**  
    - Type: Custom node for web search  
    - Role: Searches the web to provide up-to-date information to the AI agent.  
    - Configuration:  
      - Query dynamically set by AI agent’s override expression.  
      - Searches general topics with advanced depth, max 3 results.  
      - Requires Tavily API credential.  
    - Connections: Called internally by AI Agent as an AI tool.  
    - Edge Cases: API rate limits, no relevant search results, or network errors.  
  - **Edit Fields**  
    - Type: Set node  
    - Role: Extracts and stores the generated post content under the field `post`.  
    - Configuration: Assigns `post` = the AI agent’s output.  
    - Connections: Receives from AI Agent; sends to Telegram message node.  
    - Edge Cases: Unexpected output format from AI Agent.

#### 1.3 Human Review & Feedback

- **Overview:**  
Sends the AI-generated post to the user through Telegram and waits for approval or feedback.

- **Nodes Involved:**  
  - Send a text message  
  - Text Classifier

- **Node Details:**  
  - **Send a text message**  
    - Type: Telegram node  
    - Role: Sends the generated post to the user with a prompt "Good to go?" and waits for text response.  
    - Configuration:  
      - Uses chat ID from Telegram Trigger.  
      - Waits for free text response.  
      - Does not append attribution.  
      - Uses the same Telegram Bot credential.  
    - Connections: Input from Edit Fields; output to Text Classifier.  
    - Edge Cases: Telegram API failures, user no response, or unexpected replies.  
  - **Text Classifier**  
    - Type: Langchain Text Classifier  
    - Role: Categorizes user feedback as "Approved" or "Denied".  
    - Configuration:  
      - Input text is the user’s Telegram reply (`data.text`).  
      - Categories:  
        - Approved: confirmation phrases like "Good to go", "Send it", "OK".  
        - Denied: edit requests or negative feedback like "Make it shorter", "Add something".  
    - Connections: Input from Send a text message; outputs to either append to Google Sheet (Approved) or to revision chain (Denied).  
    - Edge Cases: Misclassification due to ambiguous feedback; empty or irrelevant replies.

#### 1.4 Post Revision

- **Overview:**  
If the post is denied, an AI agent revises the post based on the user’s feedback.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - Edit Fields1

- **Node Details:**  
  - **Basic LLM Chain**  
    - Type: Langchain Chain LLM node  
    - Role: Refines the LinkedIn post using the original draft and the human feedback.  
    - Configuration:  
      - Input text includes the post to revise and the human feedback.  
      - System message instructs to output only the post body.  
      - Uses prompt type "define".  
    - Connections: Receives input from Text Classifier (Denied path); outputs to Edit Fields1.  
    - Edge Cases: LLM runtime errors, ambiguous instructions, or incoherent revisions.  
  - **Edit Fields1**  
    - Type: Set node  
    - Role: Updates the `output` field with the revised post text.  
    - Configuration: Assigns `output` = the text from Basic LLM Chain result.  
    - Connections: Sends updated post back to Edit Fields to restart approval cycle.  
    - Edge Cases: Output data inconsistencies.

#### 1.5 Finalization & Archiving

- **Overview:**  
Approved posts are appended to a Google Sheet for archival and future publishing.

- **Nodes Involved:**  
  - Append row in sheet

- **Node Details:**  
  - **Append row in sheet**  
    - Type: Google Sheets node  
    - Role: Stores the approved LinkedIn post data in a specified Google Sheet.  
    - Configuration:  
      - Operation: Append.  
      - Document ID and sheet name set to a demo contacts sheet.  
      - Columns mapped for Name and Email but likely to be adapted for post data.  
      - Uses Google Sheets OAuth2 credentials.  
    - Connections: Receives input from Text Classifier (Approved path).  
    - Edge Cases: Google Sheets API rate limits, permission errors, or misconfigured sheet.

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                    | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                     |
|---------------------|---------------------------------|----------------------------------|------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger                 | Receive user message to trigger  | -                      | AI Agent                | ## Trigger \nStart by sending a message to your Telegram bot (e.g. “Write a LinkedIn post about AI”). |
| AI Agent            | Langchain Agent                 | Generate initial LinkedIn post   | Telegram Trigger        | Edit Fields             | ## AI Agent \nSearches the web for fresh insights to enrich the post and Drafts a professional LinkedIn post using OpenAI or any LLM model you want |
| Tavily Tool         | Tavily Web Search Tool          | Web search for fresh content     | AI Agent (ai_tool)      | AI Agent (ai_tool)      |                                                                                                |
| Edit Fields         | Set                            | Extract post from AI output      | AI Agent                | Send a text message     |                                                                                                |
| Send a text message | Telegram                       | Send post to user and get feedback | Edit Fields             | Text Classifier         | ## Approved?\nThe bot asks “Good to go?” — you can approve or give feedback. Once you approve, the post is appended to a Google Sheet — ready to publish. |
| Text Classifier     | Langchain Text Classifier       | Classify feedback as Approved/Denied | Send a text message     | Append row in sheet, Basic LLM Chain |                                                                                                |
| Basic LLM Chain     | Langchain Chain LLM             | Revise post based on feedback    | Text Classifier (Denied) | Edit Fields1            | ## Revision Agent \nIf not approved, another AI refines the post based on your feedback.        |
| Edit Fields1        | Set                            | Store revised post text          | Basic LLM Chain         | Edit Fields             |                                                                                                |
| Append row in sheet | Google Sheets                   | Archive approved posts           | Text Classifier (Approved) | -                      |                                                                                                |
| OpenAI Chat Model   | Langchain OpenAI Chat Model     | AI language model used in chain  | - (ai_languageModel)    | Basic LLM Chain, Text Classifier, AI Agent |                                                                                                |
| Sticky Note         | Sticky Note                    | Comments and explanations        | -                      | -                       |                                                                                                |
| Sticky Note1        | Sticky Note                    | Comments and explanations        | -                      | -                       |                                                                                                |
| Sticky Note2        | Sticky Note                    | Comments and explanations        | -                      | -                       |                                                                                                |
| Sticky Note3        | Sticky Note                    | Comments and explanations        | -                      | -                       |                                                                                                |
| Sticky Note4        | Sticky Note                    | Comments and explanations        | -                      | -                       |                                                                                                |
| Sticky Note5        | Sticky Note                    | Comments and explanations        | -                      | -                       |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for "message" updates.  
   - Assign Telegram Bot credentials (`n8nyasser_bot`).  
   - This node will start the workflow when a Telegram message is received.

2. **Create Tavily Tool Node**  
   - Type: Tavily Tool (custom node)  
   - Set `query` parameter to be dynamically overridden by AI agent (`{{$fromAI('Query', '', 'string')}}`).  
   - Set options: topic = "general", max_results = 3, search_depth = "advanced".  
   - Provide Tavily API credentials.  
   - This node will be used by AI agent to search the web.

3. **Create AI Agent Node**  
   - Type: Langchain Agent  
   - Input expression: `{{$json.message.text}}` (message text from Telegram).  
   - System message: instruct the agent to generate LinkedIn posts enriched with Tavily search results, including at least 5 hashtags.  
   - Set prompt type to "define".  
   - Link Tavily Tool node as AI tool for web search.  
   - Connect input from Telegram Trigger.  
   - Output connected to a Set node.

4. **Create Set Node (Edit Fields)**  
   - Type: Set  
   - Assign field `post` equal to AI Agent output (`{{$json.output}}`).  
   - Connect input from AI Agent.  
   - Output connected to Telegram node.

5. **Create Telegram Node (Send a text message)**  
   - Type: Telegram  
   - Send message to chat ID from Telegram Trigger: `{{$('Telegram Trigger').item.json.message.chat.id}}`.  
   - Message content: `"Good to go?\n\n{{$json.post}}"`.  
   - Operation: sendAndWait for user reply.  
   - Use same Telegram Bot credentials.  
   - Output connected to Text Classifier.

6. **Create Text Classifier Node**  
   - Type: Langchain Text Classifier  
   - Input text: `{{$json.data.text}}` (user reply from Telegram).  
   - Categories:  
     - "Approved": for confirmations like "Good to go", "Send it".  
     - "Denied": for feedback requiring edits.  
   - Outputs:  
     - Approved → Append row in sheet node.  
     - Denied → Basic LLM Chain node for revision.

7. **Create Langchain Chain LLM Node (Basic LLM Chain)**  
   - Type: Chain LLM  
   - Input text: includes original post (`{{$('Edit Fields').item.json.post}}`) and human feedback (`{{$('Send a text message').item.json.data.text}}`).  
   - System message: instructs to revise the post based on feedback and output only the post body.  
   - Prompt type: "define".  
   - Output connected to Set node (Edit Fields1).

8. **Create Set Node (Edit Fields1)**  
   - Type: Set  
   - Assign `output` field = revised text (`{{$json.text}}`).  
   - Output connected back to Edit Fields to restart approval cycle.

9. **Create Google Sheets Node (Append row in sheet)**  
   - Type: Google Sheets  
   - Operation: Append row.  
   - Configure Document ID and Sheet Name for your sheet (e.g., LinkedIn posts archive).  
   - Map necessary columns to store post data.  
   - Use Google Sheets OAuth2 credentials.  
   - Input from Text Classifier (Approved path).

10. **(Optional) Create OpenAI Chat Model Node**  
    - Type: Langchain OpenAI Chat Model  
    - Model: Select GPT-4.1-mini or preferred.  
    - Assign this node to be used by AI Agent and Basic LLM Chain.  
    - Provide OpenAI API credentials.

11. **Add Sticky Notes**  
    - Add descriptive sticky notes near logical blocks for documentation and clarity.

12. **Link all nodes as described in the connections:**  
    - Telegram Trigger → AI Agent  
    - AI Agent → Edit Fields → Send a text message → Text Classifier  
    - Text Classifier (Approved) → Append row in sheet  
    - Text Classifier (Denied) → Basic LLM Chain → Edit Fields1 → Edit Fields (cycle continues)

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This template creates LinkedIn posts with AI + human feedback directly in Telegram.                            | Workflow overview (Sticky Note1)                                                                       |
| Trigger: Start by sending a message to your Telegram bot (e.g. “Write a LinkedIn post about AI”).              | Telegram Trigger explanation (Sticky Note2)                                                           |
| AI Agent: Searches the web for fresh insights to enrich the post and drafts a professional LinkedIn post.     | AI Agent explanation (Sticky Note3)                                                                   |
| Approved?: The bot asks “Good to go?” — you can approve or give feedback. Once approved, post is appended to Google Sheet for publishing. | Human review explanation (Sticky Note4)                                                               |
| Revision Agent: If not approved, another AI refines the post based on your feedback.                           | Post revision explanation (Sticky Note5)                                                              |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.