Use any LangChain module in n8n (with the LangChain code node)

https://n8nworkflows.xyz/workflows/use-any-langchain-module-in-n8n--with-the-langchain-code-node--2082


# Use any LangChain module in n8n (with the LangChain code node)

### 1. Workflow Overview

This workflow demonstrates how to integrate LangChain modules directly within n8n using the LangChain Code node. Its main purpose is to fetch a YouTube video transcript using the SearchApiLoader module, then generate a detailed summary and example Q&A pairs based on the transcript. The workflow is designed for users who want to leverage advanced AI summarization capabilities with custom LangChain code inside n8n.

**Target Use Case:**  
Summarizing YouTube video content (specifically podcasts) by video ID and creating example questions and answers that can later be used for interactive bots or further AI tasks.

**Logical Blocks:**

- **1.1 Input Reception**: Manual trigger and setting the YouTube video ID.
- **1.2 AI Language Model Setup**: Initializing the OpenAI GPT-4o-mini model for summarization.
- **1.3 LangChain Processing**: Using a LangChain Code node to load video transcripts, split them, and run a summarization/refinement chain with custom prompts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block allows the user to manually start the workflow and set the YouTube video ID for which the summary is to be generated.

**Nodes Involved:**  
- When clicking "Execute Workflow" (Manual Trigger)  
- Set YouTube video ID

**Node Details:**

- **When clicking "Execute Workflow"**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually.  
  - Configuration: No parameters; triggers workflow execution on demand.  
  - Inputs: None  
  - Outputs: Connects to "Set YouTube video ID" node.  
  - Edge Cases: None; user must manually trigger.

- **Set YouTube video ID**  
  - Type: Set  
  - Role: Assigns a static YouTube video ID to the workflow data.  
  - Configuration: Sets a string named `videoId` with a default value "OsMVtuuwOXc" (example video ID).  
  - Inputs: Receives trigger from manual node.  
  - Outputs: Passes data (videoId) to the LangChain Code node.  
  - Edge Cases: If video ID is incorrect or video unavailable, later steps may fail or return empty transcripts.

---

#### 1.2 AI Language Model Setup

**Overview:**  
Initializes the OpenAI GPT-4o-mini model, which is used by the LangChain summarization chain to generate summaries and example questions.

**Nodes Involved:**  
- OpenAI Chat Model

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides a large language model interface to n8n for use in LangChain chains.  
  - Configuration: Model set to "gpt-4o-mini"; no additional options configured.  
  - Credentials: Uses OpenAI API credentials (OAuth2 token or API key).  
  - Inputs: None (invoked as an AI language model input connection by LangChain Code node).  
  - Outputs: Passes the model instance to LangChain Code node for summarization.  
  - Edge Cases:  
    - Auth errors if credentials are invalid or expired.  
    - Rate limits or timeouts from OpenAI API.

---

#### 1.3 LangChain Processing

**Overview:**  
Core logic that uses LangChain modules to load the YouTube video transcript, split it into manageable chunks, and run a refinement-based summarization chain. It produces a summary and example Q&A pairs.

**Nodes Involved:**  
- LangChain Code

**Node Details:**

- **LangChain Code**  
  - Type: LangChain Code node (custom JavaScript with LangChain imports)  
  - Role: Implements custom LangChain workflow logic including document loading, splitting, prompt construction, and chain execution.  
  - Configuration & Code Highlights:  
    - Requires user to replace `<YOUR API KEY>` with a valid `searchapi.io` API key (used to access YouTube transcripts).  
    - Imports:  
      - `loadSummarizationChain` from LangChain chains.  
      - `SearchApiLoader` from community document loaders for YouTube transcripts.  
      - `PromptTemplate` and `TokenTextSplitter` from LangChain core modules.  
    - Loads video transcript using `SearchApiLoader` for the given `videoId` from input data.  
    - Splits transcripts into chunks of 10,000 tokens with 250 tokens overlap for processing.  
    - Uses two prompt templates: one for initial summary and questions, one for refining summary with new transcript chunks.  
    - Loads the OpenAI model instance from input connection named `ai_languageModel`.  
    - Builds a refinement-type summarization chain with verbose logging.  
    - Runs the summarization chain on the split transcript documents.  
    - Returns JSON output containing `summary` with the generated content.  
  - Inputs:  
    - `input`: receives video ID from "Set YouTube video ID" node.  
    - `ai_languageModel`: receives OpenAI model instance from "OpenAI Chat Model".  
  - Outputs:  
    - Main output delivers a JSON object with the summary and example questions.  
  - Version Requirements:  
    - Requires n8n version 1.19.4 or later (due to LangChain Code node and module imports).  
  - Edge Cases and Failure Scenarios:  
    - Missing or invalid `searchapi.io` API key will throw an explicit error.  
    - Failure to load transcripts (e.g., invalid video ID, API limits, or network errors).  
    - Errors in LangChain module imports or internal code exceptions.  
    - OpenAI API errors from the model call (e.g., auth, rate limits).  
    - Token limits exceeded if transcript is very large (though chunking mitigates this).  
  - Sub-workflows: None, but relies on external API and LangChain modules within the node.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                        | Input Node(s)            | Output Node(s)         | Sticky Note                                                                                  |
|-------------------------|----------------------------------|-------------------------------------|--------------------------|------------------------|----------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                  | Starts workflow manually             | None                     | Set YouTube video ID    |                                                                                              |
| Set YouTube video ID     | Set                              | Assigns YouTube video ID to data     | When clicking "Execute Workflow" | LangChain Code         |                                                                                              |
| OpenAI Chat Model        | LangChain OpenAI Chat Model      | Provides GPT-4o-mini LLM instance    | None                     | LangChain Code (ai_languageModel) |                                                                                              |
| LangChain Code           | LangChain Code                   | Loads transcript, summarizes, and refines | Set YouTube video ID, OpenAI Chat Model | None                   | Before executing, replace `YOUR_API_KEY` with an API key for searchapi.io                     |
| Sticky Note              | Sticky Note                     | Workflow description and usage note  | None                     | None                   | ## About\nThis workflow shows how you can write LangChain code in n8n (and import its modules if required).\n\nThe workflow fetches a video from YouTube and produces a textual summary of it. |
| Sticky Note7             | Sticky Note                     | API key reminder                     | None                     | None                   | Before executing, replace `YOUR_API_KEY` with an API key for searchapi.io                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a node of type "Manual Trigger" named `When clicking "Execute Workflow"`.  
   - No special configuration needed. This will be the workflow entry point.

2. **Create Set Node for YouTube Video ID**  
   - Add a "Set" node named `Set YouTube video ID`.  
   - Configure to add a new string field named `videoId` with a default value (e.g., "OsMVtuuwOXc").  
   - Connect the output of the Manual Trigger node to this Set node.

3. **Add OpenAI Chat Model Node**  
   - Add a LangChain OpenAI Chat Model node named `OpenAI Chat Model`.  
   - Set the model to `gpt-4o-mini` (choose from the model list).  
   - Leave options default.  
   - Add OpenAI API credentials: configure OAuth2 or API key with valid OpenAI account credentials.  
   - This node does not connect to any previous node directly; it is used as an AI model input connection.

4. **Add LangChain Code Node**  
   - Add a LangChain Code node named `LangChain Code`.  
   - Paste the JavaScript code that:  
     - Declares a variable `searchApiKey` and instructs replacing it with a valid API key for `searchapi.io`.  
     - Imports LangChain modules (`loadSummarizationChain`, `SearchApiLoader`, `PromptTemplate`, `TokenTextSplitter`).  
     - Creates a `SearchApiLoader` instance with engine `"youtube_transcripts"`, the input `videoId`, and the API key.  
     - Throws error if API key is not set.  
     - Loads transcripts, splits them into chunks.  
     - Defines two prompt templates for summary and refinement.  
     - Loads the OpenAI model from input connection `ai_languageModel`.  
     - Creates a summarization chain of type `refine`.  
     - Runs the chain on the split documents and returns JSON output with summary.  
   - Connect the output of the `Set YouTube video ID` node to the main input of this LangChain Code node.  
   - Connect the `OpenAI Chat Model` node as the `ai_languageModel` input connection to this node.

5. **Add Sticky Notes (optional but recommended)**  
   - Add a sticky note near the LangChain Code node reminding to replace `<YOUR API KEY>` with a valid `searchapi.io` API key.  
   - Add another sticky note describing the workflow’s purpose and usage instructions for clarity.

6. **Save and Execute**  
   - Ensure n8n version is 1.19.4 or higher to support LangChain Code node and module imports.  
   - Enter a valid API key for `searchapi.io` in the LangChain Code node code.  
   - Trigger the workflow manually.  
   - The output JSON will contain the summary and example Q&A pairs for the YouTube video transcript.

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                             |
|--------------------------------------------------------------------------------------------------|--------------------------------------------|
| Workflow requires n8n version 1.19.4 or later due to LangChain Code node and module imports.     | n8n Release Notes / Upgrade Guide          |
| LangChain framework official site for detailed info on modules and usage: https://www.langchain.com/ | LangChain official website                  |
| Replace `<YOUR API KEY>` placeholder in LangChain Code node with your actual `searchapi.io` API key before execution. | API key requirement for transcript loading |
| OpenAI GPT-4o-mini model used — ensure OpenAI credentials are configured correctly in n8n.       | OpenAI API documentation                    |
| This workflow fetches YouTube transcripts via SearchApiLoader — ensure videoId is valid and public.| YouTube video data limitations              |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with current content policies and does not contain any illegal, offensive, or protected elements. All manipulated data is legal and public.