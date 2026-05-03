Convert Reddit Business Pain Points to Viral LinkedIn Content with AI and Freepik

https://n8nworkflows.xyz/workflows/convert-reddit-business-pain-points-to-viral-linkedin-content-with-ai-and-freepik-8926


# Convert Reddit Business Pain Points to Viral LinkedIn Content with AI and Freepik

### 1. Workflow Overview

This workflow automates the process of transforming Reddit business pain points from various niche communities into viral LinkedIn posts enriched with AI-generated text and relevant images from Freepik. It aims to support social media content creators and marketers by streamlining content ideation, generation, and posting.

The workflow is logically divided into these main blocks:

- **1.1 Scheduled Reddit Data Collection:** Periodically fetches Reddit posts from multiple business-related subreddits relevant to different niches (e.g., Dentist, Accounting, Ecommerce).
- **1.2 Data Merging and Filtering:** Merges all Reddit posts into a single stream and filters or batches them for processing.
- **1.3 Pain Points Extraction:** Processes Reddit posts to extract explicit business pain points using AI.
- **1.4 Solution and Content Generation:** Uses AI to generate potential solutions and craft engaging LinkedIn post content based on the pain points.
- **1.5 Image Generation and Download:** Produces image prompts via AI, requests images from Freepik, downloads them, and prepares for post attachment.
- **1.6 LinkedIn Post Creation:** Publishes generated posts along with appropriate images to LinkedIn.
- **1.7 Data Aggregation and Logging:** Aggregates processed data and appends results back to Google Sheets for record-keeping.
- **1.8 Workflow Orchestration:** Controls triggers, loops, conditional logic, and manages execution flow.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Reddit Data Collection

**Overview:**  
This block periodically triggers the workflow and fetches posts from multiple Reddit communities related to different business niches.

**Nodes Involved:**  
- Schedule Trigger  
- Get many posts Dentist  
- Get many posts Beauty  
- Get many posts Accounting  
- Get many posts Real Estate  
- Get many posts lawyer  
- Get many posts Ecommerce  
- Get many posts HR  
- Get many posts Online Education  
- Get many posts Online Coach  
- Get many posts interior_design  
- Merger

**Node Details:**  

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow execution on a configured schedule (periodicity not specified).  
  - *Configuration:* Default schedule trigger node; likely configured for periodic runs.  
  - *Input/Output:* No input; outputs to multiple Reddit nodes.  
  - *Edge Cases:* Missed triggers if n8n is offline; no retry logic.  

- **Get many posts [Multiple Reddit nodes]**  
  - *Type:* Reddit node  
  - *Role:* Fetches posts from specific subreddits relevant to each business niche.  
  - *Configuration:* Each node targets a different subreddit or Reddit query.  
  - *Input/Output:* Inputs from Schedule Trigger; outputs to Merger node.  
  - *Edge Cases:* Reddit API rate limits, subreddit access issues, empty results.  

- **Merger**  
  - *Type:* Merge node  
  - *Role:* Combines multiple Reddit post streams into a single unified output for downstream processing.  
  - *Input/Output:* Inputs from all Reddit nodes; outputs to filter or batching nodes.  
  - *Edge Cases:* Data stream synchronization issues.

---

#### 2.2 Data Merging and Filtering

**Overview:**  
Manages the combined Reddit data stream, filters relevant posts, and splits them into manageable batches for processing.

**Nodes Involved:**  
- +5 UPS (Filter)  
- Loop Over Items (SplitInBatches)  
- If  
- No Operation, do nothing

**Node Details:**  

- **+5 UPS (Filter)**  
  - *Type:* Filter node  
  - *Role:* Applies criteria to select posts meeting specific conditions (e.g., minimum upvotes).  
  - *Configuration:* Likely filters posts with more than 5 upvotes (inferred from name).  
  - *Input/Output:* Input from Merger node; outputs to Loop Over Items.  
  - *Edge Cases:* Possible empty filtered sets if criteria too strict.  

- **Loop Over Items (SplitInBatches)**  
  - *Type:* SplitInBatches node  
  - *Role:* Processes posts in batches to control workflow load and API usage.  
  - *Configuration:* Batch size unspecified; probably set to a manageable number.  
  - *Input/Output:* Input from +5 UPS; outputs to conditional node and AI processing nodes.  
  - *Edge Cases:* Batch size too large or small affects performance or throughput.  

- **If**  
  - *Type:* Conditional node  
  - *Role:* Evaluates conditions to decide next actions, e.g., skipping empty batches.  
  - *Configuration:* Details unspecified; likely checks if batch contains valid data.  
  - *Input/Output:* Input from Loop Over Items; outputs to either Edit Fields1 or Loop Over Items (to continue).  
  - *Edge Cases:* Expression evaluation errors if input data missing.

- **No Operation, do nothing**  
  - *Type:* NoOp node  
  - *Role:* Placeholder to halt or skip workflow path cleanly.  
  - *Configuration:* Default.  
  - *Input/Output:* Input from Loop Over Items; no output downstream.  
  - *Edge Cases:* None.

---

#### 2.3 Pain Points Extraction

**Overview:**  
Extracts business pain points from Reddit posts using AI language models to identify key challenges faced by the community.

**Nodes Involved:**  
- agente_pain_points (Chain LLM)  
- Code2  
- Extraer_pain_points (Google Sheets)  
- Aggregate  
- Edit Fields

**Node Details:**  

- **agente_pain_points**  
  - *Type:* LangChain Chain LLM node  
  - *Role:* Applies AI prompt chaining to analyze Reddit post content and extract pain points.  
  - *Configuration:* Uses OpenAI 4.1 Mini model for natural language understanding.  
  - *Input/Output:* Input from Loop Over Items; outputs to Code2.  
  - *Edge Cases:* AI response errors, rate limits, misunderstood context.  

- **Code2**  
  - *Type:* Code node  
  - *Role:* Processes AI output for formatting or validation before conditional filtering.  
  - *Configuration:* Custom JavaScript logic (details abstracted).  
  - *Input/Output:* Input from agente_pain_points; outputs to If node.  
  - *Edge Cases:* Script errors due to unexpected AI output.  

- **Extraer_pain_points (Google Sheets)**  
  - *Type:* Google Sheets node  
  - *Role:* Reads or writes extracted pain points from/to Google Sheets for persistence or further processing.  
  - *Configuration:* Specific sheet and range not detailed.  
  - *Input/Output:* Input from Code1; outputs to Aggregate.  
  - *Edge Cases:* Auth failures, API quota limits, sheet access issues.  

- **Aggregate**  
  - *Type:* Aggregate node  
  - *Role:* Consolidates extracted data for next steps.  
  - *Configuration:* Aggregation logic unspecified.  
  - *Edge Cases:* Data inconsistency if inputs incomplete.

- **Edit Fields**  
  - *Type:* Set node  
  - *Role:* Prepares or modifies fields in the aggregated data for AI post generation.  
  - *Edge Cases:* Expression errors if fields missing.

---

#### 2.4 Solution and Content Generation

**Overview:**  
Generates AI-driven solutions and crafts engaging LinkedIn post content based on extracted pain points.

**Nodes Involved:**  
- agente_soluciones (Chain LLM)  
- Code3  
- Edit Fields1  
- agente_post (Agent Language Model)  
- nicho_aleatorio (Chain LLM)  
- Code1

**Node Details:**  

- **agente_soluciones**  
  - *Type:* LangChain Chain LLM node  
  - *Role:* Creates solution-oriented content from pain points.  
  - *Configuration:* Uses OpenAI 4.1 Mini model.  
  - *Input/Output:* Input from Edit Fields1; outputs to Code3.  
  - *Edge Cases:* AI model latency, incomplete outputs.  

- **Code3**  
  - *Type:* Code node  
  - *Role:* Formats and validates solution content before appending to Google Sheets.  
  - *Edge Cases:* Script runtime errors.  

- **Edit Fields1**  
  - *Type:* Set node  
  - *Role:* Adjusts fields for solution generation input.  
  - *Edge Cases:* Missing input fields.  

- **agente_post**  
  - *Type:* LangChain Agent node  
  - *Role:* Combines solution content to generate final LinkedIn post text.  
  - *Configuration:* Uses Gemini 2.5 Pro and OpenAI 4.1 Mini language models.  
  - *Edge Cases:* API limits, model integration errors.  

- **nicho_aleatorio**  
  - *Type:* Chain LLM node  
  - *Role:* Selects or generates a random niche as part of content personalization.  
  - *Edge Cases:* Model response failures.  

- **Code1**  
  - *Type:* Code node  
  - *Role:* Processes niche selection results for downstream use.  

---

#### 2.5 Image Generation and Download

**Overview:**  
Creates AI-generated image prompts, requests images from Freepik API, downloads images for final LinkedIn posts.

**Nodes Involved:**  
- prompt_imagen (Chain LLM)  
- Crear Imagen (HTTP Request)  
- Wait  
- Get_freepik (HTTP Request)  
- Descargar_imagen1 (HTTP Request)

**Node Details:**  

- **prompt_imagen**  
  - *Type:* Chain LLM node  
  - *Role:* Generates descriptive image prompts based on post content.  
  - *Edge Cases:* AI text generation errors.  

- **Crear Imagen**  
  - *Type:* HTTP Request node  
  - *Role:* Sends request to Freepik or image generation API to create image resource.  
  - *Edge Cases:* API failures, rate limits, network errors.  

- **Wait**  
  - *Type:* Wait node  
  - *Role:* Delays execution to allow image generation to complete before next step.  
  - *Edge Cases:* Timeout if images not ready in time.  

- **Get_freepik**  
  - *Type:* HTTP Request node  
  - *Role:* Retrieves image metadata or download link from Freepik API.  
  - *Edge Cases:* Invalid credentials, API errors.  

- **Descargar_imagen1**  
  - *Type:* HTTP Request node  
  - *Role:* Downloads the image file to be attached to LinkedIn post.  
  - *Edge Cases:* Download failures, broken links.

---

#### 2.6 LinkedIn Post Creation

**Overview:**  
Publishes the AI-generated content and images as LinkedIn posts.

**Nodes Involved:**  
- Create a post1 (LinkedIn node)

**Node Details:**  

- **Create a post1**  
  - *Type:* LinkedIn node  
  - *Role:* Posts content and image attachments to LinkedIn user account.  
  - *Configuration:* Requires LinkedIn OAuth2 credentials and post parameters (text, media).  
  - *Edge Cases:* Auth token expiration, API posting errors, media upload failures.

---

#### 2.7 Data Aggregation and Logging

**Overview:**  
Stores generated content details and metadata back into Google Sheets for tracking and analytics.

**Nodes Involved:**  
- Append row in sheet (Google Sheets)  
- Loop Over Items

**Node Details:**  

- **Append row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends new rows with post data into a spreadsheet.  
  - *Edge Cases:* Google API quota, permission errors.  

- **Loop Over Items**  
  - *Type:* SplitInBatches node  
  - *Role:* Iterates through data to append each entry individually.  

---

#### 2.8 Workflow Orchestration and Miscellaneous

**Overview:**  
Controls workflow execution, manages field editing, and contains several utility nodes.

**Nodes Involved:**  
- Edit Fields  
- Edit Fields1  
- Code nodes  
- Sticky Note nodes (various)  
- Schedule Trigger1

**Node Details:**  

- **Edit Fields / Edit Fields1**  
  - Used for field manipulation and preparation between steps.  

- **Code nodes (Code, Code1, Code2, Code3)**  
  - Contain custom JavaScript for intermediate data transformations.  

- **Sticky Notes**  
  - Visual annotations for human operators; no functional impact.  

- **Schedule Trigger1**  
  - Secondary schedule trigger that initiates niche randomization and related steps.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                           | Input Node(s)                                     | Output Node(s)                        | Sticky Note                              |
|---------------------------|----------------------------|------------------------------------------|--------------------------------------------------|-------------------------------------|----------------------------------------|
| Schedule Trigger           | Schedule Trigger           | Initiates workflow runs                   | None                                             | Get many posts Dentist, etc.         |                                        |
| Get many posts Dentist     | Reddit                    | Fetch Reddit posts for Dentist niche     | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts Beauty      | Reddit                    | Fetch Reddit posts for Beauty niche      | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts Accounting  | Reddit                    | Fetch Reddit posts for Accounting niche  | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts Real Estate | Reddit                    | Fetch Reddit posts for Real Estate niche | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts lawyer      | Reddit                    | Fetch Reddit posts for Lawyer niche      | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts Ecommerce   | Reddit                    | Fetch Reddit posts for Ecommerce niche   | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts HR          | Reddit                    | Fetch Reddit posts for HR niche          | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts Online Education | Reddit              | Fetch Reddit posts for Online Education  | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts Online Coach| Reddit                    | Fetch Reddit posts for Online Coach      | Schedule Trigger                                 | Merger                              |                                        |
| Get many posts interior_design | Reddit               | Fetch Reddit posts for interior design   | Schedule Trigger                                 | Merger                              |                                        |
| Merger                    | Merge                     | Combines Reddit posts from all niches    | Multiple Get many posts nodes                    | +5 UPS (Filter)                    |                                        |
| +5 UPS                    | Filter                    | Filters posts by criteria (e.g., upvotes)| Merger                                           | Loop Over Items                    |                                        |
| Loop Over Items           | SplitInBatches            | Batches posts for processing              | +5 UPS                                           | If, No Operation, agente_pain_points |                                        |
| If                        | If                        | Conditional routing                       | Loop Over Items                                  | Edit Fields1 / Loop Over Items      |                                        |
| No Operation, do nothing  | NoOp                      | Placeholder to skip processing            | Loop Over Items                                  | None                              |                                        |
| agente_pain_points         | Chain LLM (LangChain)     | AI extraction of pain points              | Loop Over Items                                  | Code2                             |                                        |
| Code2                     | Code                      | Processes AI pain points output           | agente_pain_points                               | If                               |                                        |
| Extraer_pain_points        | Google Sheets             | Reads/Writes pain points data             | Code1                                            | Aggregate                        |                                        |
| Aggregate                 | Aggregate                 | Aggregates pain points data                | Extraer_pain_points                              | Edit Fields                      |                                        |
| Edit Fields               | Set                       | Prepares data for post generation          | Aggregate                                        | agente_post                     |                                        |
| agente_soluciones          | Chain LLM (LangChain)     | Generates solutions from pain points       | Edit Fields1                                     | Code3                           |                                        |
| Code3                     | Code                      | Formats solution output                    | agente_soluciones                                | Append row in sheet             |                                        |
| Edit Fields1              | Set                       | Prepares data for solution generation      | If                                               | agente_soluciones              |                                        |
| agente_post               | Agent (LangChain)         | Generates final LinkedIn post content      | Edit Fields, SerpAPI                             | Code                            | AVERIS TECH                           |
| nicho_aleatorio           | Chain LLM (LangChain)     | Generates or selects random niche          | Schedule Trigger1                                | Code1                          | AVERIS TECH                           |
| Code1                     | Code                      | Processes niche selection                   | nicho_aleatorio                                  | Extraer_pain_points             |                                        |
| prompt_imagen             | Chain LLM (LangChain)     | Generates image prompts for Freepik         | Code                                              | Crear Imagen                   | AVERIS TECH                           |
| Crear Imagen              | HTTP Request              | Requests image creation from Freepik        | prompt_imagen                                    | Wait                           |                                        |
| Wait                      | Wait                      | Delays for image availability                | Crear Imagen                                     | Get_freepik                    |                                        |
| Get_freepik               | HTTP Request              | Retrieves image download info                  | Wait                                             | Descargar_imagen1              |                                        |
| Descargar_imagen1         | HTTP Request              | Downloads image file                            | Get_freepik                                      | Create a post1                |                                        |
| Create a post1            | LinkedIn                  | Publishes post and image                        | Descargar_imagen1                                | None                          |                                        |
| Append row in sheet       | Google Sheets             | Logs post data                                    | Code3                                            | Loop Over Items              |                                        |
| Schedule Trigger1         | Schedule Trigger          | Secondary trigger for niche randomization      | None                                             | nicho_aleatorio              |                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Set to desired periodicity (e.g., daily).  
   - Connect output to multiple Reddit nodes.

2. **Create Reddit Nodes for Each Niche**  
   - Node Type: Reddit  
   - Configure subreddit or search query per niche (Dentist, Beauty, Accounting, etc.).  
   - Connect each Reddit node output to a Merge node.

3. **Create Merge Node**  
   - Type: Merge  
   - Mode: Merge Inputs (Append).  
   - Connect all Reddit nodes into this node.

4. **Create Filter Node ("+5 UPS")**  
   - Filter posts based on criteria, e.g., minimum 5 upvotes.  
   - Input from Merge node.

5. **Create SplitInBatches Node ("Loop Over Items")**  
   - Define batch size (e.g., 5 or 10).  
   - Input from Filter node.

6. **Create If Node**  
   - Condition: Check if batch is not empty or meets processing criteria.  
   - Connect True output to a Set node ("Edit Fields1").  
   - Connect False output to a NoOp node ("No Operation, do nothing").

7. **Create Chain LLM Node ("agente_pain_points")**  
   - Model: OpenAI 4.1 Mini  
   - Purpose: Extract pain points from Reddit post batch.  
   - Input from Loop Over Items (True path).

8. **Create Code Node ("Code2")**  
   - Process AI output to desired format.  
   - Connect from agente_pain_points output.  
   - Connect to If node.

9. **Create Google Sheets Node ("Extraer_pain_points")**  
   - Set to read/write pain points sheet.  
   - Input from Code1 (see below).

10. **Create Aggregate Node**  
    - Aggregate data from Google Sheets.  
    - Input from Extraer_pain_points.

11. **Create Set Node ("Edit Fields")**  
    - Prepare aggregated data for post generation.  
    - Input from Aggregate.

12. **Create Chain LLM Node ("agente_soluciones")**  
    - Model: OpenAI 4.1 Mini  
    - Purpose: Generate solutions content.  
    - Input from Edit Fields1.

13. **Create Code Node ("Code3")**  
    - Format solutions for posting and sheet logging.  
    - Input from agente_soluciones.  
    - Output to Google Sheets append node.

14. **Create Google Sheets Append Node ("Append row in sheet")**  
    - Append processed posts data and metadata.  
    - Input from Code3.

15. **Create Chain LLM Node ("agente_post")**  
    - Models: Gemini 2.5 Pro & OpenAI 4.1 Mini (configured as language models and AI tools).  
    - Purpose: Generate final LinkedIn post text.  
    - Input from Edit Fields and SerpAPI node.

16. **Create Chain LLM Node ("nicho_aleatorio")**  
    - Model: OpenAI 4.1 Mini  
    - Purpose: Select a random niche for content personalization.  
    - Input from second Schedule Trigger node ("Schedule Trigger1").

17. **Create Code Node ("Code1")**  
    - Process niche selection output.  
    - Input from nicho_aleatorio.

18. **Connect Code1 output to Google Sheets node ("Extraer_pain_points")**  
    - To feed niche data into pain points extraction.

19. **Create Chain LLM Node ("prompt_imagen")**  
    - Generate descriptive image prompts from post content.  
    - Input from Code node ("Code").

20. **Create HTTP Request Node ("Crear Imagen")**  
    - Calls Freepik or image generation API with prompt from prompt_imagen.  
    - Input from prompt_imagen.

21. **Create Wait Node ("Wait")**  
    - Delay to allow image generation completion.  
    - Input from Crear Imagen.

22. **Create HTTP Request Node ("Get_freepik")**  
    - Fetch image download links or metadata from Freepik.  
    - Input from Wait.

23. **Create HTTP Request Node ("Descargar_imagen1")**  
    - Downloads image file.  
    - Input from Get_freepik.

24. **Create LinkedIn Node ("Create a post1")**  
    - Posts content and image to LinkedIn.  
    - Input from Descargar_imagen1.  
    - Configure with LinkedIn OAuth2 credentials.

25. **Configure Credentials**  
    - Reddit API credentials for Reddit nodes.  
    - Google Sheets OAuth2 credentials for Sheets nodes.  
    - OpenAI and Gemini API keys for LangChain nodes (models).  
    - LinkedIn OAuth2 credentials for post node.  
    - Freepik API credentials for HTTP requests.

26. **Add Sticky Notes**  
    - Optional: Add sticky notes for documentation or reminders.

27. **Validate and Test**  
    - Test each block independently.  
    - Monitor rate limits and errors.  
    - Adjust batch sizes and timeouts as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow automates social media content generation from Reddit business pain points.          | General project description                      |
| AI models used include OpenAI 4.1 Mini and Gemini 2.5 Pro via LangChain integration.           | AI processing                                    |
| Freepik API used for image sourcing based on AI-generated prompts.                            | Image generation integration                      |
| LinkedIn posts created via OAuth2 secured API calls.                                         | Social media publishing                           |
| Workflow designed and annotated by AVERIS TECH.                                              | Project credit                                   |
| Sticky notes in workflow provide additional human-readable annotations without affecting flow.| n8n sticky notes use                              |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal or protected material. All data handled is public and lawful.