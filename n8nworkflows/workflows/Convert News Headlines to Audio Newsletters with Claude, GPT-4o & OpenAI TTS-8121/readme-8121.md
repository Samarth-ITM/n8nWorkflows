Convert News Headlines to Audio Newsletters with Claude, GPT-4o & OpenAI TTS

https://n8nworkflows.xyz/workflows/convert-news-headlines-to-audio-newsletters-with-claude--gpt-4o---openai-tts-8121


# Convert News Headlines to Audio Newsletters with Claude, GPT-4o & OpenAI TTS

### 1. Workflow Overview

This workflow automates the transformation of daily news headlines into personalized audio newsletters. It is designed for content creators, busy professionals, podcasters, and teams who want daily news briefings in audio format delivered via email.

**Target Use Cases:**  
- Automated creation of audio news summaries  
- AI-driven content rewriting and script generation  
- Text-to-speech conversion for natural audio newsletters  
- Scheduled email delivery of audio newsletters  

**Logical Blocks:**  
- **1.1 Schedule Trigger:** Initiates the workflow on a daily schedule.  
- **1.2 News Retrieval:** Fetches the latest US news headlines from NewsAPI.  
- **1.3 Data Extraction:** Splits and extracts individual news articles for processing.  
- **1.4 AI Content Processing:** Uses Claude AI to rewrite each article into a newsletter format.  
- **1.5 Aggregation:** Combines all rewritten news items into a single collection.  
- **1.6 Script Generation:** Uses GPT-4o-mini to create a 2-minute audio-ready script from the aggregated news.  
- **1.7 Text-to-Speech Conversion:** Converts the script into natural-sounding audio using OpenAI TTS.  
- **1.8 Email Delivery:** Sends the audio newsletter as an email attachment to subscribers.  

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:**  
Triggers the workflow automatically at a specified daily interval to start the news processing.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - Type: Schedule Trigger  
  - Configuration: Runs on a daily interval (default unspecified but customizable)  
  - Inputs: None (trigger node)  
  - Outputs: Initiates the "Get Latest News" node  
  - Edge Cases: Misconfigured schedule or disabled workflow will prevent execution.

#### 1.2 News Retrieval

- **Overview:**  
Retrieves the latest US news headlines from NewsAPI using a REST HTTP request.

- **Nodes Involved:**  
  - Get Latest News

- **Node Details:**  
  - Type: HTTP Request  
  - Configuration: GET request to `https://newsapi.org/v2/top-headlines?country=us&apiKey=YOUR_NEWSAPI_KEY`  
  - Key Parameters: Replace `YOUR_NEWSAPI_KEY` with a valid NewsAPI key  
  - Inputs: Triggered by Schedule Trigger  
  - Outputs: JSON response containing news articles, passed to "Split Out"  
  - Edge Cases: API key missing/invalid, rate limits, network failures, empty or malformed responses

#### 1.3 Data Extraction

- **Overview:**  
Splits the JSON response to process each news article individually.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**  
  - Type: Split Out  
  - Configuration: Splits the `articles` array from the API response into separate items  
  - Inputs: Output from "Get Latest News"  
  - Outputs: Individual news article objects forwarded to "Daily News Extractor"  
  - Edge Cases: Empty `articles` array results in no downstream processing; malformed JSON

#### 1.4 AI Content Processing

- **Overview:**  
Uses Claude AI to rewrite each news article into a polished newsletter post, blending in author and date details naturally.

- **Nodes Involved:**  
  - Daily News Extractor  
  - Model (Claude AI)

- **Node Details:**

  - **Daily News Extractor**  
    - Type: LangChain Agent  
    - Configuration: Template prompt includes author, title, description, content, and publish date details; system message instructs rewriting into newsletter style with date and author blended in  
    - Inputs: Individual news article JSON from "Split Out"  
    - Outputs: Rewritten newsletter posts for each article going to "Aggregate"  
    - Edge Cases: Missing article fields (author, content, date) could affect output quality; AI rate limits or errors  

  - **Model**  
    - Type: LangChain LM Chat OpenRouter (Claude)  
    - Configuration: Uses Claude-3.7-sonnet model via OpenRouter credentials  
    - Inputs: Connected to "Daily News Extractor" as its language model backend  
    - Outputs: AI-generated rewritten content  
    - Edge Cases: API key/auth errors, throttling, model unavailability

#### 1.5 Aggregation

- **Overview:**  
Combines all rewritten newsletter posts into a single aggregated list for further processing.

- **Nodes Involved:**  
  - Aggregate

- **Node Details:**  
  - Type: Aggregate  
  - Configuration: Aggregates field `output` from "Daily News Extractor" into a single array named `news`  
  - Inputs: Multiple rewritten news items from "Daily News Extractor"  
  - Outputs: Aggregated news array to "Newsletter Agent"  
  - Edge Cases: No input items results in empty aggregation; incorrect field mapping

#### 1.6 Script Generation

- **Overview:**  
Transforms the aggregated news posts into a cohesive 2-minute script suitable for audio narration.

- **Nodes Involved:**  
  - Newsletter Agent

- **Node Details:**  
  - Type: LangChain OpenAI Node  
  - Configuration: Uses GPT-4o-mini model; prompt instructs rewriting the newsletter into a 2-minute audio script avoiding special characters; includes a scripted intro ("Max here brings you...")  
  - Inputs: Aggregated news JSON from "Aggregate"  
  - Outputs: Script text for transcription passed to "Transcribe Newsletter"  
  - Edge Cases: Large input size might exceed token limits; API errors; prompt misinterpretation

#### 1.7 Text-to-Speech Conversion

- **Overview:**  
Converts the generated script text into natural-sounding audio using OpenAI’s text-to-speech resource.

- **Nodes Involved:**  
  - Transcribe Newsletter

- **Node Details:**  
  - Type: LangChain OpenAI Node (audio resource)  
  - Configuration: Input set to the script content; uses OpenAI API credentials for TTS  
  - Inputs: Script text from "Newsletter Agent"  
  - Outputs: Audio binary data sent to "Notify Subscriber"  
  - Edge Cases: API quota, unsupported characters causing TTS errors, audio generation failures

#### 1.8 Email Delivery

- **Overview:**  
Sends the generated audio newsletter as an email attachment to a predefined subscriber list.

- **Nodes Involved:**  
  - Notify Subscriber

- **Node Details:**  
  - Type: Gmail Node  
  - Configuration:  
    - Sends to `YOUR_EMAIL@example.com` (replace with actual recipient)  
    - Subject includes current date  
    - Body is a polite message referencing the news date  
    - Attaches audio binary from "Transcribe Newsletter"  
    - Uses Gmail OAuth2 credentials  
  - Inputs: Audio binary from "Transcribe Newsletter"  
  - Outputs: None (end node)  
  - Edge Cases: Invalid recipient email, OAuth token expiry, Gmail API quota limits, attachment size limits

---

### 3. Summary Table

| Node Name           | Node Type                            | Functional Role                        | Input Node(s)         | Output Node(s)         | Sticky Note                                                   |
|---------------------|------------------------------------|-------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger                   | Triggers workflow daily              | —                     | Get Latest News        | Step 1: Daily News Schedule Automatically triggers news collection at your chosen time. |
| Get Latest News      | HTTP Request                      | Fetches latest US news headlines     | Schedule Trigger       | Split Out              | Step 2: Fetch News Headlines Retrieves latest news from NewsAPI. |
| Split Out           | Split Out                         | Splits articles array into items     | Get Latest News        | Daily News Extractor   | Step 3: Data Extraction Extract The important details from the API |
| Daily News Extractor | LangChain Agent                   | Rewrites news articles to newsletter | Split Out              | Aggregate              | Step 4: AI Content Processing Claude AI rewrites news articles into newsletter format. |
| Model                | LangChain LM Chat OpenRouter     | Claude AI language model              | Daily News Extractor   | Daily News Extractor   | Step 4: AI Content Processing Claude AI rewrites news articles into newsletter format. |
| Aggregate            | Aggregate                        | Combines rewritten news items        | Daily News Extractor   | Newsletter Agent       | Step 5: Data Transformation Merge all the News into a single list |
| Newsletter Agent     | LangChain OpenAI (GPT-4o-mini)   | Generates 2-minute audio script       | Aggregate              | Transcribe Newsletter  | Step 6: Script Generation GPT-4 creates 2-minute audio-ready script. |
| Transcribe Newsletter| LangChain OpenAI (Audio resource) | Converts script text to speech audio  | Newsletter Agent       | Notify Subscriber      | Step 7: Text-to-Speech Conversion OpenAI converts script to natural-sounding audio. |
| Notify Subscriber    | Gmail                            | Sends email with audio attachment    | Transcribe Newsletter  | —                      | Step 8: Email Delivery Sends audio newsletter to subscribers. |
| Sticky Note          | Sticky Note                      | Documentation                       | —                     | —                      | 📰 Automated Daily News to Audio Newsletter. Workflow overview and setup instructions. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run daily at desired time (default is daily interval)  
   - No credentials needed

2. **Create Get Latest News node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://newsapi.org/v2/top-headlines?country=us&apiKey=YOUR_NEWSAPI_KEY` (replace API key)  
   - No authentication (API key in URL)  
   - Connect Schedule Trigger → Get Latest News

3. **Create Split Out node**  
   - Type: Split Out  
   - Field to split out: `articles`  
   - Connect Get Latest News → Split Out

4. **Create Model node (Claude AI)**  
   - Type: LangChain LM Chat OpenRouter  
   - Model: `anthropic/claude-3.7-sonnet`  
   - Credentials: Configure OpenRouter API credentials with your account  
   - No direct connection here; used internally by "Daily News Extractor"

5. **Create Daily News Extractor node**  
   - Type: LangChain Agent  
   - Text template:  
     ```
     News Details:
     Author -  {{ $json.author }}
     title - {{ $json.title }}
     description - {{ $json.description }}
     content - {{ $json.content }}
     publish date - {{ $json.publishedAt }}
     ```
   - System message prompt:  
     ```
     take the details and rewrite a newsletter post

     include the date, and author too, let it blend in
     ```
   - Prompt type: define  
   - Connect Split Out → Daily News Extractor  
   - Assign Model node as the LM for this agent

6. **Create Aggregate node**  
   - Type: Aggregate  
   - Field to aggregate: `output`  
   - Rename output field to `news`  
   - Connect Daily News Extractor → Aggregate

7. **Create Newsletter Agent node**  
   - Type: LangChain OpenAI  
   - Model: `gpt-4o-mini` (select from model list)  
   - Credentials: Configure OpenAI API key  
   - Messages:  
     - User message: `={{ $json.news.toJsonString() }}`  
     - Assistant message:  
       ```
       Rewrite all this newsletter into a 2-minute script that will be transcribe into audio 

       dont put spcial characters, because a audio will transcribe the text to audio

       you can start with Max here bring you the top new around the world
       ```
   - Connect Aggregate → Newsletter Agent

8. **Create Transcribe Newsletter node**  
   - Type: LangChain OpenAI  
   - Resource: Audio (text-to-speech)  
   - Input: `={{ $json.message.content }}` (script text from Newsletter Agent)  
   - Credentials: Use same OpenAI API credentials  
   - Connect Newsletter Agent → Transcribe Newsletter

9. **Create Notify Subscriber node**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials  
   - Send To: Replace with actual recipient email  
   - Subject: `=Top News Headline for {{ $now.format('yyyy-MM-dd') }}`  
   - Message:  
     ```
     Dear Sir,

     Kindly Find the Latest News for {{ $now.format('yyyy-MM-dd') }}

     Regards,
     Max,
     Your business name
     ```
   - Attachments: Use binary data from Transcribe Newsletter  
   - Connect Transcribe Newsletter → Notify Subscriber

10. **Test and validate the full flow**  
    - Replace all placeholder API keys and email addresses  
    - Ensure API quota and credentials are valid  
    - Adjust schedule timing and prompts as needed  
    - Confirm email delivery and audio quality  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 📰 Automated Daily News to Audio Newsletter. Workflow transforms daily news headlines into personalized audio newsletters automatically with AI rewriting, script generation, text-to-speech, and email delivery.                   | Workflow overview sticky note                                                                    |
| Setup instructions include obtaining NewsAPI key, configuring OpenRouter and OpenAI credentials, updating recipient email, and customizing prompts.                                                                             | Workflow overview sticky note                                                                    |
| Ideal for content creators, professionals, podcasters, internal team updates, and AI-powered media automation.                                                                                                                  | Workflow overview sticky note                                                                    |
| Stepwise sticky notes detail each workflow step, from scheduling, fetching news, data extraction, AI processing, aggregation, script generation, TTS conversion, to email sending.                                                | Sticky notes attached to respective node groups                                                  |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.