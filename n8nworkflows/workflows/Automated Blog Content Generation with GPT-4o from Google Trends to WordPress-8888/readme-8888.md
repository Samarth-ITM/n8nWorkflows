Automated Blog Content Generation with GPT-4o from Google Trends to WordPress

https://n8nworkflows.xyz/workflows/automated-blog-content-generation-with-gpt-4o-from-google-trends-to-wordpress-8888


# Automated Blog Content Generation with GPT-4o from Google Trends to WordPress

### 1. Workflow Overview

This workflow automates the generation and publication of SEO-friendly blog posts on WordPress based on trending topics identified via Google Trends data. It targets content creators, bloggers, and digital marketers seeking to streamline content creation around latest trends with minimal manual intervention.

The workflow comprises the following logical blocks:

- **1.1 Scheduled Trigger & Initialization:** Activates the workflow on a daily schedule and sets the target country for trend analysis.
- **1.2 Trend Retrieval & Filtering:** Fetches current trending topics from Google Trends and limits them to the top 3.
- **1.3 Detailed Search & Formatting:** Performs detailed Google searches on each trending topic via SerpApi, then formats the results into structured text.
- **1.4 AI Content Generation:** Uses GPT-4o via OpenAI to generate blog post content based on the formatted search data, enforcing SEO-friendly, natural language style without revealing sources.
- **1.5 Post Creation:** Publishes the generated content as draft posts to a WordPress site.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Initialization

**Overview:**  
This block schedules the workflow execution daily at a specific time and sets the country code parameter for localized trend data retrieval.

**Nodes Involved:**  
- Schedule Trigger  
- Set Target Country  

**Node Details:**  

- **Schedule Trigger**  
  - **Type:** Schedule Trigger  
  - **Role:** Initiates the workflow daily at 20:45 (8:45 PM) local time (Asia/Tehran timezone).  
  - **Configuration:** Runs once daily at hour 20, minute 45.  
  - **Connections:** Outputs to Set Target Country.  
  - **Edge Cases:** Misconfiguration of timezone or schedule could cause missed runs.

- **Set Target Country**  
  - **Type:** Set  
  - **Role:** Assigns a static "Country" field with value "IR" (Iran) for use in subsequent API calls.  
  - **Configuration:** Direct assignment of string "IR" to variable named `Country`.  
  - **Connections:** Outputs to Fetch Trending Topics.  
  - **Edge Cases:** Hardcoded country limits flexibility; user must update manually for other regions.

---

#### 1.2 Trend Retrieval & Filtering

**Overview:**  
Fetches real-time trending topics from Google Trends for the defined country and filters to top 3 trends for further processing.

**Nodes Involved:**  
- Fetch Trending Topics  
- Extract Trending Searches  
- Limit to Top 3 Trends  

**Node Details:**  

- **Fetch Trending Topics**  
  - **Type:** SerpApi Node (Google Trends Trending Now operation)  
  - **Role:** Retrieves current top trending searches localized by country code from the Set node.  
  - **Configuration:** Uses `geo` parameter from `Country` variable ("IR"). Uses SerpApi credentials.  
  - **Connections:** Outputs to Extract Trending Searches.  
  - **Edge Cases:** API quota limits, network errors, or invalid API key may cause failures.

- **Extract Trending Searches**  
  - **Type:** Code (JavaScript)  
  - **Role:** Extracts the array of trending search topics from SerpApi response JSON (`trending_searches`).  
  - **Configuration:** Returns `$input.first().json.trending_searches`.  
  - **Connections:** Outputs to Limit to Top 3 Trends.  
  - **Edge Cases:** If response structure changes or no trending searches returned, may fail or return empty.

- **Limit to Top 3 Trends**  
  - **Type:** Limit  
  - **Role:** Restricts the number of trending topics processed to a maximum of 3 for resource management.  
  - **Configuration:** `maxItems` set to 3.  
  - **Connections:** Outputs to Search Trend Details.  
  - **Edge Cases:** If fewer than 3 trends exist, all are passed through.

---

#### 1.3 Detailed Search & Formatting

**Overview:**  
Conducts detailed Google searches for each top trending topic and formats the search results into structured text summaries for AI input.

**Nodes Involved:**  
- Search Trend Details  
- Format Search Results  

**Node Details:**  

- **Search Trend Details**  
  - **Type:** SerpApi Node (Google Search operation)  
  - **Role:** Performs organic Google search queries for each trending topic (`q` parameter set to trending topic text).  
  - **Configuration:** Uses `q = {{ $json.query }}` from limited trends. Uses SerpApi credentials.  
  - **Connections:** Outputs to Format Search Results.  
  - **Edge Cases:** API rate limits, invalid queries, or network issues can cause errors.

- **Format Search Results**  
  - **Type:** Code (JavaScript)  
  - **Role:** Processes the search results JSON, extracts titles and snippets, and concatenates them into a readable text string for AI consumption.  
  - **Configuration:**  
    - Iterates over all input items.  
    - For each, constructs a string:  
      - Starts with "Google Search result for [query]"  
      - Lists results as "Result N" followed by title and snippet separated by "---".  
    - Stores the formatted text in `json.text`.  
  - **Connections:** Outputs to Generate Blog Content with AI.  
  - **Edge Cases:** Empty or malformed search results will yield incomplete text, potentially affecting AI output quality.

---

#### 1.4 AI Content Generation

**Overview:**  
Generates SEO-optimized blog content using GPT-4o based on the formatted search summaries.

**Nodes Involved:**  
- Generate Blog Content with AI  
- OpenAI Chat Model  
- Structured Output Parser  

**Node Details:**  

- **Generate Blog Content with AI**  
  - **Type:** Langchain Chain LLM node  
  - **Role:** Sends prompt and context to OpenAI GPT-4o to create a blog post based on search results text.  
  - **Configuration:**  
    - Prompt instructs the AI to act as an expert weblog writer, produce SEO-friendly content, avoid mentioning sources, and use info from the search results.  
    - Uses `{{ $json.text }}` as input content.  
    - Has output parser enabled.  
  - **Connections:**  
    - Sends text input to OpenAI Chat Model.  
    - Receives parsed structured output from Structured Output Parser.  
    - Outputs to Create a post.  
  - **Edge Cases:** API key issues, prompt failures, or output parsing errors can cause failures or incomplete blog posts.

- **OpenAI Chat Model**  
  - **Type:** Langchain Chat Model  
  - **Role:** Executes the GPT-4o model call to OpenAI using the input prompt generated by the previous node.  
  - **Configuration:**  
    - Model set to "chatgpt-4o-latest".  
    - Uses OpenAI API credentials.  
  - **Connections:**  
    - Input from Generate Blog Content with AI node.  
    - Output to Structured Output Parser.  
  - **Edge Cases:** API rate limits, auth failures, or network issues.

- **Structured Output Parser**  
  - **Type:** Langchain Output Parser Structured  
  - **Role:** Parses AI output to extract structured JSON containing `title` and `content` fields for the blog post.  
  - **Configuration:**  
    - Manual JSON schema specifying an object with properties: `title` (string), `content` (string).  
  - **Connections:**  
    - Output feeds back to Generate Blog Content with AI node's parser input.  
  - **Edge Cases:** If AI output deviates from schema, parsing errors may occur.

---

#### 1.5 Post Creation

**Overview:**  
Creates a draft blog post in WordPress using the AI-generated title and content.

**Nodes Involved:**  
- Create a post  

**Node Details:**  

- **Create a post**  
  - **Type:** WordPress node  
  - **Role:** Publishes the generated blog post as a draft on WordPress site.  
  - **Configuration:**  
    - Title set to `{{ $json.output.title }}` from AI output parser.  
    - Content set to `{{ $json.output.content }}`.  
    - Status set to `draft`.  
    - Uses WordPress API credentials.  
  - **Connections:** Receives input from Generate Blog Content with AI.  
  - **Edge Cases:** Authentication errors, WordPress API errors, or invalid content may prevent post creation.

---

### 3. Summary Table

| Node Name              | Node Type                              | Functional Role                       | Input Node(s)         | Output Node(s)              | Sticky Note                                                                                                            |
|------------------------|--------------------------------------|-------------------------------------|-----------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger                     | Starts the workflow on schedule     | —                     | Set Target Country            |                                                                                                                        |
| Set Target Country     | Set                                 | Sets country code to "IR"            | Schedule Trigger       | Fetch Trending Topics        |                                                                                                                        |
| Fetch Trending Topics  | SerpApi (Google Trends)              | Fetches trending search topics      | Set Target Country     | Extract Trending Searches    |                                                                                                                        |
| Extract Trending Searches | Code (JavaScript)                 | Extracts trending searches array    | Fetch Trending Topics  | Limit to Top 3 Trends        |                                                                                                                        |
| Limit to Top 3 Trends  | Limit                               | Limits trending topics to 3          | Extract Trending Searches | Search Trend Details       |                                                                                                                        |
| Search Trend Details   | SerpApi (Google Search)              | Performs detailed Google searches   | Limit to Top 3 Trends  | Format Search Results        | "This node performs detailed Google searches for each trending topic to gather comprehensive information"             |
| Format Search Results  | Code (JavaScript)                   | Formats search results into text    | Search Trend Details   | Generate Blog Content with AI | "This code node processes and formats the Google search results into readable text for the AI:\n\nCombines all search results into a structured text format\nCreates a summary that includes: search query, result titles, and snippets\nFormats the data as: \"Google Search result for [topic]\\nResult 1\\n[title]\\n[snippet]\\n---\"" |
| Generate Blog Content with AI | Langchain Chain LLM            | Generates SEO-friendly blog content | Format Search Results  | Create a post               |                                                                                                                        |
| OpenAI Chat Model      | Langchain Chat Model                | Calls GPT-4o model                  | Generate Blog Content with AI | Structured Output Parser |                                                                                                                        |
| Structured Output Parser | Langchain Output Parser Structured | Parses AI output to structured JSON | OpenAI Chat Model      | Generate Blog Content with AI (parser input) |                                                                                                                        |
| Create a post          | WordPress                          | Creates draft post on WordPress     | Generate Blog Content with AI | —                           |                                                                                                                        |
| Sticky Note            | Sticky Note                        | Documentation and setup instructions | —                     | —                           | "# Automated Blog Content Generation from Google Trends to WordPress\n\nThis n8n workflow automatically generates SEO-friendly blog content based on trending topics from Google Trends and publishes it to WordPress. Perfect for content creators, bloggers, and digital marketers who want to stay on top of trending topics with minimal manual effort.\n\nThe AI is specifically prompted to create engaging, SEO-optimized content without revealing the automated sources, ensuring natural-sounding blog posts.\n\n## How to set up\n\n1. **Install required community nodes**:\n   - `n8n-nodes-serpapi` for Google Trends and Search functionality\n\n2. **Configure credentials**:\n   - **SerpApi**: Sign up at serpapi.com and add your API key\n   - **OpenAI**: Add your OpenAI API key for GPT-4o access\n   - **WordPress**: Configure your WordPress site credentials\n\n3. **Customize the country code**: Change the \"Country\" field in the \"Edit Fields\" node (currently set to \"IR\" for Iran)\n\n4. **Adjust the schedule**: Modify the \"Schedule Trigger\" to run at your preferred time\n\n5. **Test the workflow**: Run it manually first to ensure all connections work properly\n\n## Requirements\n\n- **SerpApi account** (for Google Trends and Search data)\n- **OpenAI API access** (for content generation using GPT-4o)\n- **WordPress site** with API access enabled" |
| Sticky Note1           | Sticky Note                        | Explains detailed search node       | —                     | —                           | "This node performs detailed Google searches for each trending topic to gather comprehensive information"             |
| Sticky Note2           | Sticky Note                        | Explains search results formatting  | —                     | —                           | "This code node processes and formats the Google search results into readable text for the AI:\n\nCombines all search results into a structured text format\nCreates a summary that includes: search query, result titles, and snippets\nFormats the data as: \"Google Search result for [topic]\\nResult 1\\n[title]\\n[snippet]\\n---\"" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run daily at 20:45 (Asia/Tehran timezone)  
   - No credentials needed  
   - Connect output to Set Target Country node  

2. **Create Set node named "Set Target Country"**  
   - Assign a string variable named `Country` with value `"IR"`  
   - Connect output to Fetch Trending Topics node  

3. **Create SerpApi node named "Fetch Trending Topics"**  
   - Operation: `google_trends_trending_now`  
   - Set `geo` parameter to `={{ $json.Country }}`  
   - Configure SerpApi credentials with valid API key  
   - Connect output to Extract Trending Searches node  

4. **Create Code node named "Extract Trending Searches"**  
   - Language: JavaScript  
   - Code: `return $input.first().json.trending_searches;`  
   - Connect output to Limit to Top 3 Trends node  

5. **Create Limit node named "Limit to Top 3 Trends"**  
   - Set maxItems to 3  
   - Connect output to Search Trend Details node  

6. **Create SerpApi node named "Search Trend Details"**  
   - Operation: Google Search  
   - Set `q` parameter to `={{ $json.query }}` (the trending topic text)  
   - Configure SerpApi credentials  
   - Connect output to Format Search Results node  

7. **Create Code node named "Format Search Results"**  
   - Language: JavaScript  
   - Code:  
   ```javascript
   for (const item of $input.all()) {
     let text = `Google Search result for ${item.json.search_parameters.q}\n`;
     let i = 0;
     for (const result of item.json.organic_results) {
       i++;
       text += `Result ${i}\n${result.title}\n${result.snippet}\n---`;
     }
     item.json.text = text;
   }
   return $input.all();
   ```  
   - Connect output to Generate Blog Content with AI node  

8. **Create Langchain Chain LLM node named "Generate Blog Content with AI"**  
   - Set prompt text:  
     ```
     You are expert weblog writer.
     Write a post about this content.
     The information gathered from Google Trends and Google Search about a Trending topic.
     Don't mention the source of the information and the goal is to generate seo friendly content.
     Use the fact that exist in the search result.
     {{ $json.text }}
     ```  
   - Enable output parser  
   - Connect output to OpenAI Chat Model and Structured Output Parser nodes (see below)  

9. **Create Langchain Chat Model node named "OpenAI Chat Model"**  
   - Model: `chatgpt-4o-latest`  
   - Configure OpenAI credentials with valid API key  
   - Connect output to Structured Output Parser node  

10. **Create Langchain Output Parser Structured node named "Structured Output Parser"**  
    - Schema type: Manual JSON schema with properties:  
      ```json
      {
        "type": "object",
        "properties": {
          "title": { "type": "string" },
          "content": { "type": "string" }
        }
      }
      ```  
    - Connect output back to Generate Blog Content with AI node's parser input  

11. **Connect Generate Blog Content with AI node's main output to WordPress node**  

12. **Create WordPress node named "Create a post"**  
    - Set post title to `={{ $json.output.title }}`  
    - Set post content to `={{ $json.output.content }}`  
    - Set status to `draft`  
    - Configure WordPress API credentials with proper site access  
    - No outputs needed (end node)  

13. **Optional: Create Sticky Note nodes as documentation aids in the workflow canvas**  

14. **Test workflow manually before enabling schedule**  
    - Ensure all API credentials are valid  
    - Check that country code is correctly set  
    - Confirm that WordPress post creation works as expected  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow depends on community nodes `n8n-nodes-serpapi` for Google Trends and Search integration, and Langchain nodes for AI interaction.                                                                                                                                                                                                                                                                                                                                                                         | Setup instruction in Sticky Note                                                                |
| Requires valid API keys for SerpApi (https://serpapi.com/), OpenAI (https://openai.com/), and WordPress with API access enabled.                                                                                                                                                                                                                                                                                                                                                                                       | Setup instruction in Sticky Note                                                                |
| The AI prompt is designed to produce natural, SEO-friendly blog posts without referencing automated data sources, ensuring content quality and authenticity.                                                                                                                                                                                                                                                                                                                                                            | Workflow overview and prompt text explanation                                                    |
| Scheduled execution timezone is Asia/Tehran; adjust if deploying in other regions.                                                                                                                                                                                                                                                                                                                                                                                                                                       | Schedule Trigger node configuration                                                             |
| For modifications: adjust country code in the "Set Target Country" node, change max items in "Limit to Top 3 Trends", or customize AI prompt to tailor content style.                                                                                                                                                                                                                                                                                                                                                     | General customization advice                                                                    |
| Useful links:  
- SerpApi documentation: https://serpapi.com/  
- OpenAI API docs: https://platform.openai.com/docs/  
- WordPress REST API docs: https://developer.wordpress.org/rest-api/                                                                                                                                                                                                                                                                                                                                                                                     | External resources for API integrations                                                        |

---

**Disclaimer:**  
The text documented here is derived solely from an automated n8n workflow and complies with all current content policies. No illegal, offensive, or protected content is included. All processed data is legal and publicly accessible.