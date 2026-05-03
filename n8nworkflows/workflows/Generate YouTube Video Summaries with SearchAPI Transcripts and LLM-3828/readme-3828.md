Generate YouTube Video Summaries with SearchAPI Transcripts and LLM

https://n8nworkflows.xyz/workflows/generate-youtube-video-summaries-with-searchapi-transcripts-and-llm-3828


# Generate YouTube Video Summaries with SearchAPI Transcripts and LLM

### 1. Workflow Overview

This workflow automates the generation of concise summaries for YouTube videos by leveraging SearchAPI.io’s YouTube transcript retrieval and a Large Language Model (LLM) summarization chain. It is designed for users who want to quickly extract key insights from lengthy video content without manual transcription or extensive viewing.

**Target Use Cases:**  
- Content creators summarizing competitor or reference videos  
- Students and educators distilling lecture or tutorial content  
- Digital marketers analyzing video content for insights  
- Researchers needing rapid synthesis of video material

**Logical Blocks:**

- **1.1 Input Reception:** Manual trigger to start the workflow, accepting a YouTube video ID.  
- **1.2 Transcript Retrieval:** Calls SearchAPI.io to fetch the full transcript of the specified YouTube video.  
- **1.3 Transcript Processing:** Splits and concatenates raw transcript data to prepare it for summarization.  
- **1.4 Summarization Chain:** Uses an LLM (via OpenRouter) and a recursive character text splitter to generate a concise summary from the transcript.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Initiates the workflow manually, serving as the entry point for the YouTube video ID.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow to input parameters (e.g., video ID).  
    - Configuration: No parameters by default; video ID is hardcoded downstream but can be parameterized.  
    - Input connections: None (start node)  
    - Output connections: Triggers SearchAPI node  
    - Version: 1  
    - Edge cases: User must provide valid video ID; workflow does not handle missing/invalid IDs dynamically.

#### 2.2 Transcript Retrieval

- **Overview:**  
  Fetches the complete transcript for the specified YouTube video using the SearchAPI.io service.

- **Nodes Involved:**  
  - SearchAPI  
  - Split Out

- **Node Details:**  
  - **SearchAPI**  
    - Type: Custom API node for SearchAPI.io  
    - Role: Executes a request to the "youtube_transcripts" engine with parameters: video_id (hardcoded here as "2XTdEG0Sus0") and language "en" (English).  
    - Configuration: Uses SearchAPI.io credentials; the video ID is statically set but can be replaced with dynamic input.  
    - Input: Trigger from manual node  
    - Output: Returns transcript data with potentially multiple entries  
    - Version: 1  
    - Edge cases: API errors, invalid video ID, no transcript available, rate limiting, credential issues.

  - **Split Out**  
    - Type: Split Out  
    - Role: Extracts the "transcripts" field from the SearchAPI response, splitting it into individual transcript entries for processing.  
    - Configuration: Field to split is "transcripts"  
    - Input: SearchAPI node output  
    - Output: Multiple items each representing one transcript segment  
    - Version: 1  
    - Edge cases: Empty transcripts array, unexpected data format.

#### 2.3 Transcript Processing and Summarization Preparation

- **Overview:**  
  Aggregates and concatenates the split transcript segments into a continuous text string to prepare for summarization.

- **Nodes Involved:**  
  - Summarize

- **Node Details:**  
  - **Summarize**  
    - Type: Summarize (native n8n node)  
    - Role: Concatenates the "text" field from each transcript segment into a single combined string, separating by space.  
    - Configuration: Fields to summarize: "text", separated by spaces, aggregation method is concatenation.  
    - Input: Output from "Split Out" node (individual transcript segments)  
    - Output: One item with concatenated transcript text  
    - Version: 1.1  
    - Edge cases: Missing "text" fields, very large text that might exceed node limits.

#### 2.4 Summarization Chain

- **Overview:**  
  Uses a recursive character text splitter and an LLM summarization chain to generate a concise summary from the transcript text.

- **Nodes Involved:**  
  - Recursive Character Text Splitter  
  - OpenRouter Chat Model  
  - Summarization Chain

- **Node Details:**  
  - **Recursive Character Text Splitter**  
    - Type: LangChain Text Splitter (Recursive Character)  
    - Role: Splits large transcript text into chunks of max 6000 characters for efficient LLM processing.  
    - Configuration: Chunk size set to 6000 characters  
    - Input: Output from "Summarize" node (concatenated transcript)  
    - Output: Split text chunks  
    - Version: 1  
    - Edge cases: Very short texts may not split; very long texts truncated if chunk size exceeded.

  - **OpenRouter Chat Model**  
    - Type: LangChain LLM Chat Model node  
    - Role: Connects to OpenRouter API using a qwen3-0.6b model for generating summaries.  
    - Configuration: Model set to "qwen/qwen3-0.6b-04-28:free"; no additional options specified.  
    - Input: Connected as the LLM provider to the summarization chain  
    - Output: LLM responses (summary text)  
    - Version: 1  
    - Edge cases: API authentication failure, rate limiting, model unavailability.

  - **Summarization Chain**  
    - Type: LangChain Summarization Chain  
    - Role: Coordinates the summarization process, receiving chunked text and the LLM model to produce the final summary.  
    - Configuration: Chunking mode set to "advanced" (likely optimized chunk handling)  
    - Input: Receives text chunks (from Recursive Character Text Splitter) and LLM (from OpenRouter Chat Model)  
    - Output: Final summarized text of the YouTube video transcript  
    - Version: 2  
    - Edge cases: Failures in chunking, LLM processing errors, timeouts.

---

### 3. Summary Table

| Node Name                  | Node Type                                    | Functional Role                             | Input Node(s)              | Output Node(s)                | Sticky Note                                                                                                    |
|----------------------------|----------------------------------------------|---------------------------------------------|----------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                              | Workflow entry point; manual start          | None                       | SearchAPI                    | ## Youtube Video Summary<br>Given a **video_id** from Youtube, we concatenate the data and pass it to a summarization chain. To run this workflow, you need to have the credentials for Searchapi.io and some LLM provider. |
| SearchAPI                  | Custom SearchAPI.io node                      | Fetch YouTube video transcripts              | When clicking ‘Test workflow’ | Split Out                    |                                                                                                               |
| Split Out                  | Split Out node                               | Splits transcript array into individual items | SearchAPI                  | Summarize                    | ## Tip<br>You can pass the **video_id** from previous nodes to make a better automation                       |
| Summarize                  | Summarize node                              | Concatenates transcript segments into one text | Split Out                  | Summarization Chain          |                                                                                                               |
| Recursive Character Text Splitter | LangChain Text Splitter (Character Recursive) | Splits large transcript text into chunks     | Summarize                  | Summarization Chain          |                                                                                                               |
| OpenRouter Chat Model      | LangChain LLM Chat Model                     | Provides LLM summarization via OpenRouter API | None (connected as ai_languageModel) | Summarization Chain          |                                                                                                               |
| Summarization Chain        | LangChain Summarization Chain                | Combines chunking and LLM to produce summary | Summarize, Recursive Character Text Splitter, OpenRouter Chat Model | None                       |                                                                                                               |
| Sticky Note                | Sticky Note                                  | Informational note                           | None                       | None                         | ## Youtube Video Summary<br>Given a **video_id** from Youtube, we concatenate the data and pass it to a summarization chain. To run this workflow, you need to have the credentials for Searchapi.io and some LLM provider. |
| Sticky Note1               | Sticky Note                                  | Tip note                                    | None                       | None                         | ## Tip<br>You can pass the **video_id** from previous nodes to make a better automation                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: When clicking ‘Test workflow’  
   - Purpose: Start the workflow manually.

2. **Add SearchAPI Node**  
   - Type: Custom node for SearchAPI.io (engine: youtube_transcripts)  
   - Name: SearchAPI  
   - Configure credentials: Link to SearchAPI.io account credentials.  
   - Parameters:  
     - engine: youtube_transcripts  
     - parameters:  
       - video_id: (set default value, e.g., "2XTdEG0Sus0"; replace with dynamic input if preferred)  
       - lang: "en"  
   - Connect output from Manual Trigger node to this node.

3. **Add Split Out Node**  
   - Type: Split Out  
   - Name: Split Out  
   - Parameter: fieldToSplitOut = "transcripts"  
   - Connect input from SearchAPI node.  
   - Purpose: To split the array of transcript segments into individual items.

4. **Add Summarize Node**  
   - Type: Summarize  
   - Name: Summarize  
   - Parameters:  
     - fieldsToSummarize: select "text" field  
     - separateBy: " " (space)  
     - aggregation: "concatenate"  
   - Connect input from Split Out node.  
   - Purpose: Concatenate all transcript segments text into one continuous string.

5. **Add Recursive Character Text Splitter Node**  
   - Type: LangChain Text Splitter (Recursive Character)  
   - Name: Recursive Character Text Splitter  
   - Parameters:  
     - chunkSize: 6000 characters  
   - Connect input from Summarize node.  
   - Purpose: Split large concatenated transcript into manageable chunks for LLM.

6. **Add OpenRouter Chat Model Node**  
   - Type: LangChain LLM Chat Model  
   - Name: OpenRouter Chat Model  
   - Configure credentials: Link OpenRouter API credentials.  
   - Parameters:  
     - model: "qwen/qwen3-0.6b-04-28:free" (or preferred LLM model)  
     - options: leave empty or configure as needed.  
   - No input connection from previous nodes (linked later as LLM provider).

7. **Add Summarization Chain Node**  
   - Type: LangChain Summarization Chain  
   - Name: Summarization Chain  
   - Parameters:  
     - chunkingMode: "advanced"  
   - Connect input connections as follows:  
     - Main input: from Summarize node (concatenated transcript text)  
     - ai_textSplitter input: from Recursive Character Text Splitter node  
     - ai_languageModel input: from OpenRouter Chat Model node  
   - Purpose: Executes the summarization chain using the text splitter and LLM model.

8. **Add Sticky Notes (Optional)**  
   - Add two sticky notes with content:  
     - Youtube Video Summary explanation  
     - Tip about passing dynamic video_id from previous nodes.

9. **Verify Connections and Credentials**  
   - Ensure SearchAPI credentials are valid and linked.  
   - Ensure OpenRouter API credentials are configured correctly.  
   - Confirm all nodes are connected in the order: Manual Trigger → SearchAPI → Split Out → Summarize → Summarization Chain (with Recursive Character Text Splitter and OpenRouter Chat Model as inputs).

10. **Test the Workflow**  
    - Manually trigger the workflow.  
    - Provide or modify video_id as needed.  
    - Inspect output summary from Summarization Chain node.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                               |
|----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| This workflow relies on SearchAPI.io for YouTube transcript retrieval; sign up and configure API keys accordingly.         | https://www.searchapi.io/                      |
| The summarization is powered by LangChain nodes integrated with OpenRouter API; ensure you have valid credentials.          | https://openrouter.ai/                         |
| Customize prompt templates within the Summarization Chain node to modify summary style (bullet points, paragraphs, etc.).  | n8n LangChain Summarization Chain documentation |
| Consider extending the workflow by exporting summaries to Google Drive, Slack, or email for automated content distribution. | n8n integrations documentation                 |
| Video on using SearchAPI with n8n for YouTube transcripts available for deeper understanding (search on n8n community).     | n8n community forum / YouTube                   |

---

This reference document fully describes the "Generate YouTube Video Summaries with SearchAPI Transcripts and LLM" workflow, enabling both human users and automation agents to understand, reproduce, and modify it effectively.