Intelligent Email Organization with AI-Powered Content Classification for Gmail

https://n8nworkflows.xyz/workflows/intelligent-email-organization-with-ai-powered-content-classification-for-gmail-4557


# Intelligent Email Organization with AI-Powered Content Classification for Gmail

---
### 1. Workflow Overview

This workflow automates the labeling of incoming Gmail emails by leveraging AI-powered content classification via OpenAI. Its main purpose is to intelligently categorize emails into predefined labels based on their content, reducing manual inbox management and improving email organization at scale. The workflow targets users or organizations who want to automate email sorting with sophisticated natural language understanding, making sure emails receive the most appropriate label for streamlined processing.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Email Retrieval:** Periodically fetches recent Gmail messages.
- **1.2 Email Filtering:** Filters out emails that already have specific excluded labels, avoiding reprocessing.
- **1.3 Batch Processing:** Iterates over filtered emails in batches for manageable processing.
- **1.4 Email Detail Extraction:** Retrieves detailed content of each email for AI analysis.
- **1.5 AI Content Classification:** Uses OpenAI (via Langchain agent) to assign a single best-fit label to the email based on its content.
- **1.6 Label Management:** Checks if the AI-suggested label exists in the user's Gmail labels; creates it if missing.
- **1.7 Label Application:** Applies the appropriate label to the email in Gmail.
- **1.8 Workflow Continuation:** Prepares for the next batch or cycle.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Email Retrieval

- **Overview:**  
  Initiates the workflow every 2 minutes to fetch the latest Gmail messages (up to 20 per run), including read and unread emails.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Gmail - Get All Messages  

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Interval trigger node  
    - Configuration: Fires every 2 minutes  
    - Inputs: None (start node)  
    - Outputs: Connects to Gmail - Get All Messages  
    - Edge Cases: Workflow only triggers as scheduled; no retries if Gmail API is down; rate limits possible.

  - **Gmail - Get All Messages**  
    - Type: Gmail API node for message listing  
    - Configuration: Fetch 20 messages regardless of read status; returns full message metadata, not simplified  
    - Credentials: OAuth2 Gmail account connected  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Passes message list to Email Filtering node  
    - Edge Cases: API errors, rate limits, OAuth token expiry must be handled externally.

#### 2.2 Email Filtering

- **Overview:**  
  Excludes emails that already contain any labels from a defined exclusion list to prevent redundant processing.

- **Nodes Involved:**  
  - Filter Emails Without Excluded Labels (Code node)  

- **Node Details:**  
  - **Filter Emails Without Excluded Labels**  
    - Type: JavaScript code node  
    - Configuration: Filters out any email containing labels matching a hard-coded exclusion list of label IDs  
    - Input: Array of Gmail messages from previous node  
    - Output: Filtered array of emails without excluded labels  
    - Edge Cases: Label IDs must be up-to-date and accurate; if label IDs change in Gmail, filter may miss or wrongly exclude emails.

#### 2.3 Batch Processing

- **Overview:**  
  Splits filtered emails into batches for sequential detailed processing.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches node)  

- **Node Details:**  
  - **Loop Over Items**  
    - Type: Batch processing node  
    - Configuration: Default batch size (not explicitly limited here)  
    - Input: Filtered emails array  
    - Output: Emits one email per batch to detailed processing  
    - Edge Cases: Large volumes may increase workflow execution time; batching helps avoid timeout.

#### 2.4 Email Detail Extraction

- **Overview:**  
  Retrieves the full content of each email individually, then extracts essential fields (id, from, subject, body text) for AI classification.

- **Nodes Involved:**  
  - Gmail - Single Message  
  - If (conditional node)  
  - Merge  
  - Extract Email Data (Set node)  

- **Node Details:**  
  - **Gmail - Single Message**  
    - Type: Gmail API node for getting full email details  
    - Configuration: Uses message ID from batch item; returns full content (not simplified)  
    - Input: One email ID per batch  
    - Output: Full email JSON passed downstream  
    - Edge Cases: Message may get deleted or moved before retrieval; API limits apply.

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if a specific label ID is present (currently a tautology comparing the same label ID to itself - likely placeholder or redundant)  
    - Input: Email details from Gmail - Single Message  
    - Output: If true, continues; else merges  
    - Edge Cases: Condition logic unclear; may need revision.

  - **Merge**  
    - Type: Data merging node  
    - Configuration: Merges data streams from condition outcomes  
    - Input: From If node fallback and Gmail - Single Message node  
    - Output: Unified email data for extraction  
    - Edge Cases: Merge conflicts if data structure changes.

  - **Extract Email Data**  
    - Type: Set node  
    - Configuration: Extracts and assigns four key fields: id, from header, subject header, and email text body  
    - Input: Unified email data  
    - Output: Simplified structured data for AI processing  
    - Edge Cases: Email structure variability may cause missing fields; bodies with HTML or attachments not handled explicitly.

#### 2.5 AI Content Classification

- **Overview:**  
  Sends extracted email data to an AI model (OpenAI via Langchain agent) to classify the email with one of 17 predefined labels, returning only the label name.

- **Nodes Involved:**  
  - Categorize Email with AI (Langchain agent)  
  - Store AI Category (Set node)  
  - OpenAI Chat Model (OpenAI LM Chat node)  

- **Node Details:**  
  - **Categorize Email with AI**  
    - Type: Langchain agent node (AI reasoning)  
    - Configuration: Constructs prompt combining email from, subject, and body; instructs AI to pick exactly one label with strict output rules; uses a detailed system message defining label meanings and rules.  
    - Input: Email data from Extract Email Data  
    - Output: AI label output string  
    - Edge Cases: AI may return unexpected output if prompt malformed; rate limits and API errors possible.

  - **Store AI Category**  
    - Type: Set node  
    - Configuration: Stores AI output label into a field named "output"  
    - Input: AI classification result  
    - Output: Prepares label for existence check  
    - Edge Cases: Empty or invalid AI output may cause downstream errors.

  - **OpenAI Chat Model**  
    - Type: OpenAI Chat LM node  
    - Configuration: Uses GPT-4.1-nano model variant; no additional options; credentialed with OpenAI API key  
    - Input: Prompt from Categorize Email with AI  
    - Output: AI text completion  
    - Edge Cases: API key expiration, rate limiting.

#### 2.6 Label Management

- **Overview:**  
  Checks if the AI-assigned label exists in Gmail; if not, creates it before applying.

- **Nodes Involved:**  
  - List All Gmail Labels  
  - Check if Label Exists (Compare Datasets node)  
  - Apply New Label (Gmail label creation node)  

- **Node Details:**  
  - **List All Gmail Labels**  
    - Type: Gmail API node to fetch all labels  
    - Configuration: Returns all labels, no limit  
    - Input: Triggered after storing AI category  
    - Output: Label list for comparison  
    - Edge Cases: Large label sets may slow response.

  - **Check if Label Exists**  
    - Type: CompareDatasets node  
    - Configuration: Fuzzy compares AI output label against Gmail label names; merges on matching field  
    - Input: AI label and Gmail label list  
    - Output: Branches based on existence (exists / does not exist)  
    - Edge Cases: Fuzzy matching may cause false positives or misses; label naming must be consistent.

  - **Apply New Label**  
    - Type: Gmail API label creation node  
    - Configuration: Creates new label with AI output name  
    - Input: Triggered if label missing  
    - Output: New label ID for application  
    - Edge Cases: Gmail label creation limits; name conflicts.

#### 2.7 Label Application

- **Overview:**  
  Applies the determined (existing or newly created) label to the original email message.

- **Nodes Involved:**  
  - Create New Label (Gmail add label to message)  
  - Apply Label to Email (Gmail add label to message)  
  - Replace Me (NoOp node for workflow continuation)  

- **Node Details:**  
  - **Create New Label**  
    - Type: Gmail API node for adding label to message (after creating label)  
    - Configuration: Uses new label ID and email ID to tag message  
    - Input: New label ID from Apply New Label and email ID from Extract Email Data  
    - Output: Passes to Replace Me node  
    - Edge Cases: API errors if label/message not found.

  - **Apply Label to Email**  
    - Type: Gmail API node for adding label to message (existing label)  
    - Configuration: Uses existing label ID and email ID to tag message  
    - Input: Label ID from Check if Label Exists and email ID  
    - Output: Passes to Replace Me node  
    - Edge Cases: Same as Create New Label.

  - **Replace Me**  
    - Type: NoOp placeholder node  
    - Configuration: No operation, serves as endpoint or loop back placeholder  
    - Input: From label application nodes  
    - Output: Connects back to Loop Over Items for next batch  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name                    | Node Type                         | Functional Role                     | Input Node(s)                  | Output Node(s)              | Sticky Note                                                               |
|------------------------------|----------------------------------|-----------------------------------|-------------------------------|-----------------------------|---------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger                 | Triggers workflow every 2 minutes | None                          | Gmail - Get All Messages     |                                                                           |
| Gmail - Get All Messages      | Gmail API (getAll messages)      | Fetches recent emails              | Schedule Trigger              | Filter Emails Without Excluded Labels |                                                                           |
| Filter Emails Without Excluded Labels | Code (JS)                      | Filters out emails with excluded labels | Gmail - Get All Messages      | Loop Over Items             |                                                                           |
| Loop Over Items               | SplitInBatches                   | Processes emails one by one        | Filter Emails Without Excluded Labels | Gmail - Single Message       |                                                                           |
| Gmail - Single Message        | Gmail API (get message)          | Retrieves full email content       | Loop Over Items               | If, Merge                   |                                                                           |
| If                           | Conditional                     | Checks label presence (placeholder) | Gmail - Single Message        | Merge (on false)             |                                                                           |
| Merge                        | Merge                           | Merges email data streams          | If, Gmail - Single Message     | Extract Email Data          |                                                                           |
| Extract Email Data            | Set                            | Extracts key email fields          | Merge                        | Categorize Email with AI    |                                                                           |
| Categorize Email with AI      | Langchain Agent (AI)             | Classifies email with AI label     | Extract Email Data            | Store AI Category           |                                                                           |
| Store AI Category             | Set                            | Stores AI label output             | Categorize Email with AI      | Check if Label Exists, List All Gmail Labels |                                                                           |
| List All Gmail Labels         | Gmail API (get labels)           | Gets all Gmail labels              | Store AI Category            | Check if Label Exists       |                                                                           |
| Check if Label Exists         | CompareDatasets                 | Checks if AI label exists          | Store AI Category, List All Gmail Labels | Apply New Label (if no), Apply Label to Email (if yes) |                                                                           |
| Apply New Label              | Gmail API (create label)          | Creates label if missing            | Check if Label Exists         | Create New Label            |                                                                           |
| Create New Label             | Gmail API (add label to message) | Applies new label to email          | Apply New Label              | Replace Me                 |                                                                           |
| Apply Label to Email          | Gmail API (add label to message) | Applies existing label to email     | Check if Label Exists         | Replace Me                 |                                                                           |
| Replace Me                   | NoOp                           | Placeholder for loop continuation  | Create New Label, Apply Label to Email | Loop Over Items             |                                                                           |
| OpenAI Chat Model             | OpenAI LM Chat                  | Provides AI language model access  | Categorize Email with AI      | Categorize Email with AI    |                                                                           |
| Sticky Note                  | Sticky Note                    | Title and workflow description     | None                         | None                       | ## Auto Gmail Labeling (Powered by OpenAI)                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Set to run every 2 minutes.

2. **Add a Gmail - Get All Messages Node**  
   - Operation: getAll messages  
   - Limit: 20  
   - Read status: both read and unread  
   - Credentials: Connect your Gmail OAuth2 credentials  
   - Connect output of Schedule Trigger to input of this node.

3. **Add a Code Node (Filter Emails Without Excluded Labels)**  
   - Paste the JavaScript filtering code that excludes emails containing any of the predefined label IDs (Newsletter, Inquiry, Invoice, etc.).  
   - Connect the output of Gmail - Get All Messages to this node.

4. **Add a SplitInBatches Node (Loop Over Items)**  
   - Connect output of Code Node to this node.  
   - Use default batch size to process emails one-by-one.

5. **Add a Gmail - Single Message Node**  
   - Operation: get message  
   - Message ID: set to `{{$json["id"]}}` from batch input  
   - Credentials: same Gmail OAuth2  
   - Connect output of SplitInBatches node here.

6. **Add an If Node**  
   - Configure a condition to check for a label ID (currently a placeholder comparing the same value).  
   - Connect output of Gmail - Single Message node to this node.

7. **Add a Merge Node**  
   - Set to merge data from both true and false branches of If node appropriately.  
   - Connect the false output of If node and the Gmail - Single Message node output to this node.

8. **Add a Set Node (Extract Email Data)**  
   - Assign variables:  
     - `id` ← `$json["id"]`  
     - `from` ← `$json["headers"]["from"]`  
     - `headers.subject` ← `$json["headers"]["subject"]`  
     - `text` ← `$json["text"]`  
   - Connect output of Merge node here.

9. **Add a Langchain Agent Node (Categorize Email with AI)**  
   - Prompt setup: Combine the email "from", "subject", and "text" fields into a prompt.  
   - System message: Use the provided detailed label definitions and instructions forcing output to be exactly one label name from the 17 available labels.  
   - Connect output of Extract Email Data node here.

10. **Add an OpenAI Chat Model Node**  
    - Model: GPT-4.1-nano  
    - Credentials: OpenAI API key connected  
    - Connect Langchain Agent’s AI language model input to this node.

11. **Add a Set Node (Store AI Category)**  
    - Store AI output under field named `output`.  
    - Connect output of Langchain Agent node here.

12. **Add a Gmail List Labels Node**  
    - Operation: list all labels  
    - Credentials: Gmail OAuth2  
    - Connect output of Store AI Category node here.

13. **Add a CompareDatasets Node (Check if Label Exists)**  
    - Compare AI output `output` to Gmail label `name` with fuzzy matching enabled.  
    - Connect outputs from Store AI Category and List All Gmail Labels nodes to this node.

14. **Add a Gmail Create Label Node (Apply New Label)**  
    - Operation: create label  
    - Label name: `{{$json["output"]}}` (AI output)  
    - Credentials: Gmail OAuth2  
    - Connect first output (label does not exist) from CompareDatasets node here.

15. **Add a Gmail Add Label to Message Node (Create New Label)**  
    - Operation: add label to message  
    - Label ID: from Apply New Label node output - `id`  
    - Message ID: from Extract Email Data node - `id`  
    - Credentials: Gmail OAuth2  
    - Connect output of Apply New Label node here.

16. **Add a Gmail Add Label to Message Node (Apply Label to Email)**  
    - Operation: add label to message  
    - Label ID: from CompareDatasets output (existing label id)  
    - Message ID: from Extract Email Data node - `id`  
    - Credentials: Gmail OAuth2  
    - Connect second output (label exists) from CompareDatasets node here.

17. **Add a NoOp Node (Replace Me)**  
    - No operation, acts as loop continuation or placeholder.  
    - Connect outputs of both Gmail Add Label to Message nodes (Create New Label and Apply Label to Email) to this node.

18. **Connect output of NoOp Node back to SplitInBatches Node**  
    - To continue processing next email in batch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| Workflow is titled "Auto Gmail Labeling (Powered by OpenAI)" and is designed to intelligently label Gmail messages automatically | Title and workflow purpose                                                        |
| The AI system message includes detailed label definitions and strict output rules for consistent classification              | Important for prompt engineering best practices                                   |
| The workflow uses OAuth2 credentials for Gmail and OpenAI, requiring appropriate API key setup                                | Credential requirements                                                           |
| Exclusion label IDs in the code node must be kept up-to-date to correctly filter already labeled emails                      | Maintenance note for label ID management                                          |
| Langchain agent node integrates OpenAI model GPT-4.1-nano for classification                                                  | AI integration detail                                                             |
| The "If" node contains a tautological condition that may serve as a placeholder or require adjustment                         | Potential logic refinement needed                                                 |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---