Classify Emails and Send Replies with GPT-4o and gotoHuman for Supervision

https://n8nworkflows.xyz/workflows/classify-emails-and-send-replies-with-gpt-4o-and-gotohuman-for-supervision-7301


# Classify Emails and Send Replies with GPT-4o and gotoHuman for Supervision

### 1. Workflow Overview

This n8n workflow automates the classification, drafting, and human-supervised replying of incoming Gmail emails using OpenAI GPT-4o-mini and gotoHuman for review. It targets use cases where organizations want to streamline email handling by automatically categorizing emails, generating draft replies, and involving humans only for approval or refinement, thus balancing automation and quality control.

The workflow is structured into these logical blocks:

- **1.1 Input Reception:** Listens for new Gmail emails.
- **1.2 AI Classification:** Uses GPT-4o-mini to classify emails by category, importance, and reply necessity.
- **1.3 Decision Routing:** Decides if an email requires a reply, human attention, or no action.
- **1.4 AI Drafting:** If reply needed, generates a draft response using GPT-4o-mini.
- **1.5 Human Review:** Sends drafts to humans via gotoHuman for approval, rejection, or retry.
- **1.6 Final Actions:** Sends approved replies via Gmail or ends flow when no reply required.

Supporting nodes include structured output parsers for AI responses, prompt setters, switches for decision-making, and sticky notes for documentation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Detects new incoming emails in Gmail to trigger the workflow.
- **Nodes Involved:**  
  - New Email (Gmail Trigger)
- **Node Details:**  
  - **New Email**  
    - Type: Gmail Trigger  
    - Configuration: Polls Gmail every minute, fetches full email details (not simple)  
    - Inputs: External email events  
    - Outputs: Email JSON data containing sender, recipient, subject, and body  
    - Edge Cases: Gmail API authentication errors, rate limits, missing permissions  
    - Credentials: Gmail OAuth2 required  

#### 2.2 AI Classification

- **Overview:** Sends the incoming email content to GPT-4o-mini to classify it into predefined categories and decide if a reply is needed or if the email is important.  
- **Nodes Involved:**  
  - AI Classifier (LangChain Agent)  
  - OpenAI Chat Model (GPT-4o-mini)  
  - Structured Output1 (Output Parser)  
  - Sticky Note (comments on classification)  
- **Node Details:**  
  - **AI Classifier**  
    - Type: LangChain Agent  
    - Configuration:  
      - Input text composed of From, To, Subject, Body from the "New Email" node  
      - System message instructs to classify into categories and flag reply/importance  
      - Uses a prompt with output parser expecting JSON with fields: `classifiedAs`, `needsReply`, `important`  
    - Inputs: Email JSON  
    - Outputs: Structured JSON classification result  
    - Edge Cases: GPT timeout, malformed email content, parsing errors  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Model: gpt-4o-mini  
    - Used as language model provider for AI Classifier  
  - **Structured Output1**  
    - Parses AI Classifier output into structured JSON  
- Sticky Note Content (covers classification block):  
  > "## Classify Email  
  > We ask AI to classify the received email into one of the categories defined in the prompt. It also determines whether a reply is needed and it is deemed important."

#### 2.3 Decision Routing

- **Overview:** Routes emails based on AI classification results to decide if reply drafting, human review, or no action is needed.  
- **Nodes Involved:**  
  - Needs Human Attention? (Switch)  
  - Not important (noOp)  
  - No reply (noOp)  
- **Node Details:**  
  - **Needs Human Attention?**  
    - Type: Switch  
    - Rules:  
      - "Needs reply": if `needsReply` is true  
      - "Needs no reply but important": if `needsReply` false but `important` true  
      - "Don't bother": if neither reply nor important  
    - Inputs: AI Classifier structured output  
    - Outputs: Routes to respective processing paths  
  - **Not important**  
    - Type: noOp  
    - Role: Terminates flow for unimportant emails not needing reply  
  - **No reply**  
    - Type: noOp  
    - Role: Terminates flow for emails that do not require a reply  
- Edge Cases: Boolean evaluation errors if AI output malformed  

#### 2.4 AI Drafting

- **Overview:** Drafts an email reply using GPT-4o-mini when a reply is needed, incorporating a helpful prompt.  
- **Nodes Involved:**  
  - Set Prompt (Set)  
  - AI Email Writer (LangChain Agent)  
  - OpenAI Chat Model2 (GPT-4o-mini)  
  - Structured Output (Output Parser)  
  - Sticky Note (comments on drafting)  
- **Node Details:**  
  - **Set Prompt**  
    - Type: Set node  
    - Assigns a static, detailed prompt instructing the AI to draft a reply matching the original email style, with placeholders if needed  
  - **AI Email Writer**  
    - Type: LangChain Agent  
    - Input text composed by interpolating original email fields and prompt from "Set Prompt"  
    - Uses GPT-4o-mini to generate draft  
    - Has output parser expecting JSON with key `textEmailDraft` containing the draft reply body  
  - **OpenAI Chat Model2**  
    - Same model as classifier, used here for drafting  
  - **Structured Output**  
    - Parses AI Email Writer output into structured JSON  
- Sticky Note Content (covers drafting block):  
  > "## Draft Email Response  
  > If it was determined that a reply is required, we ask AI to draft a response. It might incl. placeholders for a human to replace during review."

#### 2.5 Human Review

- **Overview:** Sends the AI-generated draft to a human reviewer via gotoHuman for approval, rejection, or retry with edited prompts.  
- **Nodes Involved:**  
  - gotoHuman: Human review (gotoHuman node)  
  - Human response (Switch)  
  - Set (edited) prompt (Set)  
  - Sticky Notes (comments and branding)  
- **Node Details:**  
  - **gotoHuman: Human review**  
    - Type: gotoHuman node for human task distribution  
    - Sends email content, classification, AI draft, and sender info for human review  
    - Supports review template selection and update for reviewId if draft is edited  
    - Inputs: Draft from AI Email Writer, classification, original email content  
    - Outputs: Human responses including approval, rejection, or retry requests  
    - Edge Cases: API key misconfiguration, communication errors, user delays  
  - **Human response**  
    - Type: Switch  
    - Rules:  
      - "Rejected": If response = "rejected" → route to No reply (end flow)  
      - "Approved": If response = "approved" → route to send reply node  
      - "Retry": If response type = "chat" → route back to AI Email Writer with updated prompt for iteration  
  - **Set (edited) prompt**  
    - Type: Set node  
    - Captures updated prompt from human review for retrying AI draft generation  
- Sticky Note Content (covers human review block):  
  > "## Human Review  
  > Ask a human to review the received email and approve any AI-drafted response via gotoHuman.  
  > Drafts can be manually edited or retried in gotoHuman (Retries loop back to the AI email writer node)."  
  >  
  > "## Human asked for Retry  
  > In gotoHuman the reviewer clicked retry or edited the prompt to iterate on the LLM output."  
  >  
  > ![gotoHuman logo](https://cdn.prod.website-files.com/6605a2979ff17b2cd1939cd4/6877ff9cd93f480ff6eb4def_677c04b5cd6a77eb434526bf9c0eaaca_gotoHuman%20full%20logo.svg)  
  >  
  > ![gotoHuman - Reviewing](https://cdn.prod.website-files.com/6605a2979ff17b2cd1939cd4/689b5e250227be98ca4d11e2_gth-review.JPG)  
  >  
  > ![gotoHuman - Retrying with AI](https://cdn.prod.website-files.com/6605a2979ff17b2cd1939cd4/689b5eb249cfe598f6ef0305_gth-chat.JPG)  

#### 2.6 Final Actions - Sending Reply

- **Overview:** Sends the approved AI-generated email reply as a Gmail reply to the original email’s thread.  
- **Nodes Involved:**  
  - Reply to thread (Gmail)  
  - Sticky Note (comments on sending)  
- **Node Details:**  
  - **Reply to thread**  
    - Type: Gmail node (send email)  
    - Operation: Reply to existing message by messageId from "New Email"  
    - Message: Uses human-approved draft from gotoHuman response  
    - EmailType: Text  
    - Edge Cases: Gmail API limits, messageId invalid, auth errors  
- Sticky Note Content (covers sending block):  
  > "## Send approved Reply  
  > Send the approved response as a reply to the email thread"

---

### 3. Summary Table

| Node Name            | Node Type                                | Functional Role             | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                       |
|----------------------|----------------------------------------|----------------------------|------------------------|----------------------------|-------------------------------------------------------------------------------------------------|
| New Email            | Gmail Trigger                          | Input Reception            | -                      | AI Classifier               | "## AI Email Assistant 🤖📧\n### Classifies, Drafts and Sends Human-Approved Reply\n\nThis shows an AI-based email assistant..." |
| AI Classifier        | LangChain Agent                       | AI Classification         | New Email               | Needs Human Attention?      | "## Classify Email\nWe ask AI to classify the received email into one of the categories..."      |
| OpenAI Chat Model    | LangChain OpenAI Chat Model           | LM provider for Classifier | -                      | AI Classifier (lmChat)      |                                                                                                 |
| Structured Output1   | LangChain Output Parser (Structured) | Parse AI Classifier output | AI Classifier           | Needs Human Attention?      |                                                                                                 |
| Needs Human Attention? | Switch                              | Decision Routing           | AI Classifier           | Set Prompt, gotoHuman, Not important |                                                                                         |
| Not important        | noOp                                  | End flow for unimportant   | Needs Human Attention?  | -                          |                                                                                                 |
| Set Prompt           | Set                                   | Set AI drafting prompt     | Needs Human Attention?  | AI Email Writer             | "## Draft Email Response\nIf it was determined that a reply is required, we ask AI to draft..." |
| AI Email Writer      | LangChain Agent                       | AI Drafting                | Set Prompt              | gotoHuman                  |                                                                                                 |
| OpenAI Chat Model2   | LangChain OpenAI Chat Model           | LM provider for drafting   | -                      | AI Email Writer (lmChat)    |                                                                                                 |
| Structured Output    | LangChain Output Parser (Structured) | Parse AI draft output      | AI Email Writer         | gotoHuman                  |                                                                                                 |
| gotoHuman: Human review | gotoHuman Node                    | Human Review               | AI Email Writer         | Human response             | "## Human Review\nAsk a human to review the received email and approve any AI-drafted response..." |
| Human response       | Switch                               | Handle human decision      | gotoHuman               | No reply, Reply to thread, Set (edited) prompt |                                                                                     |
| No reply             | noOp                                  | End flow for no reply      | Human response          | -                          |                                                                                                 |
| Reply to thread      | Gmail                                | Send approved reply        | Human response          | -                          | "## Send approved Reply\nSend the approved response as a reply to the email thread"             |
| Set (edited) prompt  | Set                                   | Capture human-edited prompt | Human response          | AI Email Writer             | "## Human asked for Retry\nIn gotoHuman the reviewer clicked retry or edited the prompt..."      |
| Sticky Note          | Sticky Note                          | Documentation / Branding   | -                      | -                          | See individual sticky notes content above                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Name: `New Email`  
   - Type: Gmail Trigger  
   - Credentials: Connect your Gmail account via OAuth2  
   - Polling: Every minute  
   - Parameters: Full email details (not simple)  

2. **Create LangChain OpenAI Chat Model node (AI Classifier LM)**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - No special options needed  
   - No credentials embedded here (set in LangChain Agent nodes)  

3. **Create LangChain Agent node for classification**  
   - Name: `AI Classifier`  
   - Type: LangChain Agent  
   - Language Model: Select `OpenAI Chat Model` node  
   - Input text: Compose from `New Email` fields: From, To, Subject, Body  
   - System message: Instruct AI to classify into categories (Customer Support, Partnership Inquiry, etc.), flag if reply needed and importance  
   - Output parser: Enable with schema expecting JSON keys: `classifiedAs`, `needsReply`, `important`  
   - Version: Use v2.2 or latest stable  

4. **Create LangChain Output Parser node (Structured Output1)**  
   - Name: `Structured Output1`  
   - Type: LangChain Output Parser (Structured)  
   - JSON Schema Example:  
     ```json
     {
       "classifiedAs": "support",
       "needsReply": true,
       "important": false
     }
     ```  
   - Connect output from `AI Classifier` to this parser  

5. **Create Switch node for routing**  
   - Name: `Needs Human Attention?`  
   - Type: Switch  
   - Rules (all Boolean strict):  
     - Needs reply: `{{ $json.output.needsReply === true }}`  
     - Needs no reply but important: `{{ !$json.output.needsReply && $json.output.important }}`  
     - Don't bother: `{{ !$json.output.needsReply && !$json.output.important }}`  

6. **Create noOp node for unimportant emails**  
   - Name: `Not important`  
   - Type: noOp  
   - Connect from "Needs no reply but important" output  

7. **Create noOp node for no reply needed**  
   - Name: `No reply`  
   - Type: noOp  
   - Connect from "Don't bother" output and from "Rejected" human response later  

8. **Create Set node for AI draft prompt**  
   - Name: `Set Prompt`  
   - Assign string field `prompt` with detailed instructions:  
     ```
     You are a helpful email assistant. Please draft a reply to the email passed by the user. Match the writing style of the received email. If you cannot answer questions or draft a complete reply, incl. placeholders. A user will review your draft after this and can fill in more info. Incl. the body of the email only, we are responding within the thread.
     ```  
   - Connect from "Needs reply" output of the Switch  

9. **Create LangChain OpenAI Chat Model node for drafting**  
   - Name: `OpenAI Chat Model2`  
   - Same model and setup as the classifier LM  

10. **Create LangChain Agent node for drafting**  
    - Name: `AI Email Writer`  
    - Language Model: Select `OpenAI Chat Model2`  
    - Input text: Compose from `New Email` fields and prompt from `Set Prompt` node  
    - System message: Use prompt field from `Set Prompt`  
    - Enable output parser with JSON schema expecting:  
      ```json
      {
        "textEmailDraft": "Hello Jack..."
      }
      ```  

11. **Create LangChain Output Parser node for draft**  
    - Name: `Structured Output`  
    - Connect output from `AI Email Writer`  

12. **Create gotoHuman node for human review**  
    - Name: `gotoHuman: Human review`  
    - Credentials: Create gotoHuman account, obtain API key, configure in n8n credentials  
    - Parameters:  
      - Map fields:  
        - `email` → original email as HTML text  
        - `sender` → original email sender  
        - `emailDraft` → AI draft from `Structured Output` (or default "No reply needed - FYI only")  
        - `classification` → classification from `AI Classifier`  
      - Review Template: Import template with ID `v81wzxwYoFYvWpmuIBgX` in gotoHuman and select it here  
      - Configure updateForReviewId to handle edited prompts for retries  
    - Connect from `AI Email Writer` node output  

13. **Create Switch node for human decision**  
    - Name: `Human response`  
    - Rules based on human reviewer response:  
      - "Rejected" → connect to `No reply` node  
      - "Approved" → connect to `Reply to thread` node  
      - "Retry" → connect to `Set (edited) prompt` node for re-drafting  

14. **Create Set node for edited prompt**  
    - Name: `Set (edited) prompt`  
    - Capture updated prompt text from human review (field `reviewToUpdate`)  
    - Connect output back to `AI Email Writer` node input to retry drafting  

15. **Create Gmail node to send reply**  
    - Name: `Reply to thread`  
    - Type: Gmail  
    - Credentials: Same Gmail OAuth2 credentials as `New Email`  
    - Operation: Reply  
    - Message: Use human-approved draft from `gotoHuman` output  
    - MessageId: Use original email thread ID from `New Email` node  
    - Connect from "Approved" output of `Human response`  

16. **Add noOp nodes to end flows as needed**  
    - For unimportant or no-reply paths, ensure flow terminates cleanly  

17. **Add Sticky Notes for documentation**  
    - Recreate sticky notes with text and images as per original workflow for clarity and branding  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| How to set up gotoHuman node before importing this template or some settings will be missing.                                                                                                                                                                    | Workflow Setup Instructions                                                                                    |
| Create gotoHuman account and copy API key.                                                                                                                                                                                                                       | gotoHuman Portal                                                                                                |
| Import gotoHuman review template with ID `v81wzxwYoFYvWpmuIBgX` for "Email Smart Reply."                                                                                                                                                                         | Review Template Import [gotoHuman UI Screenshot](https://cdn.prod.website-files.com/6605a2979ff17b2cd1939cd4/689484a8c752885e94ef212e_import-menu.JPG) |
| Requirements: gotoHuman account for human supervision, OpenAI account for classification and drafting, Gmail account for email handling.                                                                                                                        | Workflow Prerequisites                                                                                          |
| Branding images and screenshots illustrating gotoHuman review and retry process included as sticky notes.                                                                                                                                                         | Visual aids embedded in workflow sticky notes                                                                  |
| This workflow balances AI automation with human oversight to ensure quality email responses while reducing manual effort.                                                                                                                                         | Project Philosophy                                                                                              |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow, adhering strictly to content policies and containing no illegal, offensive, or protected elements. All processed data is legal and public.