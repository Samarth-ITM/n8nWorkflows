Extract Meeting Tasks from Transcripts with AI and Sync to Trello

https://n8nworkflows.xyz/workflows/extract-meeting-tasks-from-transcripts-with-ai-and-sync-to-trello-8652


# Extract Meeting Tasks from Transcripts with AI and Sync to Trello

### 1. Workflow Overview

This workflow automates the extraction of actionable tasks from meeting transcript files and synchronizes these tasks with Trello boards. It is designed for teams who want to convert meeting notes into managed Trello tasks seamlessly, avoiding duplicates and ensuring clear task assignment and deadlines.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives the meeting transcript file (.txt) from a user via a form.
- **1.2 Transcript Extraction**: Extracts raw text from the uploaded transcript file.
- **1.3 AI Task Extraction and Processing**: Uses an AI agent to analyze the transcript, extract clear, actionable tasks with metadata (title, description, assignee, deadline), deduplicate them, and prepare a structured JSON output.
- **1.4 Trello Synchronization Sub-Agent**: For each extracted task, checks against Trello boards to avoid duplicates and creates new Trello cards if needed.
- **1.5 Final Output and User Feedback**: Parses the AI-generated structured output and responds to the user with a concise summary of the tasks processed and their Trello sync status.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures the meeting transcript file uploaded by the user through a web form.

- **Nodes Involved:**  
  - Receive Transcript file

- **Node Details:**

  - **Receive Transcript file**  
    - Type: Form Trigger  
    - Role: Entry point that waits for a user to upload a transcript file (.txt).  
    - Configuration: Single file upload field labeled "Transcript archive", accepts `.txt` files only, required field. Form titled "Tasks from meeting" with a description guiding users to create Trello tasks from the transcript.  
    - Input: User-uploaded file via HTTP form  
    - Output: Binary file data forwarded to the next node  
    - Edge Cases: User submits no file or unsupported file type; form validation manages required field  
    - Version: 2.3

---

#### 2.2 Transcript Extraction

- **Overview:**  
  Extracts plain text from the uploaded transcript file for AI consumption.

- **Nodes Involved:**  
  - Get Transcription

- **Node Details:**

  - **Get Transcription**  
    - Type: Extract From File  
    - Role: Converts the uploaded transcript archive binary data into raw text.  
    - Configuration: Operation set to "text", binary property name set to "Transcript_archive" to extract text content.  
    - Input: Binary file from form trigger  
    - Output: JSON data with plain text field `data` containing the transcript text  
    - Edge Cases: File corrupted or empty; extraction might fail or yield empty text  
    - Version: 1

---

#### 2.3 AI Task Extraction and Processing

- **Overview:**  
  Uses an AI agent to analyze the transcript text, extract actionable tasks with attributes (title, description, assignee, deadline), deduplicate tasks, and prepare a JSON output. Tasks are also sent to the Trello sub-agent for syncing.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - Type: LangChain Agent Node  
    - Role: Core logic node that runs the AI prompt instructing extraction of actionable tasks and syncing with Trello.  
    - Configuration:  
      - Input text template includes the transcript text.  
      - System message defines detailed instructions: extract tasks, deduplicate, sync each task with Trello via the Trello Sub-Agent, build user-facing JSON summary and tasks array.  
      - Uses Trello Agent as a sub-agent for Trello syncing.  
    - Inputs: Transcript text from "Get Transcription" node, OpenAI Chat Model for LLM processing.  
    - Outputs: Structured JSON with summary and tasks.  
    - Edge Cases: AI may misinterpret transcript, no tasks found, API timeout or errors, expression failures.  
    - Version: 2.2  

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the language model (GPT-4.1-mini) used by the AI Agent to process the transcript and generate outputs.  
    - Configuration: Model set to "gpt-4.1-mini", OpenAI credentials configured.  
    - Input: AI Agent prompt text.  
    - Output: AI-generated text responses.  
    - Edge Cases: API rate limits, auth failures, network issues.  
    - Version: 1.2  

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses the AI Agent’s JSON output into structured JSON objects for further processing or display.  
    - Configuration: Example JSON schema provided matching the expected AI output format.  
    - Input: AI Agent raw JSON text.  
    - Output: Parsed structured JSON.  
    - Edge Cases: Malformed JSON or deviation from schema may cause parsing errors.  
    - Version: 1.3

---

#### 2.4 Trello Synchronization Sub-Agent

- **Overview:**  
  Handles Trello API interactions for each task: checking for duplicates across all board lists and creating new cards if no duplicates are found.

- **Nodes Involved:**  
  - Trello Agent  
  - OpenAI Chat Model1  
  - Structured Output Parser1  
  - Get many lists in Trello  
  - Get all cards in a list in Trello  
  - Create a card in Trello

- **Node Details:**

  - **Trello Agent**  
    - Type: LangChain Agent Tool (Sub-Agent)  
    - Role: Sub-agent invoked by the main AI Agent to sync individual tasks with Trello.  
    - Configuration: Receives task JSON (title, description, assignee, deadline). Uses tools to get board lists, fetch cards, and create cards. Implements normalization and fuzzy matching logic to detect duplicates.  
    - Inputs: Task data from AI Agent, OpenAI Chat Model1 for processing similarity and logic.  
    - Outputs: JSON indicating task status on Trello: `exists` or `created` along with card IDs.  
    - Edge Cases: Trello API failures, credential errors, similarity false positives/negatives, rate limits.  
    - Version: 2.2  

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Supports the Trello Agent for AI-driven duplicate detection and logic.  
    - Configuration: Uses GPT-4.1-mini model with OpenAI credentials.  
    - Edge Cases: Same as other OpenAI nodes.  
    - Version: 1.2  

  - **Structured Output Parser1**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses Trello Agent’s output JSON schema indicating Trello sync results.  
    - Configuration: Manual JSON schema requiring `status` and optionally `card_id`, `list_id`, `created_in_default_list`, `message`.  
    - Edge Cases: Parsing errors if Trello Agent returns unexpected JSON.  
    - Version: 1.3  

  - **Get many lists in Trello**  
    - Type: Trello API Tool  
    - Role: Retrieves all lists on the configured Trello board to scan for duplicates.  
    - Configuration: Board ID required in parameters, returns all lists.  
    - Inputs: Triggered by Trello Agent.  
    - Outputs: List of list IDs.  
    - Edge Cases: API auth failures, invalid board ID.  
    - Version: 1  

  - **Get all cards in a list in Trello**  
    - Type: Trello API Tool  
    - Role: Retrieves cards in a specific Trello list including their names and descriptions for duplicate detection.  
    - Configuration: List ID dynamically set from Trello Agent, returns all cards with fields `name` and `desc`.  
    - Edge Cases: API failures, invalid list ID.  
    - Version: 1  

  - **Create a card in Trello**  
    - Type: Trello API Tool  
    - Role: Creates a new Trello card in a preconfigured default list if no duplicate is found.  
    - Configuration: Card name and description dynamically set from task title and description, list ID fixed to a specific list.  
    - Edge Cases: Creation failures, permission issues.  
    - Version: 1  

---

#### 2.5 Final Output and User Feedback

- **Overview:**  
  Sends a final confirmation and summary message back to the user via the form response, showing tasks created vs. existing.

- **Nodes Involved:**  
  - Respond to Form

- **Node Details:**

  - **Respond to Form**  
    - Type: Form Node  
    - Role: Sends a completion message back to the user who uploaded the transcript, containing the summary generated by the AI Agent.  
    - Configuration: Completion title "Graded transcripts and tasks created in Trello", message dynamically injected from AI Agent’s summary text.  
    - Input: AI Agent output (summary text).  
    - Output: HTTP response to form submission.  
    - Edge Cases: Failures to send message, user disconnects.  
    - Version: 2.3

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                               | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                              |
|---------------------------|--------------------------------------|-----------------------------------------------|------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| Receive Transcript file    | Form Trigger                         | Receives transcript file from user via form  | (Start)                      | Get Transcription           | **Form Trigger node** Receives the transcript file (.txt) from the user.                              |
| Get Transcription          | Extract From File                    | Extracts raw text from transcript file        | Receive Transcript file       | AI Agent                   | **Get Transcription node** Extracts raw text from the uploaded transcript file.                      |
| AI Agent                  | LangChain Agent                     | Extracts tasks from transcript, syncs Trello, produces JSON output | Get Transcription, OpenAI Chat Model | Respond to Form             | **AI Agent node** Main agent that extracts tasks, checks duplicates via Trello Sub-Agent, and builds the JSON output (summary + tasks). |
| OpenAI Chat Model          | LangChain OpenAI Chat Model          | Provides LLM processing for AI Agent           | AI Agent                     | AI Agent                   |                                                                                                        |
| Structured Output Parser   | LangChain Structured Output Parser   | Parses AI Agent JSON output                      | AI Agent                     | AI Agent                   |                                                                                                        |
| Trello Agent               | LangChain Agent Tool (Sub-Agent)     | Sub-agent syncing tasks with Trello             | AI Agent, OpenAI Chat Model1  | AI Agent                   | **Trello Agent node** Sub-agent responsible for Trello sync. Uses Get many lists + Get all cards in a list to check duplicates. Creates new cards if no duplicate. |
| OpenAI Chat Model1         | LangChain OpenAI Chat Model          | Provides LLM processing for Trello Agent        | Trello Agent                 | Trello Agent               |                                                                                                        |
| Structured Output Parser1  | LangChain Structured Output Parser   | Parses Trello Agent's Trello sync output        | Trello Agent                 | Trello Agent               |                                                                                                        |
| Get many lists in Trello   | Trello API Tool                     | Retrieves all lists in Trello board             | Trello Agent                 | Trello Agent               |                                                                                                        |
| Get all cards in a list in Trello | Trello API Tool                 | Retrieves all cards in specified Trello list   | Trello Agent                 | Trello Agent               |                                                                                                        |
| Create a card in Trello    | Trello API Tool                     | Creates new card in Trello default list         | Trello Agent                 | Trello Agent               |                                                                                                        |
| Respond to Form            | Form Node                          | Sends summary message back to form user         | AI Agent                     | (End)                      | **Respond to Form node** Returns final confirmation message with summary of tasks created vs existing. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "Receive Transcript file"  
   - Configure form with one file field:  
     - Label: "Transcript archive"  
     - Accept only `.txt` files  
     - Required field  
   - Set form title: "Tasks from meeting"  
   - Form description: "Create Trello tasks from a meeting transcript"  
   - Save and activate webhook for form submissions.

2. **Create Extract From File Node**  
   - Type: Extract From File  
   - Name: "Get Transcription"  
   - Set operation to "text"  
   - Set binary property name to "Transcript_archive" (matches form upload field)  
   - Connect "Receive Transcript file" output to this node input.

3. **Create OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Name: "OpenAI Chat Model"  
   - Select model: "gpt-4.1-mini"  
   - Configure OpenAI credentials  
   - No additional options required.

4. **Create Trello Agent Sub-Agent Setup**  
   - Create a LangChain Agent Tool node named "Trello Agent"  
   - Use system message instructions defining Trello sync logic including duplicate detection and card creation (as per workflow description)  
   - Set up tools used by the agent:  
     - "Get many lists in Trello" node (configured with your Trello Board ID)  
     - "Get all cards in a list in Trello" node (dynamic list ID input)  
     - "Create a card in Trello" node (configured with your default Trello list ID)  
   - Connect these tools as AI tools within the Trello Agent node.  
   - Create a supporting OpenAI Chat Model node "OpenAI Chat Model1" also using "gpt-4.1-mini" and credentials for Trello Agent's AI processing.  
   - Add a Structured Output Parser node "Structured Output Parser1" with the JSON schema for Trello sync results.

5. **Create Main AI Agent Node**  
   - Type: LangChain Agent  
   - Name: "AI Agent"  
   - Configure system message with detailed instructions to:  
     - Extract actionable tasks from the transcript text  
     - Deduplicate tasks  
     - For each task, invoke the "Trello Agent" sub-agent with task data  
     - Build JSON output with summary and tasks array (per provided JSON schema)  
   - Use the "OpenAI Chat Model" node as the LLM.  
   - Set the input text to include the transcript text extracted from "Get Transcription" node.  
   - Connect the "Trello Agent" as a sub-agent AI tool inside the AI Agent node.  
   - Add a Structured Output Parser node "Structured Output Parser" with example JSON schema to parse the AI Agent output.

6. **Create Trello API Nodes**  
   - "Get many lists in Trello":  
     - Resource: list  
     - Operation: getAll  
     - Provide your Trello Board ID  
   - "Get all cards in a list in Trello":  
     - Resource: list  
     - Operation: getCards  
     - List ID dynamically received from Trello Agent  
     - Return all cards with fields `name` and `desc`  
   - "Create a card in Trello":  
     - Resource: card  
     - Operation: create  
     - Name and Description dynamically set from task data  
     - List ID set to your default Trello list ID  
   - Provide Trello API credentials for all Trello nodes.

7. **Create Final Form Response Node**  
   - Type: Form Node  
   - Name: "Respond to Form"  
   - Operation: completion  
   - Completion title: "Graded transcripts and tasks created in Trello"  
   - Completion message set to use the AI Agent’s summary text output  
   - Connect "AI Agent" main output to this node.

8. **Connect Nodes in Sequence:**  
   - Receive Transcript file → Get Transcription → AI Agent → Respond to Form  
   - AI Agent uses OpenAI Chat Model and Trello Agent sub-agent internally  
   - Trello Agent uses its OpenAI Chat Model1 and Trello API nodes internally.

9. **Credential Setup:**  
   - Configure OpenAI API credentials with valid keys for OpenAI Chat Model nodes.  
   - Configure Trello API credentials with access tokens for all Trello API nodes.

10. **Default Values & Constraints:**  
    - Transcript file must be a `.txt` file.  
    - Trello Board ID and default List ID must be replaced with your actual Trello environment values.  
    - Model choice set to GPT-4.1-mini for performance and cost balance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| This workflow processes meeting transcripts to extract actionable tasks and syncs them to Trello, preventing duplicates via fuzzy matching on titles and descriptions. It generates a user-friendly summary in the user’s language (fallback English).                                                                                                              | Workflow description and use case.                                                                                   |
| Requires Trello account with Board and List IDs configured, and OpenAI API credentials for GPT-based task extraction and duplicate detection.                                                                                                                                                                                                                   | Credentials requirement.                                                                                              |
| Sticky notes in the workflow clarify node purposes, e.g., input reception, AI task extraction, Trello sub-agent logic, and user response formatting.                                                                                                                                                                                                            | Workflow annotations.                                                                                                |
| For detailed Trello duplicate detection logic, normalization includes lowercasing, trimming, punctuation and emoji stripping, and optional stopword removal. Matching thresholds are defined for title and description similarities to decide duplicates conservatively.                                                                                          | Trello Agent system message instructions.                                                                             |
| The Trello Sub-Agent returns structured JSON with status (`exists` or `created`), card IDs, and optionally list IDs or error messages. The main AI Agent aggregates these results for the final output summary.                                                                                                                                                     | JSON schema for Trello sync results.                                                                                  |
| Example transcript and expected outputs are documented in the AI Agent prompt, demonstrating practical results with assignees and deadlines.                                                                                                                                                                                                                     | Example usage scenario.                                                                                               |
| For further exploration of LangChain and AI agent setup in n8n, consult official n8n documentation and LangChain resources.                                                                                                                                                                                                                                     | n8n Docs: https://docs.n8n.io/ , LangChain: https://langchain.com/                                                    |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected elements. All data handled is legal and public.