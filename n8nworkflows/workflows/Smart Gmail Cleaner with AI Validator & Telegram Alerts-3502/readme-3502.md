Smart Gmail Cleaner with AI Validator & Telegram Alerts

https://n8nworkflows.xyz/workflows/smart-gmail-cleaner-with-ai-validator---telegram-alerts-3502


# Smart Gmail Cleaner with AI Validator & Telegram Alerts

### 1. Workflow Overview

This workflow automates cleaning a Gmail inbox by leveraging AI (Google Gemini) to validate whether emails should be deleted or skipped. It is designed for users who want to save time managing their inbox while maintaining control and transparency through Telegram notifications.

The workflow operates in adjustable 2-week batches, scanning emails, classifying them via AI, and then either deleting unwanted emails or labeling/skipping others. Notifications with AI reasoning are sent via Telegram for all actions, including errors.

**Logical Blocks:**

- **1.1 Initialization & Loop Control:** Setup and increment batch paging variables to process emails in 2-week intervals.
- **1.2 Gmail Email Retrieval:** Fetch emails from Gmail inbox based on the current batch date range and excluding previously skipped emails.
- **1.3 AI Email Classification:** Use Google Gemini AI to classify emails with confidence scores for unwanted, marketing, and spam categories, including a brief reason.
- **1.4 Decision Making:** Evaluate AI scores to decide whether to delete or skip emails.
- **1.5 Email Deletion & Labeling:** Delete unwanted emails or apply a label to skipped emails to avoid reprocessing.
- **1.6 Telegram Notifications:** Send Telegram messages reporting email deletions, skips, and AI errors.
- **1.7 Loop Aggregation & Continuation:** Aggregate results and prepare for the next batch iteration.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Loop Control

**Overview:**  
Initializes and increments the loop variable controlling the batch paging for 2-week intervals. This enables processing emails in sequential date ranges.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Initialize Loop Vars  
- Increment Loop Var  
- Forward Prev Page Num  

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Config: No parameters; triggers the workflow on demand.  
  - Inputs: None  
  - Outputs: Connects to Initialize Loop Vars  
  - Failures: None expected  
  - Notes: Used for testing or manual runs.

- **Initialize Loop Vars**  
  - Type: Set  
  - Role: Initializes the paging variable `page` to 0 at workflow start.  
  - Config: Sets `page` = 0  
  - Inputs: From manual trigger  
  - Outputs: Connects to Increment Loop Var  
  - Failures: None expected

- **Increment Loop Var**  
  - Type: Set  
  - Role: Increments the `page` variable by 1 for each iteration.  
  - Config:  
    - If `Forward Prev Page Num` executed, sets `page` = previous page + 1  
    - Else, uses initial `page` from `Initialize Loop Vars`  
  - Inputs: From Initialize Loop Vars or Forward Prev Page Num  
  - Outputs: Connects to Gmail Get Email  
  - Failures: Expression errors if referenced nodes missing or data malformed.

- **Forward Prev Page Num**  
  - Type: Set  
  - Role: Stores the current `page` value as `prevPage` for next iteration.  
  - Config: Sets `prevPage` = current `page` from Increment Loop Var  
  - Inputs: From Aggregate node (end of batch processing)  
  - Outputs: Connects back to Initialize Loop Vars (loop restart)  
  - Failures: None expected

---

#### 2.2 Gmail Email Retrieval

**Overview:**  
Fetches all emails from Gmail inbox within the current 2-week batch date range, excluding emails labeled as skipped to prevent reprocessing.

**Nodes Involved:**  
- Gmail Get Email  

**Node Details:**

- **Gmail Get Email**  
  - Type: Gmail node (OAuth2)  
  - Role: Retrieves emails filtered by date range and label exclusion.  
  - Config:  
    - Query filter:  
      - `before:` date calculated as current date minus (14 * page) days  
      - `after:` date calculated as current date minus (14 * (page + 1)) days  
      - Excludes emails with label `n8n-skipped`  
    - Returns all matching emails (no limit)  
    - Excludes spam and trash  
  - Inputs: From Increment Loop Var  
  - Outputs: Connects to AI Check Email  
  - Failures: Gmail API auth errors, rate limits, malformed query expressions.

---

#### 2.3 AI Email Classification

**Overview:**  
Classifies each email using Google Gemini AI model to determine confidence scores for unwanted, marketing, and spam categories, plus a brief reason for classification.

**Nodes Involved:**  
- AI Check Email  
- Google Gemini Chat Model  
- Unwanted Email Output Parser  

**Node Details:**

- **AI Check Email**  
  - Type: LangChain Agent node  
  - Role: Sends email data to AI for classification and receives structured output.  
  - Config:  
    - Prompt instructs AI to classify email with decimal confidence scores (0 to 1) for unwanted, marketing, spam.  
    - Baseline for deletion set at 0.5  
    - Includes email JSON data in prompt  
    - Output parser enabled for structured response  
  - Inputs: From Gmail Get Email  
  - Outputs:  
    - Main: If Unwanted Marketing or Spam (decision node)  
    - Error: Telegram Sent AI Error Notification  
  - Failures: AI API errors, timeout, malformed prompt or response.

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini model node  
  - Role: Provides AI language model backend for classification.  
  - Config: Uses model `models/gemini-1.5-flash`  
  - Inputs: From AI Check Email (ai_languageModel input)  
  - Outputs: To AI Check Email (ai_languageModel output)  
  - Failures: API auth errors, quota limits.

- **Unwanted Email Output Parser**  
  - Type: LangChain Output Parser (structured)  
  - Role: Parses AI response JSON into structured fields: emailId, confidence scores, briefReason, emailFrom.  
  - Config: JSON schema defines required fields and types.  
  - Inputs: From AI Check Email (ai_outputParser input)  
  - Outputs: To AI Check Email (ai_outputParser output)  
  - Failures: Parsing errors if AI response malformed.

---

#### 2.4 Decision Making

**Overview:**  
Evaluates AI confidence scores to decide if an email should be deleted or skipped.

**Nodes Involved:**  
- If Unwanted Marketing or Spam  

**Node Details:**

- **If Unwanted Marketing or Spam**  
  - Type: If node  
  - Role: Checks if any confidence score (unwanted, marketing, spam) exceeds 0.5 threshold.  
  - Config: Logical OR condition on:  
    - `isUnwantedConfidence > 0.5`  
    - `isMarketingConfidence > 0.5`  
    - `isSpamConfidence > 0.5`  
  - Inputs: From AI Check Email  
  - Outputs:  
    - True: GmailDeleteEmail (delete email)  
    - False: Gmail (apply label to skip)  
  - Failures: Expression evaluation errors if data missing.

---

#### 2.5 Email Deletion & Labeling

**Overview:**  
Deletes unwanted emails or applies a label to skipped emails to avoid rechecking in future runs.

**Nodes Involved:**  
- GmailDeleteEmail  
- Gmail  

**Node Details:**

- **GmailDeleteEmail**  
  - Type: Gmail node (OAuth2)  
  - Role: Deletes email by messageId.  
  - Config:  
    - Operation: delete  
    - MessageId from AI output `emailId`  
    - On error: continue (do not stop workflow)  
  - Inputs: From If Unwanted Marketing or Spam (true branch)  
  - Outputs:  
    - Success: Telegram Sent Email Deleted Notification  
    - Failure: Telegram Sent Delete Email Failed Notification  
  - Failures: Gmail API errors, auth issues, message not found.

- **Gmail**  
  - Type: Gmail node (OAuth2)  
  - Role: Adds label `n8n-skipped` to emails that are skipped (not deleted).  
  - Config:  
    - Operation: addLabels  
    - LabelIds: `Label_1321570453811516949` (assumed to be `n8n-skipped`)  
    - MessageId from AI output `emailId`  
  - Inputs: From If Unwanted Marketing or Spam (false branch)  
  - Outputs: Telegram Sent Email Not Deleted Notification  
  - Failures: Gmail API errors, label not found.

---

#### 2.6 Telegram Notifications

**Overview:**  
Sends Telegram messages reporting email deletions, skips, and AI errors with AI reasoning included for transparency.

**Nodes Involved:**  
- Telegram Sent Email Deleted Notification  
- Telegram Sent Email Not Deleted Notification  
- Telegram Sent AI Error Notification  
- Telegram Sent Delete Email Failed Notification  

**Node Details:**

- **Telegram Sent Email Deleted Notification**  
  - Type: Telegram node  
  - Role: Notifies user that an email was deleted, including sender and AI brief reason.  
  - Config:  
    - Text: `Email Deleted | {{ emailFrom }} | {{ briefReason }}`  
    - ChatId: `273696245` (user/channel ID)  
    - Append Attribution: false  
  - Inputs: From GmailDeleteEmail (success)  
  - Outputs: Success node  
  - Failures: Telegram API errors, invalid chat ID.

- **Telegram Sent Email Not Deleted Notification**  
  - Type: Telegram node  
  - Role: Notifies user that an email was skipped, including sender and AI brief reason.  
  - Config:  
    - Text: `Skipping Email | {{ emailFrom }} | {{ briefReason }}`  
    - ChatId: `273696245`  
    - Append Attribution: false  
  - Inputs: From Gmail (label applied)  
  - Outputs: Success node  
  - Failures: Telegram API errors.

- **Telegram Sent AI Error Notification**  
  - Type: Telegram node  
  - Role: Notifies user of AI classification errors with error details.  
  - Config:  
    - Text: `AI Error | Can't Check Email | Error: {{ JSON.stringify($json) }}`  
    - ChatId: `273696245`  
    - Append Attribution: false  
  - Inputs: From AI Check Email (error output)  
  - Outputs: Success node  
  - Failures: Telegram API errors.

- **Telegram Sent Delete Email Failed Notification**  
  - Type: Telegram node  
  - Role: Notifies user if email deletion failed.  
  - Config:  
    - Text: `Can't Delete Email`  
    - ChatId: `273696245`  
    - Append Attribution: false  
  - Inputs: From GmailDeleteEmail (error output)  
  - Outputs: Success node  
  - Failures: Telegram API errors.

---

#### 2.7 Loop Aggregation & Continuation

**Overview:**  
Aggregates batch results and loops back to process the next 2-week batch until no more emails remain.

**Nodes Involved:**  
- Success  
- Aggregate  
- Forward Prev Page Num  

**Node Details:**

- **Success**  
  - Type: NoOp  
  - Role: Marks successful completion of notification or deletion steps.  
  - Config: No parameters  
  - Inputs: From Telegram notification nodes  
  - Outputs: Aggregate node (only from Success node connected to Aggregate)  
  - Failures: None

- **Aggregate**  
  - Type: Aggregate  
  - Role: Aggregates all item data from previous steps to prepare for next iteration.  
  - Config: Aggregate all item data  
  - Inputs: From Success node (only from Telegram Sent Email Deleted Notification)  
  - Outputs: Forward Prev Page Num  
  - Failures: None

- **Forward Prev Page Num**  
  - (Already described in 2.1)  
  - Loops back to Initialize Loop Vars to continue processing next batch.

---

### 3. Summary Table

| Node Name                              | Node Type                     | Functional Role                          | Input Node(s)                      | Output Node(s)                               | Sticky Note                                                                                   |
|--------------------------------------|-------------------------------|----------------------------------------|----------------------------------|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’         | Manual Trigger                | Manual start of workflow                | None                             | Initialize Loop Vars                         |                                                                                               |
| Initialize Loop Vars                  | Set                          | Initialize paging variable `page`       | When clicking ‘Test workflow’    | Increment Loop Var                           |                                                                                               |
| Increment Loop Var                   | Set                          | Increment paging variable `page`        | Initialize Loop Vars, Forward Prev Page Num | Gmail Get Email                             |                                                                                               |
| Gmail Get Email                      | Gmail                        | Retrieve emails for current batch       | Increment Loop Var               | AI Check Email                               |                                                                                               |
| AI Check Email                      | LangChain Agent              | Classify emails with AI                  | Gmail Get Email                 | If Unwanted Marketing or Spam, Telegram Sent AI Error Notification |                                                                                               |
| Google Gemini Chat Model             | LangChain Google Gemini Model | AI language model backend                | AI Check Email (ai_languageModel) | AI Check Email (ai_languageModel)            |                                                                                               |
| Unwanted Email Output Parser         | LangChain Output Parser      | Parse AI classification output          | AI Check Email (ai_outputParser) | AI Check Email (ai_outputParser)              |                                                                                               |
| If Unwanted Marketing or Spam        | If                           | Decide delete or skip based on AI scores | AI Check Email                  | GmailDeleteEmail (true), Gmail (false)       |                                                                                               |
| GmailDeleteEmail                    | Gmail                        | Delete unwanted email                    | If Unwanted Marketing or Spam   | Telegram Sent Email Deleted Notification, Telegram Sent Delete Email Failed Notification |                                                                                               |
| Gmail                              | Gmail                        | Label skipped emails to avoid rechecking | If Unwanted Marketing or Spam   | Telegram Sent Email Not Deleted Notification |                                                                                               |
| Telegram Sent Email Deleted Notification | Telegram                    | Notify email deletion with AI reason    | GmailDeleteEmail (success)       | Success                                      |                                                                                               |
| Telegram Sent Email Not Deleted Notification | Telegram                    | Notify email skipped with AI reason     | Gmail (success)                 | Success                                      |                                                                                               |
| Telegram Sent AI Error Notification | Telegram                    | Notify AI classification errors         | AI Check Email (error)           | Success                                      |                                                                                               |
| Telegram Sent Delete Email Failed Notification | Telegram                    | Notify email deletion failure            | GmailDeleteEmail (error)         | Success                                      |                                                                                               |
| Success                            | NoOp                         | Marks successful completion              | Telegram notification nodes      | Aggregate                                    |                                                                                               |
| Aggregate                         | Aggregate                    | Aggregate batch results                   | Success                        | Forward Prev Page Num                         |                                                                                               |
| Forward Prev Page Num               | Set                          | Store current page for next iteration    | Aggregate                      | Initialize Loop Vars                          |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No parameters.

2. **Create Set Node to Initialize Loop Vars**  
   - Name: `Initialize Loop Vars`  
   - Type: Set  
   - Assign variable `page` = 0  
   - Connect input from Manual Trigger.

3. **Create Set Node to Increment Loop Var**  
   - Name: `Increment Loop Var`  
   - Type: Set  
   - Assign variable `page` with expression:  
     ```
     =($('Forward Prev Page Num').isExecuted) ? $('Forward Prev Page Num').first().json.prevPage + 1 : $('Initialize Loop Vars').first().json.page
     ```  
   - Connect input from Initialize Loop Vars and Forward Prev Page Num.

4. **Create Gmail Node to Get Emails**  
   - Name: `Gmail Get Email`  
   - Type: Gmail (OAuth2)  
   - Credentials: Connect your Gmail OAuth2 credentials.  
   - Operation: `getAll`  
   - Filters:  
     - Query:  
       ```
       =before:{{ $now.minus(14 * $('Increment Loop Var').first().json.page, 'days').format('yyyy/MM/dd') }} after:{{ $now.minus(14 * ($('Increment Loop Var').first().json.page + 1), 'days').format('yyyy/MM/dd') }} -label:n8n-skipped
       ```  
     - Include Spam and Trash: false  
   - Return All: true  
   - Connect input from Increment Loop Var.

5. **Create LangChain Agent Node for AI Classification**  
   - Name: `AI Check Email`  
   - Type: LangChain Agent  
   - Prompt:  
     ```
     Classify the email with decimal values (0 to 1) for isUnwantedConfidence, isMarketingConfidence, and isSpamConfidence, where 0 means clearly wanted (e.g., billing, invoices, orders, job applications, security) and 1 means clearly unwanted (e.g., promotions, setup reminders, irrelevant alerts); treat system-generated alerts or device activity (like sound played, device found, location pinged) as unwanted unless they are security-related; use 0.5 as the baseline for deletion and provide a concise briefReason explaining the classification.

     {{ JSON.stringify($json) }}
     ```  
   - Enable Output Parser.  
   - Connect input from Gmail Get Email.

6. **Create LangChain Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Type: LangChain Google Gemini Model  
   - Model Name: `models/gemini-1.5-flash`  
   - Connect as AI language model input/output to AI Check Email.

7. **Create LangChain Output Parser Node**  
   - Name: `Unwanted Email Output Parser`  
   - Type: LangChain Output Parser (Structured)  
   - Input Schema: JSON schema defining required fields: emailId, isUnwantedConfidence, isMarketingConfidence, isSpamConfidence, briefReason, emailFrom.  
   - Connect as AI output parser input/output to AI Check Email.

8. **Create If Node for Decision Making**  
   - Name: `If Unwanted Marketing or Spam`  
   - Type: If  
   - Condition: OR of  
     - `{{$json.output.isUnwantedConfidence}} > 0.5`  
     - `{{$json.output.isMarketingConfidence}} > 0.5`  
     - `{{$json.output.isSpamConfidence}} > 0.5`  
   - Connect input from AI Check Email.

9. **Create Gmail Node to Delete Email**  
   - Name: `GmailDeleteEmail`  
   - Type: Gmail (OAuth2)  
   - Operation: delete  
   - MessageId: `={{ $json.output.emailId }}`  
   - On Error: Continue  
   - Connect input from If node (true branch).

10. **Create Gmail Node to Add Label to Skipped Emails**  
    - Name: `Gmail`  
    - Type: Gmail (OAuth2)  
    - Operation: addLabels  
    - LabelIds: Use your label ID for `n8n-skipped` (e.g., `Label_1321570453811516949`)  
    - MessageId: `={{ $json.output.emailId }}`  
    - Connect input from If node (false branch).

11. **Create Telegram Nodes for Notifications**  
    - Credentials: Connect your Telegram API credentials.  
    - **Telegram Sent Email Deleted Notification**  
      - Text: `Email Deleted | {{ $('If Unwanted Marketing or Spam').item.json.output.emailFrom }} | {{ $('If Unwanted Marketing or Spam').item.json.output.briefReason }}`  
      - ChatId: Your Telegram chat/channel ID (e.g., `273696245`)  
      - Connect input from GmailDeleteEmail (success).  
    - **Telegram Sent Delete Email Failed Notification**  
      - Text: `Can't Delete Email`  
      - ChatId: same as above  
      - Connect input from GmailDeleteEmail (error).  
    - **Telegram Sent Email Not Deleted Notification**  
      - Text: `Skipping Email | {{ $('If Unwanted Marketing or Spam').item.json.output.emailFrom }} | {{ $('If Unwanted Marketing or Spam').item.json.output.briefReason }}`  
      - ChatId: same as above  
      - Connect input from Gmail (success).  
    - **Telegram Sent AI Error Notification**  
      - Text: `AI Error | Can't Check Email | Error: {{ JSON.stringify($json) }}`  
      - ChatId: same as above  
      - Connect input from AI Check Email (error).

12. **Create NoOp Node for Success**  
    - Name: `Success`  
    - Type: NoOp  
    - Connect inputs from all Telegram notification nodes (success and error outputs).

13. **Create Aggregate Node**  
    - Name: `Aggregate`  
    - Type: Aggregate  
    - Aggregate all item data  
    - Connect input from Success node.

14. **Create Set Node to Forward Prev Page Num**  
    - Name: `Forward Prev Page Num`  
    - Type: Set  
    - Assign `prevPage` = `={{ $('Increment Loop Var').first().json.page }}`  
    - Connect input from Aggregate node.

15. **Connect Forward Prev Page Num back to Initialize Loop Vars**  
    - This completes the loop for next batch processing.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Thanks to n8n's modular design, you can swap Google Gemini AI with other AI models like OpenAI or Claude without code changes, just node replacement. | Workflow description section.                                                                   |
| Telegram notifications can be replaced with other messaging platforms such as Discord, Slack, or email by swapping the Telegram nodes. | Workflow description section.                                                                   |
| Adjust the AI baseline threshold (currently 0.5) in the If node to control filtering sensitivity.    | Decision Making block.                                                                           |
| Gmail label `n8n-skipped` is used to mark emails that are skipped to avoid reprocessing. Ensure this label exists in your Gmail account. | Email Deletion & Labeling block.                                                                |
| Telegram Chat ID `273696245` is used for notifications; replace with your own chat or channel ID.    | Telegram Notifications block.                                                                   |
| Google Gemini API credentials require setup with Google PaLM API access.                             | AI Email Classification block.                                                                  |
| Gmail OAuth2 credentials must have appropriate scopes for reading, deleting, and labeling emails.   | Gmail Email Retrieval and Email Deletion & Labeling blocks.                                     |
| Manual trigger node allows testing the workflow on demand.                                          | Initialization block.                                                                           |

---

This document provides a comprehensive reference to understand, reproduce, and modify the "Smart Gmail Cleaner with AI Validator & Telegram Alerts" workflow. It covers all nodes, logic, configurations, and integration points for robust operation and easy customization.