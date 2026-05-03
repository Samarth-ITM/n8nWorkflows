Generate & Publish Professional COE Blogs with Gemini AI and Google Drive

https://n8nworkflows.xyz/workflows/generate---publish-professional-coe-blogs-with-gemini-ai-and-google-drive-5619


# Generate & Publish Professional COE Blogs with Gemini AI and Google Drive

### 1. Workflow Overview

This workflow, titled **"AI-Powered COE Blog Generator with Chat Interface"**, automates the generation, refinement, publication, and distribution of professional Center of Excellence (COE) blog posts using advanced AI models and Google Drive integrations. It is designed for users who want to create high-quality, executive-ready blog content through an interactive chat interface, leveraging Google’s Gemini AI models for content generation and review, and then store and share the final blog via Google Drive.

The workflow is logically segmented into the following blocks:

- **1.1 Input Reception and Initial Outline Generation**  
  Captures user blog topic requests via a chat interface and generates a structured blog outline.

- **1.2 Outline Review and Blog Writing**  
  Refines the outline to ensure quality and completeness, then produces a fully developed blog post using professional writing guidelines.

- **1.3 Text Cleanup and Google Drive Publishing**  
  Cleans formatting artifacts from the AI-generated text, saves the blog as a Google Document, shares it with stakeholders, makes it publicly accessible, and sends a direct link back to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Outline Generation

- **Overview:**  
  This block accepts user input via a chat interface, then generates an initial blog post outline using AI.

- **Nodes Involved:**  
  - Start Blog Request  
  - Create Blog Outline  
  - AI Brain for Outline

- **Node Details:**

  - **Start Blog Request**  
    - *Type:* LangChain Chat Trigger (Webhook)  
    - *Role:* Entry point receiving user blog topic requests via chat  
    - *Configuration:* Public webhook with initial greeting message ("Hi there! 👋 My name is Ajay. How can I assist you today?")  
    - *Inputs:* User chat messages  
    - *Outputs:* Chat input text to next node  
    - *Edge Cases:* Webhook downtime, malformed input, user abandonment  
    - *Version:* 1.1  

  - **Create Blog Outline**  
    - *Type:* LangChain Agent  
    - *Role:* Generates a structured outline for the blog based on user input  
    - *Configuration:* System prompt instructs AI to produce section titles and key points in a structured outline  
    - *Key Expression:* `{{$json.chatInput}}` passes user message as prompt text  
    - *Inputs:* Chat input from Start Blog Request  
    - *Outputs:* Raw outline text to Review & Fix Outline node  
    - *Edge Cases:* AI output may be incomplete or off-topic, requires fallback or re-prompting  
    - *Version:* 1.7  

  - **AI Brain for Outline**  
    - *Type:* LangChain Google Gemini LM Chat  
    - *Role:* Language model backend for Create Blog Outline node  
    - *Configuration:* Uses Gemini 2.0 Flash Thinking experimental model  
    - *Credentials:* Google PaLM API account  
    - *Inputs:* Text prompt from Create Blog Outline  
    - *Outputs:* AI-generated outline text  
    - *Edge Cases:* API rate limits, latency, authentication failures  
    - *Version:* 1  

#### 2.2 Outline Review and Blog Writing

- **Overview:**  
  This block refines the blog outline to ensure logical flow and completeness, then writes the full professional blog post with stringent quality criteria.

- **Nodes Involved:**  
  - Review & Fix Outline  
  - Write Full Blog Post  
  - AI Brain for Review  
  - AI Brain for Writing

- **Node Details:**

  - **Review & Fix Outline**  
    - *Type:* LangChain Agent  
    - *Role:* Evaluates and revises the initial outline ensuring introduction, section breakdown, flow, and conclusion  
    - *Configuration:* System message explicitly requests only the revised outline as output  
    - *Key Expression:* `{{$json.output}}` from Create Blog Outline  
    - *Inputs:* Initial outline text  
    - *Outputs:* Revised outline to Write Full Blog Post  
    - *Edge Cases:* AI might omit key sections, requires validation or fallback  
    - *Version:* 1.7  

  - **Write Full Blog Post**  
    - *Type:* LangChain Agent  
    - *Role:* Composes a detailed, evidence-based, executive-level blog post following strict stylistic and content guidelines  
    - *Configuration:* Extensive system message specifying writing style, tone, content structure, anti-AI patterns, and prohibiting personal or client references  
    - *Key Expression:* `{{$json.output}}` from Review & Fix Outline  
    - *Inputs:* Revised outline  
    - *Outputs:* Full blog text with rich content  
    - *Edge Cases:* AI hallucinations, verbosity, content repetition, or style drift require monitoring  
    - *Version:* 1.7  

  - **AI Brain for Review**  
    - *Type:* LangChain Google Gemini LM Chat  
    - *Role:* Language model backend for Review & Fix Outline node  
    - *Configuration:* Gemini 2.0 Flash Thinking experimental model  
    - *Credentials:* Google PaLM API account  
    - *Inputs:* Text prompt for outline review  
    - *Outputs:* Revised outline text  
    - *Edge Cases:* API limits, auth errors, latency  
    - *Version:* 1  

  - **AI Brain for Writing**  
    - *Type:* LangChain Google Gemini LM Chat  
    - *Role:* Language model backend for Write Full Blog Post node  
    - *Configuration:* Gemini 2.0 Pro experimental model for higher-quality output  
    - *Credentials:* Google PaLM API account  
    - *Inputs:* Writing prompt with revised outline  
    - *Outputs:* Complete blog post text  
    - *Edge Cases:* Same as above  
    - *Version:* 1  

#### 2.3 Text Cleanup and Google Drive Publishing

- **Overview:**  
  Cleans formatting artifacts from AI output, saves the blog to Google Drive as a Google Doc, shares it via email, sets public permissions, and returns the shareable link to the user.

- **Nodes Involved:**  
  - Clean Up Text Format  
  - Save Blog to Google Drive  
  - Email Blog to Stakeholder  
  - Make Blog Public  
  - Send Blog Link to User

- **Node Details:**

  - **Clean Up Text Format**  
    - *Type:* Code (JavaScript)  
    - *Role:* Removes markdown bold formatting (`**text**`) from AI-generated blog text  
    - *Configuration:* Regex replaces `**...**` with plain text; returns cleaned output  
    - *Inputs:* Full blog post text from Write Full Blog Post  
    - *Outputs:* Cleaned blog content for saving  
    - *Edge Cases:* Unexpected markdown formats, null or empty input  
    - *Version:* 2  

  - **Save Blog to Google Drive**  
    - *Type:* Google Drive node  
    - *Role:* Creates a new Google Document containing the blog content  
    - *Configuration:*  
      - Document name set to user’s original chat input (blog title)  
      - Content is cleaned blog text  
      - Converts text to Google Doc format with indexable text enabled  
      - Saves to root folder by default  
    - *Credentials:* Google Drive OAuth2 account  
    - *Inputs:* Cleaned blog text from previous node  
    - *Outputs:* Google Drive file metadata (including file ID)  
    - *Edge Cases:* API quota limits, permission errors, invalid content types  
    - *Version:* 3  

  - **Email Blog to Stakeholder**  
    - *Type:* Google Drive node (Share operation)  
    - *Role:* Shares the saved blog document with a specific stakeholder via email  
    - *Configuration:*  
      - Uses file ID from Save Blog to Google Drive  
      - Sends notification email with message "This is my new blog please check it"  
      - Grants “writer” permission to email ajay2343@gmail.com  
    - *Credentials:* Google Drive OAuth2 account  
    - *Inputs:* File ID from saved blog  
    - *Outputs:* Share confirmation  
    - *Edge Cases:* Invalid email address, permission denied, API errors  
    - *Version:* 3  

  - **Make Blog Public**  
    - *Type:* Google Drive node (Share operation)  
    - *Role:* Makes the blog publicly readable by “anyone with the link”  
    - *Configuration:*  
      - Uses file ID from Save Blog to Google Drive  
      - Sets permission role to “reader” and type to “anyone”  
    - *Credentials:* Google Drive OAuth2 account  
    - *Inputs:* File ID from saved blog  
    - *Outputs:* Updated permission confirmation  
    - *Edge Cases:* Permission conflicts, API quota limits  
    - *Version:* 3  

  - **Send Blog Link to User**  
    - *Type:* Set node  
    - *Role:* Constructs a shareable Google Drive URL and returns it along with the blog title back to the user  
    - *Configuration:*  
      - URL format: `https://drive.google.com/file/d/{fileId}/view` where fileId is from Save Blog to Google Drive  
      - Includes original blog title from user input  
    - *Inputs:* File ID and original chat input  
    - *Outputs:* JSON containing URL and title for downstream consumption or UI display  
    - *Edge Cases:* Missing file ID, malformed URL  
    - *Version:* 3.4  

---

### 3. Summary Table

| Node Name               | Node Type                      | Functional Role                             | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                  |
|-------------------------|--------------------------------|---------------------------------------------|----------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Start Blog Request       | LangChain Chat Trigger          | Receives user blog topic requests            | —                          | Create Blog Outline           | ## Start Blog Request, Create Blog Outline, AI Brain for Outline                             |
| Create Blog Outline      | LangChain Agent                 | Generates structured outline                  | Start Blog Request          | Review & Fix Outline          | ## Start Blog Request, Create Blog Outline, AI Brain for Outline                             |
| AI Brain for Outline     | LangChain Google Gemini LM Chat| AI backend for outline generation             | Create Blog Outline (ai_lm) | Create Blog Outline (ai_lm)  | ## Start Blog Request, Create Blog Outline, AI Brain for Outline                             |
| Review & Fix Outline     | LangChain Agent                 | Refines outline to ensure completeness        | Create Blog Outline         | Write Full Blog Post          | ## Review & Fix Outline, Write Full Blog Post, AI Brain for Review, AI Brain for Writing     |
| AI Brain for Review      | LangChain Google Gemini LM Chat| AI backend for outline review                  | Review & Fix Outline (ai_lm)| Review & Fix Outline (ai_lm) | ## Review & Fix Outline, Write Full Blog Post, AI Brain for Review, AI Brain for Writing     |
| Write Full Blog Post     | LangChain Agent                 | Writes full professional blog post             | Review & Fix Outline        | Clean Up Text Format          | ## Review & Fix Outline, Write Full Blog Post, AI Brain for Review, AI Brain for Writing     |
| AI Brain for Writing     | LangChain Google Gemini LM Chat| AI backend for blog writing                     | Write Full Blog Post (ai_lm)| Write Full Blog Post (ai_lm) | ## Review & Fix Outline, Write Full Blog Post, AI Brain for Review, AI Brain for Writing     |
| Clean Up Text Format     | Code Node (JavaScript)          | Removes markdown formatting                     | Write Full Blog Post        | Save Blog to Google Drive     | ## Clean Up Text Format, Save Blog to Google Drive, Email Blog to Stakeholder, Make Blog Public, Send Blog Link to User |
| Save Blog to Google Drive| Google Drive                   | Saves cleaned blog as Google Document          | Clean Up Text Format        | Email Blog to Stakeholder     | ## Clean Up Text Format, Save Blog to Google Drive, Email Blog to Stakeholder, Make Blog Public, Send Blog Link to User |
| Email Blog to Stakeholder| Google Drive                   | Shares blog document via email to stakeholder | Save Blog to Google Drive   | Make Blog Public              | ## Clean Up Text Format, Save Blog to Google Drive, Email Blog to Stakeholder, Make Blog Public, Send Blog Link to User |
| Make Blog Public         | Google Drive                   | Sets blog document visibility to public        | Email Blog to Stakeholder   | Send Blog Link to User        | ## Clean Up Text Format, Save Blog to Google Drive, Email Blog to Stakeholder, Make Blog Public, Send Blog Link to User |
| Send Blog Link to User   | Set Node                      | Constructs and returns shareable blog URL      | Make Blog Public            | —                            | ## Clean Up Text Format, Save Blog to Google Drive, Email Blog to Stakeholder, Make Blog Public, Send Blog Link to User |
| Sticky Note             | Sticky Note                   | Annotation block                                | —                          | —                            | ## Start Blog Request, Create Blog Outline, AI Brain for Outline                             |
| Sticky Note1            | Sticky Note                   | Annotation block                                | —                          | —                            | ## Review & Fix Outline, Write Full Blog Post, AI Brain for Review, AI Brain for Writing     |
| Sticky Note2            | Sticky Note                   | Annotation block                                | —                          | —                            | ## Clean Up Text Format, Save Blog to Google Drive, Email Blog to Stakeholder, Make Blog Public, Send Blog Link to User |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node**  
   - Add **LangChain Chat Trigger** node named "Start Blog Request".  
   - Set it as a public webhook with an initial greeting message:  
     `"Hi there! 👋\nMy name is Ajay. How can I assist you today?"`

2. **Create the Outline Generation Agent Node**  
   - Add **LangChain Agent** node named "Create Blog Outline".  
   - Set prompt to use user chat input: `={{ $json.chatInput }}`.  
   - Use system message instructing expert outline writing:  
     `"You are an expert outline writer. Your job is to generate a structured outline for a blog post with section titles and key points."`  
   - Connect output of “Start Blog Request” to this node's input.

3. **Add Gemini AI Brain for Outline**  
   - Add **LangChain LM Chat Google Gemini** node named "AI Brain for Outline".  
   - Select model `models/gemini-2.0-flash-thinking-exp-01-21`.  
   - Link its output to “Create Blog Outline” AI language model input.  
   - Configure Google PaLM API credentials.

4. **Create Outline Review Agent Node**  
   - Add **LangChain Agent** node named "Review & Fix Outline".  
   - Input text: `={{ $json.output }}` from "Create Blog Outline".  
   - System message:  
     ```
     You are an expert blog evaluator.
     Revise this outline and ensure it covers the following criteria:
     Introduction
     Clear section breakdown
     Logical flow
     Conclusion

     ## Output
     only output the revised outline
     ```  
   - Connect "Create Blog Outline" output to this node.

5. **Add Gemini AI Brain for Review**  
   - Add **LangChain LM Chat Google Gemini** node named "AI Brain for Review".  
   - Use same model as Outline AI Brain: `models/gemini-2.0-flash-thinking-exp-01-21`.  
   - Link output to "Review & Fix Outline" AI language model input.

6. **Create Blog Writing Agent Node**  
   - Add **LangChain Agent** node named "Write Full Blog Post".  
   - Input text: `={{ $json.output }}` from "Review & Fix Outline".  
   - System message: Provide detailed instructions specifying:  
     - Evidence-based leadership citing examples and technologies  
     - Boardroom-ready, structured writing style  
     - Authentic expertise signaling  
     - Anti-AI pattern strategies  
     - No personal role references or client mentions  
     - Clear, professional, accessible language  
     - No meta-commentary at start  
   - Connect "Review & Fix Outline" output to this node.

7. **Add Gemini AI Brain for Writing**  
   - Add **LangChain LM Chat Google Gemini** node named "AI Brain for Writing".  
   - Use model `models/gemini-2.0-pro-exp-02-05` for higher-quality prose.  
   - Link output to "Write Full Blog Post" AI language model input.

8. **Add Code Node to Clean Text**  
   - Add **Code** node named "Clean Up Text Format".  
   - JavaScript code to remove markdown bold formatting:  
     ```js
     const formatBoldText = (value) => {
       let value2 = value.output.replace(/\*\*(.*?)\*\*/g, '$1');  
       let arr1 = [{"output": value2}];
       return arr1;
     };

     let value = $input.first();
     return formatBoldText(value.json);
     ```  
   - Connect output of "Write Full Blog Post" to this node.

9. **Add Google Drive Node to Save Blog**  
   - Add **Google Drive** node named "Save Blog to Google Drive".  
   - Operation: Create from Text  
   - Name: `={{ $('Start Blog Request').item.json.chatInput }}` (user’s blog title)  
   - Content: `={{ $json.output }}` (cleaned blog text)  
   - Drive: “My Drive” (default)  
   - Folder: root or specify if needed  
   - Enable convert to Google Document and indexable text options  
   - Connect output of "Clean Up Text Format" to this node.  
   - Configure Google Drive OAuth2 credentials.

10. **Add Google Drive Node to Email Blog**  
    - Add **Google Drive** node named "Email Blog to Stakeholder".  
    - Operation: Share  
    - File ID: `={{ $json.id }}` from saved file  
    - Permissions: Add user with writer role, email `ajay2343@gmail.com`  
    - Send notification email with message: "This is my new blog please check it"  
    - Connect output of "Save Blog to Google Drive" to this node.  
    - Use same Google Drive OAuth2 credentials.

11. **Add Google Drive Node to Make Blog Public**  
    - Add **Google Drive** node named "Make Blog Public".  
    - Operation: Share  
    - File ID: `={{ $('Save Blog to Google Drive').item.json.id }}`  
    - Permissions: role “reader”, type “anyone” (public link)  
    - Connect output of "Email Blog to Stakeholder" to this node.

12. **Add Set Node to Send Blog Link to User**  
    - Add **Set** node named "Send Blog Link to User".  
    - Create two fields:  
      - URL: `=https://drive.google.com/file/d/{{ $('Save Blog to Google Drive').item.json.id }}/view`  
      - COE Title: `={{ $('Start Blog Request').item.json.chatInput }}`  
    - Connect output of "Make Blog Public" to this node.

13. **Add Sticky Notes (Optional)**  
    - Add sticky notes grouping logical blocks for documentation clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                               |
|----------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow uses Google Gemini AI models (Gemini 2.0 Flash Thinking and Gemini 2.0 Pro) via Google PaLM API credentials. | Google Gemini API documentation and access needed.           |
| The blog writing style is highly specialized for COE executive communications with anti-AI pattern strategies.      | Internal style guide embedded in prompt system messages.     |
| Google Drive integration requires OAuth2 credentials with permissions to create, share, and modify Google Docs.      | Google Drive API and OAuth2 setup instructions apply.        |
| This workflow supports public sharing and stakeholder email notifications for collaborative blog review.             | Google Drive sharing permissions management recommended.     |
| Initial user interaction is via chat webhook with LangChain Chat Trigger node.                                       | n8n LangChain integration documentation for chat triggers.   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.