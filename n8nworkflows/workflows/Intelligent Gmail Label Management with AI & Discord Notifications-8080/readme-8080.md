Intelligent Gmail Label Management with AI & Discord Notifications

https://n8nworkflows.xyz/workflows/intelligent-gmail-label-management-with-ai---discord-notifications-8080


# Intelligent Gmail Label Management with AI & Discord Notifications

### 1. Workflow Overview

This workflow automates intelligent Gmail email labeling using AI-generated categorizations and sends notifications via Discord about labeled emails. It leverages AI (Langchain with OpenAI-compatible models) to analyze incoming emails, suggest existing or new Gmail labels, create missing labels dynamically, apply the labels to the emails, and notify a Discord channel with a fun, emoji-enhanced summary message.

Logical blocks:

- **1.1 Input Reception & Preparation**: Trigger on new Gmail emails and retrieve existing Gmail labels.
- **1.2 AI Processing & Label Suggestion**: Use an LLM to summarize the email content and propose relevant labels from existing ones or new labels.
- **1.3 Label Management**: Determine which suggested labels already exist and which need creation; create new Gmail labels as needed.
- **1.4 Label Application**: Apply all relevant labels (new and existing) to the email message.
- **1.5 Discord Notification**: Generate a fun, emoji-rich summary of the labeling action and send it as a message to a Discord channel.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Preparation

- **Overview**: This block listens for new emails via Gmail Trigger, splits batch processing for each email, and retrieves the user's existing Gmail labels to provide options for AI labeling.

- **Nodes Involved**:  
  - Gmail Trigger  
  - Loop Over Items (splitInBatches)  
  - Get many labels (Gmail)  
  - Filter (filter labels starting with "Label")  
  - Aggregate1 (collect filtered labels)  
  - Merge labels into list (merge node combining emails and labels)  

- **Node Details**:

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Config: Polls every minute for new emails  
    - Credentials: OAuth2 Gmail account  
    - Outputs: New email data including sender, subject, snippet, internalDate, id  
    - Potential Failures: OAuth token expiry, network issues  

  - **Loop Over Items**  
    - Type: splitInBatches to process emails one by one  
    - Config: Default batch size (1)  
    - Input: New emails from Gmail Trigger  
    - Output: Single email per batch  
    - Edge Cases: Large email volumes may slow processing  

  - **Get many labels**  
    - Type: Gmail node to list all labels  
    - Config: Return all labels with no limit  
    - Credentials: Same Gmail OAuth2  
    - Output: List of all Gmail labels for the account  
    - Failure: API limits, auth errors  

  - **Filter**  
    - Type: Filter node  
    - Config: Keeps only labels whose ID starts with "Label" (standard Gmail label IDs)  
    - Input: All Gmail labels  
    - Output: Filtered label list for AI to choose from  

  - **Aggregate1**  
    - Type: Aggregate  
    - Config: Aggregates all filtered label items into one array field named `existing_labels`  
    - Output: Single item with array of existing labels  

  - **Merge labels into list (for llm to pick)**  
    - Type: Merge node (combineBySql, LEFT JOIN)  
    - Combines the current email data with the existing labels list  
    - Output: Email data enriched with existing labels  

---

#### 1.2 AI Processing & Label Suggestion

- **Overview**: Uses a Language Model chain to analyze each email and suggest a JSON summary including up to 5 labels selected from existing or newly created ones.

- **Nodes Involved**:  
  - Basic LLM Chain  
  - Add old_labels [] (Set node)  
  - Merge2  
  - Structured Output Parser  

- **Node Details**:

  - **Basic LLM Chain**  
    - Type: Langchain chain LLM node  
    - Config: Prompt instructs AI to summarize email (From, Subject, Content snippet), extract entities, and choose up to 5 labels from existing ones or create new ones  
    - Inputs: Email data + existing labels list  
    - Outputs: JSON with summary, people, and labels array  
    - Retries on failure with 100ms delay  
    - Edge Cases: AI misinterpretation, malformed output handled by Structured Output Parser  

  - **Add old_labels []**  
    - Type: Set node  
    - Initializes `output.old_labels` and `output.added_label_names` as empty arrays, copying `output` object  
    - Prepares structure to hold label IDs and names  

  - **Merge2**  
    - Type: Merge (combineBySql, LEFT JOIN)  
    - Joins the AI output with the original email data for further processing  

  - **Structured Output Parser**  
    - Type: Langchain output parser  
    - Config: JSON schema example provided to auto-fix and parse AI JSON output  
    - Input: Raw AI text output  
    - Output: Structured JSON for downstream nodes  

---

#### 1.3 Label Management

- **Overview**: Processes suggested labels to identify duplicates with existing labels and new labels needing creation; creates new Gmail labels accordingly.

- **Nodes Involved**:  
  - Code (JavaScript)  
  - Split Out (splitOut node)  
  - Create a label (Gmail node)  
  - Aggregate2  
  - Merge3  
  - Sticky Note3 (comment on label creation)  

- **Node Details**:

  - **Code**  
    - Type: Code node (JS)  
    - Logic:  
      - Compares AI suggested labels (`new_labels`) with existing Gmail labels (`old_labels`)  
      - Separates labels into new labels to create and existing ones to reuse  
      - Collects label IDs for existing labels and label names for created ones  
      - Sets `output.labels` as new labels to create and `output.old_labels` as existing label IDs  
    - Input: Merged email and AI output with existing labels  
    - Output: Annotated email with label creation lists  
    - Edge Cases: Labels with same names but different casing, empty label arrays  

  - **Split Out**  
    - Type: splitOut to split array of new labels to create into individual items  
    - Input: List of new labels (`output.labels`) to create  

  - **Create a label**  
    - Type: Gmail node  
    - Operation: Create label with name from split items  
    - Credentials: Gmail OAuth2  
    - On error: Continue (to avoid blocking on label creation errors)  
    - Output: Created label details including ID  

  - **Aggregate2**  
    - Type: Aggregate node  
    - Aggregates all created label items into `created_labels` array  

  - **Merge3**  
    - Type: Merge node (combineBySql)  
    - Combines created labels with previous data for final label list preparation  

  - **Sticky Note3**  
    - Content: "### Create additional gmail labels if needed."  
    - Context: Covers label creation nodes  

---

#### 1.4 Label Application

- **Overview**: Combines newly created label IDs and existing label IDs, applies them to the Gmail message, and prepares data for notification.

- **Nodes Involved**:  
  - Edit Fields3 (Set node)  
  - Add label to message (Gmail node)  
  - Merge  
  - Sticky Note1 (comment on labeling)  

- **Node Details**:

  - **Edit Fields3**  
    - Type: Set node  
    - Assignments:  
      - Keeps `output.messageId`  
      - Defines `labels_to_add` as concatenation of existing label IDs (`output.old_labels`) and created label IDs (`created_labels.map(id)`)  
    - Prepares parameters for Gmail label application  

  - **Add label to message**  
    - Type: Gmail node  
    - Operation: Add labels to message using `labels_to_add` array and `messageId`  
    - Credentials: Gmail OAuth2  
    - Output: Confirmation of applied labels  

  - **Merge**  
    - Type: Merge node (combineBySql)  
    - Combines labeling results with notification message preparation  

  - **Sticky Note1**  
    - Content: "### Label the message with new and existing labels"  
    - Covers labeling and message ID handling  

---

#### 1.5 Discord Notification

- **Overview**: Creates an engaging, emoji-rich summary message of labeling results and sends it to a specified Discord channel.

- **Nodes Involved**:  
  - Basic LLM Chain1  
  - Send a message (Discord node)  
  - Sticky Note2 (comment on notification)  

- **Node Details**:

  - **Basic LLM Chain1**  
    - Type: Langchain chain LLM node  
    - Config:  
      - Prompt to rewrite the labeling summary message with appropriate emojis based on content type (financial, advertisement, etc.)  
      - Prepends "MESSAGE STARTS HERE" for clarity  
      - Uses fields like From, Subject, Summary, Entity, Date, Labels Added, Appended Labels  
    - Input: Final labeling data with created and added labels, email metadata  
    - Output: Text message ready for Discord  

  - **Send a message**  
    - Type: Discord node  
    - Operation: Send message to a specific guild (server) and channel (general)  
    - Credentials: Discord Bot API credentials  
    - Message content: Parsed from LLM output after "MESSAGE STARTS HERE"  
    - Edge Cases: Discord API rate limits, bot permissions  

  - **Sticky Note2**  
    - Content: "### Rewrite and send message notification to discord"  
    - Context: Notification message formatting and sending  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                             | Input Node(s)               | Output Node(s)             | Sticky Note                               |
|-------------------------------|--------------------------------|--------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------|
| Gmail Trigger                 | Gmail Trigger                  | Trigger on new Gmail emails                 | —                           | Loop Over Items             |                                           |
| Loop Over Items               | splitInBatches                 | Process emails one by one                    | Gmail Trigger               | Get many labels, Merge labels into list |                                           |
| Get many labels               | Gmail                         | Retrieve all Gmail labels                    | Loop Over Items             | Filter                      | ### Find existing Gmail labels as options for LLM to choose from. |
| Filter                       | Filter                        | Keep labels whose ID starts with "Label"   | Get many labels             | Aggregate1                  | ### Find existing Gmail labels as options for LLM to choose from. |
| Aggregate1                   | Aggregate                     | Aggregate filtered labels into array        | Filter                      | Merge labels into list       | ### Find existing Gmail labels as options for LLM to choose from. |
| Merge labels into list (for llm to pick) | Merge                         | Combine email data with existing labels     | Aggregate1, Loop Over Items | Merge2                      | ### Find existing Gmail labels as options for LLM to choose from. |
| Basic LLM Chain              | Langchain Chain LLM            | Summarize email and suggest labels          | Merge labels into list       | Add old_labels []           | ### LLM chooses labels or defines new ones |
| Add old_labels []            | Set                           | Initialize label arrays for tracking         | Basic LLM Chain             | Merge2                      | ### LLM chooses labels or defines new ones |
| Merge2                      | Merge                         | Combine AI output with email data             | Add old_labels [], Merge labels into list | Code                        | ### LLM chooses labels or defines new ones |
| Structured Output Parser     | Langchain Output Parser        | Parse AI JSON output into structured data    | Basic LLM Chain             | Basic LLM Chain             |                                           |
| Code                        | Code                          | Identify new labels to create and existing labels | Merge2                      | Split Out                   | ### Create additional gmail labels if needed. |
| Split Out                   | splitOut                      | Split new labels array into individual labels | Code                        | Create a label              | ### Create additional gmail labels if needed. |
| Create a label              | Gmail                         | Create Gmail labels if not existing           | Split Out                   | Aggregate2                  | ### Create additional gmail labels if needed. |
| Aggregate2                  | Aggregate                     | Aggregate created label objects                | Create a label              | Merge3                      | ### Create additional gmail labels if needed. |
| Merge3                      | Merge                         | Combine created labels with previous data     | Aggregate2, Code            | Edit Fields3                | ### Create additional gmail labels if needed. |
| Edit Fields3                | Set                           | Prepare final label list for application       | Merge3                      | Add label to message, Merge | ### Label the message with new and existing labels |
| Add label to message        | Gmail                         | Apply labels to the email message               | Edit Fields3                | Merge                       | ### Label the message with new and existing labels |
| Merge                       | Merge                         | Combine labeling confirmation with notification | Add label to message, Edit Fields3 | Basic LLM Chain1           | ### Label the message with new and existing labels |
| Basic LLM Chain1            | Langchain Chain LLM            | Generate emoji-rich Discord notification       | Merge                       | Send a message              | ### Rewrite and send message notification to discord |
| Send a message              | Discord                       | Send notification message to Discord channel   | Basic LLM Chain1            | —                           | ### Rewrite and send message notification to discord |
| Sticky Note                 | Sticky Note                   | Comment: Find existing Gmail labels            | —                           | —                           | ### Find existing Gmail labels as options for LLM to choose from. |
| Sticky Note1                | Sticky Note                   | Comment: Label message with new and existing labels | —                           | —                           | ### Label the message with new and existing labels |
| Sticky Note2                | Sticky Note                   | Comment: Rewrite and send Discord message      | —                           | —                           | ### Rewrite and send message notification to discord |
| Sticky Note3                | Sticky Note                   | Comment: Create additional Gmail labels if needed | —                           | —                           | ### Create additional gmail labels if needed. |
| Sticky Note4                | Sticky Note                   | Comment: LLM chooses labels or defines new ones | —                           | —                           | ### LLM chooses labels or defines new ones |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Credentials: OAuth2 Gmail account  
   - Poll interval: Every minute  

2. **Add Split In Batches node ("Loop Over Items")**  
   - Purpose: Process each incoming email individually  

3. **Add Gmail node ("Get many labels")**  
   - Operation: List all labels  
   - Return all: true  
   - Credentials: Same Gmail OAuth2  

4. **Add Filter node**  
   - Condition: Label ID startsWith "Label"  
   - Input: Output from "Get many labels"  

5. **Add Aggregate node ("Aggregate1")**  
   - Aggregate all filtered labels into `existing_labels` array  

6. **Add Merge node ("Merge labels into list (for llm to pick)")**  
   - Mode: combineBySql (LEFT JOIN)  
   - Inputs: Email item and aggregated label list  

7. **Add Langchain Chain LLM node ("Basic LLM Chain")**  
   - Prompt: Provide summary JSON with Id, summary, people, and up to 5 labels from existing or new ones  
   - Use templated variables: From, Subject, snippet, existing_labels names  
   - Enable retry on fail with 100ms wait  
   - Connect input from Merge node  

8. **Add Set node ("Add old_labels []")**  
   - Initialize: `output.old_labels` as empty array, `output.added_label_names` as empty array, pass along `output` object  

9. **Add Merge node ("Merge2")**  
   - Combine outputs from "Add old_labels []" and "Merge labels into list"  

10. **Add Langchain Output Parser node ("Structured Output Parser")**  
    - Use example JSON schema to parse AI output reliably  

11. **Add Code node ("Code")**  
    - Implement logic to:  
      - Compare AI suggested labels with existing labels  
      - Separate new labels to create and existing labels to reuse  
      - Prepare arrays `output.labels` (new labels), `output.old_labels` (existing label IDs), and `output.added_label_names` (existing label names)  

12. **Add Split Out node ("Split Out")**  
    - Field to split: `output.labels` (new labels to create)  

13. **Add Gmail node ("Create a label")**  
    - Operation: Create label  
    - Name: Use label name from split items  
    - Credentials: Gmail OAuth2  
    - On error: Continue  

14. **Add Aggregate node ("Aggregate2")**  
    - Aggregate all created labels into array `created_labels`  

15. **Add Merge node ("Merge3")**  
    - Mode: combineBySql (LEFT JOIN)  
    - Inputs: Aggregate2 and Code node outputs  

16. **Add Set node ("Edit Fields3")**  
    - Assign:  
      - `output.messageId` from email id  
      - `labels_to_add` as concatenation of `output.old_labels` and `created_labels` ids  

17. **Add Gmail node ("Add label to message")**  
    - Operation: Add labels to message  
    - Parameters: `labelIds` from `labels_to_add`, `messageId` from `output.messageId`  
    - Credentials: Gmail OAuth2  

18. **Add Merge node ("Merge")**  
    - Combine outputs of "Add label to message" and "Edit Fields3"  

19. **Add Langchain Chain LLM node ("Basic LLM Chain1")**  
    - Prompt: Rewrite message summary with fun emojis, prepend "MESSAGE STARTS HERE", format message with From, Subject, Summary, Entity, Date, Id, Labels Added, Appended Labels  
    - Input: From merged node  

20. **Add Discord node ("Send a message")**  
    - Operation: Send message to Discord channel  
    - Credentials: Discord Bot API  
    - Parameters:  
      - Content: Extract text after "MESSAGE STARTS HERE" from LLM output  
      - Guild ID and Channel ID: Set to your Discord server and channel  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                 |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------|
| The workflow uses Langchain nodes to integrate AI models for email content summarization and label suggestion. | Requires credentials for Gmail OAuth2 and Discord Bot API                       |
| Labels starting with "Label" are treated as valid Gmail user labels for categorization.                         | Gmail label ID filtering via Filter node                                        |
| The workflow gracefully continues label creation even if some labels fail to create.                            | "Create a label" node has error mode set to continue                           |
| Discord messages include emojis to enhance readability and engagement.                                          | Uses AI to rewrite messages with appropriate emojis                            |
| For advanced usage, OpenAI-compatible models can be swapped in the Langchain nodes with proper credentials.    | Models are configured with local or remote OpenAI endpoints                    |
| Slack or other notification nodes can replace Discord node if desired with minimal adjustments.                 | Node type and parameters would change accordingly                             |

---

_Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, respecting all content policies. It contains no illegal, offensive, or protected content and processes only legal and public data._