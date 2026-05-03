Find Instagram Collaboration Leads using Apify Scraping and GPT-4o Evaluation

https://n8nworkflows.xyz/workflows/find-instagram-collaboration-leads-using-apify-scraping-and-gpt-4o-evaluation-7019


# Find Instagram Collaboration Leads using Apify Scraping and GPT-4o Evaluation

### 1. Workflow Overview

This workflow automates the process of finding potential Instagram collaboration leads by scraping Instagram posts with a specific hashtag, extracting profile data, and evaluating leads using AI. It targets users who want to identify Instagram influencers or creators fitting specific collaboration criteria, especially around follower count and profile content.

The workflow is logically divided into these blocks:

- **1.1 Manual Trigger & Hashtag Setup:** Start the workflow manually and define the Instagram hashtag to search.
- **1.2 Instagram Hashtag Scraping:** Use Apify’s Instagram Hashtag Scraper to get recent posts tagged with the chosen hashtag.
- **1.3 Profile Scraping:** For each post’s owner, scrape detailed Instagram profile data using Apify’s Instagram Profile Scraper.
- **1.4 Data Extraction:** Extract relevant fields (bio and follower count) from the profile data into clean variables.
- **1.5 AI Lead Evaluation:** Pass the extracted data to a GPT-4o-based AI agent that scores the lead as “Yes” or “No” for collaboration potential, providing reasoning.
- **1.6 Structured Output Parsing:** Parse the AI output into a structured JSON format for further automation or reporting.

---

### 2. Block-by-Block Analysis

#### 1.1 Manual Trigger & Hashtag Setup

- **Overview:** Allows manual execution of the workflow and sets the Instagram hashtag for scraping.
- **Nodes Involved:** `When clicking ‘Execute workflow’`, `Create Search Term`

##### Node: When clicking ‘Execute workflow’

- **Type:** Manual Trigger  
- **Role:** Entry point for workflow execution; enables manual start during testing or on-demand runs.  
- **Config:** No parameters; default manual trigger node.  
- **Input/Output:** No input; outputs trigger to `Create Search Term`.  
- **Edge cases:** None; manual triggers are robust by nature.

##### Node: Create Search Term

- **Type:** Set  
- **Role:** Defines the Instagram hashtag to search for.  
- **Config:** Sets variable `Search_Term` with default value `"n8n"` (editable to any hashtag).  
- **Input:** Trigger from manual node.  
- **Output:** JSON with `Search_Term` to `Find Recent Posts`.  
- **Edge cases:** User must ensure valid hashtag string; empty or invalid hashtags will return no posts.

#### 1.2 Instagram Hashtag Scraping

- **Overview:** Retrieves recent Instagram posts containing the specified hashtag using Apify’s Instagram Hashtag Scraper API.
- **Nodes Involved:** `Find Recent Posts`

##### Node: Find Recent Posts

- **Type:** HTTP Request  
- **Role:** Calls Apify API for hashtag scraping to fetch recent posts.  
- **Config:**  
  - Method: POST  
  - URL: `https://api.apify.com/v2/acts/apify~instagram-hashtag-scraper/run-sync-get-dataset-items`  
  - JSON Body includes:  
    - `hashtags`: list with `Search_Term`  
    - `resultsLimit`: 5 (limits to 5 posts for efficiency)  
    - `resultsType`: "posts"  
  - Authentication: HTTP Query Auth (token in URL query param)  
- **Input:** JSON with `Search_Term` from `Create Search Term`.  
- **Output:** JSON array of posts to `Scrape Accounts`.  
- **Edge cases:**  
  - API rate limits or token invalidity cause HTTP errors.  
  - No posts found if hashtag is invalid or unpopular.  
- **Credentials:** Requires Apify token configured as HTTP Query Auth credential.

#### 1.3 Profile Scraping

- **Overview:** For each Instagram post owner username, scrape detailed profile data using Apify’s Instagram Profile Scraper API.
- **Nodes Involved:** `Scrape Accounts`

##### Node: Scrape Accounts

- **Type:** HTTP Request  
- **Role:** Calls Apify API with usernames to get profile details.  
- **Config:**  
  - Method: POST  
  - URL: `https://api.apify.com/v2/acts/apify~instagram-profile-scraper/run-sync-get-dataset-items`  
  - JSON Body includes:  
    - `usernames`: list with `ownerUsername` from posts  
  - Authentication: Same HTTP Query Auth as `Find Recent Posts`  
- **Input:** Post owner usernames from `Find Recent Posts`.  
- **Output:** Profile JSON data to `Set bio and follower count`.  
- **Edge cases:**  
  - API token or request failures.  
  - Private or unavailable profiles return limited/no data.

#### 1.4 Data Extraction

- **Overview:** Extracts clean and relevant data fields (biography and follower count) from the profile JSON for AI input.
- **Nodes Involved:** `Set bio and follower count`

##### Node: Set bio and follower count

- **Type:** Set  
- **Role:** Maps `biography` and `followersCount` from profile JSON to variables `bio` and `followersCount`.  
- **Config:**  
  - `bio` = `{{ $json.biography }}`  
  - `followersCount` = `{{ $json.followersCount }}` (number type)  
- **Input:** Profile JSON from `Scrape Accounts`.  
- **Output:** Clean JSON with `bio` and `followersCount` to `AI Agent`.  
- **Edge cases:** Missing or malformed fields can cause empty or zero values.

#### 1.5 AI Lead Evaluation

- **Overview:** Uses GPT-4o-mini to analyze bio and follower count, deciding if the lead is suitable for collaboration.
- **Nodes Involved:** `AI Agent`, `OpenAI Chat Model`, `Structured Output Parser`

##### Node: AI Agent

- **Type:** LangChain Agent  
- **Role:** Coordinates AI prompt creation and parses AI responses.  
- **Config:**  
  - Text prompt template: `"followers: {{ $json.followersCount }} bio: {{ $json.bio }}"`  
  - System message instructs the AI to output JSON with fields:  
    - `lead status`: "Yes" or "No"  
    - `Reasoning`: explanation for decision  
  - Requires output parser (linked to `Structured Output Parser`)  
- **Input:** JSON with `bio` and `followersCount` from `Set bio and follower count`.  
- **Output:** AI response passed through parser to structured JSON.  
- **Edge cases:**  
  - AI service downtime or quota exceeded.  
  - Unexpected AI output format (mitigated via structured output parser).  
- **Sub-workflow:** None.

##### Node: OpenAI Chat Model

- **Type:** LangChain OpenAI Chat Model  
- **Role:** Executes the GPT-4o-mini model call.  
- **Config:**  
  - Model: `gpt-4o-mini` (latest GPT-4 optimized variant)  
  - Credentials: OpenAI API key (configured via n8n OpenAI API credential)  
- **Input:** Prompt from `AI Agent`.  
- **Output:** Raw AI response to `AI Agent`.  
- **Edge cases:** API key invalidity or rate limits.

##### Node: Structured Output Parser

- **Type:** LangChain Structured Output Parser  
- **Role:** Parses AI text response into JSON object with `lead status` and `Reasoning`.  
- **Config:** JSON schema example provided to enforce expected output shape.  
- **Input:** Raw AI response from `AI Agent`.  
- **Output:** Clean JSON passed back to `AI Agent`.  
- **Edge cases:** Parsing errors if AI output deviates from schema.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                        | Input Node(s)                | Output Node(s)              | Sticky Note                                                                                              |
|------------------------|----------------------------------|-------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual start of workflow             | None                        | Create Search Term           | Allows you to run the workflow manually while testing.                                                  |
| Create Search Term      | Set                              | Defines Instagram hashtag to search | When clicking ‘Execute workflow’ | Find Recent Posts           | Sets `"n8n"` as default hashtag; editable to any hashtag.                                              |
| Find Recent Posts       | HTTP Request                     | Scrapes recent Instagram posts by hashtag | Create Search Term          | Scrape Accounts              | Uses Apify Instagram Hashtag Scraper API; requires HTTP Query Auth credential with Apify token.        |
| Scrape Accounts         | HTTP Request                     | Scrapes Instagram profiles for usernames | Find Recent Posts           | Set bio and follower count   | Uses Apify Instagram Profile Scraper API; shares same credential as previous node.                      |
| Set bio and follower count | Set                             | Extracts bio and follower count fields | Scrape Accounts             | AI Agent                    | Extracts `biography` and `followersCount` from profile JSON for AI input.                              |
| AI Agent               | LangChain Agent                  | Evaluates lead suitability using AI | Set bio and follower count  | None (internal to AI Agent)  | Uses GPT-4o-mini; outputs JSON with lead status and reasoning.                                         |
| OpenAI Chat Model       | LangChain OpenAI Chat Model      | Executes GPT-4o-mini model           | AI Agent (ai_languageModel) | AI Agent (ai_languageModel)  | Requires OpenAI API credentials; uses GPT-4o-mini model.                                               |
| Structured Output Parser | LangChain Structured Output Parser | Parses AI JSON response              | AI Agent (ai_outputParser)  | AI Agent (ai_outputParser)   | Parses AI response into structured JSON with lead status and reasoning.                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add `Manual Trigger` node named `When clicking ‘Execute workflow’`.  
   - No configuration needed.

2. **Create Set Node for Hashtag**  
   - Add `Set` node named `Create Search Term`.  
   - Add a string field `Search_Term` with default value `"n8n"`.  
   - Connect output of manual trigger to this node.

3. **Create HTTP Request Node for Hashtag Scraping**  
   - Add `HTTP Request` node named `Find Recent Posts`.  
   - Set method to POST.  
   - Set URL to: `https://api.apify.com/v2/acts/apify~instagram-hashtag-scraper/run-sync-get-dataset-items`  
   - Under Body Parameters, select JSON and enter:  
     ```json
     {
       "hashtags": ["{{$json["Search_Term"]}}"],
       "resultsLimit": 5,
       "resultsType": "posts"
     }
     ```  
   - Enable "Send Body" as JSON.  
   - Set Authentication to HTTP Query Auth type.  
   - Create and select HTTP Query Auth credential with Apify token as query param named `token`.  
   - Connect `Create Search Term` output to this node.

4. **Create HTTP Request Node for Profile Scraping**  
   - Add `HTTP Request` node named `Scrape Accounts`.  
   - Set method to POST.  
   - Set URL to: `https://api.apify.com/v2/acts/apify~instagram-profile-scraper/run-sync-get-dataset-items`  
   - Set JSON Body:  
     ```json
     {
       "usernames": ["{{$json["ownerUsername"]}}"]
     }
     ```  
   - Enable "Send Body" as JSON.  
   - Use same HTTP Query Auth credential as previous node.  
   - Connect output of `Find Recent Posts` to this node.

5. **Create Set Node for Bio and Followers**  
   - Add `Set` node named `Set bio and follower count`.  
   - Add string field `bio` with value `{{$json["biography"]}}`.  
   - Add number field `followersCount` with value `{{$json["followersCount"]}}`.  
   - Connect output of `Scrape Accounts` to this node.

6. **Create LangChain OpenAI Chat Model Node**  
   - Add node named `OpenAI Chat Model` of type `@n8n/n8n-nodes-langchain.lmChatOpenAi`.  
   - Select model `gpt-4o-mini`.  
   - Create and assign OpenAI API credential with your API key.  
   - No other options needed.  

7. **Create LangChain Structured Output Parser Node**  
   - Add node named `Structured Output Parser` of type `@n8n/n8n-nodes-langchain.outputParserStructured`.  
   - Paste JSON example schema:  
     ```json
     {
       "lead status": "Yes or No",
       "Reasoning": "reasoning for why they are a good or bad lead."
     }
     ```

8. **Create LangChain Agent Node**  
   - Add node named `AI Agent` of type `@n8n/n8n-nodes-langchain.agent`.  
   - Set prompt text to:  
     ```
     followers: {{$json["followersCount"]}} bio: {{$json["bio"]}}
     ```  
   - In options, enter system message:  
     ```
     You are a helpful assistant. I'm lookig for n8n creators to collaborate with. read their bio, and output yes or no if they are a good person for me to reach out to in order to collaborate on a project. I need the follower count to be 2k at leaset. 

     output like this. 

     {
       "lead status": "Yes or No",
       "Reasoning": "reasoning for why they are a good or bad lead. "
     }
     ```  
   - Set prompt type to `define`.  
   - Enable Use Output Parser.  
   - Connect `OpenAI Chat Model` node as AI Language Model input.  
   - Connect `Structured Output Parser` node as AI Output Parser input.  
   - Connect output of `Set bio and follower count` to `AI Agent`.

9. **Configure Connections:**  
   - `When clicking ‘Execute workflow’` → `Create Search Term`  
   - `Create Search Term` → `Find Recent Posts`  
   - `Find Recent Posts` → `Scrape Accounts`  
   - `Scrape Accounts` → `Set bio and follower count`  
   - `Set bio and follower count` → `AI Agent`  
   - `OpenAI Chat Model` → `AI Agent` (AI Language Model input)  
   - `Structured Output Parser` → `AI Agent` (AI Output Parser input)

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Apify Instagram Hashtag Scraper API documentation and usage details                              | https://console.apify.com/actors/apify/instagram-hashtag-scraper                                          |
| Apify Instagram Profile Scraper API documentation and usage details                              | https://console.apify.com/actors/apify/instagram-profile-scraper                                          |
| Instructions for creating Apify API token and setting up HTTP Query Auth credential in n8n      | See Sticky Note in workflow; create token in Apify Console and configure token as HTTP Query Auth in n8n  |
| OpenAI API key creation and credential setup instructions                                       | https://platform.openai.com/account/api-keys                                                             |
| Contact for workflow assistance: Robert Breen, Automation Consultant & n8n Expert               | Email: robert@ynteractive.com, LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/               |

---

This completes the full structured reference for the "Find Instagram Collaboration Leads using Apify Scraping and GPT-4o Evaluation" workflow.  
The documentation enables advanced users to understand, replicate, and modify the workflow effectively while anticipating integration and operational issues.