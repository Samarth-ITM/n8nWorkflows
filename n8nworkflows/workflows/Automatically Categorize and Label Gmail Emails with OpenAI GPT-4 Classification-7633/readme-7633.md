Automatically Categorize and Label Gmail Emails with OpenAI GPT-4 Classification

https://n8nworkflows.xyz/workflows/automatically-categorize-and-label-gmail-emails-with-openai-gpt-4-classification-7633


# Automatically Categorize and Label Gmail Emails with OpenAI GPT-4 Classification

### 1. Workflow Overview

This workflow automates the process of categorizing and labeling Gmail emails using OpenAI's GPT-5 model. It targets users or organizations aiming to intelligently organize incoming emails by priority and type, reducing manual inbox management effort.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger and Email Fetching:** Periodically triggers the workflow and fetches recent emails from Gmail.
- **1.2 Email Processing Loop:** Iterates over each fetched email individually for processing.
- **1.3 AI-Based Classification:** Uses OpenAI GPT-5 to classify each email into predefined categories based on subject and body content.
- **1.4 Label Application:** Applies Gmail labels corresponding to the AI classification result.
- **1.5 Workflow Control Loop:** Ensures all emails are processed by looping back after labeling.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger and Email Fetching

**Overview:**  
This block periodically triggers the workflow (every 5 minutes) and fetches up to 10 recent emails received within the last 5 minutes from Gmail.

**Nodes Involved:**  
- Schedule Trigger  
- Gmail - Get Emails  
- Sticky Note6  

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger node  
  - Configuration: Interval set to trigger every 5 minutes (field: minutes).  
  - Input: None (start node)  
  - Output: Triggers Gmail email fetching node.  
  - Potential Failures: Scheduling misconfiguration, node downtime.

- **Gmail - Get Emails**  
  - Type: Gmail node (Get All operation)  
  - Configuration:  
    - Limit: 10 emails  
    - Filter: Received after current time minus 50000 minutes (likely a misconfiguration; intended to be 5 minutes but set as 50000)  
    - Credentials: OAuth2 for Gmail account "Billy Email 2"  
  - Input: Triggered by Schedule Trigger  
  - Output: Passes fetched emails to Loop Over Items node  
  - Edge Cases:  
    - API rate limits or authentication errors  
    - Empty email fetch if no new emails  
    - Potential misfiltering due to incorrect filter time (50000 minutes ≈ 34 days)  
  - Notes: The filter should be corrected to 5 minutes for intended functionality.

- **Sticky Note6**  
  - Type: Sticky Note  
  - Content: Advises on configuring Gmail email fetching parameters such as time range and email limit.

---

#### 1.2 Email Processing Loop

**Overview:**  
Splits the batch of fetched emails into individual items to process each email separately.

**Nodes Involved:**  
- Loop Over Items  

**Node Details:**  

- **Loop Over Items**  
  - Type: Split In Batches node  
  - Configuration: Default batch size (processes one email at a time)  
  - Input: Receives array of emails from Gmail - Get Emails  
  - Output: Sends single email record to Label Classifier node  
  - Edge Cases:  
    - Large email batches could increase processing time  
    - Failure if input data is empty or malformed

---

#### 1.3 AI-Based Classification

**Overview:**  
Classifies each email's subject and body into one of three categories (High Priority, Personal, Promotions) using OpenAI GPT-5 model via a LangChain text classifier.

**Nodes Involved:**  
- Label Classifier  
- OpenAI Chat Model  
- Sticky Note5  

**Node Details:**  

- **Label Classifier**  
  - Type: LangChain Text Classifier node  
  - Configuration:  
    - Input Text: Concatenation of email subject and body from JSON fields `headers.subject` and `text`  
    - Categories:  
      - High Priority: Emails requiring immediate attention  
      - Personal: Non-work, private matters  
      - Promotions: Marketing or advertisement messages  
    - Fallback category: "other"  
  - Input: Receives one email from Loop Over Items  
  - Output: Routes email for labeling based on classification  
  - Edge Cases:  
    - Misclassification due to ambiguous or malformed email text  
    - Failure if email body or subject fields are missing  
  - Notes: Categories and descriptions are configurable to suit user needs.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model node  
  - Configuration: Uses GPT-5 model (latest OpenAI language model)  
  - Credentials: OpenAI API key  
  - Role: Supplies AI model for text classification node  
  - Input/Output: Connected only as AI language model provider to Label Classifier  
  - Edge Cases:  
    - API rate limiting or authentication failures  
    - Timeout or unexpected API errors

- **Sticky Note5**  
  - Explains that the label classifier categories should be adjusted and described for best results.

---

#### 1.4 Label Application

**Overview:**  
Applies Gmail labels to each email thread according to the AI classification result. Labels include "IMPORTANT," "CATEGORY_PERSONAL," and "CATEGORY_PROMOTIONS."

**Nodes Involved:**  
- Add Label (High Priority)  
- Add Label (Personal)  
- Add Label (Promotions)  
- Sticky Note4  

**Node Details:**  

- **Add Label (High Priority), Add Label (Personal), Add Label (Promotions)**  
  - Type: Gmail node (Add Labels operation on thread resource)  
  - Configuration:  
    - Label IDs correspond to Gmail system or custom labels: "IMPORTANT," "CATEGORY_PERSONAL," "CATEGORY_PROMOTIONS"  
    - Uses `threadId` from the email JSON to target the correct email thread  
    - Credentials: Gmail OAuth2 for "Billy Email 2" account  
  - Input: Receives emails filtered by classification from Label Classifier  
  - Output: Returns to Loop Over Items node to process next email  
  - Edge Cases:  
    - Failure to add label due to invalid label ID or Gmail API errors  
    - Authentication expiration  
    - Thread ID missing or invalid  
  - Notes: User should add or remove label nodes based on their label setup.

- **Sticky Note4**  
  - Advises users to add or remove label nodes according to the number and names of their Gmail labels.

---

#### 1.5 Workflow Control Loop

**Overview:**  
After labeling an email, the workflow loops back to process the next email in the batch until all are handled.

**Nodes Involved:**  
- Loop Over Items (loop back connections)  
- Label Classifier (loop back connections)  
- Label application nodes (loop back connections)  

**Node Details:**  

- Loop connections between Label Classifier, Add Label nodes, and Loop Over Items ensure continuous processing of email batches.  
- Edge Cases: Infinite loop risk if node outputs are misconfigured; ensure batch completes.

---

### 3. Summary Table

| Node Name               | Node Type                                  | Functional Role                         | Input Node(s)            | Output Node(s)                        | Sticky Note                                                                                         |
|-------------------------|--------------------------------------------|---------------------------------------|--------------------------|-------------------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                           | Periodic workflow trigger              | None                     | Gmail - Get Emails                   |                                                                                                   |
| Gmail - Get Emails      | Gmail (Get All)                            | Fetch recent emails                    | Schedule Trigger          | Loop Over Items                     | Advises on configuring email fetch parameters (time range, limit)                                |
| Loop Over Items         | Split In Batches                           | Process emails one-by-one              | Gmail - Get Emails        | Label Classifier                    |                                                                                                   |
| Label Classifier        | LangChain Text Classifier                  | Classify email content                 | Loop Over Items           | Add Label (High Priority), Add Label (Personal), Add Label (Promotions), Loop Over Items | Advises to adjust categories and descriptions for best classification results                     |
| OpenAI Chat Model       | LangChain OpenAI Chat Model                 | Provides GPT-5 model for classification | None (AI model provider)  | Label Classifier                    |                                                                                                   |
| Add Label (High Priority) | Gmail (Add Labels on Thread)               | Apply "IMPORTANT" label to email      | Label Classifier          | Loop Over Items                    | Advises to add/remove label nodes based on user labels                                            |
| Add Label (Personal)    | Gmail (Add Labels on Thread)               | Apply "CATEGORY_PERSONAL" label        | Label Classifier          | Loop Over Items                    | Advises to add/remove label nodes based on user labels                                            |
| Add Label (Promotions)  | Gmail (Add Labels on Thread)               | Apply "CATEGORY_PROMOTIONS" label      | Label Classifier          | Loop Over Items                    | Advises to add/remove label nodes based on user labels                                            |
| Sticky Note             | Sticky Note                               | Setup instructions                    | None                     | None                              | Workflow Configurations and required credentials                                                 |
| Sticky Note1            | Sticky Note                               | Workflow process overview             | None                     | None                              | Overview of workflow steps                                                                       |
| Sticky Note2            | Sticky Note                               | Workflow purpose and capabilities    | None                     | None                              | Describes automation benefits and AI classification approach                                     |
| Sticky Note4            | Sticky Note                               | Label node customization advice      | None                     | None                              | Advises on adding label nodes based on label count                                               |
| Sticky Note5            | Sticky Note                               | Label classifier category advice     | None                     | None                              | Advises to adjust categories and descriptions                                                    |
| Sticky Note6            | Sticky Note                               | Gmail get emails configuration notes | None                     | None                              | Advises on email fetch settings                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run every 5 minutes (interval: minutes, value: 5).

2. **Create Gmail - Get Emails Node**  
   - Type: Gmail node  
   - Operation: Get All  
   - Set limit to 10 emails  
   - Set filter: Received after current time minus 5 minutes  
     - Use expression: `={{ $now.minus({ minutes: 5 }).toMillis() }}`  
   - Connect Schedule Trigger output to this node.  
   - Attach Gmail OAuth2 credentials (setup with Gmail account).  

3. **Create Loop Over Items Node**  
   - Type: Split In Batches  
   - Default batch size (process one email at a time)  
   - Connect Gmail - Get Emails output to this node input.

4. **Create OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-5  
   - Provide OpenAI API credentials.  

5. **Create Label Classifier Node**  
   - Type: LangChain Text Classifier  
   - Input Text:  
     ```
     Email Subject:
     {{ $json.headers.subject }}

     Email Body:
     {{ $json.text }}
     ```  
   - Define categories:  
     - High Priority: Emails that require immediate attention or urgent action.  
     - Personal: Personal or non-work related matters.  
     - Promotions: Marketing or advertising content.  
   - Set fallback category to "other".  
   - Connect Loop Over Items output to this node input.  
   - Connect OpenAI Chat Model node as AI language model provider.

6. **Create Add Label Nodes for Each Category**  
   For each category, create a Gmail node configured to add the corresponding label to the email thread:  

   - Node Name: Add Label (High Priority)  
     - Label ID: IMPORTANT  
     - Resource: thread  
     - Thread ID: `={{ $json.threadId }}`  
     - Operation: addLabels  
     - Connect Label Classifier's "High Priority" output to this node.  
     - Attach Gmail OAuth2 credentials.

   - Node Name: Add Label (Personal)  
     - Label ID: CATEGORY_PERSONAL  
     - Same configuration as above.  
     - Connect Label Classifier's "Personal" output.

   - Node Name: Add Label (Promotions)  
     - Label ID: CATEGORY_PROMOTIONS  
     - Same configuration as above.  
     - Connect Label Classifier's "Promotions" output.

7. **Connect Each Add Label Node Back to Loop Over Items**  
   - This ensures the workflow loops over remaining emails until all are processed.

8. **Add Sticky Notes for Documentation** (optional but recommended)  
   - Add setup instructions about configuring labels and credentials.  
   - Add workflow overview and purpose.  
   - Add notes about adjusting classifier categories and labels.

9. **Credential Setup**  
   - Gmail OAuth2: Configure with Gmail account scopes allowing reading emails and modifying labels.  
   - OpenAI API Key: Configure with access to GPT-5 or GPT-4 (depending on availability).

10. **Test Workflow**  
    - Run the workflow manually or wait for scheduled trigger.  
    - Monitor logs for errors or API limits.  
    - Adjust time filter and batch size as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                               |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Workflow automatically classifies emails every 5 minutes using GPT-5 for intelligent inbox sorting.| Workflow description                                          |
| Update label names and classifier categories as per your Gmail labels and business needs.          | Sticky Note4 and Sticky Note5                                 |
| Gmail OAuth2 credentials require proper scopes for reading emails and modifying labels.             | Gmail OAuth2 credential configuration                         |
| OpenAI API key must have access rights to GPT-5 or GPT-4 models used for text classification.        | OpenAI API credential configuration                           |
| Make sure to adjust the time filter in Gmail node to match your email fetch window (e.g., last 5 minutes). | Sticky Note6 and node configuration                           |
| This workflow is designed to be modular: you can add more label nodes for additional categories.    | General workflow design                                       |
| For more info on n8n Gmail node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ | Official n8n documentation                                   |
| For more info on LangChain text classifiers in n8n: https://docs.n8n.io/integrations/builtin/app-nodes/langchain/ | Official n8n documentation                                   |

---

**Disclaimer:** The provided text derives exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.