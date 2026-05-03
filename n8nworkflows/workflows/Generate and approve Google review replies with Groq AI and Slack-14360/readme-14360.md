Generate and approve Google review replies with Groq AI and Slack

https://n8nworkflows.xyz/workflows/generate-and-approve-google-review-replies-with-groq-ai-and-slack-14360


# Generate and approve Google review replies with Groq AI and Slack

## 1. Workflow Overview

This workflow monitors new Google Business Profile reviews, generates a reply with Groq AI, and then decides whether to publish the reply automatically or route it through Slack for human approval first.

Its main use case is review operations for businesses that want:
- fast automated responses to positive reviews,
- controlled handling of less-positive or potentially sensitive reviews,
- a consistent brand voice using AI-generated drafts.

The workflow is organized into three functional blocks plus documentation notes:

### 1.1 Review Intake and Data Preparation
A Google Business Profile trigger detects newly posted reviews. The workflow then normalizes the incoming review data into a simpler structure for downstream processing.

### 1.2 AI Reply Generation
The prepared review data is passed into an AI Agent that uses a Groq chat model to generate a suggested response.

### 1.3 Decision, Approval, and Publishing
The workflow checks the star rating. Four- and five-star reviews are replied to automatically. Other ratings are sent to Slack with an approval action. If approved in Slack, the reply is posted to Google Business Profile.

### 1.4 Embedded Documentation / Sticky Notes
Several sticky notes document the purpose of the workflow and the intent of each major stage.

---

## 2. Block-by-Block Analysis

## 2.1 Review Intake and Data Preparation

**Overview:**  
This block listens for new Google reviews and extracts the fields needed for reply generation and routing logic. It converts the original Google review payload into a simplified item containing reviewer name, review text, rating, and review identifier.

**Nodes Involved:**  
- Fetch New Reviews
- Edit Fields

### Node: Fetch New Reviews
- **Type and technical role:** `n8n-nodes-base.googleBusinessProfileTrigger`  
  Trigger node that polls Google Business Profile for new reviews.
- **Configuration choices:**  
  - Configured for a specific Google Business Profile account and location.
  - Polling schedule is set to every minute.
- **Key expressions or variables used:**  
  - No custom expressions shown.
- **Input and output connections:**  
  - Entry point of the workflow.
  - Outputs to **Edit Fields**.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
  - Requires a compatible n8n version with the Google Business Profile Trigger node available.
- **Edge cases or potential failure types:**  
  - Google authentication failure or expired credentials.
  - Missing account/location configuration.
  - API polling limits or temporary Google API unavailability.
  - Duplicate-event behavior may depend on trigger internals and polling state persistence.
- **Sub-workflow reference:**  
  - None.

### Node: Edit Fields
- **Type and technical role:** `n8n-nodes-base.set`  
  Data shaping node used to remap review payload fields into simpler keys.
- **Configuration choices:**  
  - Creates four fields:
    - `reviewerName ` ← `{{$json.reviewer.displayName}}`
    - `reviewText ` ← `{{$json.comment}}`
    - `rating ` ← `{{$json.starRating}}`
    - `reviewName ` ← `{{$json.name}}`
  - Important: all four field names contain a trailing space.
- **Key expressions or variables used:**  
  - `{{$json.reviewer.displayName}}`
  - `{{$json.comment}}`
  - `{{$json.starRating}}`
  - `{{$json.name}}`
- **Input and output connections:**  
  - Input from **Fetch New Reviews**
  - Output to **AI Agent**
- **Version-specific requirements:**  
  - Uses `typeVersion: 3.4`.
- **Edge cases or potential failure types:**  
  - If the incoming review lacks `reviewer.displayName`, `comment`, `starRating`, or `name`, mapped fields may be empty.
  - The trailing spaces in field names can easily cause later expression mistakes.
  - Empty review text may reduce AI response quality.
- **Sub-workflow reference:**  
  - None.

---

## 2.2 AI Reply Generation

**Overview:**  
This block generates the proposed review reply. The AI Agent receives the prepared data and uses the Groq chat model as its language model backend.

**Nodes Involved:**  
- AI Agent
- Groq Chat Model

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  LangChain-based AI orchestration node that produces the final suggested reply text.
- **Configuration choices:**  
  - No explicit prompt or advanced options are shown in the JSON beyond default `options`.
  - It is connected to a Groq chat model as its language model input.
- **Key expressions or variables used:**  
  - Downstream nodes rely on `$('AI Agent').item.json.output`, so this node is expected to produce an `output` field.
- **Input and output connections:**  
  - Main input from **Edit Fields**
  - AI language model input from **Groq Chat Model**
  - Main output to **If**
- **Version-specific requirements:**  
  - Uses `typeVersion: 3`.
  - Requires n8n with LangChain/AI nodes enabled.
- **Edge cases or potential failure types:**  
  - Missing Groq credentials.
  - Model rate limits or context-window issues.
  - Weak results if no system prompt/instructions are configured elsewhere.
  - Unexpected output structure if the node is later reconfigured, which would break references to `.json.output`.
- **Sub-workflow reference:**  
  - None.

### Node: Groq Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatGroq`  
  Language model connector used by the AI Agent.
- **Configuration choices:**  
  - Model: `llama-3.3-70b-versatile`
  - No additional options configured.
- **Key expressions or variables used:**  
  - None.
- **Input and output connections:**  
  - Outputs via `ai_languageModel` connection to **AI Agent**
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
  - Requires Groq API credentials configured in n8n.
- **Edge cases or potential failure types:**  
  - Invalid or expired API key.
  - Model availability changes on Groq.
  - Request timeout or rate limiting.
- **Sub-workflow reference:**  
  - None.

---

## 2.3 Decision, Approval, and Publishing

**Overview:**  
This block decides whether to auto-post the AI reply or require a human approver. Ratings of FOUR or FIVE are published immediately; all other ratings go to Slack for approval before posting.

**Nodes Involved:**  
- If
- Reply to review
- Send message and wait for response
- If1
- Reply to review1

### Node: If
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router that checks whether the review rating is positive enough for automatic reply.
- **Configuration choices:**  
  - OR logic
  - Condition 1: `{{$('Edit Fields').item.json["rating "]}} == "FOUR"`
  - Condition 2: `{{$('Edit Fields').item.json["rating "]}} == "FIVE"`
- **Key expressions or variables used:**  
  - `{{$('Edit Fields').item.json["rating "]}}`
- **Input and output connections:**  
  - Input from **AI Agent**
  - True branch to **Reply to review**
  - False branch to **Send message and wait for response**
- **Version-specific requirements:**  
  - Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Depends on Google review ratings being emitted as uppercase enum values such as `FOUR` and `FIVE`.
  - Any mismatch in field naming due to the trailing space in `rating ` will break the condition.
  - Ratings like `THREE`, `TWO`, `ONE`, or missing values go to Slack by default.
- **Sub-workflow reference:**  
  - None.

### Node: Reply to review
- **Type and technical role:** `n8n-nodes-base.googleBusinessProfile`  
  Google Business Profile action node that posts a reply directly to the review.
- **Configuration choices:**  
  - Resource: `review`
  - Operation: `reply`
  - Reply text: `{{$('AI Agent').item.json.output}}`
  - Review ID: `{{$('Code in JavaScript1').item.json.reviewId}}`
  - Account and location are configured via list selectors.
- **Key expressions or variables used:**  
  - `{{$('AI Agent').item.json.output}}`
  - `{{$('Code in JavaScript1').item.json.reviewId}}`
- **Input and output connections:**  
  - Input from the true branch of **If**
  - No downstream node shown.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - **Critical issue:** the expression references **Code in JavaScript1**, but no such node exists in this workflow JSON.
  - Because of that, this node will fail unless the expression is corrected.
  - Google permission issues or missing account/location values can also block posting.
  - Replying may fail if the review has already been replied to or the review ID is invalid.
- **Sub-workflow reference:**  
  - None.

### Node: Send message and wait for response
- **Type and technical role:** `n8n-nodes-base.slack`  
  Slack approval node that posts a message and pauses execution until a human approves or rejects.
- **Configuration choices:**  
  - Operation: `sendAndWait`
  - Authentication: OAuth2
  - Channel is selected by `channelId`
  - Approval type: `double`
  - Message includes:
    - customer name,
    - rating,
    - review text,
    - AI suggested reply.
- **Key expressions or variables used:**  
  - `{{$('Edit Fields').item.json['reviewerName ']}}`
  - `{{$('Edit Fields').item.json['rating ']}}`
  - `{{$('Edit Fields').item.json['reviewText ']}}`
  - `{{$('AI Agent').item.json.output}}`
- **Input and output connections:**  
  - Input from the false branch of **If**
  - Output to **If1**
- **Version-specific requirements:**  
  - Uses `typeVersion: 2.4`.
  - Includes a `webhookId`, which n8n uses internally for the waiting/approval callback.
- **Edge cases or potential failure types:**  
  - Placeholder value `YOUR_SLACK_CHANNEL_ID` must be replaced.
  - Slack OAuth2 credentials must be set correctly.
  - Slack app must have permissions for interactive approval/wait behavior.
  - Approval can stall indefinitely if no one responds.
  - Message formatting may degrade if review text is empty or very long.
- **Sub-workflow reference:**  
  - None.

### Node: If1
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional gate that checks Slack approval result.
- **Configuration choices:**  
  - Single boolean condition:
    - `{{$json.data.approved}} == true`
- **Key expressions or variables used:**  
  - `{{$json.data.approved}}`
- **Input and output connections:**  
  - Input from **Send message and wait for response**
  - True branch to **Reply to review1**
- **Version-specific requirements:**  
  - Uses `typeVersion: 2.3`.
- **Edge cases or potential failure types:**  
  - Assumes Slack approval response returns `data.approved`.
  - If Slack schema changes or approval payload is malformed, the condition may not match.
  - No explicit rejection handling path is defined.
- **Sub-workflow reference:**  
  - None.

### Node: Reply to review1
- **Type and technical role:** `n8n-nodes-base.googleBusinessProfile`  
  Google Business Profile action node that posts the approved reply after Slack approval.
- **Configuration choices:**  
  - Resource: `review`
  - Operation: `reply`
  - Reply text: `{{$('AI Agent').item.json.output}}`
  - Review ID: `{{$('Code in JavaScript1').item.json.reviewId}}`
- **Key expressions or variables used:**  
  - `{{$('AI Agent').item.json.output}}`
  - `{{$('Code in JavaScript1').item.json.reviewId}}`
- **Input and output connections:**  
  - Input from the true branch of **If1**
  - No downstream node shown.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - **Critical issue:** same broken reference to non-existent **Code in JavaScript1**.
  - Review reply can fail for auth, permissions, invalid ID, or duplicate reply reasons.
  - No branch handles rejected approvals or posting failures.
- **Sub-workflow reference:**  
  - None.

---

## 2.4 Embedded Documentation / Sticky Notes

**Overview:**  
These nodes provide in-canvas explanations for operators building or maintaining the workflow. They do not participate in execution but are important for understanding intended behavior and setup.

**Nodes Involved:**  
- Sticky Note6
- Sticky Note7
- Sticky Note8
- Sticky Note9

### Node: Sticky Note6
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation node describing the full workflow.
- **Configuration choices:**  
  - Contains purpose, benefits, flow summary, and setup checklist.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

### Node: Sticky Note7
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation node for Step 1.
- **Configuration choices:**  
  - Explains capture and preparation of review data.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

### Node: Sticky Note8
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation node for Step 2.
- **Configuration choices:**  
  - Explains AI reply generation.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

### Node: Sticky Note9
- **Type and technical role:** `n8n-nodes-base.stickyNote`  
  Documentation node for Step 3.
- **Configuration choices:**  
  - Explains decision logic and Slack approval routing.
- **Input and output connections:**  
  - None.
- **Version-specific requirements:**  
  - Uses `typeVersion: 1`.
- **Edge cases or potential failure types:**  
  - None.
- **Sub-workflow reference:**  
  - None.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Fetch New Reviews | Google Business Profile Trigger | Poll for new Google reviews |  | Edit Fields | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 1 : Capture & Prepare Review Data<br>Fetch new Google reviews and extract key fields like name, rating, and comment to prepare clean input for AI processing. |
| Edit Fields | Set | Normalize review payload into simplified fields | Fetch New Reviews | AI Agent | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 1 : Capture & Prepare Review Data<br>Fetch new Google reviews and extract key fields like name, rating, and comment to prepare clean input for AI processing. |
| AI Agent | LangChain Agent | Generate suggested reply text | Edit Fields, Groq Chat Model | If | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 2 : Generate AI Reply<br>AI creates a professional, human-like response based on review content, rating, and business tone guidelines. |
| Groq Chat Model | Groq Chat Model | Provide LLM backend to AI Agent |  | AI Agent | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 2 : Generate AI Reply<br>AI creates a professional, human-like response based on review content, rating, and business tone guidelines. |
| If | If | Route positive reviews to auto-reply, others to approval | AI Agent | Reply to review, Send message and wait for response | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 3 : Decision & Response Handling<br>Positive reviews are auto-replied. Negative reviews are sent to Slack for approval before posting the final response. |
| Reply to review | Google Business Profile | Auto-post reply for positive reviews | If |  | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 3 : Decision & Response Handling<br>Positive reviews are auto-replied. Negative reviews are sent to Slack for approval before posting the final response. |
| Send message and wait for response | Slack | Send review + AI draft to Slack and wait for approval | If | If1 | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 3 : Decision & Response Handling<br>Positive reviews are auto-replied. Negative reviews are sent to Slack for approval before posting the final response. |
| If1 | If | Check whether Slack approver approved the draft | Send message and wait for response | Reply to review1 | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 3 : Decision & Response Handling<br>Positive reviews are auto-replied. Negative reviews are sent to Slack for approval before posting the final response. |
| Reply to review1 | Google Business Profile | Post approved reply after Slack approval | If1 |  | # Product Review Monitoring & AI Response Bot<br>This workflow automates how businesses handle Google reviews by combining real-time triggers, AI-generated responses, and optional human approval. Whenever a new review is posted, the workflow captures the data, generates a professional reply using AI, and decides whether to post it automatically or send it for manual approval via Slack.<br>This approach ensures fast responses for positive reviews while maintaining control over sensitive or negative feedback. It helps businesses improve customer engagement, maintain brand tone, and save time without compromising quality.<br>### How it works<br>- Trigger fires when a new Google review is added<br>- Review data is cleaned and structured<br>- AI generates a contextual reply<br>- Positive reviews → auto-reply<br>- Negative reviews → sent to Slack for approval<br>- Based on approval → reply is posted<br>### Setup<br>- Connect Google Business Profile credentials<br>- Add AI model credentials (Groq/OpenAI)<br>- Connect Slack account for approval flow<br>- Configure IF conditions for rating logic<br>- Ensure correct mapping for review name & reply fields<br><br>## Step 3 : Decision & Response Handling<br>Positive reviews are auto-replied. Negative reviews are sent to Slack for approval before posting the final response. |
| Sticky Note6 | Sticky Note | In-canvas documentation for workflow purpose and setup |  |  |  |
| Sticky Note7 | Sticky Note | In-canvas documentation for Step 1 |  |  |  |
| Sticky Note8 | Sticky Note | In-canvas documentation for Step 2 |  |  |  |
| Sticky Note9 | Sticky Note | In-canvas documentation for Step 3 |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   Name it: **Generate and approve Google review replies with Groq AI and Slack**.

2. **Add the trigger node: Google Business Profile Trigger**
   - Node type: **Google Business Profile Trigger**
   - Name: **Fetch New Reviews**
   - Configure:
     - Select your **Google Business Profile credentials**
     - Choose the target **account**
     - Choose the target **location**
     - Set polling to **Every Minute**
   - This node becomes the workflow entry point.

3. **Add a Set node to normalize review fields**
   - Node type: **Set**
   - Name: **Edit Fields**
   - Add these assignments:
     - `reviewerName ` → expression `{{$json.reviewer.displayName}}`
     - `reviewText ` → expression `{{$json.comment}}`
     - `rating ` → expression `{{$json.starRating}}`
     - `reviewName ` → expression `{{$json.name}}`
   - Important: this workflow uses field names with trailing spaces. To reproduce it exactly, keep them as-is.  
     However, for a cleaner rebuild, it is strongly recommended to remove trailing spaces and then update every later expression accordingly.

4. **Connect Fetch New Reviews → Edit Fields**

5. **Add the AI model node**
   - Node type: **Groq Chat Model**
   - Name: **Groq Chat Model**
   - Configure:
     - Credentials: your **Groq API credentials**
     - Model: **llama-3.3-70b-versatile**

6. **Add the AI generation node**
   - Node type: **AI Agent**
   - Name: **AI Agent**
   - Leave default options if you want to match the provided workflow structure.
   - Connect the **Groq Chat Model** to the AI Agent using the **AI language model** connection.
   - Connect **Edit Fields → AI Agent** using the main connection.

7. **Configure AI Agent behavior**
   - The JSON does not show a custom prompt, but in a real rebuild you should provide clear instructions such as:
     - reply professionally,
     - acknowledge the customer,
     - adapt tone to rating,
     - keep reply concise,
     - avoid inventing facts,
     - avoid offering compensation unless supported by policy.
   - Ensure the node outputs a field accessible as `output`, since later nodes use `{{$('AI Agent').item.json.output}}`.

8. **Add the rating decision node**
   - Node type: **If**
   - Name: **If**
   - Set condition logic to **OR**
   - Add condition 1:
     - Left value: `{{$('Edit Fields').item.json["rating "]}}`
     - Operator: equals
     - Right value: `FOUR`
   - Add condition 2:
     - Left value: `{{$('Edit Fields').item.json["rating "]}}`
     - Operator: equals
     - Right value: `FIVE`
   - Connect **AI Agent → If**

9. **Add the auto-reply node for positive reviews**
   - Node type: **Google Business Profile**
   - Name: **Reply to review**
   - Configure:
     - Resource: **review**
     - Operation: **reply**
     - Account: select the same Google Business Profile account
     - Location: select the same location
     - Reply text: `{{$('AI Agent').item.json.output}}`
     - Review identifier: use the current review item ID
   - Important correction: the original JSON uses `{{$('Code in JavaScript1').item.json.reviewId}}`, but that node does not exist.  
     To make the workflow functional, replace that with the actual review identifier from the trigger payload, for example:
     - either `{{$json.name}}`
     - or `{{$('Edit Fields').item.json["reviewName "]}}`
   - Connect the **true** output of **If** to **Reply to review**.

10. **Add the Slack approval node for non-positive reviews**
    - Node type: **Slack**
    - Name: **Send message and wait for response**
    - Configure:
      - Authentication: **OAuth2**
      - Operation: **Send and Wait**
      - Select by: **channel**
      - Channel ID: replace `YOUR_SLACK_CHANNEL_ID` with your real Slack channel
      - Approval type: **double**
    - Message content:
      ```text
      🚨 *New Review Needs Approval*

      👤 *Customer:* {{ $('Edit Fields').item.json['reviewerName '] }} 

      ⭐ *Rating:* {{ $('Edit Fields').item.json['rating '] }}  

      📝 *Review:*  {{ $('Edit Fields').item.json['reviewText '] }}

      🤖 *AI Suggested Reply:*  {{ $('AI Agent').item.json.output }}

      👉 Please review and take action below:
      ```
    - Connect the **false** output of **If** to this Slack node.

11. **Configure Slack credentials**
    - Create or select Slack OAuth2 credentials in n8n.
    - Ensure the Slack app has the required scopes for:
      - posting messages,
      - interactive approvals / callbacks,
      - waiting for responses in n8n.
    - If self-hosted, ensure webhook/callback URLs are reachable publicly.

12. **Add the approval result check**
    - Node type: **If**
    - Name: **If1**
    - Add one boolean condition:
      - Left value: `{{$json.data.approved}}`
      - Operator: equals
      - Right value: `true`
    - Connect **Send message and wait for response → If1**

13. **Add the post-approval Google reply node**
    - Node type: **Google Business Profile**
    - Name: **Reply to review1**
    - Configure identically to the earlier reply node:
      - Resource: **review**
      - Operation: **reply**
      - Reply text: `{{$('AI Agent').item.json.output}}`
      - Review identifier: use the real review ID or review name field
    - Again, do **not** keep the broken reference to `Code in JavaScript1` unless you also create such a node.
    - Connect the **true** output of **If1** to **Reply to review1**

14. **Optionally add a rejection-handling branch**
    - The original workflow has none.
    - Recommended additions:
      - a Slack follow-up message,
      - a log record,
      - a manual review queue,
      - or a branch allowing an edited reply before posting.

15. **Add documentation sticky notes**
    - Create four Sticky Note nodes if you want to mirror the original canvas:
      - General workflow explanation
      - Step 1: Capture & Prepare Review Data
      - Step 2: Generate AI Reply
      - Step 3: Decision & Response Handling

16. **Test the trigger and payload mapping**
    - Confirm the trigger emits:
      - reviewer display name,
      - comment,
      - star rating,
      - review name / identifier.
    - Verify whether the review identifier needed by the reply node is `name` or another field.

17. **Test the AI output**
    - Run with a sample review.
    - Confirm **AI Agent** returns `output`.
    - If not, update downstream expressions to the actual output path.

18. **Test the positive-review path**
    - Use a `FOUR` or `FIVE` review.
    - Verify it routes directly to **Reply to review** and posts successfully.

19. **Test the approval path**
    - Use a lower rating.
    - Verify Slack receives the message.
    - Approve it in Slack.
    - Confirm **If1** sees `data.approved = true` and posts the reply.

20. **Harden the workflow before production**
    - Replace trailing-space field names with clean names if possible.
    - Add error handling nodes or an error workflow.
    - Add fallback text if the AI response is empty.
    - Add moderation rules for sensitive reviews.
    - Consider storing review/reply logs in a database or sheet.

### Required Credentials
- **Google Business Profile credentials**
  - Needed for both trigger and reply nodes.
- **Groq API credentials**
  - Needed for the Groq Chat Model node.
- **Slack OAuth2 credentials**
  - Needed for the approval step.

### Input/Output Expectations
- **Input from trigger:** Google review object including reviewer info, comment, rating, and review identifier.
- **AI output:** text reply in `output`.
- **Slack approval output:** object containing `data.approved`.
- **Final action:** Google Business Profile review reply posted.

### Sub-workflow Setup
- No sub-workflows are used in this workflow.
- No Execute Workflow nodes are present.
- Only one entry point exists: **Fetch New Reviews**.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Product Review Monitoring & AI Response Bot: Automates Google review handling with real-time triggers, AI-generated responses, and optional Slack approval. | Workflow purpose |
| Positive reviews are auto-replied; negative reviews are sent to Slack for approval before posting. | Decision strategy |
| Setup requires Google Business Profile credentials, AI model credentials (Groq/OpenAI), Slack account connection, IF logic configuration, and correct review/reply field mapping. | Operational setup |
| Critical implementation issue: both Google reply nodes reference `Code in JavaScript1`, but that node does not exist in the workflow JSON. This must be corrected for the workflow to run successfully. | Important technical note |