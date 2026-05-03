Learn Anything from HN - Get Top Resource Recommendations from Hacker News

https://n8nworkflows.xyz/workflows/learn-anything-from-hn---get-top-resource-recommendations-from-hacker-news-2697


# Learn Anything from HN - Get Top Resource Recommendations from Hacker News

### 1. Workflow Overview

This workflow automates the discovery and delivery of top community-recommended learning resources on any user-specified topic by leveraging Hacker News discussions. It is designed for learners who want curated, high-quality educational content without manually sifting through forums.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the user’s learning topic and email via a web form.
- **1.2 Hacker News Data Retrieval:** Searches Hacker News for relevant "Ask HN" posts and fetches their top-level comments.
- **1.3 Data Aggregation and AI Analysis:** Aggregates comments into a single text blob and uses a Large Language Model (Google Gemini) to analyze and extract top learning resources.
- **1.4 Formatting and Delivery:** Converts the AI output from Markdown to HTML and sends a personalized email to the user with the curated resource list.
- **1.5 Workflow Completion:** Marks the workflow as finished.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block collects the topic the user wants to learn and their email address through a simple web form.

- **Nodes Involved:**  
  - GetTopicFromToLearn

- **Node Details:**  
  - **GetTopicFromToLearn**  
    - Type: Form Trigger  
    - Role: Entry point that triggers the workflow when a user submits the form.  
    - Configuration:  
      - Form path: `/learn`  
      - Button label: "Submit"  
      - Form title: "What do You want to learn ?"  
      - Fields:  
        - "I want to learn" (text input, placeholder: "Python, DevOps, Ai, or just about anything")  
        - "What's your email ?" (email input, required)  
      - Response: Displays "We'll shortly send you an email with top recommendations." after submission.  
    - Inputs: HTTP request from form submission  
    - Outputs: JSON object containing user inputs, passed downstream  
    - Edge Cases:  
      - Missing or invalid email input (form validation handles this)  
      - Empty or vague topic input may reduce search relevance  

#### 2.2 Hacker News Data Retrieval

- **Overview:**  
  Searches Hacker News for "Ask HN" posts matching the user’s topic and retrieves the IDs of their top-level comments.

- **Nodes Involved:**  
  - SearchAskHN  
  - SplitOutChildrenIDs  
  - FindHNComments

- **Node Details:**  
  - **SearchAskHN**  
    - Type: Hacker News node  
    - Role: Searches Hacker News for posts tagged "ask_hn" containing the user’s topic keyword.  
    - Configuration:  
      - Limit: 150 posts  
      - Tags: ["ask_hn"]  
      - Keyword: dynamically set to user input from "I want to learn" field  
    - Inputs: User topic from form trigger  
    - Outputs: List of posts matching criteria  
    - Edge Cases:  
      - No matching posts found (empty output)  
      - API rate limits or downtime  
  - **SplitOutChildrenIDs**  
    - Type: Split Out  
    - Role: Extracts and splits the `children` field (comment IDs) from each post to process comments individually.  
    - Configuration: Field to split out: `children`  
    - Inputs: Posts from SearchAskHN  
    - Outputs: Individual comment IDs  
    - Edge Cases:  
      - Posts without comments (empty children array)  
  - **FindHNComments**  
    - Type: HTTP Request  
    - Role: Fetches the JSON data for each comment ID from Hacker News API.  
    - Configuration:  
      - URL template: `https://hacker-news.firebaseio.com/v0/item/{{ $json.children }}.json?print=pretty`  
    - Inputs: Comment IDs from SplitOutChildrenIDs  
    - Outputs: Comment JSON objects including text  
    - Edge Cases:  
      - Missing or deleted comments (null or error response)  
      - API timeouts or failures  

#### 2.3 Data Aggregation and AI Analysis

- **Overview:**  
  Aggregates all comment texts into a single string and uses Google Gemini LLM to analyze and extract the best learning resources, formatted in Markdown.

- **Nodes Involved:**  
  - CombineIntoSingleText  
  - Google Gemini Chat Model  
  - Basic LLM Chain

- **Node Details:**  
  - **CombineIntoSingleText**  
    - Type: Aggregate  
    - Role: Concatenates all comment texts into one aggregated text field for AI processing.  
    - Configuration: Aggregate field: `text`  
    - Inputs: Comment JSONs from FindHNComments  
    - Outputs: Single JSON with combined text  
    - Edge Cases:  
      - No comments to aggregate (empty output)  
  - **Google Gemini Chat Model**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Provides the underlying AI language model for the LLM chain.  
    - Configuration:  
      - Model: `models/gemini-1.5-flash`  
      - Credentials: Google PaLM API credentials (must be configured)  
    - Inputs: Prompt from Basic LLM Chain  
    - Outputs: AI-generated text response  
    - Edge Cases:  
      - API authentication errors  
      - Rate limits or model unavailability  
  - **Basic LLM Chain**  
    - Type: LangChain LLM Chain  
    - Role: Defines the prompt and calls the Google Gemini Chat Model to analyze comments and extract top resources.  
    - Configuration:  
      - Prompt template dynamically inserts user topic and aggregated comments.  
      - Instructions to categorize resources by type and difficulty, perform sentiment analysis, and output Markdown formatted recommendations.  
    - Inputs: Aggregated comment text from CombineIntoSingleText, user topic from form  
    - Outputs: Markdown text with resource recommendations  
    - Edge Cases:  
      - Prompt formatting errors  
      - AI output missing expected structure  

#### 2.4 Formatting and Delivery

- **Overview:**  
  Converts the AI-generated Markdown into HTML and sends a personalized email with the top learning resources to the user.

- **Nodes Involved:**  
  - Convert2HTML  
  - SendEmailWithTopResources

- **Node Details:**  
  - **Convert2HTML**  
    - Type: Markdown  
    - Role: Converts Markdown text from AI output into HTML for email formatting.  
    - Configuration: Mode: markdownToHtml  
    - Inputs: Markdown text from Basic LLM Chain  
    - Outputs: HTML content  
    - Edge Cases:  
      - Malformed Markdown causing conversion issues  
  - **SendEmailWithTopResources**  
    - Type: Email Send  
    - Role: Sends an email to the user with the HTML formatted top resource recommendations.  
    - Configuration:  
      - Subject: "Here are Top HN Recommendations for Learning <topic>" (dynamic)  
      - To: User email from form input  
      - From: Fixed email address (configured SMTP account)  
      - HTML body: Includes count of comments read and the HTML content from Convert2HTML  
    - Credentials: SMTP credentials (configured separately)  
    - Inputs: HTML content from Convert2HTML, user email and topic from form  
    - Outputs: Confirmation of email sent  
    - Edge Cases:  
      - SMTP authentication failure  
      - Invalid recipient email  
      - Email delivery issues  

#### 2.5 Workflow Completion

- **Overview:**  
  Marks the end of the workflow execution.

- **Nodes Involved:**  
  - Finished

- **Node Details:**  
  - **Finished**  
    - Type: No Operation (NoOp)  
    - Role: Acts as a terminal node to indicate workflow completion.  
    - Inputs: Email sent confirmation  
    - Outputs: None  
    - Edge Cases: None  

---

### 3. Summary Table

| Node Name                | Node Type                                | Functional Role                          | Input Node(s)           | Output Node(s)             | Sticky Note                                                                                              |
|--------------------------|-----------------------------------------|----------------------------------------|------------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| GetTopicFromToLearn      | Form Trigger                            | Captures user topic and email          | (Trigger)              | SearchAskHN                | "We'll find the best resources from HackerNews and send you an email"                                  |
| SearchAskHN              | Hacker News Search                      | Searches "Ask HN" posts by topic       | GetTopicFromToLearn    | SplitOutChildrenIDs        |                                                                                                        |
| SplitOutChildrenIDs      | Split Out                              | Extracts comment IDs from posts        | SearchAskHN            | FindHNComments             |                                                                                                        |
| FindHNComments           | HTTP Request                           | Fetches comment data from Hacker News | SplitOutChildrenIDs    | CombineIntoSingleText      |                                                                                                        |
| CombineIntoSingleText    | Aggregate                             | Aggregates all comment texts           | FindHNComments         | Basic LLM Chain            |                                                                                                        |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model    | Provides AI model for analysis         | Basic LLM Chain (ai_languageModel) | Basic LLM Chain            | Requires Google Gemini API credentials (Google PaLM API)                                               |
| Basic LLM Chain          | LangChain LLM Chain                   | Analyzes comments and extracts resources | CombineIntoSingleText  | Convert2HTML               | Prompt dynamically inserts user topic and comments; outputs Markdown formatted recommendations          |
| Convert2HTML             | Markdown                             | Converts Markdown to HTML              | Basic LLM Chain        | SendEmailWithTopResources  |                                                                                                        |
| SendEmailWithTopResources| Email Send                           | Sends personalized email to user      | Convert2HTML           | Finished                   | Requires SMTP credentials; email subject and body dynamically include user topic and comment count      |
| Finished                 | No Operation (NoOp)                   | Marks workflow completion              | SendEmailWithTopResources | (None)                    |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Name: `GetTopicFromToLearn`  
   - Path: `/learn`  
   - Button Label: "Submit"  
   - Form Title: "What do You want to learn ?"  
   - Form Fields:  
     - Text input labeled "I want to learn" with placeholder "Python, DevOps, Ai, or just about anything"  
     - Email input labeled "What's your email ?" (required) with placeholder "john.doe@example.com"  
   - Response Text: "We'll shortly send you an email with top recommendations."

2. **Add a Hacker News node**  
   - Name: `SearchAskHN`  
   - Operation: Search posts  
   - Limit: 150  
   - Tags: ["ask_hn"]  
   - Keyword: Set expression to `{{$json["I want to learn"]}}` from `GetTopicFromToLearn` node output  
   - Connect `GetTopicFromToLearn` main output to this node.

3. **Add a Split Out node**  
   - Name: `SplitOutChildrenIDs`  
   - Field to split out: `children` (comment IDs)  
   - Connect `SearchAskHN` output to this node.

4. **Add an HTTP Request node**  
   - Name: `FindHNComments`  
   - HTTP Method: GET  
   - URL: Use expression `https://hacker-news.firebaseio.com/v0/item/{{$json["children"]}}.json?print=pretty`  
   - Connect `SplitOutChildrenIDs` output to this node.

5. **Add an Aggregate node**  
   - Name: `CombineIntoSingleText`  
   - Aggregate field: `text` (concatenate all comment texts)  
   - Connect `FindHNComments` output to this node.

6. **Add a LangChain Google Gemini Chat Model node**  
   - Name: `Google Gemini Chat Model`  
   - Model Name: `models/gemini-1.5-flash`  
   - Credentials: Configure Google PaLM API credentials (Google Gemini API)  
   - No direct input connection; will be linked as AI model in next node.

7. **Add a LangChain LLM Chain node**  
   - Name: `Basic LLM Chain`  
   - Prompt Type: Define  
   - Text Prompt: Use a multi-line expression that:  
     - Inserts user topic from `GetTopicFromToLearn` (`I want to learn`)  
     - Inserts aggregated comments text from `CombineIntoSingleText` (`$json.text`)  
     - Instructions to find top learning resources, categorize by type and difficulty, perform sentiment analysis, and output Markdown formatted list (see prompt in workflow overview)  
   - Connect `CombineIntoSingleText` output to this node.  
   - Set AI language model to `Google Gemini Chat Model` node.

8. **Add a Markdown node**  
   - Name: `Convert2HTML`  
   - Mode: markdownToHtml  
   - Markdown input: Use expression `{{$json.text}}` from `Basic LLM Chain` output  
   - Connect `Basic LLM Chain` output to this node.

9. **Add an Email Send node**  
   - Name: `SendEmailWithTopResources`  
   - To Email: Expression from `GetTopicFromToLearn` (`What's your email ?`)  
   - From Email: Your configured sender email (e.g., `allsmallnocaps@gmail.com`)  
   - Subject: Expression `"Here are Top HN Recommendations for Learning " + {{$json["I want to learn"]}}`  
   - HTML Body: Expression combining comment count from `SplitOutChildrenIDs` (`{{$node["SplitOutChildrenIDs"].all().length}}`) and HTML content from `Convert2HTML` (`{{$json.data}}`)  
   - Credentials: Configure SMTP credentials for sending email  
   - Connect `Convert2HTML` output to this node.

10. **Add a No Operation node**  
    - Name: `Finished`  
    - Connect `SendEmailWithTopResources` output to this node.

11. **Activate the workflow**  
    - Ensure all credentials are set (Google Gemini API and SMTP)  
    - Test by submitting the form at `/learn` path.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Add your [Google Gemini API credentials](https://n8n.io/integrations/n8n-nodes-base.googlepalm/)   | Required for AI language model node to function                                                  |
| Add your [SMTP credentials](https://n8n.io/integrations/n8n-nodes-base.emailsend/#operations)      | Required for sending emails                                                                     |
| Workflow built as part of [#100DaysOfAgenticAi](https://github.com/ibrhdotme/100DaysOfAgenticAi/)  | Project credit and source                                                                        |
| Form and email subject can be customized as needed                                                 | Modify `GetTopicFromToLearn` form fields and `SendEmailWithTopResources` subject parameter       |
| Screenshot available in workflow metadata                                                          | Visual reference for node arrangement and form UI                                               |

---

This documentation fully describes the workflow’s structure, logic, and configuration, enabling reproduction, modification, and troubleshooting by advanced users or automation agents.