Generate Business Ideas & Social Media Content from Reddit with AI Analysis to Telegram

https://n8nworkflows.xyz/workflows/generate-business-ideas---social-media-content-from-reddit-with-ai-analysis-to-telegram-7744


# Generate Business Ideas & Social Media Content from Reddit with AI Analysis to Telegram

### 1. Workflow Overview

This workflow automates the generation of business ideas and social media content by mining relevant Reddit posts, analyzing them with AI models, and delivering curated educational and sales-oriented content to a Telegram channel. It targets entrepreneurs, AI automation experts, and social media strategists aiming to leverage Reddit discussions for thought leadership and marketing.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Reddit Data Collection:** Automatically triggers daily to fetch recent posts from targeted subreddits related to AI automation, n8n, and entrepreneurship.
- **1.2 Data Consolidation and Preparation:** Merges different Reddit search results and extracts focused text content for AI processing.
- **1.3 AI Classification and Analysis:** Uses AI language models to classify posts as questions or requests and conducts detailed analysis via custom AI agents, scoring relevancy and feasibility.
- **1.4 Content Formatting and Aggregation:** Prepares, edits, and merges AI outputs into structured summaries and social media content drafts.
- **1.5 Telegram Notification Delivery:** Formats the final content into a message suited for Telegram, splitting it if needed, and sends it to a specific Telegram chat.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Reddit Data Collection

**Overview:**  
This block triggers the workflow every day at noon and performs three parallel Reddit searches to collect recent posts relevant to AI automation, n8n, and AI business ideas.

**Nodes Involved:**  
- Daily Schedule  
- Search AI Automation  
- Search n8n Posts  
- Search AI Business  

**Node Details:**

- **Daily Schedule**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow at 12:00 PM daily  
  - Config: Triggers once per day at hour 12  
  - Input: None, start node  
  - Output: Three parallel connections to Reddit search nodes  
  - Edge cases: Missed trigger if n8n instance is down at scheduled time

- **Search AI Automation**  
  - Type: Reddit Node (search operation)  
  - Role: Searches subreddit "ArtificialInteligence" for posts matching keywords related to AI/workflow/business automation  
  - Config: Limit 5 posts, sorted by newest  
  - Credentials: Reddit OAuth2  
  - Input: From Daily Schedule  
  - Output: Posts data to Merge node  
  - Potential failures: Reddit API rate limits, OAuth token expiry, no search results

- **Search n8n Posts**  
  - Type: Reddit Node (get all posts)  
  - Role: Retrieves top 5 "hot" posts from subreddit "n8n"  
  - Credentials: Reddit OAuth2 (same as above)  
  - Input: From Daily Schedule  
  - Output: Posts data to Merge node  
  - Edge cases: No hot posts available, Reddit API issues

- **Search AI Business**  
  - Type: Reddit Node (search operation)  
  - Role: Searches "entrepreneur" subreddit for AI business-related posts (limit 5, sorted hot)  
  - Credentials: Reddit OAuth2  
  - Input: From Daily Schedule  
  - Output: Posts data to Merge node  
  - Edge cases: Similar to previous Reddit nodes

---

#### 2.2 Data Consolidation and Preparation

**Overview:**  
Merges the results from the three Reddit searches into a single stream and prepares the post content text ("selftext") for the AI agents.

**Nodes Involved:**  
- Merge  
- Text Classifier  
- Prepare Focused Data1  
- Prepare Focused Data  

**Node Details:**

- **Merge**  
  - Type: Merge Node  
  - Role: Combines three inputs (from the three Reddit searches) into one output list  
  - Config: Number of inputs = 3  
  - Input: From Search AI Automation (index 0), Search n8n Posts (index 1), Search AI Business (index 2)  
  - Output: Single merged output to Text Classifier

- **Text Classifier**  
  - Type: LangChain Text Classifier Node  
  - Role: Classifies each Reddit post's body text into categories: "Questions" or "Requests" based on presence of question words or action verbs  
  - Config: Uses the post's "selftext" as input text, with category definitions and examples  
  - Input: From Merge node  
  - Output: Routes outputs based on classification: output #1 for "Questions", output #2 for "Requests"  
  - On error: Continues without failure (error output enabled)  
  - Edge cases: Ambiguous texts, empty selftext, classification errors

- **Prepare Focused Data1**  
  - Type: Set Node  
  - Role: Extracts "selftext" from posts classified as "Questions"  
  - Input: Output #1 of Text Classifier  
  - Output: To AI Agent (question analyzer)

- **Prepare Focused Data**  
  - Type: Set Node  
  - Role: Extracts "selftext" from posts classified as "Requests"  
  - Input: Output #2 of Text Classifier  
  - Output: To AI Agent1 (request analyzer)

---

#### 2.3 AI Classification and Analysis

**Overview:**  
Two AI agents analyze the focused Reddit post texts differently: one summarizes and scores questions for educational content creation, the other analyzes requests for sales and feasibility insights.

**Nodes Involved:**  
- AI Agent (questions)  
- AI Agent1 (requests)  
- OpenRouter Chat Model  
- Structured Output Parser  

**Node Details:**

- **AI Agent**  
  - Type: LangChain Agent Node  
  - Role: Processes question posts to produce a summary, relevancy score, detail score, and social media content ideas tailored for educational thought leadership  
  - Config: System message sets persona as Femi Adedayo, Nigerian entrepreneur and AI automation expert; detailed instructions for analysis framework and output format  
  - Input: From Prepare Focused Data1  
  - Output: JSON with scores, summary, and content ideas  
  - Edge cases: Poorly formed question text, model API failures, response parsing errors

- **AI Agent1**  
  - Type: LangChain Agent Node  
  - Role: Processes request posts to produce solution summaries, relevancy and feasibility scores, and sales-focused social media content  
  - Config: Similar persona and task instructions focused on sales and feasibility analysis  
  - Input: From Prepare Focused Data  
  - Output: JSON with relevant sales content and metrics  
  - Edge cases: Complex requests beyond scope, model failures

- **OpenRouter Chat Model**  
  - Type: LangChain LM Chat Model Node  
  - Role: Provides the language model backend for AI Agent nodes and the Text Classifier node  
  - Credentials: OpenRouter API  
  - Input: Receives prompts from AI Agent, AI Agent1, and Text Classifier  
  - Output: AI-generated text responses

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI output into structured JSON format to extract scores, summaries, and content  
  - Input: From OpenRouter Chat Model outputs  
  - Output: Parsed JSON to AI Agent and AI Agent1 nodes

---

#### 2.4 Content Formatting and Aggregation

**Overview:**  
This block edits and merges the AI outputs into a unified data set, formats fields, and builds a comprehensive Telegram message containing all social media content strategies.

**Nodes Involved:**  
- Edit Fields  
- Edit Fields2  
- Edit Fields1  
- Merge1  
- Code  

**Node Details:**

- **Edit Fields**  
  - Type: Set Node  
  - Role: Transfers AI Agent's output fields (relevancy score, meaning, summary, social content) into main JSON keys for easier handling  
  - Input: From AI Agent  
  - Output: To Merge1 (input 0)

- **Edit Fields2**  
  - Type: Set Node  
  - Role: Transfers AI Agent1's output fields similarly  
  - Input: From AI Agent1  
  - Output: To Merge1 (input 1)

- **Merge1**  
  - Type: Merge Node  
  - Role: Combines the edited outputs from AI Agent and AI Agent1 into a single dataset  
  - Config: Mode "combine", includes unpaired entries, combines by position  
  - Input: From Edit Fields (#0) and Edit Fields2 (#1)  
  - Output: To Edit Fields1

- **Edit Fields1**  
  - Type: Set Node  
  - Role: Prepares final fields for output, ensuring consistent keys for Meaning, Summary, and Social Media Content  
  - Input: From Merge1  
  - Output: To Code node

- **Code**  
  - Type: Code Node (JavaScript)  
  - Role: Constructs one consolidated Telegram message aggregating all opportunities, their meaning, summaries, and formatted LinkedIn and Twitter posts; applies error handling for JSON parsing of social content  
  - Input: From Edit Fields1  
  - Output: Single message JSON with full text for Telegram  
  - Edge cases: Malformed AI output JSON, empty content arrays

---

#### 2.5 Telegram Notification Delivery

**Overview:**  
Finalizes message length to comply with Telegram’s character limit by splitting into parts if necessary, then sends the messages to a specific Telegram chat.

**Nodes Involved:**  
- Code1  
- Send to Telegram  

**Node Details:**

- **Code1**  
  - Type: Code Node (JavaScript)  
  - Role: Cleans message text aggressively to remove non-ASCII and markdown characters, splits message into chunks under Telegram's 4096 character limit (with buffer), adds part headers if multiple parts  
  - Input: From Code node  
  - Output: Multiple JSON messages ready for sending  
  - Edge cases: Overly large messages, splitting logic errors

- **Send to Telegram**  
  - Type: Telegram Node  
  - Role: Sends each prepared message chunk sequentially to Telegram chat ID 7107208579  
  - Credentials: Telegram API with OAuth2  
  - Input: From Code1  
  - Output: Telegram delivery confirmation  
  - Edge cases: Telegram API errors, chat ID invalid, message size over limit

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                          | Input Node(s)                        | Output Node(s)                  | Sticky Note                                              |
|---------------------|----------------------------------|----------------------------------------|------------------------------------|--------------------------------|----------------------------------------------------------|
| Daily Schedule      | Schedule Trigger                  | Triggers workflow daily at noon        | —                                  | Search AI Automation, Search n8n Posts, Search AI Business |                                                          |
| Search AI Automation| Reddit Node                      | Searches AI automation posts on Reddit | Daily Schedule                     | Merge                          |                                                          |
| Search n8n Posts    | Reddit Node                      | Retrieves hot posts from r/n8n          | Daily Schedule                     | Merge                          |                                                          |
| Search AI Business  | Reddit Node                      | Searches AI business posts on Reddit    | Daily Schedule                     | Merge                          |                                                          |
| Merge               | Merge Node                      | Merges three Reddit search outputs      | Search AI Automation, Search n8n Posts, Search AI Business | Text Classifier              |                                                          |
| Text Classifier     | LangChain Text Classifier        | Classifies posts as Questions or Requests | Merge                             | Prepare Focused Data1 (Questions), Prepare Focused Data (Requests) |                                                          |
| Prepare Focused Data1| Set Node                       | Extracts selftext from question posts   | Text Classifier (Questions)        | AI Agent                      |                                                          |
| Prepare Focused Data | Set Node                       | Extracts selftext from request posts    | Text Classifier (Requests)         | AI Agent1                     |                                                          |
| AI Agent            | LangChain Agent                 | Analyzes questions, generates educational content | Prepare Focused Data1            | Edit Fields                   |                                                          |
| AI Agent1           | LangChain Agent                 | Analyzes requests, generates sales content | Prepare Focused Data              | Edit Fields2                  |                                                          |
| OpenRouter Chat Model| LangChain LM Chat Model          | Provides AI language model backend      | AI Agent, AI Agent1, Text Classifier, Structured Output Parser | Structured Output Parser, AI Agent, AI Agent1, Text Classifier |                                                          |
| Structured Output Parser| LangChain Output Parser       | Parses AI model output to structured JSON | OpenRouter Chat Model            | AI Agent, AI Agent1           |                                                          |
| Edit Fields         | Set Node                       | Normalizes AI Agent output fields       | AI Agent                         | Merge1                       |                                                          |
| Edit Fields2        | Set Node                       | Normalizes AI Agent1 output fields      | AI Agent1                        | Merge1                       |                                                          |
| Merge1              | Merge Node                    | Combines outputs from both AI agents    | Edit Fields, Edit Fields2          | Edit Fields1                 |                                                          |
| Edit Fields1        | Set Node                       | Prepares final fields for message build | Merge1                          | Code                         |                                                          |
| Code                | Code Node (JavaScript)           | Builds consolidated Telegram message    | Edit Fields1                     | Code1                        |                                                          |
| Code1               | Code Node (JavaScript)           | Cleans and splits message for Telegram  | Code                            | Send to Telegram             |                                                          |
| Send to Telegram    | Telegram Node                   | Sends final messages to Telegram chat   | Code1                           | —                            | Chat ID: 7107208579; No attribution appended             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add **Schedule Trigger** node named "Daily Schedule"  
   - Configure to trigger once daily at 12:00 PM  

2. **Set Up Reddit Search Nodes:**  
   - Add **Reddit** node named "Search AI Automation"  
     - Operation: Search  
     - Subreddit: ArtificialInteligence  
     - Keyword: `AI automation OR workflow automation OR business automation`  
     - Limit: 5  
     - Sort: New  
     - Connect credentials for Reddit OAuth2  
     - Connect input from "Daily Schedule" output  

   - Add **Reddit** node named "Search n8n Posts"  
     - Operation: Get All (hot category)  
     - Subreddit: n8n  
     - Limit: 5  
     - Use same Reddit OAuth2 credentials  
     - Connect input from "Daily Schedule"  

   - Add **Reddit** node named "Search AI Business"  
     - Operation: Search  
     - Subreddit: entrepreneur  
     - Keyword: `AI business OR artificial intelligence business OR automation startup`  
     - Limit: 5  
     - Sort: Hot  
     - Use Reddit OAuth2 credentials  
     - Connect input from "Daily Schedule"  

3. **Merge Reddit Results:**  
   - Add **Merge** node named "Merge"  
   - Set number of inputs: 3  
   - Connect inputs from the three Reddit search nodes respectively  
   - Output to next node  

4. **Add Text Classification:**  
   - Add **LangChain Text Classifier** node named "Text Classifier"  
   - Configure categories:  
     - "Questions" with description and examples including question words  
     - "Requests" with description and examples with action verbs  
   - Input text: expression `={{ $json.selftext }}`  
   - Connect input from "Merge"  
   - Enable continue on error  
   - This node has multiple outputs:  
     - Output #1 (Questions)  
     - Output #2 (Requests)  

5. **Prepare Data for AI Agents:**  
   - Add **Set** node named "Prepare Focused Data1" for Questions  
     - Assign field "selftext" = `={{ $json.selftext }}`  
     - Connect input from Text Classifier output #1  
     - Output to AI Agent node  

   - Add **Set** node named "Prepare Focused Data" for Requests  
     - Assign field "selftext" = `={{ $json.selftext }}`  
     - Connect input from Text Classifier output #2  
     - Output to AI Agent1 node  

6. **Add AI Agents:**  
   - Add **LangChain Agent** node named "AI Agent" for question analysis  
     - Input text: `={{ $json.selftext }}`  
     - System message: Use provided persona and detailed instructions for question analysis, scoring, and educational content generation  
     - Connect input from "Prepare Focused Data1"  

   - Add **LangChain Agent** node named "AI Agent1" for request analysis  
     - Input text: `={{ $json.selftext }}`  
     - System message: Use provided persona and detailed instructions for request analysis, scoring, feasibility, and sales content  
     - Connect input from "Prepare Focused Data"  

7. **Configure OpenRouter Language Model:**  
   - Add **LangChain LM Chat Model** node named "OpenRouter Chat Model"  
   - Use OpenRouter API credentials  
   - Connect AI Agent, AI Agent1, Text Classifier, and Structured Output Parser nodes to this model node as their language model backend  

8. **Add Structured Output Parser:**  
   - Add **LangChain Structured Output Parser** node named "Structured Output Parser"  
   - Connect input from "OpenRouter Chat Model" outputs  
   - Connect output to AI Agent and AI Agent1 nodes for parsing AI responses  

9. **Edit and Normalize AI Outputs:**  
   - Add **Set** node "Edit Fields" for AI Agent output  
     - Map AI output JSON fields (relevance_score.score, meaning, summary, social_content) to clear fields  
     - Connect input from AI Agent output  

   - Add **Set** node "Edit Fields2" for AI Agent1 output  
     - Same mapping setup for request analysis output  
     - Connect input from AI Agent1 output  

10. **Merge AI Outputs:**  
    - Add **Merge** node "Merge1"  
    - Mode: Combine  
    - Include unpaired items: yes  
    - Combine by position  
    - Connect inputs from Edit Fields (0) and Edit Fields2 (1)  

11. **Final Field Preparation:**  
    - Add **Set** node "Edit Fields1"  
    - Assign final fields "Meaning", "Summary", "Social Media Content" from merged data  
    - Connect input from Merge1  

12. **Build Telegram Message:**  
    - Add **Code** node "Code"  
    - JavaScript code to aggregate all entries into a single formatted Telegram message, parse social media posts, and handle missing data gracefully  
    - Connect input from Edit Fields1  

13. **Split Large Message:**  
    - Add **Code** node "Code1"  
    - JavaScript code to clean message text (remove non-ASCII, markdown chars), split into parts ≤3800 chars with headers if multiple parts  
    - Connect input from "Code"  

14. **Send to Telegram:**  
    - Add **Telegram** node "Send to Telegram"  
    - Configure with Telegram API credentials (OAuth2)  
    - Chat ID: 7107208579  
    - Text parameter: `={{ $json.message }}`  
    - Disable attribution append  
    - Connect input from "Code1"  

15. **Set Workflow Settings:**  
    - Configure error workflow to handle failures gracefully  
    - Ensure all credentials are linked and tested: Reddit OAuth2, OpenRouter API, Telegram API  
    - Activate and test end-to-end  

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses AI agents modeled as Femi Adedayo, an AI automation entrepreneur persona for context and branding. | Enhances personalized content generation and authority positioning in social media posts.      |
| Reddit OAuth2 credentials must be set up with sufficient scopes for reading subreddit content.                   | See Reddit API OAuth2 documentation for setup: https://www.reddit.com/dev/api/oauth             |
| Telegram API integration requires chat ID and bot token with message sending permissions.                        | Telegram Bot API docs: https://core.telegram.org/bots/api                                       |
| OpenRouter API is used as the language model backend, ensure API key is valid and quota sufficient.              | OpenRouter platform info: https://openrouter.ai                                                  |
| The final Telegram message includes both LinkedIn and Twitter content ideas, formatted for easy copy-paste.     | Designed for rapid deployment of social media strategies by the user.                           |
| Error handling in AI nodes is set to continue on failure to maximize throughput despite single post issues.     | Prevents workflow halting due to single problematic Reddit post or AI response.                  |

---

This structured documentation enables technical users and automation agents to fully understand, reproduce, maintain, or customize the "Generate Business Ideas & Social Media Content from Reddit with AI Analysis to Telegram" workflow without needing to consult the raw JSON.