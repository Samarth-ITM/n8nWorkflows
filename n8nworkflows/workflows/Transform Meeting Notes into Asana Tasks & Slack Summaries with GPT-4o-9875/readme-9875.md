Transform Meeting Notes into Asana Tasks & Slack Summaries with GPT-4o

https://n8nworkflows.xyz/workflows/transform-meeting-notes-into-asana-tasks---slack-summaries-with-gpt-4o-9875


# Transform Meeting Notes into Asana Tasks & Slack Summaries with GPT-4o

---
### 1. Workflow Overview

This workflow automates the transformation of meeting notes into actionable project tasks in Asana and a summarized message sent to a Slack channel. It targets teams and project managers who want to streamline meeting follow-ups by extracting key decisions, action items, pain points, budget mentions, and next meeting dates using AI.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collects meeting notes and relevant metadata via a form trigger.
- **1.2 AI Processing:** Uses GPT-4o to extract structured data from raw meeting notes.
- **1.3 Data Structuring:** Parses AI output and contextual form data into actionable items.
- **1.4 Task Creation in Asana:** Converts action items into Asana tasks with appropriate metadata.
- **1.5 Task Aggregation:** Aggregates created tasks for summary formatting.
- **1.6 Slack Summary Preparation and Delivery:** Formats a comprehensive Slack message and posts it to the specified channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives meeting notes and metadata from a user via a web form, serving as the workflow's entry point.

- **Nodes Involved:**  
  - Meeting Notes Form

- **Node Details:**

  - **Meeting Notes Form**  
    - *Type & Role:* Form Trigger node; initiates workflow on form submission.  
    - *Configuration:*  
      - Form titled "Meeting Notes → Tasks & Summary"  
      - Fields:  
        - Company Name / Project Name (required)  
        - Paste in your meeting notes here (textarea, required)  
        - Asana Project ID (required)  
        - Slack Channel Name (optional, default to "#general")  
      - Attribution disabled  
      - Webhook ID specified for triggering  
    - *Key Expressions:* Uses form field values for downstream AI extraction and task creation.  
    - *Input / Output:* No input; outputs form data as JSON.  
    - *Potential Failures:* Form submission errors, invalid or missing required fields, webhook connectivity issues.  
    - *Sticky Notes:* "Step 1: Submit Notes" explains form fields and suggests replacing with email trigger if desired.

#### 2.2 AI Processing

- **Overview:**  
  This block uses GPT-4o to extract structured meeting insights from the raw notes.

- **Nodes Involved:**  
  - AI Extract Data

- **Node Details:**

  - **AI Extract Data**  
    - *Type & Role:* Langchain OpenAI node; sends prompt to GPT-4o.  
    - *Configuration:*  
      - Model: GPT-4o  
      - System message positions the AI as a helpful assistant extracting structured data.  
      - Prompt instructs extraction of: key decisions, action items (with task, assignee, due date), next meeting date, pain points, and budget mentions.  
      - Output: JSON parsed from AI response.  
    - *Key Expressions:* Uses expression to insert meeting notes from form: `{{ $json["Paste in your meeting notes here"] }}`  
    - *Input / Output:* Input from form trigger; outputs structured JSON data.  
    - *Potential Failures:* AI response timeout, rate limits, malformed JSON output, missing or ambiguous data in notes.  
    - *Sticky Notes:* "Step 2: AI Extraction" details what AI extracts.

#### 2.3 Data Structuring

- **Overview:**  
  Takes AI output and form data, combines and restructures it into individual action item entries enriched with context.

- **Nodes Involved:**  
  - Structure Data

- **Node Details:**

  - **Structure Data**  
    - *Type & Role:* Code node; parses and structures data for task creation.  
    - *Configuration:*  
      - Extracts AI content from previous node's JSON.  
      - Retrieves form data values for company name, notes, Asana project ID, Slack channel (defaults to "#general" if empty).  
      - Maps each action item into a separate output item JSON, adding contextual fields like key decisions, pain points, next meeting date, and budget.  
    - *Key Expressions:* Uses references to previous nodes by name, e.g., `$('Meeting Notes Form').item.json` and `$input.item.json.message.content`.  
    - *Input / Output:* Input from AI Extract Data; outputs one item per action item for Asana task creation.  
    - *Potential Failures:* Null or undefined AI output fields, empty action items causing empty outputs, expression failures.  
    - *Sticky Notes:* None directly attached.

#### 2.4 Task Creation in Asana

- **Overview:**  
  Creates Asana tasks for each action item using OAuth2 authentication and includes detailed notes.

- **Nodes Involved:**  
  - Create Asana Task

- **Node Details:**

  - **Create Asana Task**  
    - *Type & Role:* Asana node; creates tasks in Asana project.  
    - *Configuration:*  
      - Task Name: from `task` field of input JSON  
      - Notes: includes assignee, company/project name, full meeting notes for context.  
      - Due date: from `due_date` field; accepts nulls if none.  
      - Assignee: hardcoded as "me" (the authenticated user)  
      - Project: from `asana_project_id` field  
      - Authentication: OAuth2, configured externally.  
    - *Key Expressions:* Uses dynamic expressions to fill task fields from input JSON.  
    - *Input / Output:* Input from Structure Data; outputs created task details.  
    - *Potential Failures:* Authentication errors, invalid project IDs, due date parsing errors, API rate limits.  
    - *Sticky Notes:* "Step 3: Create Tasks" emphasizes task metadata setup.

#### 2.5 Task Aggregation

- **Overview:**  
  Aggregates all created Asana tasks into a single data object to prepare for Slack summary formatting.

- **Nodes Involved:**  
  - Combine Tasks

- **Node Details:**

  - **Combine Tasks**  
    - *Type & Role:* Aggregate node; consolidates multiple task items into one.  
    - *Configuration:* Aggregates all item data into a field named "Tasks".  
    - *Input / Output:* Input from Create Asana Task; outputs one aggregated JSON object.  
    - *Potential Failures:* Empty input array, data inconsistency in aggregated tasks.  
    - *Sticky Notes:* None directly attached.

#### 2.6 Slack Summary Preparation and Delivery

- **Overview:**  
  Formats a rich Slack message summarizing meeting outcomes and posts it to the specified channel.

- **Nodes Involved:**  
  - Format Slack Message  
  - Send to Slack

- **Node Details:**

  - **Format Slack Message**  
    - *Type & Role:* Code node; builds Slack message string from aggregated tasks and meeting context.  
    - *Configuration:*  
      - Extracts aggregated tasks and contextual info: company name, Slack channel, key decisions, pain points, next meeting date, budget.  
      - Formats message sections with markdown for: meeting summary, key decisions, clickable action items with links, pain points, budget, next meeting date, and project name.  
      - Handles missing data gracefully (empty arrays, nulls).  
    - *Input / Output:* Input from Combine Tasks; outputs JSON with Slack `channel` and `message`.  
    - *Potential Failures:* Missing task properties (e.g., URL), Slack formatting errors, empty data.  
    - *Sticky Notes:* "Step 4: Slack Summary" explains message content structure.

  - **Send to Slack**  
    - *Type & Role:* Slack node; sends message to Slack channel using OAuth2.  
    - *Configuration:*  
      - Text: from formatted message JSON  
      - Channel: dynamically from JSON channel field  
      - Authentication: OAuth2, configured externally  
    - *Input / Output:* Input from Format Slack Message; no output needed.  
    - *Potential Failures:* Slack API rate limits, invalid channel names, auth errors, network timeouts.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                           | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                   |
|---------------------|----------------------------|-----------------------------------------|-----------------------|-------------------------|-----------------------------------------------------------------------------------------------|
| Meeting Notes Form   | Form Trigger               | Collects meeting notes and metadata     | -                     | AI Extract Data          | "Step 1: Submit Notes" describes form fields and usage.                                      |
| AI Extract Data      | Langchain OpenAI           | Extracts structured data from notes     | Meeting Notes Form     | Structure Data           | "Step 2: AI Extraction" details AI output elements.                                          |
| Structure Data       | Code                       | Parses AI output & form data into items | AI Extract Data        | Create Asana Task        |                                                                                               |
| Create Asana Task    | Asana                      | Creates Asana tasks for action items    | Structure Data         | Combine Tasks            | "Step 3: Create Tasks" explains task metadata setup.                                         |
| Combine Tasks        | Aggregate                  | Aggregates all created Asana tasks      | Create Asana Task      | Format Slack Message     |                                                                                               |
| Format Slack Message | Code                       | Formats meeting summary for Slack       | Combine Tasks          | Send to Slack            | "Step 4: Slack Summary" describes Slack message contents.                                    |
| Send to Slack        | Slack                      | Sends formatted summary to Slack channel| Format Slack Message   | -                       |                                                                                               |
| Sticky Note6         | Sticky Note                | Notes on overall workflow purpose       | -                     | -                       | "Uses AI to extract key info, creates Asana tasks, sends Slack summary, works with any notes."|
| Sticky Note7         | Sticky Note                | Notes on form submission step            | -                     | -                       | "Step 1: Submit Notes" form details and tip for email trigger.                               |
| Sticky Note8         | Sticky Note                | Notes on AI extraction step              | -                     | -                       | "Step 2: AI Extraction" details extracted data points.                                       |
| Sticky Note9         | Sticky Note                | Notes on task creation step               | -                     | -                       | "Step 3: Create Tasks" task properties description.                                          |
| Sticky Note10        | Sticky Note                | Notes on Slack summary step               | -                     | -                       | "Step 4: Slack Summary" Slack message formatting and team alignment.                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: `Meeting Notes Form`  
   - Type: Form Trigger  
   - Configure form with title: "Meeting Notes → Tasks & Summary"  
   - Add fields:  
     - *Company Name / Project Name* (required, single-line text)  
     - *Paste in your meeting notes here* (required, textarea)  
     - *Asana Project ID* (required, single-line text)  
     - *Slack Channel Name* (optional, single-line text)  
   - Disable attribution option  
   - Save webhook URL for triggering workflow.

2. **Add Langchain OpenAI Node**  
   - Name: `AI Extract Data`  
   - Type: Langchain OpenAI  
   - Select model: GPT-4o  
   - Set system message: "You are a helpful, intelligent personal assistant. You extract structured data from meeting notes."  
   - Add user message with prompt:  
     ```
     Extract the following from these meeting notes:

     Meeting notes:
     {{ $json["Paste in your meeting notes here"] }}

     Return a JSON object with these exact keys:
     {
       "key_decisions": ["decision 1", "decision 2"],
       "action_items": [
         {
           "task": "task description",
           "assignee": "person name or 'Unassigned'",
           "due_date": "YYYY-MM-DD or null"
         }
       ],
       "next_meeting_date": "YYYY-MM-DD or null",
       "pain_points": ["pain point 1", "pain point 2"],
       "budget_mentioned": "dollar amount or null"
     }

     Rules:
     - If no action items found, return empty array
     - If no assignee mentioned, use "Unassigned"
     - If no due date mentioned, use null
     - Extract dates in YYYY-MM-DD format
     ```  
   - Enable JSON output parsing.

3. **Add Code Node for Data Structuring**  
   - Name: `Structure Data`  
   - Type: Code  
   - Use JavaScript to:  
     - Extract AI response content from `AI Extract Data`.  
     - Retrieve form data from `Meeting Notes Form`.  
     - Map each action item into output items with meeting context fields: company name, notes, Asana project ID, Slack channel (default "#general" if missing), key decisions, pain points, next meeting date, budget.  
   - Connect `AI Extract Data` output to this node.

4. **Add Asana Node to Create Tasks**  
   - Name: `Create Asana Task`  
   - Type: Asana  
   - Authentication: OAuth2 (configure Asana OAuth credentials in n8n)  
   - Set task name to `={{ $json.task }}`  
   - Set notes with multiline template including assignee, company/project, and notes from meeting.  
   - Set due date to `={{ $json.due_date }}` (accept nulls)  
   - Set assignee to "me" (authenticated user)  
   - Set project to `={{ $json.asana_project_id }}`  
   - Connect `Structure Data` output to this node.

5. **Add Aggregate Node to Combine Tasks**  
   - Name: `Combine Tasks`  
   - Type: Aggregate  
   - Aggregate all item data into field named `Tasks`.  
   - Connect `Create Asana Task` output here.

6. **Add Code Node for Slack Message Formatting**  
   - Name: `Format Slack Message`  
   - Type: Code  
   - Use JavaScript to:  
     - Extract aggregated tasks from `Combine Tasks`.  
     - Retrieve meeting context from `Structure Data` first item.  
     - Format a Slack message with sections: meeting summary header, key decisions, clickable action items with their URLs, pain points, budget, next meeting date, and project name.  
     - Default Slack channel to "#general" if missing.  
   - Output JSON with keys: `channel` and `message`.  
   - Connect `Combine Tasks` output to this node.

7. **Add Slack Node to Send Message**  
   - Name: `Send to Slack`  
   - Type: Slack  
   - Authentication: OAuth2 (configure Slack OAuth credentials in n8n)  
   - Set text to `={{ $json.message }}`  
   - Set channel to `={{ $json.channel }}` (dynamic)  
   - Connect `Format Slack Message` output to this node.

8. **Connect Workflow**  
   - Connect nodes in the following order:  
     `Meeting Notes Form` → `AI Extract Data` → `Structure Data` → `Create Asana Task` → `Combine Tasks` → `Format Slack Message` → `Send to Slack`.

9. **Test and Validate**  
   - Submit test meeting notes via form.  
   - Verify Asana tasks created with correct metadata.  
   - Confirm Slack message posts correctly and is well formatted.  
   - Handle errors such as missing fields, AI response issues, or API failures.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                        |
|----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| The workflow eliminates manual meeting follow-up work by leveraging GPT-4o AI to extract actionable items.| Sticky Note6: Overview of workflow benefits and supported note formats.               |
| Form trigger can be replaced by email trigger for automated note intake.                                  | Sticky Note7: Suggestion for replacing manual form with email input for automation.   |
| GPT-4o extract includes decisions, action items, due dates, pain points, budget, and next meeting dates. | Sticky Note8: Detailed AI extraction data points.                                    |
| Each action item creates a detailed Asana task with context for better tracking.                          | Sticky Note9: Task creation metadata details.                                        |
| Slack summary message includes clickable links to tasks and key meeting insights to align teams.         | Sticky Note10: Slack summary formatting and team communication rationale.             |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.