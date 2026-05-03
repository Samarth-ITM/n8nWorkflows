Generate YouTube Scripts for Shorts & Long-Form with Gemini AI and Tavily Research

https://n8nworkflows.xyz/workflows/generate-youtube-scripts-for-shorts---long-form-with-gemini-ai-and-tavily-research-9342


# Generate YouTube Scripts for Shorts & Long-Form with Gemini AI and Tavily Research

---

### 1. Workflow Overview

This workflow automates the generation of YouTube video scripts for both Shorts and long-form videos based on a single user-provided topic. It is designed for content creators who want to efficiently produce engaging video scripts with the help of AI-powered research and writing tools.

The workflow logically splits into two main branches after receiving the user input:  
- **1.1 YouTube Shorts Script Generation:** Fast, concise script creation using a quick web search summary and an AI agent tailored for short video scripts.  
- **1.2 YouTube Long-Form Video Script Generation:** Detailed script creation using an advanced web search with deep research, outline creation, and a comprehensive AI-written script for longer videos.

Additionally, the workflow manages the creation and update of Google Docs files to save the generated scripts automatically.

Logical Blocks:  
- **1.1 Input Reception & Branching** — Collects topic and video type; determines which script generation path to follow  
- **1.2 YouTube Shorts Script Generation** — Quick research, AI script writing, and saving to Google Docs  
- **1.3 YouTube Long-Form Video Script Generation** — In-depth research, summary, outline creation, detailed AI script generation, and saving to Google Docs

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Branching

**Overview:**  
This block triggers the workflow when a user submits a form with the video topic and desired video type, then routes the flow to either Shorts or Long video script generation.

**Nodes Involved:**  
- On form submission  
- If  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point, collects user input for "Provide topic" and "Choose video type" (Short video or Long video).  
  - Configuration: A form with two required fields: a text input for the topic and a dropdown for video type.  
  - Input/Output: Outputs user-submitted JSON with the topic and video type.  
  - Failure cases: Missing required fields or webhook connectivity issues.

- **If**  
  - Type: Conditional node  
  - Role: Routes the workflow based on the "Choose video type" field.  
  - Configuration: Checks if video type equals "Short video" to decide which path to follow.  
  - Input: Output from "On form submission"  
  - Output: Two branches, one for Shorts, one for Long videos.  
  - Edge cases: Unexpected values in video type field; no fallback branch.

---

#### 1.2 YouTube Shorts Script Generation

**Overview:**  
Handles the creation of a concise, engaging YouTube Shorts script using a quick web search summary and an AI agent specialized for short scripts, then saves the script to a Google Doc.

**Nodes Involved:**  
- Tavily (basic search)  
- Websearch (Google Gemini research assistant)  
- Reference Script (set node with style reference)  
- AI Agent (Langchain agent for short script generation)  
- Create Doc (Google Docs creation)  
- Update Doc (Google Docs update)  

**Node Details:**

- **Tavily**  
  - Type: HTTP Request  
  - Role: Performs a basic search query on the user-provided topic using Tavily API.  
  - Configuration: POST to Tavily API with "basic" search depth, max 3 results, includes answers and raw content.  
  - Inputs: Topic from form submission  
  - Outputs: Search results JSON  
  - Edge cases: API key missing/invalid, rate limits, network errors.

- **Websearch**  
  - Type: Google Gemini AI (Langchain node)  
  - Role: Processes Tavily search results to generate a cohesive research summary tailored for the Shorts script.  
  - Configuration: Custom prompt instructing extraction of relevant info and producing a single summary for the given topic.  
  - Input: Tavily search results JSON  
  - Output: Text summary content  
  - Edge cases: AI model errors, malformed input, rate limits.

- **Reference Script**  
  - Type: Set node  
  - Role: Provides a style reference script for the AI agent as a tone and formatting guide.  
  - Configuration: Contains example short scripts (“Experimental”, “Listicle”, “How to”) to influence AI output style.  
  - Input: None (static data)  
  - Output: JSON with style reference text

- **AI Agent**  
  - Type: Langchain Agent node  
  - Role: Writes the full YouTube Shorts script based on the video title and summary, following a structured format (hook, intro, body, CTA).  
  - Configuration: System prompt defines the agent as a senior YouTube short script writer; uses the video title and Websearch summary as context.  
  - Inputs: Video title, style reference, Websearch summary  
  - Outputs: Generated script text  
  - Edge cases: AI generation failure, prompt errors.

- **Create Doc**  
  - Type: Google Docs node  
  - Role: Creates a new Google Doc titled with the user topic plus "_Shorts".  
  - Configuration: Default folder, OAuth2 credentials for Google Docs.  
  - Inputs: None from previous node but uses expression for title from form submission.  
  - Outputs: Document metadata including document URL or ID.  
  - Edge cases: Google API auth errors, quota exceeded.

- **Update Doc**  
  - Type: Google Docs node  
  - Role: Inserts the AI-generated script text into the newly created Google Doc.  
  - Configuration: Uses the document ID from "Create Doc" to update content.  
  - Inputs: Document ID, script text from AI Agent  
  - Outputs: Updated document metadata  
  - Edge cases: Document not found, update permission errors.

---

#### 1.3 YouTube Long-Form Video Script Generation

**Overview:**  
Generates a detailed YouTube video script for long-form content by performing an advanced web search, summarizing the research, creating a structured outline, then writing a comprehensive script. The final output is saved into Google Docs.

**Nodes Involved:**  
- Tavily1 (advanced search)  
- WebSearch Summary (Google Gemini AI for detailed summary)  
- Create Outline (Langchain chain LLM node)  
- Reference Script1 (set node with style reference for long videos)  
- AI Agent1 (Langchain agent for long script generation)  
- Create Doc1 (Google Docs creation)  
- Update Doc1 (Google Docs update)  

**Node Details:**

- **Tavily1**  
  - Type: HTTP Request  
  - Role: Performs an advanced search query using Tavily API with deeper search depth and up to 10 results.  
  - Configuration: POST with "advanced" search depth, max_results 10, includes answers and raw content.  
  - Input: Topic from form submission  
  - Output: Detailed search results JSON  
  - Edge cases: Similar to Tavily node, including API errors and rate limits.

- **WebSearch Summary**  
  - Type: Google Gemini AI (Langchain node)  
  - Role: Reads detailed Tavily1 results and generates a ~1000-word research summary with structured sections (intro, main summary, conclusion).  
  - Configuration: Custom prompt instructing detailed research summary suitable for long-form video scripting.  
  - Input: Tavily1 search results JSON  
  - Output: Text summary of research  
  - Edge cases: Large input size errors, generation timeouts.

- **Create Outline**  
  - Type: Langchain chain LLM node  
  - Role: Creates a detailed YouTube video outline with headings and sub-headings based on the research summary and topic.  
  - Configuration: Prompt instructs creation of focused, essential outline items without fluff or numbered sections.  
  - Input: Text summary from WebSearch Summary  
  - Output: Outline text with headings/subheadings  
  - Edge cases: Outline too vague or incomplete if input summary is insufficient.

- **Reference Script1**  
  - Type: Set node  
  - Role: Provides style references and example scripts for the long-form video AI agent to emulate.  
  - Configuration: Contains three script style templates: Experimental, Listicle, and How-to, with detailed instructions and examples.  
  - Input: None (static content)  
  - Output: JSON with style reference text

- **AI Agent1**  
  - Type: Langchain Agent node  
  - Role: Writes the full detailed YouTube script (2500–3000 words) following the outline, matching requested video type, including intro, body, and outro.  
  - Configuration: System prompt sets tone as senior YouTube script writer; uses video title, outline, style references, and video type input.  
  - Input: Video title, outline, style reference, video type from form  
  - Output: Long-form script text  
  - Edge cases: Long generation time, AI model token limits, prompt misinterpretation.

- **Create Doc1**  
  - Type: Google Docs node  
  - Role: Creates a new Google Doc titled with the user topic for storing the long-form script.  
  - Configuration: Default folder, OAuth2 credentials.  
  - Input: Topic from form submission  
  - Output: Document metadata  
  - Edge cases: Same as Create Doc node.

- **Update Doc1**  
  - Type: Google Docs node  
  - Role: Inserts the long-form AI-generated script into the Google Doc created by Create Doc1.  
  - Configuration: Uses document ID from Create Doc1 and AI Agent1 output text.  
  - Output: Updated document metadata  
  - Edge cases: Same as Update Doc node.

---

### 3. Summary Table

| Node Name          | Node Type                                  | Functional Role                                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                    |
|--------------------|--------------------------------------------|-----------------------------------------------------|-------------------------|-------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger                              | Receive user input: video topic and video type       | None                    | If                      |                                                                                                               |
| If                 | Conditional                               | Route based on video type (Short or Long)            | On form submission      | Tavily, Tavily1          |                                                                                                               |
| Tavily             | HTTP Request                             | Basic web search for Shorts path                      | If (Short video branch) | Websearch                | ## Add Tavily API KEYS                                                                                        |
| Websearch          | Google Gemini AI (Langchain)              | Generate research summary for Shorts script           | Tavily                  | Reference Script         |                                                                                                               |
| Reference Script    | Set Node                                 | Provide style reference for Shorts script             | Websearch               | AI Agent                 |                                                                                                               |
| AI Agent           | Langchain Agent                          | Generate YouTube Shorts script                        | Reference Script, Websearch, On form submission | Create Doc               | You are a senior YouTube script writer for the Website Learners channel. Write a complete script...            |
| Create Doc         | Google Docs                              | Create Google Doc for Shorts script                   | AI Agent                | Update Doc               |                                                                                                               |
| Update Doc         | Google Docs                              | Insert Shorts script text into Google Doc             | Create Doc, AI Agent    | None                    |                                                                                                               |
| Tavily1             | HTTP Request                             | Advanced web search for Long video path                | If (Long video branch)  | WebSearch Summary        | ## Add Tavily API KEYS                                                                                        |
| WebSearch Summary  | Google Gemini AI (Langchain)              | Generate detailed research summary for Long video     | Tavily1                 | Create Outline           |                                                                                                               |
| Create Outline     | Langchain Chain LLM                      | Create detailed video outline from summary            | WebSearch Summary       | Reference Script1        |                                                                                                               |
| Reference Script1  | Set Node                                 | Provide style references for Long video AI script     | Create Outline          | AI Agent1                |                                                                                                               |
| AI Agent1          | Langchain Agent                          | Generate full long-form YouTube script                 | Reference Script1, Create Outline, On form submission | Create Doc1              | You are a senior YouTube script writer for the Website Learners channel. Write a complete script...            |
| Create Doc1        | Google Docs                              | Create Google Doc for Long-form script                 | AI Agent1               | Update Doc1              |                                                                                                               |
| Update Doc1        | Google Docs                              | Insert long-form script text into Google Doc           | Create Doc1, AI Agent1  | None                    |                                                                                                               |
| Sticky Note        | Sticky Note                             | Workflow overview, usage instructions, and setup notes| None                    | None                    | 🤖 YouTube Content Generator (Shorts & Long Videos) - explains workflow purpose and setup instructions         |
| Sticky Note1       | Sticky Note                             | Label for YouTube Shorts section                        | None                    | None                    | # For Youtube Short Videos                                                                                    |
| Sticky Note3       | Sticky Note                             | Label for YouTube Long Videos section                   | None                    | None                    | # For Youtube Long Videos                                                                                      |
| Sticky Note2       | Sticky Note                             | Reminder to add Tavily API keys                         | None                    | None                    | ## Add Tavily API KEYS                                                                                        |
| Sticky Note4       | Sticky Note                             | Reminder to add Tavily API keys                         | None                    | None                    | ## Add Tavily API KEYS                                                                                        |
| Google Gemini Chat Model | Langchain Google Gemini LM         | AI language model credential node for AI Agent (Short) | AI Agent                | AI Agent                 |                                                                                                               |
| Google Gemini Chat Model1| Langchain Google Gemini LM         | AI language model credential node for AI Agent1 (Long) | AI Agent1, Create Outline| AI Agent1, Create Outline |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission"):**  
   - Set up a webhook form trigger with two required fields:  
     - "Provide topic" (text input)  
     - "Choose video type" (dropdown with options: "Short video", "Long video")

2. **Add an If Node ("If"):**  
   - Connect from "On form submission"  
   - Condition: If `{{ $json['Choose video type'] }}` equals "Short video"  
   - Output branches:  
     - True → Shorts branch  
     - False → Long-form branch

3. **Shorts Branch:**

   a. **HTTP Request Node ("Tavily"):**  
      - POST to https://api.tavily.com/search  
      - JSON body:  
        ```json
        {
          "query": "{{ $('On form submission').item.json['Provide topic'] }}",
          "search_depth": "basic",
          "max_results": 3,
          "include_answer": true,
          "include_raw_content": true
        }
        ```  
      - Authentication: HTTP Header with "Authorization: Bearer YOUR-TAVILY-API-KEY"  
      - Connect from If node (true branch)

   b. **Google Gemini Node ("Websearch"):**  
      - Model: "models/gemini-2.5-flash"  
      - Prompt: Custom research assistant prompt to summarize Tavily results for the topic  
      - Input: Use Tavily node results in prompt expressions  
      - Connect from "Tavily"

   c. **Set Node ("Reference Script"):**  
      - Set a string variable "short_style_ref" with example scripts and style reference for Shorts  
      - Connect from "Websearch"

   d. **Langchain Agent Node ("AI Agent"):**  
      - Role: Senior YouTube short script writer  
      - Prompt uses video title, summary from "Websearch", and style reference from "Reference Script"  
      - Connect from "Reference Script"

   e. **Google Docs Node ("Create Doc"):**  
      - Create a new Google Doc titled "{{ Provide topic }}_Shorts"  
      - Connect from "AI Agent"  
      - Use Google Docs OAuth2 credentials

   f. **Google Docs Node ("Update Doc"):**  
      - Update the created doc by inserting the script text from "AI Agent" output  
      - Document URL from "Create Doc" output  
      - Connect from "Create Doc"

4. **Long-form Branch:**

   a. **HTTP Request Node ("Tavily1"):**  
      - POST to https://api.tavily.com/search  
      - JSON body:  
        ```json
        {
          "query": "{{ $('On form submission').item.json['Provide topic'] }}",
          "search_depth": "advanced",
          "max_results": 10,
          "include_answer": true,
          "include_raw_content": true
        }
        ```  
      - Authentication: HTTP Header with "Authorization: Bearer YOUR-TAVILY-API-KEY"  
      - Connect from If node (false branch)

   b. **Google Gemini Node ("WebSearch Summary"):**  
      - Model: "models/gemini-2.5-flash"  
      - Prompt: Detailed research summary instructions with multiple sections for long video use  
      - Input: Use Tavily1 results  
      - Connect from "Tavily1"

   c. **Langchain Chain LLM Node ("Create Outline"):**  
      - Prompt: Generate a structured video outline with headings and sub-headings based on summary  
      - Input: Output from "WebSearch Summary"  
      - Connect from "WebSearch Summary"

   d. **Set Node ("Reference Script1"):**  
      - Set style reference strings for long-form scripts including "Experimental", "Listicle", and "How to" script templates  
      - Connect from "Create Outline"

   e. **Langchain Agent Node ("AI Agent1"):**  
      - Role: Senior YouTube script writer for long-form content  
      - Prompt uses video title, outline, style reference, and video type  
      - Connect from "Reference Script1"

   f. **Google Docs Node ("Create Doc1"):**  
      - Create a new Google Doc titled "{{ Provide topic }}"  
      - Connect from "AI Agent1"  
      - Use Google Docs OAuth2 credentials

   g. **Google Docs Node ("Update Doc1"):**  
      - Update the created doc by inserting the script text from "AI Agent1" output  
      - Document URL from "Create Doc1" output  
      - Connect from "Create Doc1"

5. **Credentials Setup:**  
   - Configure Google Docs OAuth2 credential for document creation and updates.  
   - Configure HTTP Header Auth credentials for Tavily API with your API key.  
   - Configure Google Gemini (PaLM) API credentials for AI nodes.

6. **General Connections:**  
   - Connect "On form submission" → "If"  
   - "If" true branch → Shorts nodes chain: Tavily → Websearch → Reference Script → AI Agent → Create Doc → Update Doc  
   - "If" false branch → Long nodes chain: Tavily1 → WebSearch Summary → Create Outline → Reference Script1 → AI Agent1 → Create Doc1 → Update Doc1

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| This workflow is a content multiplier that automatically creates scripts for two different video formats from a single topic submission.  | Sticky Note at workflow start explaining purpose and usage                                  |
| Configure Tavily API keys for both Tavily nodes (basic and advanced search) to enable web search capability.                               | Sticky Notes near Tavily and Tavily1 nodes                                                 |
| The workflow uses Google Gemini (PaLM) API for AI research and script generation; ensure your Google Cloud project has API access enabled. | Nodes "Google Gemini Chat Model" and "Google Gemini Chat Model1" provide credential context |
| Google Docs OAuth2 credentials must be set up to allow creation and update of documents in your Google Drive.                               | Google Docs nodes require OAuth2 credentials                                               |
| The AI agents are configured with detailed prompt templates to ensure scripts are engaging, well-structured, and audience-appropriate.      | Node notes on AI Agent and AI Agent1 nodes                                                 |

---

# Disclaimer

The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.

---