Sort Gmail Emails with GPT-4o into Action Required and No Action Labels

https://n8nworkflows.xyz/workflows/sort-gmail-emails-with-gpt-4o-into-action-required-and-no-action-labels-4053


# Sort Gmail Emails with GPT-4o into Action Required and No Action Labels

### 1. Workflow Overview

This workflow automates the classification and organization of unread Gmail emails using GPT-4o (or GPT-4o-mini). Its main goal is to reduce inbox clutter by sorting emails into two categories:

- **Action Required**: Emails that need user attention such as replying, reviewing, scheduling, or responding.
- **No Action**: Informational or promotional emails that do not require any follow-up.

The workflow runs on a scheduled interval, fetches unread emails, classifies them via GPT-4o, then applies Gmail labels accordingly and removes the Inbox label to keep the primary inbox clean without deleting or archiving any emails.

**Logical blocks:**

- **1.1 Trigger and Email Fetching:** Schedule trigger initiates the process on a customizable schedule; unread emails are retrieved from Gmail.
- **1.2 AI-Based Classification:** Emails are fed into a GPT-4o-powered classifier node that determines if each email requires action or not.
- **1.3 Label Management:** Based on classification, appropriate Gmail labels ("Action Required" or "No Action") are applied; the Inbox label is removed from all processed emails to declutter the primary inbox view.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Email Fetching

- **Overview:** This block initiates the workflow on a schedule and retrieves unread Gmail messages.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Emails

##### Schedule Trigger
- **Type:** Schedule Trigger (n8n built-in)
- **Role:** Starts workflow execution at defined intervals.
- **Configuration:** Default interval set to run periodically (empty interval array defaults to 1 minute or manual trigger if not otherwise configured).
- **Expressions/Variables:** None.
- **Connections:** Output → Get Emails.
- **Version:** 1.2
- **Potential Failures:** Misconfiguration of schedule; node downtime or n8n scheduler issues.

##### Get Emails
- **Type:** Gmail node (Get All operation)
- **Role:** Fetches all unread emails from Gmail inbox.
- **Configuration:**  
  - Filters: readStatus = "unread"  
  - Operation: getAll (fetch all matching emails)
- **Expressions/Variables:** None.
- **Connections:** Output → Email Classifier.
- **Credentials:** Gmail OAuth2 credentials required and configured.
- **Version:** 2.1
- **Potential Failures:** Gmail API rate limiting, OAuth expiration, connectivity issues, empty inbox scenario (no emails fetched).

---

#### 1.2 AI-Based Classification

- **Overview:** Classifies each email snippet using GPT-4o-mini to determine if it requires action or not.
- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Email Classifier

##### OpenAI Chat Model
- **Type:** Langchain OpenAI Chat Model node
- **Role:** Provides GPT-4o-mini access for classification.
- **Configuration:**  
  - Model: "gpt-4o-mini"  
  - No additional options configured.
- **Expressions/Variables:** None.
- **Connections:** AI language model output → Email Classifier (ai_languageModel input).
- **Credentials:** OpenAI API key configured.
- **Version:** 1.2
- **Potential Failures:** API key invalid or expired, rate limits, network issues, OpenAI service downtime.

##### Email Classifier
- **Type:** Langchain Text Classifier node
- **Role:** Uses GPT-4o-mini to categorize emails into "Action" or "No Action".
- **Configuration:**  
  - Input Text: email snippet (`{{$json.snippet}}`)  
  - Categories:  
    - "Action": Requires a response or follow-up (reply, review, schedule, decision, task, meeting).  
    - "No Action": Informational/promotional, no response needed (newsletters, confirmations, updates).
- **Expressions/Variables:** Input text dynamically set from each email snippet.
- **Connections:**  
  - Output 1 ("Action") → Add "Action Required" Label  
  - Output 2 ("No Action") → Add "No Action" Label
- **Version:** 1
- **Potential Failures:** Classification errors due to ambiguous email content, API timeouts, expression errors if snippet missing, model misinterpretation.

---

#### 1.3 Label Management

- **Overview:** Applies Gmail labels based on classification results and removes the Inbox label to clean up the inbox.
- **Nodes Involved:**  
  - Add "Action Required" Label  
  - Remove "Inbox" Label (Action)  
  - Add "No Action" Label  
  - Remove "Inbox" Label (No Action)

##### Add "Action Required" Label
- **Type:** Gmail node (Add Labels operation)
- **Role:** Labels emails classified as "Action" with the "Action Required" Gmail label.
- **Configuration:**  
  - LabelIds: ["Label_5412178974275757474"] (internal Gmail label ID for "Action Required")  
  - MessageId: Email's unique ID (`{{$json.id}}`)
- **Expressions/Variables:** MessageId dynamically from email JSON.
- **Connections:** Output → Remove "Inbox" Label (Action).
- **Credentials:** Gmail OAuth2.
- **Version:** 2.1
- **Potential Failures:** Invalid label ID, Gmail API errors, OAuth token expiration.

##### Remove "Inbox" Label (Action)
- **Type:** Gmail node (Remove Labels operation)
- **Role:** Removes the Inbox label from emails labeled as "Action Required".
- **Configuration:**  
  - LabelIds: ["INBOX"]  
  - MessageId: `{{$json.id}}`
- **Expressions/Variables:** MessageId dynamically.
- **Connections:** None (terminal).
- **Credentials:** Gmail OAuth2.
- **Version:** 2.1
- **Potential Failures:** API errors, label removal failures, OAuth issues.

##### Add "No Action" Label
- **Type:** Gmail node (Add Labels operation)
- **Role:** Labels emails classified as "No Action" with the "No Action" Gmail label.
- **Configuration:**  
  - LabelIds: ["Label_1906200659860958692"] (internal Gmail label ID for "No Action")  
  - MessageId: `{{$json.id}}`
- **Expressions/Variables:** MessageId dynamically.
- **Connections:** Output → Remove "Inbox" Label (No Action).
- **Credentials:** Gmail OAuth2.
- **Version:** 2.1
- **Potential Failures:** Same as above for Add Label.

##### Remove "Inbox" Label (No Action)
- **Type:** Gmail node (Remove Labels operation)
- **Role:** Removes Inbox label from emails labeled "No Action".
- **Configuration:**  
  - LabelIds: ["INBOX"]  
  - MessageId: `{{$json.id}}`
- **Expressions/Variables:** MessageId dynamically.
- **Connections:** None (terminal).
- **Credentials:** Gmail OAuth2.
- **Version:** 2.1
- **Potential Failures:** Same as above for Remove Label.

---

#### Additional Node

##### Sticky Note
- **Type:** Sticky Note (UI element only)
- **Role:** Provides a node-by-node breakdown overview inside the workflow editor for user reference.
- **Content:** Explains the purpose of each functional block and key nodes.
- **Connections:** None.
- **Version:** 1
- **Potential Failures:** None (non-executable).

---

### 3. Summary Table

| Node Name                 | Node Type                       | Functional Role                          | Input Node(s)          | Output Node(s)                          | Sticky Note                                                                                          |
|---------------------------|--------------------------------|----------------------------------------|------------------------|----------------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                | Initiates workflow on schedule         |                        | Get Emails                            | 🧱 NODE-BY-NODE BREAKDOWN ⏰ 1. Schedule Trigger Runs the workflow on a set interval.               |
| Get Emails                | Gmail                          | Fetch unread Gmail emails               | Schedule Trigger       | Email Classifier                      | 📥 2. Get Emails Fetches unread emails from your Gmail inbox.                                      |
| OpenAI Chat Model          | Langchain OpenAI Chat Model    | Provides GPT-4o-mini model access       |                        | Email Classifier (ai_languageModel)  | 🧠 3. Email Classifier Classifies each email as Action or No Action using GPT-4o.                  |
| Email Classifier           | Langchain Text Classifier      | Classify emails as Action or No Action  | Get Emails, OpenAI Chat Model | Add "Action Required" Label, Add "No Action" Label |                                                                                                    |
| Add "Action Required" Label| Gmail                          | Label emails requiring action           | Email Classifier       | Remove "Inbox" Label (Action)         | 🏷️ 4. Add "Action Required" Label Adds the Action Required label to actionable emails.             |
| Remove "Inbox" Label (Action)| Gmail                        | Remove Inbox label from Action emails   | Add "Action Required" Label |                                      | 🧹 5. Remove "Inbox" Label (Action) Removes the INBOX label from emails marked as Action.           |
| Add "No Action" Label      | Gmail                          | Label emails with no action needed      | Email Classifier       | Remove "Inbox" Label (No Action)      | 🏷️ 6. Add "No Action" Label Adds the No Action label to emails requiring no response.              |
| Remove "Inbox" Label (No Action)| Gmail                     | Remove Inbox label from No Action emails| Add "No Action" Label  |                                      | 🧹 7. Remove "Inbox" Label (No Action) Removes the INBOX label from emails marked as No Action.     |
| Sticky Note                | Sticky Note                    | Provides workflow node breakdown notes  |                        |                                        | See above content.                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add Schedule Trigger node:**  
   - Set the interval to desired frequency (e.g., every hour or daily).  
   - This node will start your workflow on schedule.

3. **Add Gmail node (Get All operation):**  
   - Name: "Get Emails"  
   - Operation: getAll  
   - Filter: readStatus = "unread" to fetch only unread emails.  
   - Connect the Schedule Trigger output to this node input.  
   - Set up Gmail OAuth2 credentials with appropriate scopes (read and modify emails).

4. **Add Langchain OpenAI Chat Model node:**  
   - Name: "OpenAI Chat Model"  
   - Model: Select "gpt-4o-mini" or "gpt-4o" depending on your OpenAI subscription.  
   - Connect this node as AI language model input for the classifier node.  
   - Configure OpenAI API credentials with a valid API key.

5. **Add Langchain Text Classifier node:**  
   - Name: "Email Classifier"  
   - Input Text: Set to `{{$json.snippet}}` to classify email snippet.  
   - Categories:  
     - Action: Description includes replying, reviewing, scheduling, decision-making, task completion, or meetings.  
     - No Action: Informational/promotional with no required response.  
   - Link the "Get Emails" node main output to this node input.  
   - Link the OpenAI Chat Model node to this node’s AI language model input.

6. **Add Gmail nodes for labeling:**  
   - **Add "Action Required" Label:**  
     - Operation: addLabels  
     - LabelIds: Use the Gmail label ID for "Action Required" (create this label in Gmail beforehand).  
     - MessageId: `{{$json.id}}`  
     - Connect the first output of Email Classifier (Action) to this node.

   - **Remove "Inbox" Label (Action):**  
     - Operation: removeLabels  
     - LabelIds: ["INBOX"]  
     - MessageId: `{{$json.id}}`  
     - Connect output of Add "Action Required" Label to this node.

   - **Add "No Action" Label:**  
     - Operation: addLabels  
     - LabelIds: Use the Gmail label ID for "No Action" (create this label as well).  
     - MessageId: `{{$json.id}}`  
     - Connect the second output of Email Classifier (No Action) to this node.

   - **Remove "Inbox" Label (No Action):**  
     - Operation: removeLabels  
     - LabelIds: ["INBOX"]  
     - MessageId: `{{$json.id}}`  
     - Connect output of Add "No Action" Label to this node.

7. **Set all Gmail nodes to use the same Gmail OAuth2 credentials.**

8. **Optionally add a Sticky Note node with the workflow overview for documentation purposes.**

9. **Activate the workflow and test.**

10. **Create Gmail labels "Action Required" and "No Action" in your Gmail account before running the workflow to ensure label IDs correspond correctly.**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Create Gmail labels "Action Required" and "No Action" before running to ensure correct labeling.| Gmail UI > Settings > Labels                                                                         |
| OpenAI GPT-4o or GPT-4o-mini model is required; check your OpenAI subscription and API access. | https://platform.openai.com/docs/models                                                               |
| Workflow keeps all emails in your account; it only removes the Inbox label for visual clarity.| Avoids email deletion or archiving—safe organization method.                                           |
| For Gmail OAuth2 setup in n8n, ensure scopes allow reading and modifying labels and emails.   | https://developers.google.com/gmail/api/auth/scopes                                                   |
| Sticky Note node provides inline documentation for easier maintenance and understanding.      | Helpful for team collaboration or future workflow updates.                                            |

---

**Disclaimer:** The provided content is based exclusively on an n8n automation workflow. It complies fully with all applicable content policies and contains no illegal or sensitive data. All processed emails are public or user-authorized.