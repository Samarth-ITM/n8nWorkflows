Automated Email Replies with GPT-4o, Outlook, and GoToHuman Approval

https://n8nworkflows.xyz/workflows/automated-email-replies-with-gpt-4o--outlook--and-gotohuman-approval-7049


# Automated Email Replies with GPT-4o, Outlook, and GoToHuman Approval

### 1. Workflow Overview

This workflow automates email replies using GPT-4o, Microsoft Outlook, and GoToHuman for human approval. It is designed for small and medium-sized businesses to efficiently handle incoming emails by generating AI-crafted responses, routing them for human review, and sending replies upon approval.

**Target Use Cases:**  
- Automated email triage and response generation  
- Human-in-the-loop approval for AI-generated content  
- Integration of Outlook email fetching and replying with AI and review system  

**Logical Blocks:**  
- **1.1 Input Reception & Email Filtering:** Receive trigger, get today's date, and fetch Outlook emails received today excluding emails from self.  
- **1.2 Batch Processing:** Split incoming emails into manageable batches for sequential processing.  
- **1.3 AI Response Generation:** Use GPT-4o to generate short, personalized email replies based on email content.  
- **1.4 Human Review with GoToHuman:** Submit AI responses for human approval with a review template, then decide next steps based on approval status.  
- **1.5 Conditional Reply Sending:** If approved, send the AI-generated reply via Outlook; if not approved, loop or skip.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Email Filtering

- **Overview:**  
  This block triggers workflow execution, gets today’s date formatted for Outlook search, and fetches emails from Outlook received today excluding those from the automated user to avoid self-replies.

- **Nodes Involved:**  
  - Test (Manual Trigger)  
  - Output Today's Date (Code)  
  - Microsoft Outlook  

- **Node Details:**  

  - **Test (Manual Trigger):**  
    - Type: Manual trigger node to start the workflow manually for testing or scheduling via other triggers.  
    - Config: No parameters configured.  
    - Inputs: None.  
    - Outputs: Connects to "Output Today's Date".  
    - Edge Cases: Manual trigger only; no issues expected here.  

  - **Output Today's Date (Code):**  
    - Type: Code node (JavaScript) to generate today’s date string formatted as `yyyy-mm-dd`.  
    - Config: Sets hours to zero to normalize, converts to ISO string, extracts date part.  
    - Key Expression:  
      ```js
      const today = new Date();
      today.setHours(0, 0, 0, 0);
      return [{ json: { searchQuery: `received:${today.toISOString().split('T')[0]}` } }];
      ```  
    - Inputs: From "Test".  
    - Outputs: To "Microsoft Outlook".  
    - Edge Cases: Timezone issues could arise if server time differs from Outlook mailbox time.  

  - **Microsoft Outlook:**  
    - Type: Microsoft Outlook node to fetch emails using a search filter.  
    - Config:  
      - Operation: `getAll` messages.  
      - Limit: 2 messages per execution (may be for testing).  
      - Filter: Searches emails received today excluding emails from `rbreen@ynteractive.com`.  
      - Credentials: OAuth2 configured with Microsoft Outlook account.  
    - Inputs: From "Output Today's Date".  
    - Outputs: To "Loop Over Items".  
    - Edge Cases:  
      - OAuth token expiration or refresh failure.  
      - API rate limits or connectivity issues.  
      - Search query syntax errors or empty inbox for the day.  

#### 2.2 Batch Processing

- **Overview:**  
  This block splits the fetched emails into batches, allowing the workflow to process emails one by one or in small groups.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  

- **Node Details:**  

  - **Loop Over Items:**  
    - Type: SplitInBatches node to process items sequentially or in defined batch sizes.  
    - Config: Default batch options (batch size not explicitly set, likely 1 by default).  
    - Inputs: From "Microsoft Outlook".  
    - Outputs: To "AI Agent: Create caption for linkedin" (on batch completion no output).  
    - Edge Cases:  
      - Empty input results in no processing.  
      - Large batch size may cause timeouts or memory issues.  

#### 2.3 AI Response Generation

- **Overview:**  
  This block uses GPT-4o via Langchain nodes to generate a short, personalized reply email based on the original email’s subject and body content.

- **Nodes Involved:**  
  - AI Agent: Create caption for linkedin (Langchain AI Agent)  
  - Sticky Note (content description)  

- **Node Details:**  

  - **AI Agent: Create caption for linkedin:**  
    - Type: Langchain agent node integrating OpenAI GPT-4o.  
    - Config:  
      - Uses a prompt combining subject and body of the email.  
      - System prompt: AI acts as a personal assistant specializing in automation, creates a short signed reply as "Robert Breen".  
      - Output format enforced as JSON array with `"body"` field.  
      - Output parser enabled to extract structured JSON.  
    - Inputs: From "Loop Over Items".  
    - Outputs: To "gotoHuman".  
    - Credentials: OpenAI API key configured.  
    - Key Expressions:  
      ```
      subject: {{ $json.subject }}
      Body: {{ $json.body.content }}
      ```
    - Edge Cases:  
      - OpenAI API limits or downtime.  
      - Malformed email content causing prompt generation issues.  
      - Output parser may fail if AI output does not match expected JSON schema.  

  - **Sticky Note:**  
    - Contains detailed explanation of AI prompt construction and signing instructions for the generated email.  

#### 2.4 Human Review with GoToHuman

- **Overview:**  
  Submits the AI-generated email response to GoToHuman for human review and approval using a specific review template, passing both the AI output and original email content.

- **Nodes Involved:**  
  - gotoHuman  
  - Sticky Note3 (content description)  

- **Node Details:**  

  - **gotoHuman:**  
    - Type: GoToHuman node for human review integration.  
    - Config:  
      - Maps AI output `"body"` to the review field `"email"`.  
      - Includes original email HTML content for context in `"OriginalEmail"`.  
      - Uses a predefined Review Template ID: `7nYfIV9vrefvuHcccGEB`.  
      - Executes once per item.  
    - Inputs: From "AI Agent: Create caption for linkedin".  
    - Outputs: To "If".  
    - Credentials: GoToHuman API credentials configured.  
    - Edge Cases:  
      - API authentication failure or rate limits.  
      - Mismatch between submitted data and review template fields causing validation errors.  
      - Delays or failures in human response.  

  - **Sticky Note3:**  
    - Explains that GoToHuman outputs must match review template schema, and approval status governs subsequent workflow branching.  

#### 2.5 Conditional Reply Sending

- **Overview:**  
  Based on GoToHuman review result, sends the AI-generated email reply via Outlook if approved or loops back otherwise.

- **Nodes Involved:**  
  - If (conditional node)  
  - Microsoft Outlook1 (Reply node)  
  - Loop Over Items (reused)  

- **Node Details:**  

  - **If:**  
    - Type: Conditional node evaluates approval status.  
    - Config: Checks if `response` field equals `"approved"`.  
    - Inputs: From "gotoHuman".  
    - Outputs:  
      - True branch: To "Microsoft Outlook1" to send reply.  
      - False branch: Loops back to "Loop Over Items" for potential reprocessing or skipping.  
    - Edge Cases:  
      - Missing or malformed response field leading to false negatives.  
      - Infinite loops if no exit condition on rejection.  

  - **Microsoft Outlook1:**  
    - Type: Microsoft Outlook node for replying to emails.  
    - Config:  
      - Operation: `reply` to the original message ID.  
      - Message content: Uses approved AI-generated email from GoToHuman.  
      - Credentials: Same OAuth2 Outlook account as input fetch.  
    - Inputs: From "If" true branch.  
    - Outputs: Back to "Loop Over Items" for next email processing.  
    - Edge Cases:  
      - Reply fails if message ID is invalid or deleted.  
      - API rate limits or connectivity issues.  

---

### 3. Summary Table

| Node Name                 | Node Type                                   | Functional Role                             | Input Node(s)             | Output Node(s)               | Sticky Note                                                                                  |
|---------------------------|---------------------------------------------|---------------------------------------------|---------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| Test                      | Manual Trigger                              | Workflow start trigger                       | -                         | Output Today's Date          |                                                                                              |
| Output Today's Date        | Code                                        | Generate today's date string for search     | Test                      | Microsoft Outlook            |                                                                                              |
| Microsoft Outlook         | Microsoft Outlook                           | Fetch emails received today excluding self  | Output Today's Date        | Loop Over Items              | "Use search query: `received:YYYY-MM-DD -from:rbreen@ynteractive.com` (update email)"        |
| Loop Over Items            | SplitInBatches                              | Batch processing of fetched emails          | Microsoft Outlook          | AI Agent: Create caption...  |                                                                                              |
| AI Agent: Create caption for linkedin | Langchain AI Agent                       | Generate AI short reply to email             | Loop Over Items            | gotoHuman                   | "Subject and body used in prompt; sign as Robert Breen; output as JSON array with 'body'"    |
| gotoHuman                 | GoToHuman Node                              | Submit AI reply for human approval           | AI Agent: Create caption...| If                          | "Submit using Review Template ID; map AI output and original email for review"               |
| If                        | If Node                                     | Check approval status                         | gotoHuman                  | Microsoft Outlook1 (true), Loop Over Items (false) | "If approved, send reply; else loop or skip"                                                |
| Microsoft Outlook1        | Microsoft Outlook                           | Send approved reply email                     | If (true)                  | Loop Over Items              |                                                                                              |
| Sticky Note               | Sticky Note                                 | AI prompt explanation                         | -                         | -                           | "Generate AI response with system prompt; sign as Robert Breen"                             |
| Sticky Note1              | Sticky Note                                 | Initial workflow step explanation             | -                         | -                           | "Trigger, date filter, Outlook search query usage"                                          |
| Sticky Note2              | Sticky Note                                 | Branding and contact info                      | -                         | -                           | "Contact info: LinkedIn and email Robert Breen"                                             |
| Sticky Note3              | Sticky Note                                 | GoToHuman review explanation                   | -                         | -                           | "GoToHuman approval flow and IF node decision logic"                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Test`  
   - Purpose: Start workflow manually or via scheduling.  

2. **Add Code Node to Output Today’s Date**  
   - Name: `Output Today's Date`  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const today = new Date();
     today.setHours(0, 0, 0, 0);
     const formattedDate = today.toISOString().split('T')[0];
     return [{ json: { searchQuery: `received:${formattedDate}` } }];
     ```  
   - Connect `Test` → `Output Today's Date`.  

3. **Add Microsoft Outlook Node to Fetch Emails**  
   - Name: `Microsoft Outlook`  
   - Operation: `getAll` messages  
   - Limit: 2 (adjust as needed)  
   - Filter: Use search query expression:  
     ```
     ={{ $json.searchQuery }} -from:rbreen@ynteractive.com
     ```  
   - Credentials: Configure Microsoft Outlook OAuth2 credentials.  
   - Connect `Output Today's Date` → `Microsoft Outlook`.  

4. **Add SplitInBatches Node**  
   - Name: `Loop Over Items`  
   - Purpose: Batch process emails one at a time (default batch size 1).  
   - Connect `Microsoft Outlook` → `Loop Over Items`.  

5. **Add Langchain AI Agent Node**  
   - Name: `AI Agent: Create caption for linkedin`  
   - Model: `gpt-4o`  
   - Prompt:  
     ```
     subject: {{ $json.subject }} Body: {{ $json.body.content }}
     ```  
   - System Message:  
     ```
     You are a personal assistant helping respond to emails. I am an AI automation expert specializing in helping small and medium size businesses automate processes. Create a short response to the email that you are analyzing.
     The email that you write should be written to the sender who sent the email to us.
     Sign the email as Robert Breen.
     Output the data like this:
     [
       {
         "body": ""
       }
     ]
     ```  
   - Enable Output Parser with JSON schema example:  
     ```json
     [
       {
         "body": ""
       }
     ]
     ```  
   - Credentials: Configure OpenAI API credentials.  
   - Connect `Loop Over Items` → `AI Agent: Create caption for linkedin`.  

6. **Add GoToHuman Node**  
   - Name: `gotoHuman`  
   - Review Template ID: `7nYfIV9vrefvuHcccGEB` (must be created in GoToHuman platform)  
   - Fields Mapping:  
     - `email`: `={{ $json.output[0].body }}` (AI-generated reply)  
     - `OriginalEmail`: `={{ $('Microsoft Outlook').item.json.body.content }}` (original email HTML)  
   - Credentials: Configure GoToHuman API credentials.  
   - Connect `AI Agent: Create caption for linkedin` → `gotoHuman`.  

7. **Add If Node for Approval Decision**  
   - Name: `If`  
   - Condition: Check if `={{ $json.response }} == 'approved'`  
   - Connect `gotoHuman` → `If`.  

8. **Add Microsoft Outlook Node to Reply**  
   - Name: `Microsoft Outlook1`  
   - Operation: `reply` to message ID  
   - Message ID: `={{ $('Microsoft Outlook').item.json.id }}`  
   - Message: `={{ $json.responseValues.email.value }}` (approved AI reply)  
   - Credentials: Same Microsoft Outlook OAuth2 credentials as before.  
   - Connect `If` (true branch) → `Microsoft Outlook1`.  

9. **Connect Reply Node Back to Batch Node**  
   - Connect `Microsoft Outlook1` → `Loop Over Items` to continue processing next email.  

10. **Connect If Node False Branch Back to Batch Node**  
    - Connect `If` (false branch) → `Loop Over Items` to skip or handle unapproved replies.  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| 🤖 AI-Powered Outlook Email Assistant with GoToHuman Approval Workflow                                | Workflow branding and contact: [LinkedIn Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/) |
| Message email automation expert at **robert@ynteractive.com** for support or customization inquiries. | Contact email for author and workflow support.                                                      |
| Use GoToHuman review template ID to manage approval workflow                                         | GoToHuman review template must be pre-configured to match fields submitted via the workflow.        |
| Outlook search query format example: `received:YYYY-MM-DD -from:youremail@example.com`                | Adjust your email address in the search query to avoid replying to self.                            |
| OpenAI GPT-4o model is used for AI response generation; ensure API key has access to this model.      | OpenAI account with GPT-4o access required.                                                         |

---

**Disclaimer:** The provided text is generated from an automated n8n workflow setup. It complies with all content policies and contains no illegal or protected content. All data handled is legal and publicly accessible.