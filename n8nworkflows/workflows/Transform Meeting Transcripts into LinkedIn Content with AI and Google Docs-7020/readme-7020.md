Transform Meeting Transcripts into LinkedIn Content with AI and Google Docs

https://n8nworkflows.xyz/workflows/transform-meeting-transcripts-into-linkedin-content-with-ai-and-google-docs-7020


# Transform Meeting Transcripts into LinkedIn Content with AI and Google Docs

### 1. Workflow Overview

This workflow automates the transformation of meeting transcripts into polished LinkedIn posts using AI and Google Docs. It is designed for professionals such as coaches, consultants, and content creators who want to effortlessly share insights from their meetings on LinkedIn.

The workflow logically divides into these key blocks:

- **1.1 Meeting Detection & Filtering**: Monitors Google Calendar for meeting start events and filters based on meeting summary keywords to target relevant meetings only.
- **1.2 Meeting Completion Wait**: Waits for the meeting to end before proceeding, ensuring transcripts are ready.
- **1.3 Transcript Collection via Email**: Sends an interactive Gmail form requesting the meeting transcript and user preferences (post type, tone, instructions).
- **1.4 AI Content Generation**: Uses LangChain AI agents to analyze the provided transcript and generate LinkedIn post content tailored to personal or company posts, respecting tone and brand voice.
- **1.5 Content Formatting & Storage**: Creates a structured Google Drive folder with two Google Docs — one for the transcript and one for the generated LinkedIn post.
- **1.6 Notification**: Sends an email with links to both documents for user review.

---

### 2. Block-by-Block Analysis

#### 2.1 Meeting Detection & Filtering

**Overview:**  
This block triggers the workflow at the start of a calendar event, filtering to only continue with relevant meetings based on keywords in the event summary.

**Nodes Involved:**  
- New Event Started (Google Calendar Trigger)  
- Filter Unwanted Event Type (Filter)  

**Node Details:**

- **New Event Started**  
  - Type: Google Calendar Trigger  
  - Configuration: Triggers on every event start, polling every minute. Calendar ID must be configured to the user's calendar.  
  - Input: None (trigger node)  
  - Output: Event data containing event summary, start and end times.  
  - Edge Cases: If calendar credentials expire or calendar ID is invalid, node fails.  
  - Sticky Note: Explains how the Google Calendar trigger works and filtering rationale.  
  - Link: [Google Calendar Trigger Docs](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlecalendartrigger/)  

- **Filter Unwanted Event Type**  
  - Type: Filter  
  - Configuration: Continues only if the event summary contains specified keywords (e.g., "key words") relevant to the meeting type.  
  - Input: Event data from 'New Event Started'  
  - Output: Passes filtered events forward; blocks others.  
  - Edge Cases: Expression failure if event summary is missing or empty.  
  - Sticky Note: Details on filtering to avoid unrelated meetings triggering the flow.

---

#### 2.2 Meeting Completion Wait

**Overview:**  
Waits until the meeting's scheduled end time before moving on, preventing premature requests for transcripts.

**Nodes Involved:**  
- Wait till Even End (Wait)  

**Node Details:**

- **Wait till Even End**  
  - Type: Wait  
  - Configuration: Waits until the event's end dateTime extracted from filtered event data.  
  - Input: Filtered event data with meeting end time  
  - Output: Continues after the wait period (meeting end time).  
  - Edge Cases: Misconfigured or missing end time may cause failures or immediate continuation.  
  - Sticky Note: Explains the wait node's role in avoiding interruptions during meetings.  
  - Link: [Wait Node Docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/)

---

#### 2.3 Transcript Collection via Email

**Overview:**  
Sends a Gmail form email after the meeting ends, prompting the user to provide the meeting transcript and posting preferences.

**Nodes Involved:**  
- Need Transcript to be Provided (Gmail, sendAndWait)  

**Node Details:**

- **Need Transcript to be Provided**  
  - Type: Gmail (sendAndWait)  
  - Configuration: Sends an email with detailed instructions and a custom form containing fields: Meeting Transcript (textarea, required), Post Type (dropdown, required), Tone of Voice (multiselect dropdown, required), Additional Instructions (optional).  
  - Subject includes the filtered event summary dynamically.  
  - Input: Meeting end time from Wait node  
  - Output: Waits for user reply with form data including transcript and preferences.  
  - Edge Cases: Email delivery failure, user not responding, or malformed replies.  
  - Sticky Note: Describes how the Gmail node collects user input interactively.  
  - Link: [Gmail Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)

---

#### 2.4 AI Content Generation

**Overview:**  
Based on the user inputs, routes to the appropriate AI agent to generate LinkedIn posts in either a personal or company style, using LangChain agents with stored brand memory.

**Nodes Involved:**  
- Matching Post Type (Switch)  
- Simple Memory (Memory Buffer Window, personal)  
- Simple Memory1 (Memory Buffer Window, company)  
- Personal LinkedIn Generator (LangChain Agent)  
- Company LinkedIn Generator (LangChain Agent)  
- Structured Output Parser (LangChain Output Parser, personal)  
- Structured Output Parser1 (LangChain Output Parser, company)  

**Node Details:**

- **Matching Post Type**  
  - Type: Switch  
  - Configuration: Routes based on the 'Post Type' field value ('Personal LinkedIn Post' or 'Company LinkedIn Post').  
  - Input: User form data from Gmail node  
  - Output: To either personal or company AI generator nodes.  
  - Edge Cases: Unrecognized post type values cause dead ends.  

- **Simple Memory / Simple Memory1**  
  - Type: LangChain Memory Buffer Window  
  - Configuration: Custom session key, zero context window length (likely for brand guidelines memory).  
  - Input: Used as AI memory for the respective LinkedIn Generator nodes.  
  - Output: Provides brand voice context to agents.  
  - Edge Cases: Requires correct memory initialization; missing data may degrade AI output quality.  

- **Personal LinkedIn Generator / Company LinkedIn Generator**  
  - Type: LangChain AI Agent  
  - Configuration: Uses a detailed prompt instructing the AI to generate LinkedIn posts following brand voice, tone, and length constraints. Inputs are transcript and user preferences. Output must be JSON with "post_title" and "post_content".  
  - Input: User form data and memory nodes  
  - Output: JSON post content, parsed by respective Structured Output Parser nodes.  
  - Edge Cases: AI provider API limits, malformed JSON response, or incomplete data.  
  - Sticky Note: Explains AI usage for content creation with brand consistency.  
  - Link: [LangChain Agent Node Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/)

- **Structured Output Parser / Structured Output Parser1**  
  - Type: LangChain Output Parser Structured  
  - Configuration: Validates and structures AI JSON output containing post title and content. One parser for each post type.  
  - Input: AI agent raw output  
  - Output: Parsed JSON to be used downstream.  
  - Edge Cases: Parsing errors if AI output is malformed or incomplete.

---

#### 2.5 Content Formatting & Storage

**Overview:**  
Generates a Google Drive folder and creates two Google Docs: one for the original transcript and one for the LinkedIn post content, updating both with the respective texts.

**Nodes Involved:**  
- Set Fields (Set)  
- Create New Folder (Google Drive)  
- Create Content Doc (Google Docs)  
- Update Content Doc (Google Docs)  
- Create Transcript Doc (Google Docs)  
- Update Transcript Doc (Google Docs)  

**Node Details:**

- **Set Fields**  
  - Type: Set  
  - Configuration: Assigns the parsed AI output fields "post_title" and "post_content" into the workflow data for use in document creation.  
  - Input: Parsed AI JSON output  
  - Output: Data with explicit post fields.  

- **Create New Folder**  
  - Type: Google Drive  
  - Configuration: Creates a new folder in "My Drive" root. Folder name is empty string (likely requires user customization).  
  - Input: Post data from Set Fields node  
  - Output: Folder metadata including folder ID.  
  - Edge Cases: Insufficient Google Drive permissions or quota errors.  
  - Sticky Note: Describes organization of files in Google Drive.  
  - Link: [Google Drive Node Docs](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/)

- **Create Content Doc**  
  - Type: Google Docs  
  - Configuration: Creates a Google Doc titled as the LinkedIn post title in the newly created folder.  
  - Input: Folder ID from Create New Folder  
  - Output: Document metadata including document ID.  

- **Update Content Doc**  
  - Type: Google Docs  
  - Configuration: Inserts LinkedIn post content text into the created content doc.  
  - Input: Content doc ID and post content from Set Fields  
  - Output: Updated document info.  

- **Create Transcript Doc**  
  - Type: Google Docs  
  - Configuration: Creates a Google Doc titled "Coaching's Transcript" in the new folder.  
  - Input: Folder ID  
  - Output: Document metadata.  

- **Update Transcript Doc**  
  - Type: Google Docs  
  - Configuration: Inserts the raw meeting transcript text (collected from user form) into the transcript doc.  
  - Input: Transcript doc ID and transcript text from Gmail node  
  - Output: Updated document info.  
  - Edge Cases: Google Docs API quota, permissions errors.  
  - Sticky Note: Notes on storing transcript and content side-by-side.

---

#### 2.6 Notification

**Overview:**  
Sends a final Gmail notification with links to the created Google Docs for the transcript and LinkedIn post, signaling workflow completion.

**Nodes Involved:**  
- Content Results (Gmail)  

**Node Details:**

- **Content Results**  
  - Type: Gmail (send)  
  - Configuration: Sends an email with a message body containing URLs to the Google Docs created for transcript and content, dynamically inserted. Subject is empty string (likely user to customize).  
  - Input: Document IDs from Google Docs nodes  
  - Output: Email sent confirmation  
  - Edge Cases: Email delivery failure or invalid document links.

---

### 3. Summary Table

| Node Name                    | Node Type                                | Functional Role                           | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                                                      |
|------------------------------|-----------------------------------------|-----------------------------------------|-----------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| New Event Started             | Google Calendar Trigger                  | Trigger on meeting start event          | None                        | Filter Unwanted Event Type        | Explains Google Calendar trigger and filtering. [Link](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.googlecalendartrigger/) |
| Filter Unwanted Event Type    | Filter                                  | Filter meetings by keywords              | New Event Started           | Wait till Even End                | Details filtering criteria to avoid unwanted meetings.                                                                        |
| Wait till Even End            | Wait                                    | Wait until meeting ends                   | Filter Unwanted Event Type  | Need Transcript to be Provided    | Explains wait node usage to avoid interruption. [Link](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.wait/)  |
| Need Transcript to be Provided| Gmail (sendAndWait)                      | Email form to collect transcript & prefs| Wait till Even End          | Matching Post Type                | Describes interactive transcript collection via Gmail. [Link](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) |
| Matching Post Type            | Switch                                  | Route based on user-selected post type  | Need Transcript to be Provided | Personal LinkedIn Generator, Company LinkedIn Generator |                                                                                                                               |
| Simple Memory                | LangChain Memory Buffer Window           | AI memory for personal brand guidelines  |                           | Personal LinkedIn Generator      | Stores brand voice and style for personal posts.                                                                                |
| Personal LinkedIn Generator   | LangChain AI Agent                      | Generate personal LinkedIn post           | Matching Post Type, Simple Memory | Structured Output Parser          | AI generates post content in JSON format. [Link](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/) |
| Structured Output Parser      | LangChain Output Parser Structured      | Parse AI JSON output for personal post   | Personal LinkedIn Generator | Set Fields                      |                                                                                                                                |
| Simple Memory1               | LangChain Memory Buffer Window           | AI memory for company brand guidelines    |                            | Company LinkedIn Generator       | Stores brand voice and style for company posts.                                                                                 |
| Company LinkedIn Generator    | LangChain AI Agent                      | Generate company LinkedIn post            | Matching Post Type, Simple Memory1 | Structured Output Parser1         | AI generates post content in JSON format.                                                                                      |
| Structured Output Parser1     | LangChain Output Parser Structured      | Parse AI JSON output for company post    | Company LinkedIn Generator  | Set Fields                      |                                                                                                                                |
| Set Fields                   | Set                                     | Set post_title and post_content fields    | Structured Output Parser, Structured Output Parser1 | Create New Folder               |                                                                                                                                |
| Create New Folder             | Google Drive                            | Create folder for docs                     | Set Fields                  | Create Content Doc                | Organizes content in Google Drive. [Link](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googledrive/)         |
| Create Content Doc            | Google Docs                            | Create Google Doc for LinkedIn post      | Create New Folder           | Update Content Doc               |                                                                                                                                |
| Update Content Doc            | Google Docs                            | Insert LinkedIn post content into doc     | Create Content Doc          | Create Transcript Doc            |                                                                                                                                |
| Create Transcript Doc         | Google Docs                            | Create Google Doc for meeting transcript  | Update Content Doc          | Update Transcript Doc            |                                                                                                                                |
| Update Transcript Doc         | Google Docs                            | Insert meeting transcript text into doc   | Create Transcript Doc       | Content Results                 | Stores transcript alongside content.                                                                                           |
| Content Results              | Gmail (send)                            | Send notification email with doc links    | Update Transcript Doc       | None                           |                                                                                                                                |
| Sticky Note1                 | Sticky Note                            | Workflow overview and purpose             | None                       | None                           | Describes overall workflow benefits and setup instructions.                                                                    |
| Sticky Note                  | Sticky Note                            | Meeting detection & filtering info       | None                       | None                           |                                                                                                                                |
| Sticky Note2                 | Sticky Note                            | Wait node explanation                      | None                       | None                           |                                                                                                                                |
| Sticky Note3                 | Sticky Note                            | Gmail form for transcript collection      | None                       | None                           |                                                                                                                                |
| Sticky Note4                 | Sticky Note                            | AI content generation explanation         | None                       | None                           |                                                                                                                                |
| Sticky Note5                 | Sticky Note                            | Google Docs content organization           | None                       | None                           |                                                                                                                                |
| Sticky Note6                 | Sticky Note                            | Credential and setup requirements          | None                       | None                           |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Calendar Trigger Node: "New Event Started"**  
   - Type: Google Calendar Trigger  
   - Trigger: On event started, poll interval every minute  
   - Configure with your Google Calendar OAuth2 credentials and specify calendar ID  
   - Position accordingly

2. **Add Filter Node: "Filter Unwanted Event Type"**  
   - Type: Filter  
   - Condition: Event summary contains specific keywords identifying relevant meetings  
   - Connect input from "New Event Started"  
   
3. **Add Wait Node: "Wait till Even End"**  
   - Type: Wait  
   - Set to resume at specific time  
   - Expression for dateTime: `{{$node["Filter Unwanted Event Type"].item.json.end.dateTime}}`  
   - Connect input from filter node

4. **Add Gmail Node: "Need Transcript to be Provided"**  
   - Type: Gmail (sendAndWait)  
   - OAuth2 credentials for Gmail configured  
   - Compose email body with instructions requesting transcript and user preferences  
   - Add custom form fields:  
     - Meeting Transcript (textarea, required)  
     - Post Type (dropdown: Personal LinkedIn Post, Company LinkedIn Post, required)  
     - Tone of Voice (multiselect dropdown, options like Professional, Friendly, etc., required)  
     - Additional Instructions (optional text)  
   - Subject includes event summary: `Action Required: Meeting Transcript for LinkedIn Post Creation - {{$node["Filter Unwanted Event Type"].first().json.summary}}`  
   - Connect input from Wait node

5. **Add Switch Node: "Matching Post Type"**  
   - Type: Switch  
   - Rule 1: If `{{$json.data["Post Type"]}}` equals "Personal LinkedIn Post"  
   - Rule 2: If equals "Company LinkedIn Post"  
   - Connect input from Gmail node

6. **Add LangChain Memory Nodes: "Simple Memory" and "Simple Memory1"**  
   - Type: LangChain Memory Buffer Window  
   - Set sessionIdType to customKey  
   - Context window length 0  
   - These hold brand guidelines for personal and company posts respectively  

7. **Add LangChain Agent Nodes: "Personal LinkedIn Generator" and "Company LinkedIn Generator"**  
   - Type: LangChain Agent  
   - Prompt text includes detailed instructions referencing user inputs and brand memory  
   - Configure to output a JSON object with "post_title" and "post_content" fields  
   - Connect "Personal LinkedIn Generator" to "Simple Memory" and the "Personal LinkedIn Post" output of the switch node  
   - Connect "Company LinkedIn Generator" to "Simple Memory1" and the "Company LinkedIn Post" output of the switch node

8. **Add LangChain Structured Output Parser Nodes: "Structured Output Parser" and "Structured Output Parser1"**  
   - Type: LangChain Output Parser Structured  
   - For personal and company LinkedIn generators respectively  
   - Connect AI agent output to these parsers

9. **Add Set Node: "Set Fields"**  
   - Type: Set  
   - Assign variables:  
     - post_title = `{{$json.output.post_title}}`  
     - post_content = `{{$json.output.post_content}}`  
   - Connect input from both Structured Output Parser nodes (fan-in as needed)

10. **Add Google Drive Node: "Create New Folder"**  
    - Type: Google Drive  
    - Operation: Create folder in "My Drive" root or specified location  
    - Name: Provide meaningful folder name (customize as needed)  
    - Connect input from Set Fields

11. **Add Google Docs Node: "Create Content Doc"**  
    - Type: Google Docs  
    - Title: `{{$json.post_title}}`  
    - Folder ID: `{{$node["Create New Folder"].item.json.id}}`  
    - Connect input from Create New Folder

12. **Add Google Docs Node: "Update Content Doc"**  
    - Type: Google Docs  
    - Operation: Update document  
    - Document URL/ID: `{{$node["Create Content Doc"].item.json.id}}`  
    - Insert action: Insert `{{$json.post_content}}` as text  
    - Connect input from Create Content Doc

13. **Add Google Docs Node: "Create Transcript Doc"**  
    - Type: Google Docs  
    - Title: "Coaching's Transcript" (or customizable)  
    - Folder ID: `{{$node["Create New Folder"].item.json.id}}`  
    - Connect input from Update Content Doc

14. **Add Google Docs Node: "Update Transcript Doc"**  
    - Type: Google Docs  
    - Operation: Update document  
    - Document URL/ID: `{{$node["Create Transcript Doc"].item.json.id}}`  
    - Insert action: Insert `{{$node["Need Transcript to be Provided"].item.json.data["Meeting Transcript"]}}` as text  
    - Connect input from Create Transcript Doc

15. **Add Gmail Node: "Content Results"**  
    - Type: Gmail (send)  
    - Compose email with links to:  
      - Content doc: `https://docs.google.com/document/d/{{$node["Create Content Doc"].item.json.id}}`  
      - Transcript doc: `https://docs.google.com/document/d/{{$node["Create Transcript Doc"].item.json.id}}`  
    - Subject and message customizable  
    - Connect input from Update Transcript Doc

16. **Add Sticky Notes** (Optional but recommended for maintainability)  
    - Add descriptive sticky notes at each logical block explaining purpose, usage, and references, including relevant links.

17. **Credentials Setup**  
    - Google Calendar OAuth2 credentials  
    - Gmail OAuth2 credentials  
    - Google Drive OAuth2 credentials  
    - LangChain AI provider credentials (OpenAI, Anthropic, etc.)  
    - Configure memory nodes with your brand guidelines data as needed

18. **Testing and Customization**  
    - Replace placeholders like folder names, calendar IDs, keywords, and email addresses with your own data  
    - Test with sample meetings and transcripts before going live

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                                       |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| This workflow is ideal for coaches, consultants, sales professionals, and content creators who want automated LinkedIn content. | Overview sticky note in the workflow explains the target audience and workflow benefits.                             |
| AI processing may incur costs depending on your LangChain provider and usage frequency.                                          | Sticky note on credentials and setup warns about AI provider costs.                                                  |
| The workflow supports transcripts from any meeting platform via copy-paste in the email form (Zoom, Teams, Google Meet, etc.)   | Email instructions detailed in the Gmail form node configuration.                                                    |
| Can be adapted to use webhook inputs from recording tools such as Fireflies.ai instead of manual transcript submission.          | Mentioned in the overview sticky note for possible customizations.                                                   |
| Brand guidelines for AI content generation are stored in LangChain memory buffer nodes for consistent voice and style.          | Memory nodes "Simple Memory" and "Simple Memory1" serve this purpose.                                                |
| Refer to official n8n documentation for detailed usage of key nodes: Google Calendar Trigger, Gmail, Wait, Google Docs, LangChain AI. | Links in sticky notes and node comments provide direct access to official docs: https://docs.n8n.io/                 |

---

**Disclaimer:**  
The text provided originates solely from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.