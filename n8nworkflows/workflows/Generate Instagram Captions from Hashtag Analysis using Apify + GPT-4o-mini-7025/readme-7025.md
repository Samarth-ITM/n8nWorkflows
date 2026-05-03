Generate Instagram Captions from Hashtag Analysis using Apify + GPT-4o-mini

https://n8nworkflows.xyz/workflows/generate-instagram-captions-from-hashtag-analysis-using-apify---gpt-4o-mini-7025


# Generate Instagram Captions from Hashtag Analysis using Apify + GPT-4o-mini

### 1. Workflow Overview

This n8n workflow automates the generation of Instagram caption ideas based on recent posts found under a specified hashtag. It leverages the Apify Instagram Hashtag Scraper to collect recent Instagram posts, extracts and aggregates their captions, and feeds this data into an AI agent powered by GPT-4o-mini (OpenAI) to generate relevant caption ideas and identify common post themes. The workflow is ideal for social media marketers, content creators, and brands seeking data-driven inspiration for Instagram content.

The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger and Input Setup:** Starts the workflow manually and sets the hashtag to analyze.
- **1.2 Instagram Data Retrieval:** Calls the Apify Instagram Hashtag Scraper API to fetch recent Instagram posts for the chosen hashtag.
- **1.3 Caption Extraction and Aggregation:** Extracts captions from posts and combines them into a single text block.
- **1.4 AI Caption Idea Generation:** Uses GPT-4o-mini to analyze captions and generate new caption ideas.
- **1.5 Output Parsing and Formatting:** Parses the AI output JSON and splits lists for final output.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger and Input Setup

**Overview:**  
This block initiates the workflow manually and defines the hashtag keyword used to scrape Instagram posts.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Create Search Term

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point; allows manual start for testing or one-time execution.  
  - *Configuration:* No parameters; simply triggers workflow on click.  
  - *Inputs:* None  
  - *Outputs:* Connects to `Create Search Term` node.  
  - *Edge Cases:* None typical; manual start prevents unintended runs.

- **Create Search Term**  
  - *Type:* Set  
  - *Role:* Defines the hashtag to search for Instagram posts.  
  - *Configuration:* Sets a string variable `Search_Term` with default value `"n8n"`; can be customized.  
  - *Key Expression:* `"n8n"` (modifiable to any hashtag string).  
  - *Inputs:* From Manual Trigger  
  - *Outputs:* To `Find Recent Posts`  
  - *Edge Cases:* Ensure the hashtag is valid and not empty to prevent API errors.

---

#### 1.2 Instagram Data Retrieval

**Overview:**  
This block calls the Apify Instagram Hashtag Scraper to fetch recent Instagram posts for the specified hashtag.

**Nodes Involved:**  
- Find Recent Posts

**Node Details:**

- **Find Recent Posts**  
  - *Type:* HTTP Request  
  - *Role:* Queries Apify's Instagram Hashtag Scraper API synchronously to retrieve recent posts.  
  - *Configuration:*  
    - Method: POST  
    - URL: `https://api.apify.com/v2/acts/apify~instagram-hashtag-scraper/run-sync-get-dataset-items`  
    - JSON Body includes:  
      - `hashtags`: array with the value of `Search_Term` from previous node  
      - `resultsLimit`: 20 (limits results to 20 posts)  
      - `resultsType`: `"posts"` (filters for posts)  
    - Authentication: HTTP Query Auth with API token passed as query parameter (credentials setup required)  
  - *Inputs:* From `Create Search Term`  
  - *Outputs:* To `Set bio and follower count`  
  - *Edge Cases:*  
    - API token invalid or expired → authentication error  
    - Network timeout or rate limiting by Apify  
    - Empty or invalid hashtag → no results  
  - *Notes:* Requires Apify account and API token configured in n8n credentials.

---

#### 1.3 Caption Extraction and Aggregation

**Overview:**  
Extracts captions from the retrieved posts, aggregates them into a list, and converts the list into a single text block for AI processing.

**Nodes Involved:**  
- Set bio and follower count  
- Aggregate  
- Convert table names and columns into single text for agent

**Node Details:**

- **Set bio and follower count**  
  - *Type:* Set  
  - *Role:* Extracts the `caption` field from each Instagram post JSON and assigns it to a new variable `caption`.  
  - *Configuration:* Sets `caption` to `={{ $json.caption }}`  
  - *Inputs:* From `Find Recent Posts`  
  - *Outputs:* To `Aggregate`  
  - *Edge Cases:* Missing or null captions in some posts → may produce empty strings.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Collects all `caption` fields into one aggregated array.  
  - *Configuration:* Aggregates the field `caption` from input items.  
  - *Inputs:* From `Set bio and follower count`  
  - *Outputs:* To `Convert table names and columns into single text for agent`  
  - *Edge Cases:* Empty input array leads to empty aggregation.

- **Convert table names and columns into single text for agent**  
  - *Type:* Code  
  - *Role:* Converts array of captions into a single formatted string for AI input.  
  - *Configuration:* JavaScript code concatenates captions, each preceded by a dash and stringified, separated by double newlines.  
  - *Inputs:* From `Aggregate`  
  - *Outputs:* To `AI Agent`  
  - *Edge Cases:* Unexpected caption content causing serialization errors.

---

#### 1.4 AI Caption Idea Generation

**Overview:**  
Sends the aggregated caption text to the AI agent with a system prompt to generate relevant Instagram caption ideas and identify common post themes.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Orchestrates AI prompt and response processing using the OpenAI model.  
  - *Configuration:*  
    - Text input: `"posts: {{ $json.text }}"` — passes aggregated captions.  
    - System message prompts AI to focus on the provided hashtag and recent posts, producing JSON with two arrays: `Post Idea` and `Most Common Post`.  
    - Output parser enabled to expect JSON response.  
  - *Inputs:* From `Convert table names and columns into single text for agent`  
  - *Outputs:* To `Split Out` and `Split Out1` after parsing  
  - *Edge Cases:*  
    - AI model timeout or rate limit errors  
    - AI output invalid or malformed JSON causing parsing failure

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Executes the GPT-4o-mini model call to OpenAI API.  
  - *Configuration:*  
    - Model: `gpt-4o-mini`  
    - Uses OpenAI credentials (API key must be configured in n8n).  
  - *Inputs:* From `AI Agent` via LangChain connection  
  - *Outputs:* To `Structured Output Parser`  
  - *Edge Cases:*  
    - Invalid API keys, quota exceeded  
    - Network errors

- **Structured Output Parser**  
  - *Type:* Output Parser (LangChain)  
  - *Role:* Parses AI textual JSON output into structured JSON objects.  
  - *Configuration:* Expects JSON with keys `"Post Idea"` and `"Most Common Post"`, each an array of strings.  
  - *Inputs:* From `OpenAI Chat Model`  
  - *Outputs:* To `AI Agent` node for further routing  
  - *Edge Cases:* Malformed AI output causing parse errors.

---

#### 1.5 Output Parsing and Formatting

**Overview:**  
Splits the parsed JSON arrays into individual items and merges them into a single output stream for downstream use or export.

**Nodes Involved:**  
- Split Out  
- Split Out1  
- Merge

**Node Details:**

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the `Post Idea` list into individual items.  
  - *Configuration:* Splits on path `output['Post Idea']`.  
  - *Inputs:* From `AI Agent`  
  - *Outputs:* To `Merge` (main output 0)  
  - *Edge Cases:* Empty or missing `Post Idea` field.

- **Split Out1**  
  - *Type:* Split Out  
  - *Role:* Splits the `Most Common Post` list into individual items.  
  - *Configuration:* Splits on path `output['Most Common Post']`.  
  - *Inputs:* From `AI Agent`  
  - *Outputs:* To `Merge` (main output 1)  
  - *Edge Cases:* Empty or missing `Most Common Post` field.

- **Merge**  
  - *Type:* Merge  
  - *Role:* Combines the two streams from split nodes into one unified output stream by position.  
  - *Configuration:* Mode: combine, combineBy: position  
  - *Inputs:* From `Split Out` (main 0) and `Split Out1` (main 1)  
  - *Outputs:* Final output of workflow  
  - *Edge Cases:* Mismatched list lengths produce incomplete pairs.

---

### 3. Summary Table

| Node Name                                     | Node Type                            | Functional Role                                 | Input Node(s)                      | Output Node(s)                            | Sticky Note                                                                                                                                                                                                                                                                             |
|-----------------------------------------------|------------------------------------|------------------------------------------------|----------------------------------|------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’              | Manual Trigger                     | Manual start of workflow                        | None                             | Create Search Term                       | Starts the workflow for testing or single-run execution.                                                                                                                                                                                                                                |
| Create Search Term                            | Set                                | Defines hashtag to search                       | When clicking ‘Execute workflow’ | Find Recent Posts                       | Sets the hashtag value (default `n8n`) to scan.                                                                                                                                                                                                                                        |
| Find Recent Posts                            | HTTP Request                      | Calls Apify Instagram Hashtag Scraper API      | Create Search Term               | Set bio and follower count             | Uses Apify Instagram Hashtag Scraper API; requires API token setup. More info: https://console.apify.com/actors/apify/instagram-hashtag-scraper                                                                                                                                          |
| Set bio and follower count                    | Set                                | Extracts caption from posts                     | Find Recent Posts               | Aggregate                             | Extracts the `caption` from each post for AI processing.                                                                                                                                                                                                                                |
| Aggregate                                    | Aggregate                         | Aggregates all captions into a list             | Set bio and follower count       | Convert table names and columns into single text for agent | Aggregates captions for combined AI input.                                                                                                                                                                                                                                             |
| Convert table names and columns into single text for agent | Code                              | Converts captions list into single string       | Aggregate                       | AI Agent                              | Formats captions as a single text block using JavaScript code.                                                                                                                                                                                                                        |
| AI Agent                                    | LangChain Agent                   | Sends captions to GPT-4o-mini to generate ideas | Convert table names and columns into single text for agent | Split Out, Split Out1                  | Prompts GPT-4o-mini with system message to generate caption ideas and common posts in JSON format.                                                                                                                                                                                      |
| OpenAI Chat Model                            | LangChain OpenAI Chat Model       | Executes GPT-4o-mini model call                 | AI Agent (ai_languageModel)     | Structured Output Parser              | Requires OpenAI API key configured in n8n credentials: https://platform.openai.com/account/api-keys                                                                                                                                                                                    |
| Structured Output Parser                      | LangChain Output Parser           | Parses AI JSON output into structured data      | OpenAI Chat Model (ai_outputParser) | AI Agent                             | Parses output JSON with keys `Post Idea` and `Most Common Post`.                                                                                                                                                                                                                         |
| Split Out                                    | Split Out                        | Splits `Post Idea` array into individual items | AI Agent                       | Merge                                | Splits the AI output list of caption ideas.                                                                                                                                                                                                                                            |
| Split Out1                                   | Split Out                        | Splits `Most Common Post` array into items     | AI Agent                       | Merge                                | Splits the AI output list of common posts.                                                                                                                                                                                                                                             |
| Merge                                        | Merge                            | Combines split outputs into one stream          | Split Out, Split Out1           | None (workflow output)                | Combines caption ideas and common posts into a single output stream.                                                                                                                                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add "Manual Trigger" node named `When clicking ‘Execute workflow’` to start workflow execution manually.

2. **Create Set Node for Hashtag**  
   - Add "Set" node named `Create Search Term`.  
   - In Assignments, create a string field `Search_Term` with default value `"n8n"` or your desired hashtag.

3. **Create HTTP Request Node to Apify API**  
   - Add "HTTP Request" node named `Find Recent Posts`.  
   - Set method to POST.  
   - URL: `https://api.apify.com/v2/acts/apify~instagram-hashtag-scraper/run-sync-get-dataset-items`  
   - Set Body Content Type to JSON.  
   - JSON Body:  
     ```json
     {
       "hashtags": ["{{$json.Search_Term}}"],
       "resultsLimit": 20,
       "resultsType": "posts"
     }
     ```  
   - Set authentication to "HTTP Query Auth".  
   - Create credentials with query parameter: `token=yourApifyToken` (replace `yourApifyToken` with your Apify API token).  
   - Connect input from `Create Search Term`.

4. **Create Set Node to Extract Captions**  
   - Add "Set" node named `Set bio and follower count`.  
   - Add a string field `caption` with value `={{ $json.caption }}` to extract caption from each post.  
   - Connect input from `Find Recent Posts`.

5. **Create Aggregate Node**  
   - Add "Aggregate" node named `Aggregate`.  
   - Configure to aggregate the field `caption`.  
   - Connect input from `Set bio and follower count`.

6. **Create Code Node to Combine Captions**  
   - Add "Code" node named `Convert table names and columns into single text for agent`.  
   - Insert this JavaScript code:  
     ```javascript
     return [
       {
         json: {
           text: items.map(item => `- ${JSON.stringify(item.json)}`).join('\n\n'),
         },
       },
     ];
     ```  
   - Connect input from `Aggregate`.

7. **Add LangChain AI Agent Node**  
   - Add `@n8n/n8n-nodes-langchain.agent` node named `AI Agent`.  
   - Set Text input to: `posts: {{ $json.text }}`.  
   - Define system message prompt:  
     ```
     I'm looking for ideas for posts about  {{ $('Create Search Term').item.json.Search_Term }}
     
     Here's the last 5 posts on instagram about the topic. Use those to help me generate a list of relevant captions to post on my instagram. 
     
     do not make up ideas that are not like the others in the list.
     
     output like this:
     {
       "Post Idea": ["Idea1", "Idea2"], 
       "Most Common Post": ["common post 1", "common post 2"]
     }
     ```  
   - Enable output parser.  
   - Connect input from `Convert table names and columns into single text for agent`.

8. **Add OpenAI Chat Model Node**  
   - Add `@n8n/n8n-nodes-langchain.lmChatOpenAi` node named `OpenAI Chat Model`.  
   - Select model `gpt-4o-mini`.  
   - Assign OpenAI credentials (must configure OpenAI API key in n8n credentials).  
   - Connect as AI language model input to `AI Agent`.

9. **Add Structured Output Parser Node**  
   - Add `@n8n/n8n-nodes-langchain.outputParserStructured` node named `Structured Output Parser`.  
   - Provide JSON schema example:  
     ```json
     {
       "Post Idea": ["Idea1", "Idea2"],
       "Most Common Post": ["common post 1", "common post 2"]
     }
     ```  
   - Connect AI output parser input from `OpenAI Chat Model`.  
   - Connect output back to `AI Agent`.

10. **Add Split Out Nodes for Output Lists**  
    - Add "Split Out" node named `Split Out`.  
    - Configure to split array at path `output['Post Idea']`.  
    - Connect input from `AI Agent` main output.

    - Add second "Split Out" node named `Split Out1`.  
    - Configure to split array at path `output['Most Common Post']`.  
    - Connect input from `AI Agent` main output.

11. **Add Merge Node to Combine Splits**  
    - Add "Merge" node named `Merge`.  
    - Set mode to `combine`.  
    - Set combineBy to `position`.  
    - Connect inputs from `Split Out` (main input 0) and `Split Out1` (main input 1).

12. **Finalize Workflow**  
    - Confirm all connections follow this order:  
      Manual Trigger → Set Hashtag → HTTP Request → Set Captions → Aggregate → Code → AI Agent (with OpenAI Chat Model and Output Parser) → Split Outs → Merge.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Apify Instagram Hashtag Scraper API requires API token setup. See: https://console.apify.com/actors/apify/instagram-hashtag-scraper                                                                                 | Apify Instagram Hashtag Scraper Documentation                                                         |
| OpenAI API key needed for GPT-4o-mini model access. Create and manage keys at: https://platform.openai.com/account/api-keys                                                                                   | OpenAI API Key Management                                                                              |
| Workflow authored by Robert Breen, Automation Consultant and AI Workflow Designer. Contact: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/                                | Author and Support                                                                                     |
| JavaScript code node concatenates JSON-stringified captions separated by double newlines to improve AI context comprehension.                                                                                  | Code snippet embedded in workflow                                                                      |
| Avoid generating AI captions unrelated to the hashtag by enforcing system prompt constraints.                                                                                                                 | System prompt technique for focused AI output                                                         |
| Apify API rate limits and network errors may cause workflow failures; implement retry or error handling as needed.                                                                                              | Integration risk and error consideration                                                              |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.