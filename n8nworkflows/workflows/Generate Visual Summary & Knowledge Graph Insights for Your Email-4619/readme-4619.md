Generate Visual Summary & Knowledge Graph Insights for Your Email

https://n8nworkflows.xyz/workflows/generate-visual-summary---knowledge-graph-insights-for-your-email-4619


# Generate Visual Summary & Knowledge Graph Insights for Your Email

---

### 1. Workflow Overview

This workflow is designed to generate a visual summary and knowledge graph insights based on selected Gmail emails. It targets users who want to analyze, summarize, and gain AI-driven insights from their email content through topical knowledge graphs and automated notifications.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception & Triggering:** Supports multiple trigger methods including manual, schedule, webhook, and password-protected form input to specify email filtering criteria.
- **1.2 Email Filtering and Retrieval:** Applies user-specified Gmail search criteria and label filters to find relevant emails.
- **1.3 Email Content Processing:** Decides whether to analyze email snippets or full email text, retrieves full messages if necessary, then cleans and organizes the text.
- **1.4 AI-Based Email Classification:** Optionally classifies emails using Google's Gemini AI model to filter emails by type or relevance.
- **1.5 Knowledge Graph Construction:** Builds either a social or text knowledge graph with InfraNodus API based on processed email statements.
- **1.6 AI Summary and Question Generation:** Uses InfraNodus AI to generate topical summaries and insight questions from the knowledge graph.
- **1.7 Notification & Reporting:** Sends the generated graph links, summaries, and insight questions via Telegram and email.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Triggering

- **Overview:** This block defines how the workflow receives input and begins execution. It supports form submission, manual trigger, scheduled trigger, and webhook options.
- **Nodes Involved:**  
  - User submits form  
  - When clicking ‘Test workflow’ (manual trigger)  
  - Schedule Trigger  
  - Assign Processing Settings (parameter assignment)  
  - Sticky Note (explanations)

- **Node Details:**

  - **User submits form**  
    - Type: Form Trigger  
    - Role: Enables user to input email search criteria and other processing parameters via a password-protected form.  
    - Key Config: Basic Auth enabled; form fields include search criteria, label filter, email type description, analyze full email text option, build social graph option, and graph name.  
    - Connections: Outputs to "Assign Processing Settings".  
    - Edge cases: Disabled by default; form fields must be properly filled to avoid empty values.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution during development/testing.  
    - Connections: Outputs to "Assign Processing Settings".

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Enables daily automated execution (disabled by default).  
    - Config: Triggers at 10 AM daily.  
    - Connections: Outputs to "Assign Processing Settings".

  - **Assign Processing Settings**  
    - Type: Set node  
    - Role: Sets default or input parameters for email filtering and processing, including search criteria, label filter, email type description, analysis preferences, and graph name.  
    - Key Expressions: Uses ternary expressions to provide defaults if inputs are missing.  
    - Connections: Outputs to "Get Messages by Search Criteria".  
    - Edge cases: Ensures defaults prevent empty queries; date calculation for "after" filter to limit email search.

  - **Sticky Note**  
    - Provides instructions on how to trigger the workflow and where to find Gmail search criteria documentation.

---

#### 2.2 Email Filtering and Retrieval

- **Overview:** This block uses the assigned parameters to query Gmail for emails matching the criteria and optionally filters by labels.
- **Nodes Involved:**  
  - Get Messages by Search Criteria  
  - Was label provided? (If node)  
  - Filter emails by label  
  - Should analyze snippets from filtered emails? (If node)  
  - Should analyze snippets? (If node)  
  - Sticky Notes explaining filtering and Gmail node setup

- **Node Details:**

  - **Get Messages by Search Criteria**  
    - Type: Gmail node (get all messages)  
    - Role: Fetches all Gmail messages matching the search query from "Assign Processing Settings".  
    - Key Config: Uses the "q" filter from JSON input; returns all matching emails.  
    - Credentials: Requires Gmail OAuth2.  
    - Output: List of email metadata.  
    - Edge cases: Large result sets may slow down; the recommended use of date and label filters to optimize.

  - **Was label provided?**  
    - Type: If node  
    - Role: Checks if a label filter is specified to decide next filtering steps.  
    - Condition: Label Filter not empty.

  - **Filter emails by label**  
    - Type: Filter node  
    - Role: Filters emails by labels specified (e.g., SENT, CATEGORY_PERSONAL).  
    - Condition: Checks if the email's labels array contains the specified label.  
    - Notes: Includes Gmail label suggestions for personal, promo, and sent emails.

  - **Should analyze snippets from filtered emails?**  
    - Type: If node  
    - Role: Determines whether to analyze snippets or full text for filtered emails based on user preference.

  - **Should analyze snippets?**  
    - Type: If node  
    - Role: Similar check for emails not filtered by label.

  - **Sticky Notes**  
    - Provide guidance on filtering best practices, label usage, and Gmail credentials setup.

---

#### 2.3 Email Content Processing

- **Overview:** Retrieves full email content if needed and processes either snippets or full text by cleaning HTML and formatting for knowledge graph input.
- **Nodes Involved:**  
  - Get Full Message Content (Gmail node)  
  - Should use AI to filter emails further? (If node)  
  - Message text or snippet present? (Filter node)  
  - Classify Emails (Google Gemini AI classifier)  
  - Text field present? (If node)  
  - Aggregate from full email texts (Aggregate node)  
  - Aggregate from email snippets (Aggregate node)  
  - Clean text and organize into statements (Code node)  
  - Sticky Notes explaining processing choices and formatting

- **Node Details:**

  - **Get Full Message Content**  
    - Type: Gmail node (get message by ID)  
    - Role: Retrieves full message content if full text analysis is requested.  
    - Input: Message ID from previous nodes.  
    - Credentials: Gmail OAuth2.  
    - Output: Full email JSON including text content.

  - **Should use AI to filter emails further?**  
    - Type: If node  
    - Role: Checks if AI classification should be applied based on Email Type Description presence.

  - **Message text or snippet present?**  
    - Type: Filter node  
    - Role: Filters emails that have either text or snippet content for further processing.

  - **Classify Emails**  
    - Type: LangChain Text Classifier using Google Gemini Model  
    - Role: Classifies emails as matching user-defined conditions or spam/notifications.  
    - Input: Email text or snippet.  
    - Output: Categorized emails.  
    - Credentials: Google Gemini API.  
    - Edge cases: API failures, classification errors.

  - **Text field present?**  
    - Type: If node  
    - Role: Checks if full text is present to choose aggregation method.

  - **Aggregate from full email texts**  
    - Type: Aggregate node  
    - Role: Collects and renames fields ("from", "text", "date") from full email texts for further processing.

  - **Aggregate from email snippets**  
    - Type: Aggregate node  
    - Role: Similarly aggregates fields from email snippets.

  - **Clean text and organize into statements**  
    - Type: Code node (JavaScript)  
    - Role: Cleans HTML tags, scripts, styles, comments from email text and formats it into statements suitable for InfraNodus.  
    - Key Logic: Removes HTML, decodes entities, concatenates sender and date.  
    - Output: JSON array of cleaned statements.  
    - Edge cases: Missing or malformed HTML content, empty email bodies.

---

#### 2.4 Knowledge Graph Construction

- **Overview:** Builds either a social or text knowledge graph on InfraNodus from cleaned statements depending on user choice.
- **Nodes Involved:**  
  - Type of graph to build (Switch node)  
  - InfraNodus Build a Social Knowledge Graph (HTTP Request)  
  - InfraNodus Build a Text Knowledge Graph (HTTP Request)  
  - Wait before generating questions (Wait node)  
  - Sticky Notes on graph types and InfraNodus API usage

- **Node Details:**

  - **Type of graph to build**  
    - Type: Switch node  
    - Role: Routes to social or text graph build based on "Build a Social Graph?" setting.  
    - Condition: Checks if the setting is not empty (social graph) or empty (text graph).

  - **InfraNodus Build a Social Knowledge Graph**  
    - Type: HTTP Request node  
    - Role: Sends statements to InfraNodus API to build a social knowledge graph including sender mentions.  
    - Method: POST to InfraNodus API v1/graphAndStatements endpoint.  
    - Body Parameters: Includes graph name, statements, and context settings enabling mentions processing.  
    - Authentication: Bearer token with InfraNodus API key.  
    - Output: Graph summary data.

  - **InfraNodus Build a Text Knowledge Graph**  
    - Type: HTTP Request node  
    - Role: Similar to above, but excludes mentions for a text-only knowledge graph.  
    - Context settings disable mentions and double square brackets processing.  

  - **Wait before generating questions**  
    - Type: Wait node  
    - Role: Delays workflow for 60 seconds to allow InfraNodus graph processing before retrieving summaries and questions.  
    - Webhook enabled for manual resume if needed.

  - **Sticky Notes**  
    - Provide detailed explanations of graph types, example images, and how to use the InfraNodus API.

---

#### 2.5 AI Summary and Question Generation

- **Overview:** Retrieves topical summary and insight questions from InfraNodus based on the constructed knowledge graph.
- **Nodes Involved:**  
  - InfraNodus AI Summary & Graph Link (HTTP Request)  
  - InfraNodus Question Generator (HTTP Request)  
  - Sticky Note explaining AI-generated insights

- **Node Details:**

  - **InfraNodus AI Summary & Graph Link**  
    - Type: HTTP Request node  
    - Role: Requests a summary of the knowledge graph from InfraNodus AI.  
    - Parameters: Includes graph name, optimized summary mode, and AI topics enabled.  
    - Authentication: Bearer token with InfraNodus API key.  
    - Output: AI-generated textual summary.

  - **InfraNodus Question Generator**  
    - Type: HTTP Request node  
    - Role: Requests AI-generated insight questions based on knowledge graph structural gaps.  
    - Parameters: Graph name, request mode "question", AI topics enabled.  
    - Authentication: InfraNodus API key.  
    - Output: Insight questions text.

  - **Sticky Note**  
    - Describes the use of InfraNodus GraphRAG API for generating insights and structural gap-based questions.

---

#### 2.6 Notification & Reporting

- **Overview:** Sends the generated visual summary, graph link, and insight questions to the user via Telegram and email.
- **Nodes Involved:**  
  - Send the graph link and summary via Telegram  
  - Send an insight question via Telegram  
  - Gmail (send email)  
  - Sticky Note on notification setup and Telegram bot creation

- **Node Details:**

  - **Send the graph link and summary via Telegram**  
    - Type: Telegram node  
    - Role: Sends a message with the knowledge graph link and topical summary to a specified Telegram chat.  
    - Key Config: Message text includes dynamic InfraNodus URLs and summary text.  
    - Credentials: Telegram API with bot token.  
    - Retry: Enabled on failure.  
    - Edge cases: Telegram API rate limits, invalid chat ID.

  - **Send an insight question via Telegram**  
    - Type: Telegram node  
    - Role: Sends the AI-generated insight question to the same Telegram chat with a link.  
    - Retry: Enabled on failure.

  - **Gmail (send email)**  
    - Type: Gmail node (send email)  
    - Role: Sends an email summary with graph link, topical summary, and research questions.  
    - Credentials: Gmail OAuth2.  
    - Config: Dynamic subject and message body with InfraNodus outputs.

  - **Sticky Note**  
    - Provides instructions to create a Telegram bot via @botfather and link credentials.

---

### 3. Summary Table

| Node Name                             | Node Type                         | Functional Role                                | Input Node(s)                   | Output Node(s)                                | Sticky Note                                                                                                                             |
|-------------------------------------|----------------------------------|------------------------------------------------|--------------------------------|-----------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                         | Sticky Note                      | Instruction on triggering methods               |                                |                                               | How to Trigger? Explains trigger options including form, schedule, manual, webhook, links to Gmail search criteria page                 |
| User submits form                   | Form Trigger                    | Receives user input parameters                   |                                | Assign Processing Settings                     |                                                                                                                                         |
| When clicking ‘Test workflow’       | Manual Trigger                  | Manual start for testing                         |                                | Assign Processing Settings                     |                                                                                                                                         |
| Schedule Trigger                   | Schedule Trigger                | Scheduled daily execution                        |                                | Assign Processing Settings                     |                                                                                                                                         |
| Assign Processing Settings          | Set                            | Sets default or input parameters for filtering   | User submits form, Manual, Schedule | Get Messages by Search Criteria                 | Provides default search criteria and parameters including label filter and graph name                                                    |
| Sticky Note1                       | Sticky Note                    | Overview and use cases                           |                                |                                               | Visual Summary & Knowledge Graph Insights for Gmail, use cases, setup instructions                                                      |
| Get Messages by Search Criteria     | Gmail (get all messages)        | Retrieves emails matching search criteria       | Assign Processing Settings       | Was label provided?                            | Explains filters, date usage, and Gmail credentials                                                                                     |
| Was label provided?                 | If                             | Checks if label filter specified                 | Get Messages by Search Criteria  | Filter emails by label, Should analyze snippets? |                                                                                                                                         |
| Filter emails by label              | Filter                         | Filters emails by specified label                | Was label provided?              | Should analyze snippets from filtered emails? | Notes on label usage: personal mails, promotions, sent                                                                                    |
| Should analyze snippets?            | If                             | Determines if snippets or full text are analyzed| Was label provided?              | Should use AI to filter emails further?, Get Full Message Content |                                                                                                                                         |
| Should analyze snippets from filtered emails? | If                     | Same as above but for filtered emails            | Filter emails by label           | Should use AI to filter emails further?, Get Full Message Content |                                                                                                                                         |
| Get Full Message Content            | Gmail (get message)             | Retrieves full email content if needed           | Should analyze snippets?         | Should use AI to filter emails further?       | Requires Gmail OAuth2                                                                                                                   |
| Should use AI to filter emails further? | If                        | Checks if AI classification is needed            | Get Full Message Content, Should analyze snippets? | Message text or snippet present?               |                                                                                                                                         |
| Message text or snippet present?    | Filter                         | Filters emails having text or snippet             | Should use AI to filter emails further? | Classify Emails                               |                                                                                                                                         |
| Classify Emails                    | LangChain Text Classifier       | Classifies emails using Google's Gemini model    | Message text or snippet present? | Text field present?                            | Requires Google Gemini API key                                                                                                          |
| Text field present?                | If                             | Checks if full text present to choose aggregation| Classify Emails                 | Aggregate from full email texts, Aggregate from email snippets |                                                                                                                                         |
| Aggregate from full email texts     | Aggregate                      | Aggregates sender, text, date from full texts    | Text field present?              | Clean text and organize into statements       |                                                                                                                                         |
| Aggregate from email snippets       | Aggregate                      | Aggregates sender, snippet, date from snippets   | Text field present?              | Clean text and organize into statements       |                                                                                                                                         |
| Clean text and organize into statements | Code                      | Cleans HTML from text and formats for graph input| Aggregate nodes                 | Type of graph to build                         |                                                                                                                                         |
| Type of graph to build              | Switch                        | Chooses social or text graph based on setting    | Clean text and organize into statements | InfraNodus Build a Social Knowledge Graph, InfraNodus Build a Text Knowledge Graph |                                                                                                                                         |
| InfraNodus Build a Social Knowledge Graph | HTTP Request              | Builds social knowledge graph with senders       | Type of graph to build (social) | Wait before generating questions               | Requires InfraNodus API key                                                                                                            |
| InfraNodus Build a Text Knowledge Graph | HTTP Request              | Builds text knowledge graph without senders      | Type of graph to build (text)   | Wait before generating questions               | Requires InfraNodus API key                                                                                                            |
| Wait before generating questions    | Wait                          | Waits 60 seconds for InfraNodus processing       | InfraNodus Build nodes          | InfraNodus AI Summary & Graph Link             |                                                                                                                                         |
| InfraNodus AI Summary & Graph Link  | HTTP Request                  | Retrieves AI-generated topical summary           | Wait before generating questions| InfraNodus Question Generator                  | Requires InfraNodus API key                                                                                                            |
| InfraNodus Question Generator       | HTTP Request                  | Retrieves AI-generated insight questions          | InfraNodus AI Summary & Graph Link | Send an insight question via Telegram, Send the graph link and summary via Telegram, Gmail | Requires InfraNodus API key                                                                                                            |
| Send the graph link and summary via Telegram | Telegram                  | Sends knowledge graph link and summary to Telegram| InfraNodus Question Generator  |                                               | Instructions on Telegram bot setup via @botfather                                                                                      |
| Send an insight question via Telegram | Telegram                    | Sends AI-generated insight question to Telegram  | InfraNodus Question Generator   |                                               |                                                                                                                                         |
| Gmail                              | Gmail (send email)             | Sends email summary with graph link and insights | InfraNodus Question Generator   |                                               |                                                                                                                                         |
| Sticky Note2                      | Sticky Note                    | Explains filter settings and Gmail search operators|                                |                                               |                                                                                                                                         |
| Sticky Note3                      | Sticky Note                    | Explains Gmail message retrieval and label usage |                                |                                               |                                                                                                                                         |
| Sticky Note4                      | Sticky Note                    | Explains choice between analyzing snippets or full text |                            |                                               |                                                                                                                                         |
| Sticky Note5                      | Sticky Note                    | Explains AI classifier usage and Gemini model    |                                |                                               |                                                                                                                                         |
| Sticky Note6                      | Sticky Note                    | Explains text cleaning and InfraNodus formatting |                                |                                               |                                                                                                                                         |
| Sticky Note7                      | Sticky Note                    | Explains knowledge graph building and InfraNodus API |                            |                                               |                                                                                                                                         |
| Sticky Note8                      | Sticky Note                    | Explains InfraNodus AI summary and insight question generation |                            |                                               |                                                                                                                                         |
| Sticky Note9                      | Sticky Note                    | Explains Telegram notification setup             |                                |                                               |                                                                                                                                         |
| Sticky Note10                     | Sticky Note                    | Explains types of knowledge graphs and shows examples |                            |                                               |                                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Form Trigger** node named "User submits form" with Basic Auth enabled, defining form fields for:  
     - Search Criteria (text)  
     - Label Filter (text)  
     - Email Type Description (text)  
     - Analyze Full Email Text? (dropdown: Yes/No)  
     - Build a Social Graph? (dropdown: Yes/No)  
     - Graph Name (text)  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - Add a **Schedule Trigger** node named "Schedule Trigger" (disabled by default), configured to run daily at 10:00.

2. **Set Processing Parameters:**  
   - Add a **Set** node named "Assign Processing Settings" connected from all triggers.  
   - Configure to set JSON with conditional expressions:  
     - "Label Filter" with default "CATEGORY_PERSONAL"  
     - "Email Type Description" with default "Important personalized emails, exclude mailouts and non-urgent notifications"  
     - "Analyze Full Email Text?" default empty  
     - "Build a Social Graph?" default "Yes"  
     - "Search Criteria" default to yesterday's date filter in Gmail format (e.g., after:YYYY/MM/DD)  
     - "Graph Name" default with prefix "n8n_gmail_search_" plus current date.

3. **Retrieve Emails:**  
   - Add a **Gmail** node named "Get Messages by Search Criteria" configured to:  
     - Operation: getAll  
     - Filter "q" equal to the "Search Criteria" from previous node.  
     - Use Gmail OAuth2 credentials.  
   - Connect "Assign Processing Settings" to this node.

4. **Filter by Label:**  
   - Add an **If** node named "Was label provided?" checking if "Label Filter" is not empty.  
   - Connect "Get Messages by Search Criteria" to this node.  
   - Add a **Filter** node named "Filter emails by label" to check if email labels contain the label filter string.  
   - Connect the true branch of "Was label provided?" to "Filter emails by label".

5. **Decide Text or Snippets:**  
   - Add two **If** nodes:  
     - "Should analyze snippets from filtered emails?" connected from "Filter emails by label".  
     - "Should analyze snippets?" connected from false branch of "Was label provided?".  
   - Both check if "Analyze Full Email Text?" equals "Snippets" or is empty.

6. **Retrieve Full Message Content (if needed):**  
   - Add a **Gmail** node "Get Full Message Content" with operation "get" using message ID.  
   - Connect from the false branches of the above "Should analyze snippets" nodes.

7. **Apply AI Filtering (optional):**  
   - Add an **If** node "Should use AI to filter emails further?" checking if "Email Type Description" is not empty.  
   - Connect from "Get Full Message Content" and from "Should analyze snippets?" true branch.

8. **Check for Text/Snippet Presence:**  
   - Add a **Filter** node "Message text or snippet present?" checking if either text or snippet exists.  
   - Connect from "Should use AI to filter emails further?".

9. **Classify Emails with AI:**  
   - Add a **LangChain Text Classifier** node "Classify Emails" using Google Gemini model.  
   - Input text: email text or snippet.  
   - Categories: "Matched User's Condition" with description from "Email Type Description" and "Spam and Notifications".  
   - Connect from "Message text or snippet present?".

10. **Check for Full Text:**  
    - Add an **If** node "Text field present?" to check if full text is present, connected from "Classify Emails".

11. **Aggregate Data:**  
    - Add two **Aggregate** nodes:  
      - "Aggregate from full email texts" aggregating "from.text", "text", "date" fields.  
      - "Aggregate from email snippets" aggregating "From", "snippet", "internalDate".  
    - Connect true branch of "Text field present?" to full texts aggregate, false to snippets aggregate.

12. **Clean and Prepare Statements:**  
    - Add a **Code** node "Clean text and organize into statements" to:  
      - Remove HTML, scripts, styles, comments from text  
      - Decode HTML entities  
      - Format to InfraNodus statements with sender and date.  
    - Connect from both aggregate nodes.

13. **Select Graph Type:**  
    - Add a **Switch** node "Type of graph to build" checking "Build a Social Graph?" parameter.  
    - If not empty, route to social graph; otherwise, to text graph.

14. **Build Knowledge Graph:**  
    - Add two **HTTP Request** nodes:  
      - "InfraNodus Build a Social Knowledge Graph" with POST to InfraNodus API v1/graphAndStatements with context settings to process mentions.  
      - "InfraNodus Build a Text Knowledge Graph" with POST to same endpoint but excludes mentions and double brackets.  
    - Use InfraNodus API bearer token credentials.  
    - Connect switch outputs accordingly.

15. **Wait for Processing:**  
    - Add a **Wait** node "Wait before generating questions" configured to wait 60 seconds.  
    - Connect both InfraNodus build nodes to this.

16. **Get Summary and Insight Questions:**  
    - Add two **HTTP Request** nodes:  
      - "InfraNodus AI Summary & Graph Link" POST to InfraNodus API requesting summary with optimization.  
      - "InfraNodus Question Generator" POST requesting insight questions.  
    - Connect "Wait before generating questions" to "InfraNodus AI Summary & Graph Link", then to "InfraNodus Question Generator".

17. **Send Notifications:**  
    - Add two **Telegram** nodes:  
      - "Send the graph link and summary via Telegram" sending graph link and summary text.  
      - "Send an insight question via Telegram" sending insight question text.  
    - Add a **Gmail** node to send an email summary with graph link and insights.  
    - Use Telegram bot credentials and Gmail OAuth2 credentials.  
    - Connect "InfraNodus Question Generator" to all three notification nodes.

18. **Add Sticky Notes:**  
    - Add explanatory sticky notes at relevant positions to provide user guidance on filtering, Gmail setup, AI classification, InfraNodus usage, and notification setup.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Workflow supports multiple trigger methods: form, schedule, manual, webhook. Activation of one requires deactivation of others.    | Sticky Note 1                                                                                                           |
| Gmail search operators greatly improve filtering speed and precision. See: https://support.google.com/mail/answer/7190?hl=en        | Sticky Note 1, Sticky Note 2                                                                                             |
| InfraNodus API key required to build knowledge graphs and generate AI summaries and questions. Obtain at https://infranodus.com/api-access | Sticky Notes 7 & 8                                                                                                      |
| Telegram bot creation via @botfather takes seconds and enables receiving notifications. Bot token required in Telegram node credentials. | Sticky Note 9                                                                                                           |
| Two types of knowledge graphs: Text Graph (topics only) and Social Graph (with senders). See visual examples in Sticky Note 10.    | Sticky Note 10                                                                                                          |
| Google's Gemini AI model is used for email classification, offering faster and more secure processing within Google's ecosystem.   | Sticky Note 5                                                                                                           |
| Use of wait node ensures InfraNodus has time to process the graph before summary and questions are requested.                        | Wait before generating questions node                                                                                   |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---