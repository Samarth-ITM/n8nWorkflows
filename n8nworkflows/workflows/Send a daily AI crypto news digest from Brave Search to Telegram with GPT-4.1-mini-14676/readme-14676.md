Send a daily AI crypto news digest from Brave Search to Telegram with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/send-a-daily-ai-crypto-news-digest-from-brave-search-to-telegram-with-gpt-4-1-mini-14676


# Send a daily AI crypto news digest from Brave Search to Telegram with GPT-4.1-mini

# Technical Documentation: Daily AI Crypto News Digest

## 1. Workflow Overview
This workflow automates the collection, synthesis, and delivery of cryptocurrency news. It operates on a daily schedule, fetching the latest headlines via Brave Search, processing the top five most relevant articles, and using a Large Language Model (GPT-4.1-mini) to generate a concise, plain-text digest sent directly to a Telegram chat.

### Logical Blocks
- **1.1 Trigger & Acquisition:** Handles the timing and the initial retrieval of raw news data.
- **1.2 Data Normalization:** Filters the search results and formats them into a structured list.
- **1.3 AI Synthesis:** Transforms raw article data into a human-readable, professional digest.
- **1.4 Delivery:** Transmits the final text to the end user via Telegram.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger & Acquisition
**Overview:** Initiates the workflow daily and fetches recent cryptocurrency news.
- **Nodes Involved:** `Schedule Trigger`, `Brave Search`
- **Node Details:**
    - **Schedule Trigger**: 
        - *Role*: Temporal trigger.
        - *Configuration*: Set to Cron expression `0 8 * * *` (Daily at 08:00 UTC).
        - *Potential Failures*: Timezone misalignment if the user expects a different local time.
    - **Brave Search**: 
        - *Role*: External API data fetcher.
        - *Configuration*: Operation set to "News"; Query: `crypto cryptocurrency bitcoin news`; Language: `en`; Country: `US`.
        - *Connections*: Trigger $\rightarrow$ Brave Search.
        - *Potential Failures*: API Key expiration, quota limits, or connectivity issues with Brave's API.

### 2.2 Data Normalization
**Overview:** Cleans the raw API response and prepares a text prompt for the AI.
- **Nodes Involved:** `Format Top 5`, `Build Prompt`
- **Node Details:**
    - **Format Top 5 (Code Node)**:
        - *Role*: Data filtering and mapping.
        - *Logic*: Slices the first 5 results. Extracts `title`, `description`, `url`, `hostname` (from `meta_url`), and `age`.
        - *Edge Case*: Throws an explicit error if the results array is empty to prevent the workflow from continuing with null data.
    - **Build Prompt (Code Node)**:
        - *Role*: String aggregation.
        - *Logic*: Maps the array of 5 articles into a single concatenated string using template literals for the AI to process.
        - *Connections*: Brave Search $\rightarrow$ Format Top 5 $\rightarrow$ Build Prompt.

### 2.3 AI Synthesis
**Overview:** Uses a LLM to summarize the news into a specific professional format.
- **Nodes Involved:** `Make sumarry`, `OpenAI Chat Model`
- **Node Details:**
    - **Make sumarry (Chain LLM)**:
        - *Role*: Orchestrates the prompt and the LLM response.
        - *Configuration*: 
            - *System Prompt*: Instructs the AI to act as a crypto analyst, avoiding markdown/bolding, using plain text only.
            - *User Prompt*: Requests punchy headlines, 2-3 sentence summaries, and a "Market Mood" conclusion.
        - *Connections*: Build Prompt $\rightarrow$ Make sumarry.
    - **OpenAI Chat Model**:
        - *Role*: The intelligence engine.
        - *Configuration*: Model set to `gpt-4.1-mini`.
        - *Connections*: Connected to `Make sumarry` via the `ai_languageModel` input.

### 2.4 Delivery
**Overview:** Sends the final synthesized text to the target recipient.
- **Nodes Involved:** `Send to Telegram`
- **Node Details:**
    - **Send to Telegram**:
        - *Role*: Communication output.
        - *Configuration*: Sends the `text` output from the AI chain to a specific `chatId`.
        - *Connections*: Make sumarry $\rightarrow$ Send to Telegram.
        - *Potential Failures*: Incorrect Chat ID, Bot not started/added to the group, or API token invalidation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule Trigger | Timing Execution | - | Brave Search | Fires automatically every day at 08:00 UTC via cron expression (0 8 * * *). Adjust the time to match your timezone. |
| Brave Search | Brave Search | News Retrieval | Schedule Trigger | Format Top 5 | Queries the Brave Search News API for recent crypto headlines. Returns up to 10 results filtered to English-language US sources. |
| Format Top 5 | Code | Data Filtering | Brave Search | Build Prompt | Selects the top 5 articles and normalizes each entry. Assembles them into a single formatted prompt string for the language model. |
| Build Prompt | Code | Prompt Construction | Format Top 5 | Make sumarry | Selects the top 5 articles and normalizes each entry. Assembles them into a single formatted prompt string for the language model. |
| Make sumarry | Chain LLM | Content Synthesis | Build Prompt | Send to Telegram | GPT-4.1-mini generates a plain-text daily digest — numbered headlines, 2–3 sentence summaries, and a Market Mood line at the end. |
| OpenAI Chat Model | Chat Model | LLM Provider | - | Make sumarry | GPT-4.1-mini generates a plain-text daily digest — numbered headlines, 2–3 sentence summaries, and a Market Mood line at the end. |
| Send to Telegram | Telegram | Final Notification | Make sumarry | - | Sends the completed digest to the configured chat. Ensure the bot is added to the target chat before activating. |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Trigger & Data Retrieval
1. Create a **Schedule Trigger** node. Set the interval to "Cron Expression" and use `0 8 * * *`.
2. Create a **Brave Search** node. 
    - Configure credentials using your Brave Search API key.
    - Set Operation to `News`.
    - Set Query to `crypto cryptocurrency bitcoin news`.
    - Set Country to `US` and Language to `en`.

### Step 2: Data Processing (Code)
3. Create a **Code** node named `Format Top 5`. Paste the JavaScript logic to slice the first 5 results from the JSON output and map them to a simplified object (index, title, description, url, source, published).
4. Create a second **Code** node named `Build Prompt`. Use JavaScript to join these 5 objects into a single string, separated by double newlines.

### Step 3: AI Configuration
5. Create a **Chain LLM** node named `Make sumarry`.
    - Set the prompt to include the dynamic expression `{{ $json.prompt }}`.
    - Define the system message: "You are a crypto news analyst... use plain text only — no asterisks, no bold, no hashtags."
6. Create an **OpenAI Chat Model** node and connect it to the `Make sumarry` node.
    - Use your OpenAI API credentials.
    - Select model `gpt-4.1-mini`.

### Step 4: Telegram Delivery
7. Create a **Telegram** node.
    - Configure your Telegram Bot API credentials.
    - Set the "Chat ID" to your specific numeric Telegram ID.
    - Set the "Text" field to the output of the AI node: `{{ $json.text }}`.

### Step 5: Connection Order
Connect the nodes in this sequence: 
`Schedule Trigger` $\rightarrow$ `Brave Search` $\rightarrow$ `Format Top 5` $\rightarrow$ `Build Prompt` $\rightarrow$ `Make sumarry` $\rightarrow$ `Send to Telegram`. 
*(Attach the OpenAI Model to the Chain LLM node)*.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Setup Checklist** | Ensure Brave Search News API is enabled, OpenAI API is active, and the Telegram Bot is a member of the destination chat. |
| **Customization** | To change the topic, modify the search query in the Brave Search node and update the System Prompt in the AI node. |
| **Timezone Warning** | Cron expressions in n8n are based on the instance timezone; verify the server time to ensure 08:00 UTC is correct for your needs. |