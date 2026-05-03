Generate Startup Ideas from Reddit Posts Using Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/generate-startup-ideas-from-reddit-posts-using-gemini-ai-and-google-sheets-6094


# Generate Startup Ideas from Reddit Posts Using Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow, titled **"Startup Idea Finder"**, is designed to automate the generation of startup ideas by mining Reddit posts from relevant subreddits, analyzing the content using AI models (Google Gemini via n8n’s LangChain integration), and recording the output into a Google Sheet for further review or tracking.

**Target Use Cases:**  
- Entrepreneurs and innovators looking for inspiration from online community discussions.  
- Market researchers exploring unmet needs and pain points expressed in subreddit communities.  
- AI-assisted ideation by synthesizing raw user-generated content into actionable startup concepts.

**Logical Blocks:**

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Reddit Data Collection**: Searches three specific subreddits for posts matching a keyword.
- **1.3 Data Preparation**: Standardizes and selects key fields from Reddit posts.
- **1.4 AI Processing Block**: Uses Google Gemini (PaLM) and an AI Agent node to analyze posts and generate startup ideas.
- **1.5 Data Aggregation & Output**: Merges AI-generated insights and appends results into a Google Sheet for storage.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow manually via n8n’s manual trigger node.
- **Nodes Involved:**  
  - *When clicking ‘Execute workflow’*
- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Configuration:** Default (no parameters)  
  - **Inputs:** None (starts the workflow)  
  - **Outputs:** Connects to three Reddit search nodes in parallel  
  - **Edge Cases:** None typical; user must manually execute  
  - **Version:** v1

---

#### 1.2 Reddit Data Collection

- **Overview:** Searches three subreddits ("crazyideas", "Lightbulb", "sidehustle") for posts containing the phrase "Why is there no tool that", sorted differently per subreddit.
- **Nodes Involved:**  
  - *Search for a post* (r/crazyideas)  
  - *Search for a post1* (r/Lightbulb)  
  - *Search for a post2* (r/sidehustle)  
  - *Merge1* (merges outputs from the three Reddit nodes)  
- **Node Details:**  

  - **Search for a post**, **Search for a post1**, **Search for a post2**  
    - **Type:** Reddit node (OAuth2 authenticated)  
    - **Operation:** Search posts by keyword "Why is there no tool that"  
    - **Limit:** 25 posts per subreddit  
    - **Subreddit & Sort:**  
      - *crazyideas*: sorted by number of comments  
      - *Lightbulb*: sorted by number of comments  
      - *sidehustle*: sorted by top posts  
    - **Credentials:** Reddit OAuth2  
    - **Outputs:** Each outputs a list of post JSON objects  
    - **Edge Cases:**  
      - Reddit API limits, rate limiting, OAuth token expiration or invalid credentials.  
      - Empty search results if no matching posts.  
      - Network timeouts or API failures.

  - **Merge1**  
    - **Type:** Merge node  
    - **Mode:** Combine three inputs into one consolidated output (numberInputs=3)  
    - **Purpose:** Aggregates posts from all three subreddit searches into one stream.  
    - **Inputs:** From the three Reddit nodes  
    - **Outputs:** To the *Edit Fields* node  
    - **Edge Cases:** If any input is missing or empty, merge still proceeds; downstream nodes must handle empty data gracefully.

---

#### 1.3 Data Preparation

- **Overview:** Selects and normalizes key fields from each Reddit post for further AI processing.
- **Nodes Involved:**  
  - *Edit Fields*  
- **Node Details:**  
  - **Type:** Set node  
  - **Purpose:** Extracts and renames fields: title, selftext, ups (upvotes), created (timestamp), url  
  - **Configuration:** Assigns values directly from incoming JSON using expressions such as `={{ $json.title }}`  
  - **Inputs:** From *Merge1* (combined Reddit posts)  
  - **Outputs:** To *AI Agent* and *Merge* (for merging later with AI output)  
  - **Edge Cases:** Missing or null fields in Reddit data could result in empty values; expressions depend on consistent Reddit API response structure.

---

#### 1.4 AI Processing Block

- **Overview:** Processes each Reddit post using AI to extract the core problem, check for existing solutions, and propose startup ideas with implementation insights.
- **Nodes Involved:**  
  - *AI Agent* (LangChain agent node)  
  - *Google Gemini Chat Model* (Google PaLM Gemini 1.5 chat model)  
  - *Merge* (combines edited fields and AI-generated output)  
- **Node Details:**  

  - **Google Gemini Chat Model**  
    - **Type:** LangChain Google Gemini Chat model node  
    - **Model:** "models/gemini-1.5-flash-8b-latest"  
    - **Credentials:** Google PaLM API  
    - **Inputs:** Receives text prompt from *AI Agent* node  
    - **Outputs:** AI-generated text responses  
    - **Edge Cases:**  
      - API quota limits, authentication failure, network errors  
      - Model latency or unavailability  
      - Unexpected output format requiring error handling

  - **AI Agent**  
    - **Type:** LangChain agent node  
    - **Prompt:** Structured prompt combining Reddit post details (title, selftext, upvotes, URL, created date) with instructions to:  
      1. Explain core problem/frustration  
      2. Identify existing solutions  
      3. Propose a startup idea if none exist  
      4. Define user benefits and efficiency gains  
      5. Provide an implementation overview  
    - **Inputs:** From *Edit Fields*  
    - **Outputs:** To *Merge* node  
    - **Configuration:** Uses the Google Gemini Chat Model as the language model backend  
    - **Edge Cases:**  
      - Expression failures in prompt if input data missing or malformed  
      - AI output may be inconsistent or incomplete; downstream handling needed

  - **Merge**  
    - **Type:** Merge node  
    - **Mode:** Combine inputs by position (combining the original post data and AI output)  
    - **Inputs:**  
      - Position 0: *Edit Fields* output (post data)  
      - Position 1: *AI Agent* output (AI insights)  
    - **Outputs:** To Google Sheets appending node  
    - **Edge Cases:** Missing data from either input will affect the final merged output’s completeness.

---

#### 1.5 Data Aggregation & Output

- **Overview:** Appends the combined Reddit post data and AI-generated insights into a specified Google Sheet for record keeping.
- **Nodes Involved:**  
  - *Append row in sheet*  
- **Node Details:**  
  - **Type:** Google Sheets node  
  - **Operation:** Append a row to a specified sheet and Google Sheets document  
  - **Configuration:**  
    - Sheet name and document ID are parameterized and require user input  
    - OAuth2 credentials for Google Sheets API  
  - **Inputs:** From *Merge* node  
  - **Outputs:** None (terminal node)  
  - **Edge Cases:**  
    - Invalid or missing Google Sheets document ID or sheet name  
    - Credential expiry or permission issues  
    - API rate limits or quota exceeded  
    - Data format mismatch causing append failure

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                    | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                                  |
|----------------------------|----------------------------|----------------------------------|-----------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger             | Start workflow manually           | None                        | Search for a post, Search for a post1, Search for a post2 |                                                                                                                              |
| Search for a post           | Reddit                     | Search posts in r/crazyideas      | When clicking ‘Execute workflow’ | Merge1                   |                                                                                                                              |
| Search for a post1          | Reddit                     | Search posts in r/Lightbulb        | When clicking ‘Execute workflow’ | Merge1                   |                                                                                                                              |
| Search for a post2          | Reddit                     | Search posts in r/sidehustle       | When clicking ‘Execute workflow’ | Merge1                   |                                                                                                                              |
| Merge1                     | Merge                      | Combine Reddit search results      | Search for a post, Search for a post1, Search for a post2 | Edit Fields              |                                                                                                                              |
| Edit Fields                | Set                        | Extract and normalize Reddit data  | Merge1                      | AI Agent, Merge           |                                                                                                                              |
| AI Agent                   | LangChain Agent            | Analyze post with AI, generate ideas | Edit Fields                 | Merge                    |                                                                                                                              |
| Google Gemini Chat Model   | LangChain LM Chat Model    | AI language model backend          | AI Agent (via ai_languageModel input) | AI Agent                 | Setup Instructions: After importing this workflow into your n8n instance, you must set up your own credentials for Reddit, Google Gemini (PaLM), and Google Sheets. Replace the credential names 'YOUR_REDDIT_CREDENTIAL', 'YOUR_GOOGLE_GEMINI_CREDENTIAL', and 'YOUR_GOOGLE_SHEETS_CREDENTIAL' in the workflow with the names of your own credentials configured in n8n. No API keys or sensitive data are included in this workflow. |
| Merge                      | Merge                      | Combine original post data and AI output | Edit Fields, AI Agent       | Append row in sheet       |                                                                                                                              |
| Append row in sheet        | Google Sheets              | Append combined data to Google Sheet | Merge                       | None                     |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it **"Startup Idea Finder"**.

2. **Add a Manual Trigger node**:  
   - Name: *When clicking ‘Execute workflow’*  
   - No special parameters needed.

3. **Add three Reddit nodes** configured as follows:  
   - **Node 1:**  
     - Name: *Search for a post*  
     - Operation: Search  
     - Subreddit: `crazyideas`  
     - Keyword: `"Why is there no tool that"`  
     - Limit: 25  
     - Sort by: `comments`  
     - Credentials: Reddit OAuth2 API (set your own)  
   - **Node 2:**  
     - Name: *Search for a post1*  
     - Operation: Search  
     - Subreddit: `Lightbulb`  
     - Keyword: `"Why is there no tool that"`  
     - Limit: 25  
     - Sort by: `comments`  
     - Credentials: Reddit OAuth2 API (same as above)  
   - **Node 3:**  
     - Name: *Search for a post2*  
     - Operation: Search  
     - Subreddit: `sidehustle`  
     - Keyword: `"Why is there no tool that"`  
     - Limit: 25  
     - Sort by: `top`  
     - Credentials: Reddit OAuth2 API  

4. **Connect the Manual Trigger node’s main output to all three Reddit nodes’ inputs** (parallel execution).

5. **Add a Merge node** named *Merge1*:  
   - Mode: Combine  
   - Number of Inputs: 3  
   - Connect the output of the three Reddit nodes to the three inputs of this Merge node.

6. **Add a Set node** named *Edit Fields*:  
   - Purpose: Extract and rename fields to:  
     - `title` = `{{$json["title"]}}`  
     - `selftext` = `{{$json["selftext"]}}`  
     - `ups` = `{{$json["ups"]}}` (number)  
     - `created` = `{{$json["created"]}}` (number)  
     - `url` = `{{$json["url"]}}`  
   - Connect *Merge1* output to *Edit Fields* input.

7. **Add a LangChain Google Gemini Chat Model node** named *Google Gemini Chat Model*:  
   - Model Name: `models/gemini-1.5-flash-8b-latest`  
   - Credentials: Google PaLM API (set your own Google Gemini credential)  
   - This node will be used as the language model backend for the LangChain Agent.

8. **Add a LangChain Agent node** named *AI Agent*:  
   - Prompt:  
     ```
     A Reddit user posted the following in r/SomebodyMakeThis:

     ---
     **Title**: {{ $json["title"] }}

     **Post Text**: {{ $json["selftext"] }}

     **Upvotes**: {{ $json["ups"] }}
     **Post URL**: {{ $json["url"] }}
     **Created Date**: {{ $json.created }}
     ---

     Based on the above, please do the following:

     1. **Explain the core problem or frustration described in this post.**
     2. **Determine if any existing products, tools, or companies are actively solving this issue.**
     3. **If you can't find any known solution, propose a startup idea or product that could effectively address this pain point.**
     4. **Include what type of user would benefit, and how the solution could make their life easier or more efficient.**
     5. Also give an overview of implementing this startup idea.

     Be as insightful and practical as possible.
     ```
   - Set the *Google Gemini Chat Model* node as the language model backend in the AI Agent node’s settings.
   - Connect *Edit Fields* output to the AI Agent node input.
   - Link the *Google Gemini Chat Model* node to the AI Agent node via the ai_languageModel input.

9. **Add another Merge node** named *Merge*:  
   - Mode: Combine by position (combine inputs from *Edit Fields* and *AI Agent*)  
   - Connect *Edit Fields* output to the first input of *Merge*.  
   - Connect *AI Agent* output to the second input of *Merge*.

10. **Add a Google Sheets node** named *Append row in sheet*:  
    - Operation: Append  
    - Specify your Google Sheets Document ID and Sheet Name (must exist)  
    - Credentials: Google Sheets OAuth2 (set your own)  
    - Connect *Merge* output to this node input.

11. **Test the workflow** by clicking the manual trigger and verify that data flows through each node correctly, and new rows are appended to your Google Sheet containing both Reddit post data and AI-generated startup ideas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| After importing this workflow, you must configure your own credentials for Reddit, Google Gemini (PaLM), and Google Sheets by replacing placeholder names in the nodes.        | See node descriptions for credential names: 'YOUR_REDDIT_CREDENTIAL', 'YOUR_GOOGLE_GEMINI_CREDENTIAL', 'YOUR_GOOGLE_SHEETS_CREDENTIAL' |
| Workflow uses the Google Gemini 1.5 Flash 8B latest model for AI processing via LangChain integration in n8n.                                                                   | Google PaLM API / Gemini documentation                                                                       |
| The workflow is designed to handle multiple subreddit searches in parallel and merges results for comprehensive AI analysis.                                                   |                                                                                                             |
| Reddit OAuth2 credentials must have appropriate scopes to use the Reddit search API.                                                                                           | Reddit API documentation                                                                                      |
| Google Sheets node requires the target sheet to exist with proper permission for the OAuth2 user.                                                                               | Google Sheets API documentation                                                                               |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.