Generate and Schedule AI Discussion Posts for Reddit with GPT-4 and Airtable

https://n8nworkflows.xyz/workflows/generate-and-schedule-ai-discussion-posts-for-reddit-with-gpt-4-and-airtable-6255


# Generate and Schedule AI Discussion Posts for Reddit with GPT-4 and Airtable

### 1. Workflow Overview

This workflow automates the generation and scheduling of AI-powered discussion posts for Reddit using GPT-4 and Airtable. It is designed for content creators, community managers, or marketers who want to regularly publish fresh, engaging questions on specific subreddits without manual effort. The workflow ensures that newly generated discussion topics are unique by referencing previously posted content, schedules posts at a defined time, publishes them on Reddit, and archives the posts for future reference.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger**: Defines the publishing time for new Reddit posts.
- **1.2 Retrieve Previous Discussions**: Fetches past posted questions from Airtable to prevent duplication.
- **1.3 Aggregate Previous Content**: Combines all past questions into a single text block for input to the AI model.
- **1.4 AI Discussion Generation**: Uses GPT-4 to generate a new, unique discussion question based on prior posts.
- **1.5 Post to Reddit**: Publishes the AI-generated discussion question to the specified subreddit.
- **1.6 Archive New Discussion**: Saves the newly posted discussion question back into Airtable for tracking and future deduplication.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow at a specified time daily to automate the post generation and publishing process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Trigger node (Schedule Trigger)  
    - Configuration: Trigger set to run daily at 19:00 (7 PM)  
    - Inputs: None (trigger node)  
    - Outputs: Connected to "Get Previous Discussions" node  
    - Edge Cases: Potential failure if the server time zone differs or if the node is disabled; ensure timezone settings align with expected schedule.  
    - Version: 1.2

---

#### 1.2 Retrieve Previous Discussions

- **Overview:**  
  Retrieves existing discussion questions previously posted and stored in Airtable to avoid repetition in new AI-generated content.

- **Nodes Involved:**  
  - Get Previous Discussions

- **Node Details:**

  - **Get Previous Discussions**  
    - Type: Airtable node (Search operation)  
    - Configuration:  
      - Base: Airtable base named "[TEMPLATE] Reddit Discussion"  
      - Table: "Content" table  
      - View: "r/Example" (filters relevant subreddit data)  
      - Operation: Search to fetch existing records  
    - Credentials: Airtable API token with sufficient permissions  
    - Inputs: Triggered by Schedule Trigger  
    - Outputs: Passes data to Aggregate node  
    - Edge Cases: Possible API rate limits, expired credentials, or empty results if no previous discussions exist. Handle empty data gracefully downstream.  
    - Version: 2.1

---

#### 1.3 Aggregate Previous Content

- **Overview:**  
  Combines all fetched previous discussion questions into a single text block to be used as context for the AI model, helping it avoid duplicating questions.

- **Nodes Involved:**  
  - Aggregate

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate node  
    - Configuration: Aggregates the "Copy" field from previous discussion records into one concatenated string  
    - Inputs: Data from "Get Previous Discussions"  
    - Outputs: Single aggregated text passed to "Generate New Discussion"  
    - Edge Cases: If no prior discussions exist, aggregation will output empty string which may affect AI prompt; consider default fallback.  
    - Version: 1

---

#### 1.4 AI Discussion Generation

- **Overview:**  
  Utilizes GPT-4 via Langchain to generate a short, open-ended discussion question that is unique compared to previous posts.

- **Nodes Involved:**  
  - Generate New Discussion  
  - OpenAI Chat Model

- **Node Details:**

  - **Generate New Discussion**  
    - Type: Langchain Chain LLM node  
    - Configuration:  
      - Prompt instructs GPT-4 to create a short open-ended question about a specified topic and subreddit.  
      - It explicitly requests no duplication of previous questions by referencing the aggregated previous content (`{{ $json.Copy }}`).  
      - Parameters include placeholders for [WEEK/DAY], [INSERT TOPIC HERE], and [SUBREDDIT NAME HERE] which users must customize.  
    - Inputs: Aggregated previous discussions text  
    - Outputs: Generated question text to "Post Discussion" node  
    - Edge Cases: Prompt misconfiguration or malformed expressions may cause unexpected outputs; ensure placeholders are replaced correctly.  
    - Version: 1.4

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model node  
    - Configuration:  
      - Model set to "gpt-4o" (GPT-4 optimized)  
      - Default options  
    - Credentials: OpenAI API key (configured with valid OpenAI account)  
    - Inputs: Used internally by "Generate New Discussion" as AI backend  
    - Outputs: Passes AI-generated response to "Generate New Discussion"  
    - Edge Cases: API quota exceeded, network errors, or invalid credentials may cause failure; retry policies advised.  
    - Version: 1

---

#### 1.5 Post to Reddit

- **Overview:**  
  Posts the AI-generated discussion question to the target subreddit on Reddit.

- **Nodes Involved:**  
  - Post Discussion

- **Node Details:**

  - **Post Discussion**  
    - Type: Reddit node (OAuth2 API)  
    - Configuration:  
      - Post title and text set to the generated question text (`={{ $json.text }}`)  
      - Subreddit hardcoded as "Biohackers" (user must change as needed)  
      - Post type defaults to text post; options exist for image or link posts but not currently configured  
    - Credentials: Reddit OAuth2 API with authorized Reddit account  
    - Inputs: Generated question text from "Generate New Discussion"  
    - Outputs: Passes newly created post data to "Create Archived Discussion"  
    - Edge Cases: OAuth token expiry, subreddit posting restrictions, rate limits, or API errors can cause failures; ensure proper error handling and permission scopes.  
    - Version: 1

---

#### 1.6 Archive New Discussion

- **Overview:**  
  Stores the newly posted discussion question into Airtable to maintain an archive for future deduplication and tracking.

- **Nodes Involved:**  
  - Create Archived Discussion

- **Node Details:**

  - **Create Archived Discussion**  
    - Type: Airtable node (Create operation)  
    - Configuration:  
      - Base: "[TEMPLATE] Reddit Discussion"  
      - Table: "Content"  
      - Fields:  
        - Question: Text of the posted question from "Generate New Discussion" node  
        - Subreddit: Hardcoded "r/Example" (should be updated to match actual subreddit)  
        - Published Date: Current timestamp (`{{ $now }}`)  
    - Credentials: Airtable API token with write permissions  
    - Inputs: From "Post Discussion" node  
    - Outputs: None (terminal node)  
    - Edge Cases: Write failures due to API limits or invalid credentials; missing or mismatched field names; ensure field mappings are accurate.  
    - Version: 2.1

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                               |
|-------------------------|----------------------------------|------------------------------------|-----------------------|-------------------------|---------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger        | Schedule Trigger                  | Initiates workflow at scheduled time | None                  | Get Previous Discussions  | 1. 📅 Schedule: Select when you want to publish a new discussion(daily, weekly, etc)                                        |
| Get Previous Discussions | Airtable (Search)                | Fetches previously posted questions | Schedule Trigger      | Aggregate                | 2. 📕 Get Previous Discussions: Receive previously posted questions from a subreddit; includes Airtable setup links         |
| Aggregate              | Aggregate                        | Concatenates previous questions into one text block | Get Previous Discussions | Generate New Discussion  | 3. 📚 Aggregate: Combines all previous questions for ChatGPT input; suggests limiting tokens pre-aggregation               |
| Generate New Discussion | Langchain Chain LLM              | Generates new unique discussion question | Aggregate            | Post Discussion          | 4. 🪶 Generate Discussion: Generates new question avoiding duplicates; user must customize prompt placeholders             |
| OpenAI Chat Model      | Langchain OpenAI Chat Model      | AI backend for generating text      | Generate New Discussion | Generate New Discussion  |                                                                                                                           |
| Post Discussion        | Reddit                          | Posts generated question to subreddit | Generate New Discussion | Create Archived Discussion | 5. 🔗 Post Discussion: Posts question to subreddit; configure subreddit name and post type                                  |
| Create Archived Discussion | Airtable (Create)              | Archives posted question in Airtable | Post Discussion       | None                    | 6. 🗃️ Store Discussion in Archive: Saves posted question to Airtable to prevent future duplication                        |
| Sticky Note            | Sticky Note                     | Documentation / comments             | None                  | None                    | Multiple notes covering each logical block with setup instructions and links                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add "Schedule Trigger" node:**  
   - Type: Schedule Trigger  
   - Set trigger to daily at 19:00 (adjust as needed)  
   - Connect output to next node.

3. **Add "Get Previous Discussions" node:**  
   - Type: Airtable node  
   - Operation: Search  
   - Configure credentials with Airtable API token (ensure token has read access)  
   - Set Base to your Airtable base for Reddit discussions (e.g., "[TEMPLATE] Reddit Discussion")  
   - Set Table to "Content"  
   - Use view filter for the desired subreddit (e.g., "r/Example")  
   - Connect input from Schedule Trigger.

4. **Add "Aggregate" node:**  
   - Type: Aggregate  
   - Configure to aggregate the "Copy" (or equivalent) field from previous discussions into one text string  
   - Connect input from "Get Previous Discussions".

5. **Add "Generate New Discussion" node:**  
   - Type: Langchain Chain LLM node  
   - Configure prompt as:  
     ```
     Generate a short open-ended question of the [WEEK/DAY] about [INSERT TOPIC HERE] for the r/[SUBREDDIT NAME HERE] subreddit. ONLY provide the short open-ended question and nothing else in your output. Make sure it isn't any of the following previously used below.

     {{ $json.Copy }}
     ```  
   - Replace placeholders with actual values before use  
   - Set "executeOnce" to true  
   - Connect input from "Aggregate".

6. **Add "OpenAI Chat Model" node:**  
   - Type: Langchain OpenAI Chat Model node  
   - Select model "gpt-4o" or equivalent GPT-4 model  
   - Configure OpenAI credentials with valid API key  
   - Connect as AI backend for "Generate New Discussion" node (via Langchain AI model input).

7. **Add "Post Discussion" node:**  
   - Type: Reddit node  
   - Configure Reddit OAuth2 credentials with authorized Reddit account  
   - Set title and text fields to `={{ $json.text }}` (the generated question)  
   - Set subreddit name (without "r/") to the target subreddit, e.g., "Biohackers"  
   - Connect input from "Generate New Discussion".

8. **Add "Create Archived Discussion" node:**  
   - Type: Airtable node  
   - Operation: Create  
   - Use same Airtable base and table as "Get Previous Discussions"  
   - Map fields:  
     - Question: `={{ $('Generate New Discussion').item.json.text }}`  
     - Subreddit: e.g., "r/Example" (match with actual subreddit)  
     - Published Date: `={{ $now }}` (current timestamp)  
   - Connect input from "Post Discussion".

9. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create Airtable Account: https://airtable.com/invite/r/zhjTCrgs                                       | For setting up Airtable to store discussion posts                                                  |
| Get Airtable "Personal Access Token": https://airtable.com/create/tokens                              | Required for API access to Airtable                                                               |
| Airtable Template used in this workflow: https://airtable.com/app6wzQqegKIJOiOg                       | Preconfigured Airtable base and table for Reddit discussions                                      |
| Reddit post types available: Text, Image, Link with OpenGraph thumbnail                              | Configure in "Post Discussion" node to enhance Reddit post presentation                            |
| Ensure to replace placeholder texts in prompts and node configurations with actual subreddit names and topics | Critical for correct AI prompt execution and subreddit posting                                   |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.