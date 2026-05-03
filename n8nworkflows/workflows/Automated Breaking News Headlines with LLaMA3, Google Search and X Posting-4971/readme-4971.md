Automated Breaking News Headlines with LLaMA3, Google Search and X Posting

https://n8nworkflows.xyz/workflows/automated-breaking-news-headlines-with-llama3--google-search-and-x-posting-4971


# Automated Breaking News Headlines with LLaMA3, Google Search and X Posting

---
### 1. Workflow Overview

This workflow automates the generation and posting of breaking news headlines related to a specified topic ("AI" by default) on the social media platform X (formerly Twitter). It integrates Google Custom Search to fetch the latest news articles, leverages the Groq AI API running a LLaMA3 model to create concise, objective headlines, and finally posts these headlines to X.

**Logical Blocks:**

- **1.1 Scheduled Trigger & Topic Setup:** Periodically triggers the workflow and sets the news topic.
- **1.2 Query Construction & Google Search:** Builds a search query and fetches corresponding news articles using Google Custom Search API.
- **1.3 Result Processing:** Extracts the top search result and constructs a prompt for AI headline generation.
- **1.4 AI Headline Generation:** Uses Groq AI (LLaMA3) to generate a short, formatted news headline.
- **1.5 Social Media Posting:** Publishes the generated headline on X (Twitter).

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Topic Setup

- **Overview:** This block initiates the workflow every 5 hours at a randomized minute, then sets the news topic for searching.
- **Nodes Involved:** Schedule Trigger, Set Topic

**Node Details:**

- **Schedule Trigger**
  - Type: Schedule trigger node
  - Configuration: Triggers every 5 hours at a random minute (0–59) to avoid predictability
  - Input: None (entry point)
  - Output: To "Set Topic"
  - Edge cases: Trigger delay or failure if n8n server is down; randomness may cause uneven intervals

- **Set Topic**
  - Type: Set node
  - Configuration: Sets a fixed string parameter `topic` with value "AI"
  - Input: From Schedule Trigger
  - Output: To "Build Query"
  - Edge cases: Hardcoded topic limits flexibility; no dynamic topic input

---

#### 2.2 Query Construction & Google Search

- **Overview:** Constructs a Google search query string using the topic and performs a Google Custom Search API call to retrieve news articles.
- **Nodes Involved:** Build Query, Google Config, HTTP Request

**Node Details:**

- **Build Query**
  - Type: Set node
  - Configuration: Creates a `query` string in the format `"latest <topic> news"`, dynamically injecting the topic from "Set Topic"
  - Input: From Set Topic
  - Output: To Google Config
  - Key Expression: `={{ "latest " + $node["Set Topic"].json["topic"] + " news" }}`
  - Edge cases: If topic is empty or malformed, query could be invalid or return irrelevant results

- **Google Config**
  - Type: Set node
  - Configuration: Sets Google API credentials parameters: `google_api_key` and `google_cx` (Custom Search Engine ID); placeholders require user replacement
  - Input: From Build Query
  - Output: To HTTP Request
  - Edge cases: Missing or invalid API key or CX will cause authentication or API errors

- **HTTP Request**
  - Type: HTTP Request node
  - Configuration:
    - Method: GET
    - URL built dynamically using API key, CX, and query string from previous nodes
    - Response format: JSON
  - Input: From Google Config
  - Output: To Function (processing)
  - Key Expression in URL: `=https://www.googleapis.com/customsearch/v1?key={{ $json.google_api_key }}&cx={{ $json.google_cx }}?q{{ $node["Build Query"].json["query"] }}`
  - Edge cases:
    - Incorrect URL formatting: The query parameter concatenation uses `?q{{ ... }}`, missing an equal sign or ampersand could cause URL errors.
    - API rate limits or quota exceeded errors
    - Network timeouts or malformed JSON response
  - Notes: The URL concatenation appears incorrect (should be `&q=` instead of `?q` after `cx=...`)

---

#### 2.3 Result Processing

- **Overview:** Parses the Google search results JSON, extracts the top news article's title and URL, and constructs a prompt string for headline generation.
- **Nodes Involved:** Function

**Node Details:**

- **Function**
  - Type: Code (JavaScript) node
  - Configuration:
    - Checks for search results presence; throws error if none found
    - Extracts first item’s `title` and `link`
    - Sets multiple JSON properties: `headline`, `url`, `articleTitle`, `articleURL`
    - Constructs prompt text for AI with a strict format and includes hashtags
  - Input: From HTTP Request
  - Output: To Groq AI
  - Key Code Snippet:
    ```javascript
    if (results.length === 0) throw new Error("No search results found");
    const top = results[0];
    item.json.prompt = `Write a short, objective headline (max 120 characters) for the following article, and append 1–2 relevant hashtags. Format: <headline> <link> <hashtags>. Title: ${top.title} Link: ${top.link}`;
    ```
  - Edge cases:
    - Throws error if no results returned, which could stop execution
    - Assumes `items` array always exists; if API schema changes, could fail
    - Prompt length control relies on AI model compliance

---

#### 2.4 AI Headline Generation

- **Overview:** Sends the constructed prompt to Groq AI’s LLaMA3 model via a chat completion API to generate a concise news headline formatted as specified.
- **Nodes Involved:** Groq AI

**Node Details:**

- **Groq AI**
  - Type: HTTP Request node
  - Configuration:
    - POST request to Groq API endpoint for chat completions
    - Model: llama3-70b-8192
    - Messages: System message instructs AI to generate concise headlines with exact format, user message contains the prompt from the Function node
    - Parameters: temperature 0.7, top_p 0.95, max_tokens 200, presence_penalty 0, frequency_penalty 0.4
    - Authentication: Uses predefined Groq API credentials
  - Input: From Function
  - Output: To Post to X
  - Key Expressions:
    - User content: `{{ $('Function').item.json.prompt.replace(/\n/g, '\\n') }}`
  - Edge cases:
    - API authentication failure or quota exceeded
    - Response latency or timeouts
    - AI output may not strictly follow format or length constraints
  - Version requirements: Uses HTTP Request typeVersion 4.2 for advanced JSON payloads

---

#### 2.5 Social Media Posting

- **Overview:** Posts the AI-generated headline text to the X platform using OAuth2 credentials.
- **Nodes Involved:** Post to X

**Node Details:**

- **Post to X**
  - Type: Twitter node (version 2 using OAuth2)
  - Configuration:
    - Text to post is extracted from AI response: `={{ $json.choices[0].message.content }}`
    - No additional fields configured
    - Uses OAuth2 credentials for authorized posting
  - Input: From Groq AI
  - Output: None (end of workflow)
  - Edge cases:
    - OAuth token expiration or invalid credentials
    - API rate limits or content posting restrictions
    - AI-generated content violating platform policies may cause rejection
  - Notes: The node’s name references “X” (Twitter rebranding)

---

### 3. Summary Table

| Node Name     | Node Type             | Functional Role                 | Input Node(s)     | Output Node(s) | Sticky Note                                    |
|---------------|-----------------------|--------------------------------|-------------------|----------------|-----------------------------------------------|
| Schedule Trigger | Schedule Trigger      | Periodic workflow start trigger | None              | Set Topic      |                                               |
| Set Topic     | Set                   | Sets the news topic parameter   | Schedule Trigger  | Build Query    |                                               |
| Build Query   | Set                   | Constructs the Google search query | Set Topic         | Google Config  |                                               |
| Google Config | Set                   | Sets Google API credentials     | Build Query       | HTTP Request   | Replace with your Google API key and CX value |
| HTTP Request  | HTTP Request          | Calls Google Custom Search API  | Google Config     | Function       |                                               |
| Function      | Function (Code)       | Extracts top article & builds AI prompt | HTTP Request      | Groq AI       |                                               |
| Groq AI       | HTTP Request          | Calls Groq AI for headline generation | Function          | Post to X      |                                               |
| Post to X     | Twitter (OAuth2)      | Posts generated headline to X   | Groq AI           | None           |                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Set it to trigger every 5 hours
   - Configure the trigger minute as a random number between 0 and 59 using expression: `={{Math.floor(Math.random() * 60)}}`

2. **Add a Set node named "Set Topic"**
   - Create a string variable named `topic` with value `"AI"`
   - Connect Schedule Trigger → Set Topic

3. **Add a Set node named "Build Query"**
   - Create a string variable `query`
   - Set value with expression: `={{ "latest " + $node["Set Topic"].json["topic"] + " news" }}`
   - Connect Set Topic → Build Query

4. **Add a Set node named "Google Config"**
   - Add two string variables:
     - `google_api_key`: set to your Google API key (replace placeholder)
     - `google_cx`: set to your Google Custom Search Engine ID (replace placeholder)
   - Connect Build Query → Google Config

5. **Add an HTTP Request node named "HTTP Request"**
   - Method: GET
   - URL expression:  
     `=https://www.googleapis.com/customsearch/v1?key={{ $json.google_api_key }}&cx={{ $json.google_cx }}&q={{ $node["Build Query"].json["query"] }}`
   - Response format: JSON
   - Connect Google Config → HTTP Request

6. **Add a Function node named "Function"**
   - JavaScript code:
     ```javascript
     return items.map(item => {
       const results = item.json.items || [];
       if (results.length === 0) {
         throw new Error("No search results found");
       }
       const top = results[0];
       item.json.headline = top.title;
       item.json.url = top.link;
       item.json.articleTitle = top.title;
       item.json.articleURL = top.link;
       item.json.prompt = `Write a short, objective headline (max 120 characters) for the following article, and append 1–2 relevant hashtags. Format: <headline> <link> <hashtags>. Title: ${top.title} Link: ${top.link}`;
       return item;
     });
     ```
   - Connect HTTP Request → Function

7. **Add an HTTP Request node named "Groq AI"**
   - Method: POST
   - URL: `https://api.groq.com/openai/v1/chat/completions`
   - Authentication: Add Groq API credentials (predefined credential type)
   - Body type: JSON
   - JSON body:
     ```json
     {
       "model": "llama3-70b-8192",
       "messages": [
         {
           "role": "system",
           "content": "You are a social media assistant. Your task is to write a short, news-style headline for a given article title and link. Use exactly this format: <headline> <link>. Be objective and precise. Max 200 characters total."
         },
         {
           "role": "user",
           "content": "={{ $('Function').item.json.prompt.replace(/\\n/g, '\\n') }}"
         }
       ],
       "temperature": 0.7,
       "top_p": 0.95,
       "max_tokens": 200,
       "presence_penalty": 0,
       "frequency_penalty": 0.4
     }
     ```
   - Connect Function → Groq AI

8. **Add a Twitter node named "Post to X"**
   - Use OAuth2 credentials for your X account
   - Text field expression: `={{ $json.choices[0].message.content }}`
   - Connect Groq AI → Post to X

9. **Activate the workflow**

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Replace placeholders `REPLACE_WITH_YOUR_KEY` and `REPLACE_WITH_YOUR_CX` in "Google Config" node with your credentials | Google Custom Search API documentation: https://developers.google.com/custom-search/v1/overview        |
| The Groq AI node uses the LLaMA3 70B model at 8192 context length, which requires appropriate API access and quota    | Groq AI documentation: https://docs.groq.com/api/openai                                               |
| Twitter node uses OAuth2 for authentication with X (Twitter) API; ensure tokens are valid and have posting permission | Twitter Developer Platform: https://developer.twitter.com/en/docs/authentication/oauth-2-0             |
| The schedule trigger randomizes trigger minute to reduce hitting the API at the same time every day                   | Helps avoid rate limits and detection by anti-bot systems                                             |
| Prompt formatting enforces output style for AI to generate concise and objective headlines with link and hashtags    | Important to maintain social media content quality and compliance                                     |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created with n8n, a workflow automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.