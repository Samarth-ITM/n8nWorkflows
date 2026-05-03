Scrape and Analyze Websites with Custom Prompts using Gemini, Apify, and LangChain

https://n8nworkflows.xyz/workflows/scrape-and-analyze-websites-with-custom-prompts-using-gemini--apify--and-langchain-6754


# Scrape and Analyze Websites with Custom Prompts using Gemini, Apify, and LangChain

### 1. Workflow Overview

This workflow is designed to **scrape website content and analyze it with AI** based on a user-defined custom prompt. It targets use cases where users want to extract structured information from websites, such as contact details, summaries, product data, or job listings, by leveraging the combination of Apify for web scraping, LangChain for AI orchestration, and OpenRouter’s Gemini AI models for natural language processing.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives the input parameters including URL, scraping options, and AI prompt.
- **1.2 Content Scraping**: Calls Apify’s scraping actor to fetch content from specified website pages.
- **1.3 Iterative AI Analysis**: Loops over each scraped page to process metadata and Markdown content with AI agents based on the user’s prompt.
- **1.4 Aggregation**: Collects all individual AI responses into a single aggregated output.
- **1.5 Final AI Processing**: Performs a final AI step for structured, clean response generation.
- **1.6 Documentation & Notes**: Provides detailed usage notes and customization instructions via a Sticky Note.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when executed by another workflow and accepts JSON input that specifies the scraping and AI parameters.

- **Nodes Involved:**  
  - When Executed by Another Workflow

- **Node Details:**  
  - **When Executed by Another Workflow**  
    - Type: `executeWorkflowTrigger`  
    - Role: Entry point that triggers execution with JSON input.  
    - Configuration: Input JSON example includes fields for `enqueue` (boolean to queue pages), `maxPages` (max number of pages to scrape), `url` (target website), `method` (HTTP method), and `prompt` (AI instruction).  
    - Inputs: None (trigger node)  
    - Outputs: Passes JSON to HTTP Request node.  
    - Edge Cases: Missing or malformed JSON input can cause downstream failures; validation is recommended.  
    - Version: 1.1

#### 1.2 Content Scraping

- **Overview:**  
  Sends a POST request to Apify’s scraping actor with parameters from the input JSON to start scraping website content and retrieve structured data in Markdown format.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **HTTP Request**  
    - Type: `httpRequest`  
    - Role: Calls Apify API actor to scrape website pages synchronously.  
    - Configuration:  
      - URL points to Apify actor endpoint with a placeholder for API token.  
      - Method: POST  
      - Body: JSON dynamically built from incoming JSON input, includes enqueue flag, maxPages, URL, HTTP method, and options to not get raw HTML or text but structured Markdown data.  
    - Inputs: JSON from trigger node.  
    - Outputs: Scraped pages as dataset items to Loop Over Items node.  
    - Edge Cases: API token invalid or missing, HTTP errors, rate limits, empty or incomplete scrape results.  
    - Version: 4.2

#### 1.3 Iterative AI Analysis

- **Overview:**  
  This block processes each scraped page individually by looping through dataset items and invoking AI agents to analyze content against the user prompt.

- **Nodes Involved:**  
  - Loop Over Items  
  - AI Agent  
  - OpenRouter Chat Model

- **Node Details:**  
  - **Loop Over Items**  
    - Type: `splitInBatches`  
    - Role: Iterates over each scraped page one at a time to allow sequential AI processing.  
    - Configuration: Batch size = 1 to process pages serially.  
    - Inputs: Scraped dataset items from HTTP Request node.  
    - Outputs: Single item batches to AI Agent and Aggregate nodes.  
    - Edge Cases: Large datasets may cause long runtime; consider batch size and rate limits.  
    - Version: 3
  
  - **AI Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Performs AI prompt execution per page, interpreting metadata and markdown with the user prompt.  
    - Configuration:  
      - Prompt text dynamically injects user prompt and page metadata/markdown.  
      - `promptType` set to "define" for custom prompt execution.  
    - Inputs: Single page data from Loop Over Items.  
    - Outputs: AI-generated analysis to Aggregate node.  
    - Edge Cases: AI model response failures, parsing errors.  
    - Version: 2  
  
  - **OpenRouter Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: Provides AI model backend for the AI Agent node.  
    - Configuration: Uses `google/gemini-2.5-flash` model via OpenRouter API credential.  
    - Inputs: Connected as AI language model for AI Agent.  
    - Outputs: AI responses to AI Agent.  
    - Edge Cases: API quota limits, network errors, model unavailability.  
    - Version: 1

#### 1.4 Aggregation

- **Overview:**  
  Aggregates outputs from each AI analysis of individual pages into a collection for further processing.

- **Nodes Involved:**  
  - Aggregate
  
- **Node Details:**  
  - **Aggregate**  
    - Type: `aggregate`  
    - Role: Combines all AI Agent outputs into an array by aggregating the `output` field from each batch.  
    - Configuration: Aggregates on `output` field.  
    - Inputs: AI Agent outputs from Loop Over Items.  
    - Outputs: Aggregated array to the next AI Agent1 node.  
    - Edge Cases: Empty inputs result in empty aggregation; handle downstream accordingly.  
    - Version: 1

#### 1.5 Final AI Processing

- **Overview:**  
  Takes the aggregated AI outputs and runs a final AI prompt to generate a clean, structured, and machine-readable result according to the user prompt.

- **Nodes Involved:**  
  - AI Agent1  
  - OpenRouter Chat Model1

- **Node Details:**  
  - **AI Agent1**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Final AI step that interprets combined AI results to produce a consolidated JSON response that fulfills the prompt.  
    - Configuration:  
      - Text input is stringified aggregated JSON.  
      - System message instructs the AI to collect all information and respond in JSON format aligned with the user prompt.  
      - `promptType` set to "define".  
    - Inputs: Aggregated data from Aggregate node.  
    - Outputs: Final AI response.  
    - Edge Cases: Large input size may cause token limit errors; AI parsing errors possible.  
    - Version: 2
  
  - **OpenRouter Chat Model1**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: AI model backend for final AI Agent.  
    - Configuration: Uses `google/gemini-2.5-pro-preview` model via OpenRouter API credential.  
    - Inputs: Connected as AI language model for AI Agent1.  
    - Outputs: AI-generated final structured output.  
    - Edge Cases: Same as previous OpenRouter model node.  
    - Version: 1

#### 1.6 Documentation & Notes

- **Overview:**  
  Provides comprehensive user guidance, workflow description, and usage instructions embedded as a sticky note.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note**  
    - Type: `stickyNote`  
    - Role: Displays detailed documentation including workflow purpose, input/output examples, customization options, use cases, and required credentials.  
    - Configuration: Large content markdown with links to Apify and OpenRouter, example input JSON, and explanation of each step.  
    - Inputs: None  
    - Outputs: None  
    - Version: 1

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                     | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                           |
|-----------------------------|-----------------------------------------|-----------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | executeWorkflowTrigger                | Workflow entry and input reception | None                        | HTTP Request               |                                                                                                                       |
| HTTP Request                | httpRequest                             | Calls Apify scraping actor         | When Executed by Another Workflow | Loop Over Items           |                                                                                                                       |
| Loop Over Items             | splitInBatches                         | Iterates over scraped pages        | HTTP Request                | Aggregate, AI Agent        |                                                                                                                       |
| AI Agent                   | langchain.agent                        | AI analysis per scraped page       | Loop Over Items             | Aggregate                  |                                                                                                                       |
| OpenRouter Chat Model       | langchain.lmChatOpenRouter             | AI model backend for AI Agent      | AI Agent                   | AI Agent                   |                                                                                                                       |
| Aggregate                   | aggregate                             | Aggregates AI outputs              | Loop Over Items             | AI Agent1                  |                                                                                                                       |
| AI Agent1                  | langchain.agent                        | Final AI consolidation             | Aggregate                   |                            |                                                                                                                       |
| OpenRouter Chat Model1      | langchain.lmChatOpenRouter             | AI model backend for AI Agent1     | AI Agent1                  | AI Agent1                  |                                                                                                                       |
| Sticky Note                | stickyNote                             | Documentation and instructions     | None                       | None                      | Workflow documentation with detailed usage, examples, and links to Apify and OpenRouter.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add `Execute Workflow Trigger` node named `When Executed by Another Workflow`.  
   - Configure `inputSource` to `jsonExample`.  
   - Paste example JSON input:  
     ```json
     {
       "enqueue": true,
       "maxPages": 5,
       "url": "https://apify.com",
       "method": "GET",
       "prompt": "collect all contact informations available on this website"
     }
     ```
   - This node acts as the workflow entry point.

2. **Add HTTP Request Node:**  
   - Add an `HTTP Request` node named `HTTP Request`.  
   - Set method to `POST`.  
   - Set URL to:  
     `https://api.apify.com/v2/acts/mohamedgb00714~firescraper-ai-website-content-markdown-scraper/run-sync-get-dataset-items?token=apify_api_your_apify_api_token`  
     (Replace `apify_api_your_apify_api_token` with your actual Apify API token.)  
   - Enable `Send Body` as JSON.  
   - In `JSON/RAW Parameters`, insert the following JSON with expressions:  
     ```json
     {
       "enqueue": {{$json["enqueue"]}},
       "getHtml": false,
       "getText": false,
       "maxPages": {{$json["maxPages"]}},
       "screenshot": false,
       "startUrls": [
         {
           "url": "{{$json["url"]}}",
           "method": "{{$json["method"]}}"
         }
       ]
     }
     ```
   - Connect `When Executed by Another Workflow` output to this node.

3. **Add Loop Over Items Node:**  
   - Add `SplitInBatches` node named `Loop Over Items`.  
   - Set batch size to `1` to process pages one by one.  
   - Connect output of `HTTP Request` to this node.

4. **Add AI Agent Node for Per-Page Analysis:**  
   - Add `LangChain Agent` node named `AI Agent`.  
   - Set `promptType` to `define`.  
   - In the `Text` field, insert expression:  
     ```
     {{$node["When Executed by Another Workflow"].json["prompt"]}}
     this is only analyse of one page of full website here is the  metadata and markdown of page {{$json["url"]}}
     metadata:{{ JSON.stringify($json["metadata"]) }}
     markdown:{{ $json["markdown"] }}
     ```  
   - Connect `Loop Over Items` to this node.

5. **Add OpenRouter Chat Model Node for AI Backend:**  
   - Add `LangChain OpenRouter Chat Model` node named `OpenRouter Chat Model`.  
   - Set `model` to `google/gemini-2.5-flash`.  
   - Select or create OpenRouter API credentials.  
   - Connect this node as AI language model for `AI Agent`.

6. **Add Aggregate Node:**  
   - Add `Aggregate` node named `Aggregate`.  
   - Configure it to aggregate the `output` field from incoming data.  
   - Connect output of `Loop Over Items` (main output 1) to this node.  
   - Connect output of `AI Agent` to this node as well, so it collects all AI results.

7. **Add Final AI Agent Node:**  
   - Add another `LangChain Agent` node named `AI Agent1`.  
   - Set `promptType` to `define`.  
   - In `Text` field, use expression:  
     ```
     {{ JSON.stringify($json) }}
     ```  
   - In `options.systemMessage`, insert:  
     ```
     You are a helpful assistant you should collect alll informations from input to respond to this prompt with json format {{$node["When Executed by Another Workflow"].json["prompt"]}}
     ```  
   - Connect output of `Aggregate` node to this node.

8. **Add Final OpenRouter Chat Model Node:**  
   - Add another `LangChain OpenRouter Chat Model` node named `OpenRouter Chat Model1`.  
   - Set model to `google/gemini-2.5-pro-preview`.  
   - Use the same OpenRouter API credentials.  
   - Connect this node as AI language model for `AI Agent1`.

9. **Add Sticky Note for Documentation:**  
   - Add a `Sticky Note` node.  
   - Paste the provided detailed markdown content describing workflow purpose, input/output, use cases, and credential requirements.

10. **Connect Nodes:**  
    - From `When Executed by Another Workflow` → `HTTP Request`  
    - `HTTP Request` → `Loop Over Items`  
    - `Loop Over Items` main output 0 → `AI Agent`  
    - `AI Agent` → `Aggregate`  
    - `Loop Over Items` main output 1 → `Aggregate` (if applicable)  
    - `Aggregate` → `AI Agent1`  
    - `AI Agent` uses `OpenRouter Chat Model` as AI backend  
    - `AI Agent1` uses `OpenRouter Chat Model1` as AI backend

11. **Credential Setup:**  
    - Create and configure Apify API credentials with your Apify token.  
    - Create and configure OpenRouter API credentials with your OpenRouter key.

12. **Run and Test:**  
    - Trigger the workflow via the entry node with JSON input specifying URL, prompt, and scraping parameters.  
    - Validate the final output JSON for expected structured data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                           | Context or Link                                                       |
|----------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow combines Apify scraping with OpenRouter’s Gemini AI models to enable flexible web data extraction via custom prompts.    | Official Apify: https://apify.com                                    |
| OpenRouter provides access to Gemini models `google/gemini-2.5-flash` and `google/gemini-2.5-pro-preview` used here for AI processing. | OpenRouter API: https://openrouter.ai                                |
| The workflow is designed to be triggered by other workflows, enabling integration into larger automation pipelines or AI agent systems.| n8n Workflow Integration                                             |
| Use the input JSON format to customize URL, scraping depth, HTTP method, and AI prompt for flexible reuse across many web scraping tasks.| Input JSON example included in sticky note                           |
| Potential failure points include API rate limits, invalid credentials, large input causing token overflow, or malformed input JSON.    | Error handling recommendations: validate inputs and monitor quotas. |

---

**Disclaimer:**  
The provided content originates solely from an n8n automated workflow. It strictly adheres to content policies, contains no illegal or offensive material, and only processes legal, publicly available data.