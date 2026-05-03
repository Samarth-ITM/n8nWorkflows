Send daily translated Calvin and Hobbes Comics to Discord

https://n8nworkflows.xyz/workflows/send-daily-translated-calvin-and-hobbes-comics-to-discord-2560


# Send daily translated Calvin and Hobbes Comics to Discord

---

### 1. Workflow Overview

This workflow automates the daily retrieval and sharing of Calvin and Hobbes comic strips on Discord, enriched with AI-powered translations. It is designed for fans of the comic who want to receive daily comics automatically with dialogues translated into English and Korean. The workflow runs every day at 9 AM, fetches the current day's comic strip from the official website, extracts the comic image URL, uses OpenAI models to analyze and translate the dialogue, and then posts the comic image along with the translated dialogues to a configured Discord channel.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Parameter Setup:** Triggers the workflow daily at a specific time and sets date parameters for the comic retrieval URL.
- **1.2 Comic Retrieval and Extraction:** Fetches the comic page HTML and extracts the comic image URL.
- **1.3 AI Processing and Translation:** Uses OpenAI’s language models to analyze the comic image and generate translations.
- **1.4 Posting to Discord:** Posts the comic image and translated dialogues to Discord via webhook.
- **1.5 Documentation and Notes:** Provides a sticky note summarizing the workflow purpose and setup instructions for user clarity.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling and Parameter Setup

**Overview:**  
This block sets the execution time of the workflow and prepares dynamic date parameters (year, month, day) based on the current date to construct the comic URL.

**Nodes Involved:**  
- Schedule Trigger  
- param (Set node)

**Node Details:**

- **Schedule Trigger**  
  - *Type:* scheduleTrigger  
  - *Role:* Initiates the workflow every day at 9 AM.  
  - *Configuration:* Interval set to trigger at hour 9 daily.  
  - *Input/Output:* No input; output triggers "param" node.  
  - *Edge Cases:* Workflow won't run if n8n server is down at trigger time.  
  - *Version:* 1.2

- **param (Set node)**  
  - *Type:* set  
  - *Role:* Sets three string parameters (`year`, `month`, `day`) based on current execution time.  
  - *Configuration:* Uses expressions `$now.format('yyyy')`, `$now.format('MM')`, `$now.format('dd')` to get current year, month, and day respectively.  
  - *Input:* Output of Schedule Trigger.  
  - *Output:* Feeds parameters to "HTTP Request" node.  
  - *Edge Cases:* Timezone differences could affect date if server time differs from intended locale.  
  - *Version:* 3.4

---

#### 2.2 Comic Retrieval and Extraction

**Overview:**  
This block downloads the HTML page for the daily Calvin and Hobbes comic and extracts the image URL using an AI-powered information extractor node.

**Nodes Involved:**  
- HTTP Request  
- Information Extractor

**Node Details:**

- **HTTP Request**  
  - *Type:* httpRequest  
  - *Role:* Fetches the HTML content of the comic page for the given date.  
  - *Configuration:* URL constructed dynamically as `https://www.gocomics.com/calvinandhobbes/{{ $json.year }}/{{ $json.month }}/{{ $json.day }}` using date parameters.  
  - *Input:* Output of "param" node (date parameters).  
  - *Output:* HTML content passed to "Information Extractor".  
  - *Edge Cases:* HTTP errors (404 if date has no comic), network timeouts, website structure changes.  
  - *Version:* 4.2

- **Information Extractor**  
  - *Type:* @n8n/n8n-nodes-langchain.informationExtractor  
  - *Role:* Extracts the comic image URL from the HTML content using AI prompt.  
  - *Configuration:* Prompt instructs to extract the `src` attribute value from an `<img>` tag with class `"img-fluid Lazyloaded"`.  
  - *Key Expressions:* Uses input from previous node as HTML source (`{{ $json.data }}`).  
  - *Input:* HTML from HTTP Request.  
  - *Output:* Outputs extracted URL as `cartoon_image`.  
  - *Edge Cases:* If website HTML structure changes, extractor may fail or output incorrect URL. AI prompt parsing may fail if unexpected HTML is received.  
  - *Version:* 1

---

#### 2.3 AI Processing and Translation

**Overview:**  
This block uses OpenAI models to analyze the comic image and generate translations combining the original language with Korean.

**Nodes Involved:**  
- OpenAI Chat Model (not connected in current workflow)  
- OpenAI (langchain openAi image analysis)

**Node Details:**

- **OpenAI Chat Model**  
  - *Type:* @n8n/n8n-nodes-langchain.lmChatOpenAi  
  - *Role:* Language model for chat-based AI tasks (not currently connected in main flow but available).  
  - *Configuration:* Model set to `gpt-4o-mini-2024-07-18`, no extra options.  
  - *Input:* None connected.  
  - *Output:* None connected.  
  - *Edge Cases:* Requires valid OpenAI credentials and internet connectivity.  
  - *Version:* 1

- **OpenAI (Image Analysis)**  
  - *Type:* @n8n/n8n-nodes-langchain.openAi  
  - *Role:* Analyzes the comic image URL to extract and translate dialogues.  
  - *Configuration:*  
    - Text prompt instructs to write original language and Korean translations together, with an example format.  
    - Model: `gpt-4o-mini` (OpenAI).  
    - Operation: `analyze` on image resource.  
    - Image URL: dynamically set from `{{ $json.output.cartoon_image }}` (output of Information Extractor).  
  - *Input:* Output from Information Extractor (comic image URL).  
  - *Output:* Passes translated text content to Discord node.  
  - *Edge Cases:* API rate limits, image URL invalid or inaccessible, translation inaccuracies.  
  - *Version:* 1.6  
  - *Credentials:* OpenAI API with configured API key.

---

#### 2.4 Posting to Discord

**Overview:**  
This block posts the daily comic image and translated dialogue text to a Discord channel via webhook.

**Nodes Involved:**  
- Discord

**Node Details:**

- **Discord**  
  - *Type:* discord  
  - *Role:* Posts a message containing the date, comic image URL, and translated dialogue text to Discord.  
  - *Configuration:*  
    - Content template includes date parameters, comic image URL, and the translated text from OpenAI.  
    - Uses Discord Webhook authentication.  
  - *Input:* Receives text from OpenAI node and date from param node (via OpenAI).  
  - *Output:* None (last node).  
  - *Edge Cases:* Invalid webhook URL, Discord API downtime, message formatting errors.  
  - *Version:* 2  
  - *Credentials:* Discord webhook URL configured.

---

#### 2.5 Documentation and Notes

**Overview:**  
Provides a visible sticky note to users in the n8n editor explaining workflow purpose, how it works, and setup steps.

**Nodes Involved:**  
- Sticky Note

**Node Details:**

- **Sticky Note**  
  - *Type:* stickyNote  
  - *Role:* Displays workflow documentation and branding image.  
  - *Configuration:* Markdown content describing workflow functionality and setup instructions.  
  - *Input/Output:* None; purely visual aid.  
  - *Version:* 1

---

### 3. Summary Table

| Node Name          | Node Type                         | Functional Role                         | Input Node(s)        | Output Node(s)      | Sticky Note                                                                                      |
|--------------------|----------------------------------|---------------------------------------|----------------------|---------------------|------------------------------------------------------------------------------------------------|
| Sticky Note        | stickyNote                       | Documentation and user guidance       | None                 | None                | ![](https://raw.githubusercontent.com/2innnnn0/30-Days-of-ChatGPT/refs/heads/main/datapopcorn_logo_50px.png) # Daily Cartoon (w/ AI Translate) ... |
| Schedule Trigger   | scheduleTrigger                  | Triggers workflow daily at 9 AM       | None                 | param               |                                                                                                |
| param              | set                             | Sets date parameters (year, month, day) | Schedule Trigger     | HTTP Request        |                                                                                                |
| HTTP Request       | httpRequest                     | Retrieves comic HTML page              | param                 | Information Extractor |                                                                                                |
| Information Extractor | @n8n/n8n-nodes-langchain.informationExtractor | Extracts comic image URL from HTML    | HTTP Request          | OpenAI              |                                                                                                |
| OpenAI             | @n8n/n8n-nodes-langchain.openAi | Analyzes comic image and translates   | Information Extractor | Discord             |                                                                                                |
| OpenAI Chat Model  | @n8n/n8n-nodes-langchain.lmChatOpenAi | (Available but not connected)         | None                  | None                |                                                                                                |
| Discord            | discord                         | Posts comic and translation to Discord | OpenAI                | None                |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Set to trigger daily at 9:00 AM (hour 9).  
   - No credentials needed.

2. **Create a Set Node named "param":**  
   - Add three string parameters:  
     - `year` with expression: `{{$now.format('yyyy')}}`  
     - `month` with expression: `{{$now.format('MM')}}`  
     - `day` with expression: `{{$now.format('dd')}}`  
   - Connect Schedule Trigger’s output to this node.

3. **Create an HTTP Request Node:**  
   - Configure method to GET (default).  
   - Set URL to: `https://www.gocomics.com/calvinandhobbes/{{$json.year}}/{{$json.month}}/{{$json.day}}`  
   - Connect "param" node output to this node.

4. **Create an Information Extractor Node:**  
   - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
   - Prompt:  
     ```
     Please just extract the src value in the <img class="img-fluid Lazyloaded"> tag from HTML below. I don't need anything other than the value.

     e.g.)
     EXAMPLE INPUT)
     <img class="img-fluid lazyloaded" srcset="..." data-srcset="..." sizes="..." width="100%" alt="Calvin and Hobbes Comic Strip for March 03, 2023 " src="https://assets.amuniversal.com/5ed526b06e94013bda88005056a9545d">

     EXAMPLE OUTPUT)
     https://assets.amuniversal.com/5ed526b06e94013bda88005056a9545d

     --
     (INPUT)
     {{ $json.data }}
     ```  
   - Output attribute name: `cartoon_image`  
   - Connect HTTP Request output to this node.

5. **Create an OpenAI Node for Image Analysis and Translation:**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Credentials: Add OpenAI API credentials with valid API key.  
   - Parameters:  
     - Text prompt:  
       ```
       Please write the original language and Korean together. 

       EXAMPLE)
       Calvin: "YOU'VE NEVER HAD AN OBLIGATION, AN ASSIGNMENT, OR A DEADLINE IN ALL YOUR LIFE! YOU HAVE NO RESPONSIBILITIES AT ALL! IT MUST BE NICE!" (너는 평생 한 번도 의무, 과제, 혹은 마감일 없었잖아! 전혀 책임이 없다니! 정말 좋겠다!)
       Hobbes: "WIPE THAT INSOLENT SMIRK OFF YOUR FACE!" (그 뻔뻔한 미소를 그만 지어!)
       ```  
     - Model ID: `gpt-4o-mini`  
     - Operation: `analyze`  
     - Resource: `image`  
     - Image URLs: `={{ $json.output.cartoon_image }}` (from Information Extractor)  
   - Connect Information Extractor output to this node.

6. **Create a Discord Node:**  
   - Type: `discord`  
   - Credentials: Configure Discord Webhook API with a valid webhook URL for your Discord channel.  
   - Parameters:  
     - Content:  
       ```
       Daily Cartoon ({{$json.year}}/{{$json.month}}/{{$json.day}})
       {{$json.output.cartoon_image}}

       {{$json.content}}
       ```  
   - Connect OpenAI node output to this node.

7. **Create a Sticky Note Node:**  
   - Add markdown content describing workflow purpose and setup steps (can copy from provided documentation content).  
   - Position anywhere for visibility.  
   - No connections needed.

**Connect nodes in this order:**  
Schedule Trigger → param → HTTP Request → Information Extractor → OpenAI → Discord

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow uses n8n’s LangChain OpenAI nodes for advanced AI-powered extraction and translation. Make sure to configure OpenAI API credentials with sufficient quota.                                                                                       | OpenAI API documentation: https://platform.openai.com/docs/api-reference                      |
| Discord webhook must be configured with appropriate permissions to post messages in the target channel.                                                                                                                                                   | Discord Webhooks: https://discord.com/developers/docs/resources/webhook                        |
| The comic source website (gocomics.com) may change its HTML structure, which can break the extraction logic. Regularly verify the Information Extractor prompt and update if necessary.                                                                   | https://www.gocomics.com/calvinandhobbes/                                                     |
| The workflow assumes server timezone aligns with intended publication time. Adjust expressions or server time accordingly if needed.                                                                                                                    | n8n timezone settings: https://docs.n8n.io/nodes/trigger-nodes/schedule-trigger/               |
| ![](https://raw.githubusercontent.com/2innnnn0/30-Days-of-ChatGPT/refs/heads/main/datapopcorn_logo_50px.png) Project and branding by datapopcorn.                                                                                                         | https://github.com/2innnnn0/30-Days-of-ChatGPT                                                |

---

This documentation comprehensively describes the workflow’s structure, logic, and setup, enabling robust reproduction and maintenance.