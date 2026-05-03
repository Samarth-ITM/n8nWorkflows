Complete Lesson Automation for Modern UK Teachers

https://n8nworkflows.xyz/workflows/complete-lesson-automation-for-modern-uk-teachers-4927


# Complete Lesson Automation for Modern UK Teachers

### 1. Workflow Overview

This workflow, titled **"Complete Lesson Automation for Modern UK Teachers"**, is designed to automate the creation of comprehensive, curriculum-aligned lesson packages for UK teachers who wish to generate full lesson plans without relying on traditional textbooks. It focuses on producing differentiated teaching materials, assessment tools, AI integration recommendations, and administrative support documents in a streamlined and efficient manner. The workflow culminates in the automatic creation of a formatted Google Doc, scheduling a calendar event for lesson preparation, and sending a confirmation email with the lesson package link.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception & AI Content Generation:** Captures teacher inputs via a form and sequentially generates three AI-driven outputs: lesson content, assessment tools, and AI tool integration recommendations.
  
- **1.2 Package Compilation & Document Generation:** Compiles the AI-generated outputs into a structured package, formats the package into markdown, creates a Google Document in Drive, and inserts the formatted content.
  
- **1.3 Sharing & Notification:** Fetches the created document's link, schedules a calendar event for lesson preparation, and sends a confirmation email with the lesson package link.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & AI Content Generation

**Overview:**  
This block collects detailed lesson parameters from the teacher via a form, then processes these inputs through three specialized AI agents responsible for generating the lesson plan, assessment tools, and AI integration recommendations.

**Nodes Involved:**  
- Teacher Input Form  
- Content Creation Agent  
- Assessment & Marking Agent  
- AI Integration Agent  
- OpenAI Chat Model (3 instances)  
- Sticky Note (contextual annotation)

**Node Details:**

- **Teacher Input Form**  
  - *Type:* Form Trigger  
  - *Role:* Entry point capturing teacher’s lesson parameters via a web form.  
  - *Configuration:*  
    - Webhook path: `/webhook/modern-lesson-generator`  
    - Form fields include dropdowns (Subject, Year Group, Curriculum Standard, Assessment Focus), text inputs (Topic/Chapter, Differentiation Needs), and numeric inputs (Lesson Duration, Class Size).  
    - Form description emphasizes generation of complete lesson resources without textbooks.  
  - *Inputs:* External HTTP POST via webhook  
  - *Outputs:* JSON data with form responses  
  - *Potential Failures:* Webhook connectivity issues, malformed submissions, missing required fields.

- **Content Creation Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Generates comprehensive lesson packages aligned to curriculum and differentiation needs.  
  - *Configuration:* Uses a defined prompt incorporating teacher inputs to generate lesson structure, differentiated materials, assessments, parent support, and administrative tools.  
  - *Inputs:* Output from Teacher Input Form  
  - *Outputs:* Text content of lesson package  
  - *Potential Failures:* AI API connection errors, prompt failures, rate limits.

- **Assessment & Marking Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Creates assessment tools and marking aids that save teacher time while maintaining quality feedback.  
  - *Configuration:* Defined system prompt referencing lesson content and teacher inputs to produce formative and summative assessment materials, automated feedback, and time-saving strategies.  
  - *Inputs:* Output from Content Creation Agent and Teacher Input Form (via expressions referencing nodes)  
  - *Outputs:* Text content of assessment tools  
  - *Potential Failures:* Similar to Content Creation Agent; also possible delayed AI responses due to content length.

- **AI Integration Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Recommends AI tools and automation workflows tailored to teacher’s subject, year group, and lesson content to reduce workload.  
  - *Configuration:* System prompt includes context from teacher input, lesson content, and assessment tools to suggest actionable AI tools and implementation guides.  
  - *Inputs:* Output from Assessment & Marking Agent and Teacher Input Form  
  - *Outputs:* Text content of AI tool recommendations  
  - *Potential Failures:* AI API issues, incomplete contextual data.

- **OpenAI Chat Model (3 nodes)**  
  - *Type:* Language Model (OpenAI GPT-4o-mini)  
  - *Role:* Backend AI model powering each agent node.  
  - *Configuration:* Model set to `gpt-4o-mini`, no extra options.  
  - *Inputs:* Agent prompt text and context  
  - *Outputs:* AI-generated text  
  - *Potential Failures:* API authentication, rate limits, transient network errors.

- **Sticky Note (ID: 95271c6e-d4ec-4082-9576-c2f5915bcb0a)**  
  - *Content:* "1. AI Lesson Plan, Assignment and Integration Agent Trio. Each agent with specialised prompt to do research, build resources and suggest integration ideas."  
  - *Purpose:* Provides conceptual grouping and explanation of the three AI agents.

---

#### 2.2 Package Compilation & Document Generation

**Overview:**  
This block consolidates the outputs from all AI agents into a unified comprehensive package, converts it into markdown format, and creates a Google Document in a specific Drive folder, inserting the formatted content.

**Nodes Involved:**  
- Package Compiler  
- Format Document  
- Create Google Doc  
- Add Content to Doc  
- Sticky Note2

**Node Details:**

- **Package Compiler**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Aggregates inputs from the three AI agents and teacher form, generates metadata, extracts time-saving and tool recommendations, and constructs a detailed JSON package.  
  - *Key Logic:*  
    - Creates unique package ID and timestamp  
    - Extracts time savings from assessment text via regex  
    - Detects recommended AI tools by scanning AI Integration Agent output  
    - Produces structured fields: lesson_package, assessment_tools, ai_automation, implementation_guide  
  - *Inputs:* Outputs from Content Creation Agent, Assessment & Marking Agent, AI Integration Agent, and Teacher Input Form  
  - *Outputs:* JSON object representing entire lesson package metadata and contents  
  - *Potential Failures:* JavaScript errors, missing input data, regex extraction edge cases.

- **Format Document**  
  - *Type:* Code Node (JavaScript)  
  - *Role:* Converts the compiled package JSON into a well-structured markdown document containing sections for lesson content, assessment tools, and AI recommendations.  
  - *Key Logic:* Builds markdown strings with headings, metadata summary, and content blocks.  
  - *Inputs:* Package Compiler output JSON  
  - *Outputs:* JSON with markdown string and package ID  
  - *Potential Failures:* Null or malformed package data, string encoding issues.

- **Create Google Doc**  
  - *Type:* Google Docs Node  
  - *Role:* Creates a new Google Document in a predefined folder with a dynamic title based on teacher inputs.  
  - *Configuration:*  
    - Document title template: `"Modern Lesson: {Subject} - {Topic/Chapter} - {Year Group}"`  
    - Folder ID hardcoded (Google Drive folder for lesson packages)  
    - OAuth2 authentication configured  
  - *Inputs:* Output from Format Document (for title expressions)  
  - *Outputs:* JSON containing new document ID and metadata  
  - *Potential Failures:* OAuth token expiration, permissions errors, folder ID invalid.

- **Add Content to Doc**  
  - *Type:* Google Docs Node  
  - *Role:* Inserts the formatted markdown content into the newly created Google Document.  
  - *Configuration:* Update operation with insert action containing the markdown text from Format Document.  
  - *Inputs:* Document ID from Create Google Doc, markdown content from Format Document  
  - *Outputs:* Confirmation JSON, document URL  
  - *Potential Failures:* API rate limits, document access errors.

- **Sticky Note2**  
  - *Content:* "2. Compile. And create a document in Google Drive"  
  - *Purpose:* Highlights the function of this block as compiling and document creation.

---

#### 2.3 Sharing & Notification

**Overview:**  
This final block handles retrieving the created Google Doc’s shareable link, schedules a calendar event to remind the teacher to prepare the lesson, and sends an email with the lesson package link.

**Nodes Involved:**  
- Fetch File From Drive  
- Schedule Prep Reminder  
- Send Confirmation Email  
- Sticky Note3

**Node Details:**

- **Fetch File From Drive**  
  - *Type:* Google Drive Node  
  - *Role:* Finds the Google Document by name to retrieve its web view link and export URLs for sharing.  
  - *Configuration:*  
    - Query string dynamically set to the document name generated earlier  
    - Requests fields: `webViewLink`, `exportLinks`  
    - OAuth2 authentication configured  
  - *Inputs:* Document name from Create Google Doc output  
  - *Outputs:* JSON with file metadata including shareable link  
  - *Potential Failures:* Search failures if document name changes, permission issues.

- **Schedule Prep Reminder**  
  - *Type:* Google Calendar Node  
  - *Role:* Creates a calendar event on the next day from 8:00 to 8:30 AM to remind the teacher to review and prepare the lesson package.  
  - *Configuration:*  
    - Primary calendar selected  
    - Event location: "Staffroom"  
    - Event description: "Review and prepare the AI-generated lesson package."  
    - OAuth2 authentication configured  
  - *Inputs:* Triggered after document creation  
  - *Outputs:* Confirmation of calendar event creation  
  - *Potential Failures:* OAuth token expiry, calendar API quota exceeded.

- **Send Confirmation Email**  
  - *Type:* Gmail Node  
  - *Role:* Sends an email to a fixed recipient (`ptalur@gmail.com`) with a subject and HTML body containing lesson details and a link to the Google Doc.  
  - *Configuration:*  
    - Dynamic subject: includes the lesson topic  
    - Message uses HTML formatting, embedding the document’s web view link  
    - OAuth2 authentication configured  
  - *Inputs:* Output from Fetch File From Drive (webViewLink), Teacher Input Form data  
  - *Outputs:* Email send confirmation  
  - *Potential Failures:* SMTP errors, authentication failures, email formatting issues.

- **Sticky Note3**  
  - *Content:* "3. Share. Create Calendar event for review. Email the Google Drive Link"  
  - *Purpose:* Annotates the sharing and notification purpose of this block.

---

### 3. Summary Table

| Node Name                | Node Type                       | Functional Role                                         | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                  |
|--------------------------|--------------------------------|--------------------------------------------------------|--------------------------------|-------------------------------|--------------------------------------------------------------|
| Teacher Input Form        | Form Trigger                   | Captures teacher lesson parameters                      | External webhook               | Content Creation Agent         |                                                              |
| Content Creation Agent    | Langchain Agent                | Generates comprehensive lesson package                  | Teacher Input Form             | Assessment & Marking Agent     | Covered by Sticky Note about AI Lesson Plan, Assignment, Integration Agents |
| Assessment & Marking Agent| Langchain Agent                | Creates assessment tools and marking aids               | Content Creation Agent         | AI Integration Agent           | Covered by Sticky Note about AI Lesson Plan, Assignment, Integration Agents |
| AI Integration Agent      | Langchain Agent                | Recommends AI tools and automation workflows            | Assessment & Marking Agent     | Package Compiler              | Covered by Sticky Note about AI Lesson Plan, Assignment, Integration Agents |
| OpenAI Chat Model         | Language Model (OpenAI GPT)    | Backend AI model for Content Creation Agent             | -                            | Content Creation Agent         |                                                              |
| OpenAI Chat Model1        | Language Model (OpenAI GPT)    | Backend AI model for Assessment & Marking Agent         | -                            | Assessment & Marking Agent     |                                                              |
| OpenAI Chat Model2        | Language Model (OpenAI GPT)    | Backend AI model for AI Integration Agent                | -                            | AI Integration Agent           |                                                              |
| Package Compiler          | Code Node (JavaScript)          | Aggregates AI outputs into structured lesson package     | AI Integration Agent           | Format Document               | Covered by Sticky Note2 ("Compile and create a document in Google Drive") |
| Format Document           | Code Node (JavaScript)          | Converts package JSON to markdown document               | Package Compiler              | Create Google Doc             | Covered by Sticky Note2                                        |
| Create Google Doc         | Google Docs Node                | Creates Google Doc in Drive for lesson package           | Format Document               | Add Content to Doc            | Covered by Sticky Note2                                        |
| Add Content to Doc        | Google Docs Node                | Inserts markdown content into Google Doc                 | Create Google Doc             | Fetch File From Drive, Schedule Prep Reminder | Covered by Sticky Note2                                        |
| Fetch File From Drive     | Google Drive Node               | Retrieves shareable link of created Google Doc           | Add Content to Doc            | Send Confirmation Email       | Covered by Sticky Note3 ("Share: Calendar event and Email")  |
| Schedule Prep Reminder    | Google Calendar Node            | Schedules calendar event for lesson preparation          | Add Content to Doc            | -                             | Covered by Sticky Note3                                        |
| Send Confirmation Email   | Gmail Node                     | Sends confirmation email with lesson package link        | Fetch File From Drive          | -                             | Covered by Sticky Note3                                        |
| Sticky Note               | Sticky Note                    | Annotation for AI agent trio block                        | -                            | -                             | "1. AI Lesson Plan, Assignment and Integration Agent Trio..." |
| Sticky Note2              | Sticky Note                    | Annotation for compilation and document creation         | -                            | -                             | "2. Compile and create a document in Google Drive"            |
| Sticky Note3              | Sticky Note                    | Annotation for sharing and notification                   | -                            | -                             | "3. Share: Calendar event for review and Email the Google Drive Link" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("Teacher Input Form")**  
   - Set webhook path to `/webhook/modern-lesson-generator`.  
   - Configure form fields: Subject (dropdown with UK subjects), Year Group (dropdown Years 7-11), Topic/Chapter (text), Curriculum Standard (dropdown with UK curriculum options), Lesson Duration (number), Class Size (number), Differentiation Needs (text), Assessment Focus (dropdown).  
   - Add form title and description as per original.

2. **Add Three Langchain Agent Nodes:**
   
   - **Content Creation Agent**  
     - Connect input from "Teacher Input Form".  
     - Configure system prompt with detailed lesson package instructions including differentiation, assessment, and parent materials.  
     - Use OpenAI GPT-4o-mini model credentials.

   - **Assessment & Marking Agent**  
     - Connect input from "Content Creation Agent".  
     - Configure system prompt focused on assessment tools, marking efficiency, and automated feedback.  
     - Use OpenAI GPT-4o-mini credentials.

   - **AI Integration Agent**  
     - Connect input from "Assessment & Marking Agent".  
     - Configure system prompt to recommend AI tools and automation workflows based on lesson data.  
     - Use OpenAI GPT-4o-mini credentials.

3. **Set up three OpenAI Chat Model nodes** to serve as the backend for each agent, configured with the `gpt-4o-mini` model and linked appropriately.

4. **Create a Code Node ("Package Compiler"):**  
   - Input from the AI Integration Agent node.  
   - Write JavaScript to aggregate all AI outputs and teacher inputs into a structured JSON package including metadata, content, assessment tools, AI recommendations, and implementation guide.  
   - Include functions to extract time-saving mentions and recommended AI tools.

5. **Create a Code Node ("Format Document"):**  
   - Input from Package Compiler.  
   - Convert the JSON package to a markdown string with sections for lesson content, assessments, and AI integration.

6. **Set up Google Docs Node ("Create Google Doc"):**  
   - Configure to create a new Google Doc with OAuth2 credentials.  
   - Use dynamic title template: `"Modern Lesson: {Subject} - {Topic/Chapter} - {Year Group}"`.  
   - Specify target folder ID for lesson documents.

7. **Create Google Docs Node ("Add Content to Doc"):**  
   - Connect input from "Create Google Doc" and "Format Document".  
   - Configure to update the document by inserting the markdown content.

8. **Create Google Drive Node ("Fetch File From Drive"):**  
   - Query for the document by name (matching created Google Doc).  
   - Request fields: `webViewLink` and `exportLinks`.  
   - OAuth2 credentials required.

9. **Add Google Calendar Node ("Schedule Prep Reminder"):**  
   - Schedule an event for the next day from 8:00 to 8:30 AM in primary calendar.  
   - Event location: Staffroom.  
   - Description: "Review and prepare the AI-generated lesson package."  
   - OAuth2 credentials required.

10. **Add Gmail Node ("Send Confirmation Email"):**  
    - Send to fixed email `ptalur@gmail.com`.  
    - Subject includes the lesson topic.  
    - Message is HTML formatted with lesson subject and a hyperlink to the Google Doc.  
    - OAuth2 credentials configured.

11. **Wire the nodes in sequence:**  
    - Teacher Input Form → Content Creation Agent → Assessment & Marking Agent → AI Integration Agent → Package Compiler → Format Document → Create Google Doc → Add Content to Doc → Fetch File From Drive → Send Confirmation Email.  
    - Add Content to Doc also triggers Schedule Prep Reminder.

12. **Optional:** Add Sticky Notes for clarity describing each block.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow targets UK teachers aiming to replace textbook-based lesson planning with AI-generated content. | Workflow purpose and audience                                   |
| It integrates OpenAI GPT-4o-mini model for content generation with Google Workspace (Docs, Drive, Calendar). | External services integration                                   |
| Time-saving strategies are highlighted, with estimated 95% reduction in lesson prep time communicated by email. | User communication and motivation                              |
| Folder ID and email recipient are hardcoded and should be customized for deployment environment.               | Deployment customization                                         |
| Sticky Notes provide conceptual grouping to assist understanding and maintenance.                              | Workflow documentation aid                                      |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.