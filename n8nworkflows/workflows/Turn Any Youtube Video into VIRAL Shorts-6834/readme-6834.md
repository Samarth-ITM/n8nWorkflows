Turn Any Youtube Video into VIRAL Shorts

https://n8nworkflows.xyz/workflows/turn-any-youtube-video-into-viral-shorts-6834


# Turn Any Youtube Video into VIRAL Shorts

### 1. Workflow Overview

This workflow, titled **"Turn Any Youtube Video into VIRAL Shorts,"** automates the process of transforming a long YouTube video into engaging short clips optimized for viral sharing. The main use case is to extract meaningful segments from a video transcript, generate short-form content ideas or clips, and save these clips for further use or publishing.

The workflow’s logic is divided into the following main blocks:

- **1.1 Input Reception:** Manual trigger and initial setup of input parameters such as the YouTube video URL or ID.
- **1.2 Transcript Acquisition:** HTTP request to retrieve the full transcript of the provided YouTube video.
- **1.3 Processing & Clip Selection:** Using code and AI language models (Mistral Cloud Chat Model) to analyze the transcript, select the most engaging clips, and parse AI output into structured data.
- **1.4 Clip Management & Saving:** Splitting clips for batch processing, iterating over each clip to save them via HTTP requests, and applying waits to manage request pacing.
- **1.5 Workflow Control & Finalization:** Handling the loop exit and no-operation node for potential future replacements or extensions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block receives manual execution input and prepares the initial data fields required for the workflow.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Edit Fields

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point; triggers workflow execution manually  
  - Configuration: Default manual trigger settings  
  - Connections: Outputs to Edit Fields  
  - Edge Cases: None typical; user must manually execute  

- **Edit Fields**  
  - Type: Set node  
  - Role: Sets or modifies initial input fields such as video URL, video ID, or parameters for transcript retrieval  
  - Configuration: Custom key-value pairs to prepare input data for subsequent HTTP request  
  - Key Expressions: Likely sets video identifiers or API parameters  
  - Connections: Outputs to Get Transcript  
  - Edge Cases: Misconfigured fields may cause transcript retrieval failure

---

#### 2.2 Transcript Acquisition

**Overview:**  
Fetches the transcript of the target YouTube video via HTTP request.

**Nodes Involved:**  
- Get Transcript

**Node Details:**  

- **Get Transcript**  
  - Type: HTTP Request  
  - Role: Calls an API or service to retrieve the full transcript text of the provided YouTube video  
  - Configuration: HTTP method (GET/POST), URL constructed with video ID or URL, authentication if required  
  - Key Expressions: URL or query parameters dynamically referencing previous node’s output  
  - Connections: Outputs to Code node  
  - Edge Cases: HTTP failures (timeout, 404 if transcript not found), rate limiting, invalid video ID

---

#### 2.3 Processing & Clip Selection

**Overview:**  
Processes the transcript, uses code logic and AI language models to identify and select short clips suitable for viral content, and parses AI output into structured clips.

**Nodes Involved:**  
- Code  
- Mistral Cloud Chat Model  
- Structured Output Parser  
- Select Clips

**Node Details:**  

- **Code**  
  - Type: Code node (JavaScript)  
  - Role: Processes raw transcript data, possibly cleaning or structuring it before AI processing  
  - Configuration: Custom JavaScript code likely to transform input data for AI consumption  
  - Input: Transcript data from Get Transcript  
  - Output: Processed transcript data passed to Select Clips  

- **Mistral Cloud Chat Model**  
  - Type: AI Language Model (Chat-based)  
  - Role: Generates candidate clips or extracts highlights from the transcript using advanced AI language modeling  
  - Configuration: Uses Mistral Cloud credentials and settings; model parameters may include temperature, max tokens  
  - Input: Receives prompt and transcript from Select Clips chain node  
  - Output: AI-generated text passed to Structured Output Parser  

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses the AI-generated text into structured JSON or similar format specifying clips  
  - Configuration: Uses predefined parsing schema to ensure output consistency  
  - Input: AI response from Mistral Cloud Chat Model  
  - Output: Structured clip data back to Select Clips  

- **Select Clips**  
  - Type: LangChain Chain LLM node  
  - Role: Orchestrates the chain of AI interactions and parsing to finalize clip selection  
  - Configuration: References Mistral Cloud Chat Model and Structured Output Parser as subnodes  
  - Input: Receives processed transcript from Code node  
  - Output: Sends structured clips data to Split Out node  
  - Edge Cases: AI model errors, parsing errors, incomplete or ambiguous output  

---

#### 2.4 Clip Management & Saving

**Overview:**  
Splits the array of clips, processes them in batches, and saves each clip via HTTP requests while managing pacing via wait timers.

**Nodes Involved:**  
- Split Out  
- Loop Over Items  
- Save Clips  
- Wait  
- Replace Me (No Operation placeholder)

**Node Details:**  

- **Split Out**  
  - Type: Split Out node  
  - Role: Splits the structured clips array into individual clip items for batch processing  
  - Configuration: Default splitting by array elements  
  - Input: Structured clip data from Select Clips  
  - Output: Individual clip items to Loop Over Items  

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes clips one at a time or in defined batch sizes to control rate of API calls or saving operations  
  - Configuration: Batch size configured depending on API limits or desired pacing  
  - Input: Individual clip items from Split Out  
  - Output: Processes each clip through Save Clips node and then loops or ends  
  - Edge Cases: Batch size misconfiguration causing slow processing or API overload  

- **Save Clips**  
  - Type: HTTP Request  
  - Role: Saves or uploads each selected clip to storage or a publishing endpoint  
  - Configuration: HTTP method (POST/PUT), URL, authentication, payload including clip details  
  - Input: Receives each clip item from Loop Over Items  
  - Output: Triggers Wait node to delay before next clip processing  
  - Edge Cases: Authentication failures, network errors, payload formatting issues  

- **Wait**  
  - Type: Wait node  
  - Role: Pauses execution for a defined time to avoid API rate limits or server overload  
  - Configuration: Delay duration set based on API requirements  
  - Input: After Save Clips  
  - Output: Back to Loop Over Items to process next clip  

- **Replace Me**  
  - Type: No Operation (NoOp) node  
  - Role: Placeholder for potential future logic or replacement  
  - Configuration: No action; outputs empty data  
  - Input: From Loop Over Items (secondary output)  
  - Output: None  
  - Edge Cases: None; safe to ignore or replace

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                     | Input Node(s)               | Output Node(s)            | Sticky Note |
|----------------------------|----------------------------------|-----------------------------------|-----------------------------|---------------------------|-------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Entry point for manual execution   | —                           | Edit Fields               |             |
| Edit Fields                | Set                              | Prepares initial input parameters  | When clicking ‘Execute workflow’ | Get Transcript            |             |
| Get Transcript             | HTTP Request                     | Retrieves full video transcript    | Edit Fields                 | Code                      |             |
| Code                       | Code                             | Processes transcript data          | Get Transcript              | Select Clips              |             |
| Mistral Cloud Chat Model   | AI Language Model (Chat)         | Generates clip ideas from transcript | Select Clips (ai_languageModel) | Structured Output Parser  |             |
| Structured Output Parser   | LangChain Output Parser Structured | Parses AI output into structured clips | Mistral Cloud Chat Model (ai_outputParser) | Select Clips             |             |
| Select Clips               | LangChain Chain LLM              | Coordinates AI processing and parsing | Code                       | Split Out                 |             |
| Split Out                  | Split Out                        | Splits clips array into single items | Select Clips               | Loop Over Items           |             |
| Loop Over Items            | Split In Batches                 | Processes clips in batches         | Split Out                   | Replace Me, Save Clips    |             |
| Replace Me                 | No Operation (NoOp)              | Placeholder node                   | Loop Over Items             | —                         |             |
| Save Clips                 | HTTP Request                    | Saves or uploads clips             | Loop Over Items             | Wait                      |             |
| Wait                       | Wait                            | Delays to manage pacing            | Save Clips                  | Loop Over Items           |             |
| Sticky Note                | Sticky Note                     | —                                 | —                           | —                         |             |
| Sticky Note1               | Sticky Note                     | —                                 | —                           | —                         |             |
| Sticky Note3               | Sticky Note                     | —                                 | —                           | —                         |             |
| Sticky Note4               | Sticky Note                     | —                                 | —                           | —                         |             |
| Sticky Note5               | Sticky Note                     | —                                 | —                           | —                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking ‘Execute workflow’  
   - Type: Manual Trigger  
   - No special configuration; this triggers the workflow manually.

2. **Add Set Node to Prepare Input Fields**  
   - Name: Edit Fields  
   - Type: Set  
   - Configure fields for input such as YouTube video URL or ID, API keys if needed.  
   - Connect output from Manual Trigger to this node.

3. **Add HTTP Request Node to Retrieve Transcript**  
   - Name: Get Transcript  
   - Type: HTTP Request  
   - Set method (GET or POST) and URL to call the transcript service or YouTube transcript API.  
   - Use expressions to inject video ID or URL from Edit Fields node output.  
   - Connect Edit Fields output to this node.

4. **Add Code Node to Process Transcript**  
   - Name: Code  
   - Type: Code (JavaScript)  
   - Write script to clean, format, or extract relevant text data from the HTTP response.  
   - Connect Get Transcript output to this node.

5. **Add LangChain Chain LLM Node to Select Clips**  
   - Name: Select Clips  
   - Type: Chain LLM  
   - Configure to coordinate the AI processing steps:  
     - Add Mistral Cloud Chat Model as AI language model node inside.  
     - Add Structured Output Parser node inside for parsing AI output.  
   - Connect Code node output to Select Clips input.

6. **Add Mistral Cloud Chat Model Node**  
   - Name: Mistral Cloud Chat Model  
   - Type: LangChain LLM (Chat)  
   - Configure credentials for Mistral Cloud API.  
   - Set model parameters (temperature, max tokens, etc.) as needed.  
   - This node is linked as subnode inside Select Clips.

7. **Add Structured Output Parser Node**  
   - Name: Structured Output Parser  
   - Type: LangChain Output Parser Structured  
   - Define parsing schema for clip data (e.g., JSON structure with clip start/end, text).  
   - This node is linked as subnode inside Select Clips.

8. **Add Split Out Node to Split Clips**  
   - Name: Split Out  
   - Type: Split Out  
   - Configure to split the structured array output from Select Clips into individual clip items.  
   - Connect Select Clips output to this node.

9. **Add Split In Batches Node to Loop Over Clips**  
   - Name: Loop Over Items  
   - Type: Split In Batches  
   - Set batch size (e.g., 1 for sequential processing).  
   - Connect Split Out output to this node.

10. **Add HTTP Request Node to Save Clips**  
    - Name: Save Clips  
    - Type: HTTP Request  
    - Configure to send clip data to storage or publishing API (method, URL, headers, body).  
    - Connect Loop Over Items main output to this node.

11. **Add Wait Node to Pace Requests**  
    - Name: Wait  
    - Type: Wait  
    - Set a delay (e.g., a few seconds) to respect API rate limits.  
    - Connect Save Clips output to Wait node.

12. **Loop Back Wait Node Output to Loop Over Items**  
    - Connect Wait node output back to Loop Over Items input to continue batch processing.

13. **Add No Operation Node as Placeholder**  
    - Name: Replace Me  
    - Type: No Operation (NoOp)  
    - Connect second output of Loop Over Items to this node (optional, for future extensions).

14. **Check and Configure Credentials**  
    - Add and configure credentials for:  
      - Mistral Cloud API (for AI model access)  
      - Any HTTP APIs used for transcript retrieval and clip saving.

15. **Test Workflow**  
    - Execute manually using the trigger.  
    - Monitor logs for errors (e.g., HTTP failures, AI parsing issues).

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                       |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| The workflow leverages the Mistral Cloud Chat AI model, which requires appropriate API credentials. | Ensure you have access and API keys for Mistral Cloud.                                                                |
| Managing API rate limits is critical; the Wait node controls pacing to avoid being blocked.          | Adjust delay based on your API provider’s limits.                                                                     |
| Structured Output Parser uses a schema to enforce clip data consistency; customize it for your needs.| See LangChain documentation for output parser schemas: https://js.langchain.com/docs/modules/output_parsers/         |
| This workflow can be extended by replacing the No Operation node with additional processing steps.  | For example, post-processing clips, uploading to social media platforms, or analytics integration.                     |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created in n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All data handled is lawful and public.