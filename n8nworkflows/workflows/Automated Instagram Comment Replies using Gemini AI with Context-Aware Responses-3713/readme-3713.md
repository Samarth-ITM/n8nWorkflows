Automated Instagram Comment Replies using Gemini AI with Context-Aware Responses

https://n8nworkflows.xyz/workflows/automated-instagram-comment-replies-using-gemini-ai-with-context-aware-responses-3713


# Automated Instagram Comment Replies using Gemini AI with Context-Aware Responses

### 1. Workflow Overview

This workflow automates Instagram comment replies by integrating Instagram’s webhook events with an AI agent powered by Gemini AI (via OpenRouter). It listens for new comments on Instagram posts, verifies webhook authenticity, extracts and validates comment data, fetches post context, generates context-aware AI responses, and posts replies directly under the original comments.

The workflow is logically divided into the following blocks:

- **1.1 Webhook Verification**: Handles Instagram webhook subscription verification by echoing the challenge token.
- **1.2 Comment Reception & Data Extraction**: Receives new comment events, extracts relevant fields from the payload, and structures them for processing.
- **1.3 User Validation**: Filters out comments made by the account owner to avoid self-replies and loops.
- **1.4 Post Data Retrieval**: Fetches the Instagram post’s caption and ID to provide context for AI-generated replies.
- **1.5 AI Response Generation**: Uses a custom AI agent with a detailed prompt to analyze the comment and generate a personalized, context-aware reply or decide to ignore irrelevant/spam comments.
- **1.6 Posting the Reply**: Sends the AI-generated response back to Instagram as a reply to the original comment.

---

### 2. Block-by-Block Analysis

#### 2.1 Webhook Verification

- **Overview:**  
  This block ensures the workflow securely integrates with Instagram’s webhook system by responding to verification challenges during subscription setup.

- **Nodes Involved:**  
  - Webhook  
  - Respond to Webhook  
  - Sticky Note (Webhook Verification instructions)

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (HTTP listener)  
    - Configured with a unique path matching Instagram’s webhook subscription endpoint.  
    - Listens for GET requests containing `hub.challenge` for verification.  
    - Output: Passes incoming request data to the next node.  
    - Edge Cases: Incorrect verify_token mismatch causes verification failure; network issues may block webhook calls.

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Configured to respond with the `hub.challenge` query parameter value extracted from the incoming request.  
    - Ensures Instagram receives the expected challenge response to confirm webhook subscription.  
    - Edge Cases: Missing or malformed `hub.challenge` leads to verification failure.

  - **Sticky Note**  
    - Provides instructions on ensuring the verify_token matches Instagram App settings and the need to echo `hub.challenge`.

---

#### 2.2 Comment Reception & Data Extraction

- **Overview:**  
  Receives POST webhook events for new Instagram comments and extracts key data fields into structured variables for downstream processing.

- **Nodes Involved:**  
  - get_new_comments (Webhook)  
  - data (Set)  
  - Sticky Note1 (Data Extraction instructions)

- **Node Details:**  
  - **get_new_comments**  
    - Type: Webhook (HTTP listener)  
    - Listens for POST requests at the same webhook path as verification but for comment events.  
    - Receives JSON payload containing comment details.  
    - Output: Raw webhook JSON passed to the next node.

  - **data (Set)**  
    - Type: Set  
    - Maps nested JSON fields from the webhook payload into flat, named variables:  
      - `conta.id`: Instagram account ID receiving the comment  
      - `usuario.id`: Commenter’s user ID  
      - `usuario.name`: Commenter’s username  
      - `usuario.message.id`: Comment ID  
      - `usuario.message.text`: Comment text  
      - `usuario.media.id`: Media (post) ID  
      - `endpoint`: Base Instagram Graph API URL (`https://graph.instagram.com/v22.0`)  
    - This normalization simplifies access in subsequent nodes.  
    - Edge Cases: Missing or malformed fields in payload may cause expression failures or incomplete data.

  - **Sticky Note1**  
    - Describes the importance of verifying all required fields are correctly mapped for processing.

---

#### 2.3 User Validation

- **Overview:**  
  Filters out comments made by the Instagram account owner to prevent the workflow from replying to itself.

- **Nodes Involved:**  
  - its me? (Filter)  
  - Sticky Note2 (User Validation instructions)

- **Node Details:**  
  - **its me?**  
    - Type: Filter  
    - Condition: Passes only if `conta.id` (account owner) is NOT equal to `usuario.id` (commenter).  
    - If IDs match, the comment is ignored (no further processing).  
    - Edge Cases: If IDs are missing or equal erroneously, valid comments might be skipped or self-comments replied to.

  - **Sticky Note2**  
    - Explains the logic and importance of this filtering step.

---

#### 2.4 Post Data Retrieval

- **Overview:**  
  Retrieves the Instagram post’s metadata (ID and caption) to provide richer context for the AI response.

- **Nodes Involved:**  
  - Get post data (HTTP Request)  
  - Sticky Note1 (also relevant here for data extraction context)

- **Node Details:**  
  - **Get post data**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://graph.instagram.com/v22.0/{{ $json.usuario.media.id }}?fields=id,caption`  
    - Authentication: Header Auth (configured with Instagram Graph API token)  
    - Purpose: Fetches the post’s caption to include in AI prompt context.  
    - Output: JSON with post ID and caption.  
    - Edge Cases: Invalid or expired access token, media ID not found, API rate limits, network errors.

---

#### 2.5 AI Response Generation

- **Overview:**  
  Uses a custom AI agent with a detailed system prompt to analyze the comment, classify its intent, and generate a personalized, context-aware reply or decide to ignore it.

- **Nodes Involved:**  
  - AI Agent (Langchain Agent)  
  - OpenRouter Chat Model (Gemini AI)  
  - Sticky Note3 (AI Response Generation instructions)

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent node  
    - Configuration:  
      - Custom prompt defines persona (friendly, expert in AI & automation), input variables (username, comment text, post caption), and detailed instructions for filtering and response generation.  
      - Output: Text reply or `[IGNORE]` string.  
    - Input: Receives data from filter node and post data retrieval.  
    - Output: Passes generated reply text to the next node.  
    - Edge Cases: AI model downtime, prompt errors, unexpected input formats, or generation of `[IGNORE]` to skip reply.

  - **OpenRouter Chat Model**  
    - Type: Language Model node (Gemini 2.0 flash experimental)  
    - Credentials: OpenRouter API key configured.  
    - Role: Provides the underlying AI model for the Langchain Agent.  
    - Edge Cases: API rate limits, authentication errors, network issues.

  - **Sticky Note3**  
    - Details the AI prompt design and expected behavior.

---

#### 2.6 Posting the Reply

- **Overview:**  
  Sends the AI-generated reply back to Instagram as a comment reply under the original comment.

- **Nodes Involved:**  
  - Post comment (HTTP Request)  
  - Sticky Note4 (Sending the Response instructions)

- **Node Details:**  
  - **Post comment**  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `{{ $json.endpoint }}/{{ $json.usuario.message.id }}/replies`  
    - Body: `message={{ $json.output }}` (the AI-generated reply text)  
    - Authentication: Header Auth with Instagram Graph API token  
    - Purpose: Posts the reply comment on Instagram.  
    - Edge Cases: API errors (invalid token, rate limits), malformed body, network failures, or if the AI returned `[IGNORE]` (should ideally skip posting).  
    - Note: The workflow does not explicitly handle skipping posting on `[IGNORE]` output; this should be considered for robustness.

  - **Sticky Note4**  
    - Emphasizes correct formatting and error handling for API calls.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                         | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                  |
|---------------------|--------------------------------|---------------------------------------|----------------------|----------------------|----------------------------------------------------------------------------------------------|
| Webhook             | Webhook                       | Webhook verification listener         | —                    | Respond to Webhook    | Handles the initial verification handshake with Instagram's Webhook API.                     |
| Respond to Webhook   | Respond to Webhook            | Responds to Instagram webhook challenge | Webhook              | —                    | Echoes `hub.challenge` to confirm webhook subscription.                                     |
| get_new_comments     | Webhook                       | Receives new Instagram comment events | —                    | data                 | Extracts relevant data from the incoming webhook payload.                                   |
| data                | Set                          | Maps and normalizes comment data       | get_new_comments      | Get post data         | Verify all necessary fields are correctly mapped.                                           |
| Get post data        | HTTP Request                 | Fetches Instagram post caption         | data                  | its me?               | Fetches media’s caption for richer AI context.                                              |
| its me?              | Filter                       | Filters out self-comments               | Get post data          | AI Agent              | Checks if comment is from account owner; skips if true.                                    |
| AI Agent            | Langchain Agent              | Generates AI response to comment       | its me?                | Post comment          | Uses AI to generate friendly, context-aware replies or `[IGNORE]`.                          |
| OpenRouter Chat Model| Language Model (Gemini AI)   | Provides AI model for response generation | AI Agent (ai_languageModel) | AI Agent              | Configured with OpenRouter API key.                                                        |
| Post comment         | HTTP Request                 | Posts AI-generated reply to Instagram  | AI Agent               | —                     | Sends reply back via Instagram API; handle errors carefully.                               |
| Sticky Note          | Sticky Note                  | Instructional notes                    | —                      | —                      | Various notes covering each block’s purpose and instructions.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Webhook Verification):**  
   - Type: Webhook  
   - Path: Unique string (e.g., `ea7d37ac-9e82-40d7-bbb3-e9b7ce180fc9`)  
   - HTTP Method: GET (default)  
   - Response Mode: Response Node  
   - Purpose: Receive Instagram webhook verification requests.

2. **Create Respond to Webhook Node:**  
   - Type: Respond to Webhook  
   - Connect input from Webhook node.  
   - Respond With: Text  
   - Response Body: Expression `{{$json.query['hub.challenge']}}`  
   - Purpose: Echo back the challenge token for verification.

3. **Create Webhook Node (Comment Reception):**  
   - Type: Webhook  
   - Path: Same as above (`ea7d37ac-9e82-40d7-bbb3-e9b7ce180fc9`)  
   - HTTP Method: POST  
   - Purpose: Receive new Instagram comment events.

4. **Create Set Node (Data Extraction):**  
   - Type: Set  
   - Connect input from comment reception webhook.  
   - Add fields with expressions mapping payload:  
     - `endpoint`: `"https://graph.instagram.com/v22.0"` (string)  
     - `conta.id`: `{{$json.body.entry[0].id}}`  
     - `usuario.id`: `{{$json.body.entry[0].changes[0].value.from.id}}`  
     - `usuario.name`: `{{$json.body.entry[0].changes[0].value.from.username}}`  
     - `usuario.message.id`: `{{$json.body.entry[0].changes[0].value.id}}`  
     - `usuario.message.text`: `{{$json.body.entry[0].changes[0].value.text}}`  
     - `usuario.media.id`: `{{$json.body.entry[0].changes[0].value.media.id}}`

5. **Create HTTP Request Node (Get post data):**  
   - Type: HTTP Request  
   - Connect input from Set node.  
   - Method: GET  
   - URL: Expression `https://graph.instagram.com/v22.0/{{$json.usuario.media.id}}?fields=id,caption`  
   - Authentication: HTTP Header Auth (configure with Instagram Graph API token)  
   - Purpose: Fetch post caption for AI context.

6. **Create Filter Node (User Validation):**  
   - Type: Filter  
   - Connect input from HTTP Request node.  
   - Condition: `conta.id` NOT EQUALS `usuario.id` (expression)  
   - Purpose: Skip comments from the account owner.

7. **Create OpenRouter Chat Model Node:**  
   - Type: Language Model (OpenRouter)  
   - Credentials: Configure OpenRouter API key  
   - Model: `google/gemini-2.0-flash-exp:free`  
   - Purpose: Provide AI model for Langchain Agent.

8. **Create Langchain Agent Node (AI Agent):**  
   - Type: Langchain Agent  
   - Connect input from Filter node.  
   - Configure prompt with:  
     - Persona: Expert AI assistant, friendly tone, Brazilian Portuguese.  
     - Inputs: username, comment text, post caption (use expressions referencing previous nodes).  
     - Instructions: Analyze comment intent, filter spam, generate personalized replies or `[IGNORE]`.  
   - Link AI language model input to OpenRouter Chat Model node.  
   - Purpose: Generate AI reply text.

9. **Create HTTP Request Node (Post comment):**  
   - Type: HTTP Request  
   - Connect input from AI Agent node.  
   - Method: POST  
   - URL: Expression `{{$json.endpoint}}/{{$json.usuario.message.id}}/replies`  
   - Body Parameters: `message` = `{{$json.output}}` (AI reply text)  
   - Authentication: HTTP Header Auth (Instagram Graph API token)  
   - Purpose: Post AI-generated reply as comment reply.

10. **Connect all nodes in order:**  
    - Webhook (verification) → Respond to Webhook  
    - Webhook (comment reception) → Set (data) → HTTP Request (Get post data) → Filter (its me?) → Langchain Agent (AI Agent) → HTTP Request (Post comment)  
    - OpenRouter Chat Model connected as AI language model input to Langchain Agent.

11. **Configure Credentials:**  
    - Instagram Graph API token with scopes: `instagram_basic`, `instagram_manage_comments`  
    - OpenRouter/OpenAI API key for AI model access.

12. **Test Workflow:**  
    - Publish test comment on Instagram post.  
    - Verify webhook receives event, data extraction works, AI generates reply, and reply posts successfully.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The AI Agent prompt is written in Brazilian Portuguese and tailored for a friendly, expert tone.   | Adjust prompt language or style in the Langchain Agent node as needed.                          |
| Instagram Graph API version used: v22.0                                                            | https://developers.facebook.com/docs/instagram-api/reference/                                  |
| OpenRouter Chat Model uses Gemini 2.0 flash experimental free tier                                 | https://openrouter.ai/                                                                          |
| Ensure Instagram App webhook verify_token matches the one configured in the Webhook node.          | Instagram Developer Dashboard webhook settings                                                 |
| Consider adding error handling or conditional logic to skip posting if AI returns `[IGNORE]`.      | Improves robustness by preventing unwanted replies.                                           |
| Workflow licensed under MIT, compatible with n8n version 1.88.0 and above.                         |                                                                                               |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and extend the Instagram Auto-Comment Responder workflow with Gemini AI integration.