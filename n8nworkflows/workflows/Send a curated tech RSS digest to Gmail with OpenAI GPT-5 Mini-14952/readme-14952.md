Send a curated tech RSS digest to Gmail with OpenAI GPT-5 Mini

https://n8nworkflows.xyz/workflows/send-a-curated-tech-rss-digest-to-gmail-with-openai-gpt-5-mini-14952


# Send a curated tech RSS digest to Gmail with OpenAI GPT-5 Mini

### 1. Workflow Overview

The **"Send a curated tech RSS digest to Gmail with OpenAI GPT-5 Mini"** workflow is an automated content aggregation and curation system. Its primary purpose is to monitor 27 high-quality technical RSS feeds, filter the most relevant content using artificial intelligence, and deliver a professionally formatted HTML digest via Gmail every morning.

The logic is organized into the following functional blocks:
- **1.1 Input & Source Definition:** Handles the timing of the execution and the list of target RSS feeds.
- **1.2 Content Acquisition & Sanitization:** Fetches raw data from the web and cleans it for AI processing.
- **1.3 AI Curation:** Uses a Large Language Model (LLM) to select the highest-quality articles based on diversity and impact.
- **1.4 Delivery Generation:** Converts the AI's selection into a stylized HTML email and sends it to the user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Source Definition
**Overview:** Triggers the workflow daily and provides the static list of RSS feeds to be processed.
- **Nodes Involved:** `Daily Morning Trigger`, `Define RSS Sources`.
- **Node Details:**
    - **Daily Morning Trigger** (Schedule Trigger): Configured to execute every day at 08:00.
    - **Define RSS Sources** (Code): A JavaScript node that returns an array of 27 objects containing URLs and source names. These cover categories like Tech News, Engineering, AI/ML, Startups, Science, and Technical Bloggers.
    - **Output:** A list of objects passed to the fetcher.

#### 2.2 Content Acquisition & Sanitization
**Overview:** Retrieves the latest articles from all defined sources and prepares a clean, deduplicated dataset for the AI.
- **Nodes Involved:** `Fetch RSS Feeds`, `Clean and Prepare for AI`.
- **Node Details:**
    - **Fetch RSS Feeds** (RSS Feed Read): Reads the URL provided by the previous node. 
        - *Configuration:* `ignoreSSL` is enabled to prevent failures from certificates. 
        - *Error Handling:* Set to "Continue Regular Output" so that a single broken feed doesn't stop the entire workflow.
    - **Clean and Prepare for AI** (Code): A JavaScript node that performs the following:
        - Deduplicates articles by URL.
        - Strips HTML tags from descriptions.
        - Truncates descriptions to 250 characters.
        - Sorts articles by date (newest first) and limits the list to the top 150.
        - Formats the data into a numbered list for the AI prompt and a JSON string for later reconstruction.
    - **Edge Cases:** Handles missing titles/links and invalid date formats.

#### 2.3 AI Curation
**Overview:** Acts as an "editor" by analyzing the 150 most recent articles and selecting a diverse subset.
- **Nodes Involved:** `AI Article Curator`, `OpenAI GPT-5 Mini`.
- **Node Details:**
    - **AI Article Curator** (AI Agent): A LangChain agent that receives the curated list.
        - *Prompt:* Instructs the AI to select exactly 30 articles based on timeliness, diversity, and professional interest.
        - *Requirement:* The AI must return **only** a JSON object: `{"selected": [indices]}`.
    - **OpenAI GPT-5 Mini** (Chat OpenAI): The LLM providing the intelligence.
        - *Model:* `gpt-5-mini`.
        - *Credentials:* Requires OpenAI API key.
    - **Potential Failures:** AI might return text instead of JSON; the following block contains logic to handle this.

#### 2.4 Delivery Generation
**Overview:** Transforms the AI's index selection into a visually appealing email and delivers it.
- **Nodes Involved:** `Build Digest Email`, `Send Digest Email`.
- **Node Details:**
    - **Build Digest Email** (Code): A JavaScript node that:
        - Parses the AI output (using regex to extract JSON if the AI included conversational text).
        - Maps the selected indices back to the original article data.
        - Constructs a complex HTML template featuring a purple gradient header, numbered cards, and responsive CSS.
    - **Send Digest Email** (Gmail): Sends the generated HTML.
        - *Configuration:* Uses `emailHtml` for the body and a dynamic subject line including the current date.
        - *Credentials:* Requires Gmail OAuth2.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Daily Morning Trigger | Schedule Trigger | Execution timer | - | Define RSS Sources | Fires every day at 8:00 AM |
| Define RSS Sources | Code | Feed list definition | Daily Morning Trigger | Fetch RSS Feeds | Outputs 27 curated RSS feed URLs across 7 categories |
| Fetch RSS Feeds | RSS Feed Read | Content retrieval | Define RSS Sources | Clean and Prepare for AI | Reads each feed (with error handling) |
| Clean and Prepare for AI | Code | Data cleaning/sorting | Fetch RSS Feeds | AI Article Curator | Deduplicates, strips HTML, sorts by date, keeps 150 most recent |
| AI Article Curator | AI Agent | Content selection | Clean and Prepare for AI | Build Digest Email | Reviews all articles and picks the 30 most interesting/diverse ones |
| OpenAI GPT-5 Mini | Chat OpenAI | LLM Intelligence | - | AI Article Curator | Reviews all articles and picks the 30 most interesting/diverse ones |
| Build Digest Email | Code | HTML Email generation | AI Article Curator | Send Digest Email | Generates a beautiful HTML email with a purple gradient header... |
| Send Digest Email | Gmail | Email delivery | Build Digest Email | - | Sends via Gmail |

---

### 4. Reproducing the Workflow from Scratch

1.  **Trigger Setup:** Create a **Schedule Trigger** node. Set the interval to "Daily" and the time to "08:00".
2.  **Source Definition:** Add a **Code** node. Use JavaScript to return an array of objects containing `url` and `source` for your desired RSS feeds.
3.  **Data Retrieval:** Add an **RSS Feed Read** node.
    - Set URL to `{{ $json.url }}`.
    - In "Options", enable "Ignore SSL".
    - In "Node Settings", set "On Error" to "Continue Regular Output".
4.  **Data Cleaning:** Add a **Code** node to process the RSS data.
    - Implement logic to deduplicate by URL, strip HTML from content, and sort by date.
    - Ensure the output contains `articleList` (a string for the AI) and `articlesJson` (a stringified array of all articles).
5.  **AI Setup:** 
    - Add an **AI Agent** node. Set the prompt to ask for the 30 most interesting articles and specify the output must be a JSON object `{"selected": [indices]}`.
    - Connect an **OpenAI Chat Model** node to the Agent. Set the model to `gpt-5-mini` and configure your OpenAI credentials.
6.  **Email Construction:** Add a **Code** node.
    - Create logic to parse the AI's JSON response.
    - Map these indices to the articles from the "Clean and Prepare for AI" node.
    - Construct a HTML string using a `<table>` based layout with inline CSS for the purple gradient and cards.
7.  **Delivery:** Add a **Gmail** node.
    - Action: "Send Email".
    - Recipient: Enter your email address.
    - Subject: `Tech Digest - {{ $json.emailSubject }}`.
    - Body: Use the HTML generated in the previous step.
    - Configure Gmail OAuth2 credentials.
8.  **Activation:** Set the workflow to "Active".

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| AI requires specific JSON format to prevent email breaking | AI Agent Prompting |
| 27 RSS sources used across 7 categories (Tech, Engineering, AI, etc.) | Source List |
| Professional HTML email template includes responsive CSS | UI/UX Design |
| Must replace "your@email.com" placeholder in Gmail node | Deployment Requirement |