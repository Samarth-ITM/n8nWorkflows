Generate Trend-Based Marketing Videos with Seedance AI, Perplexity, and GPT-4o

https://n8nworkflows.xyz/workflows/generate-trend-based-marketing-videos-with-seedance-ai--perplexity--and-gpt-4o-5674


# Generate Trend-Based Marketing Videos with Seedance AI, Perplexity, and GPT-4o

### 1. Workflow Overview

This n8n workflow automates the generation of trend-based short-form marketing videos using AI services and a video generation API, triggered by user prompts via Telegram. It is designed for marketers, founders, and content creators who want to transform fresh marketing trends into engaging video ads for platforms like Instagram and TikTok without manual video editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives user marketing briefs through Telegram messages.
- **1.2 Trend Research:** Uses Perplexity AI (Sonar Pro model) to identify the most relevant and recent marketing trend related to the user’s input.
- **1.3 Video Prompt Engineering:** Employs OpenAI’s GPT-4o to craft a concise, visually descriptive video prompt based on the trend and user input.
- **1.4 Video Generation Request:** Sends the generated prompt to the Seedance video generation API via Wavespeed to create a 5-second marketing video.
- **1.5 Video Status Polling and Delivery:** Periodically checks the video generation status and, once complete, sends the final video URL back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initiates the workflow by capturing user input from Telegram. The input is expected to be a marketing brief, e.g., “Create a 5-second Instagram ad for my perfume brand.”
- **Nodes Involved:**  
  - Telegram Trigger
- **Node Details:**

| Node Name       | Telegram Trigger                                |
|-----------------|------------------------------------------------|
| Type            | Telegram Trigger node                           |
| Role            | Listens for incoming Telegram messages         |
| Configuration   | Watches for "message" updates; uses Telegram API credentials configured externally |
| Variables Used  | `$json.message.text` contains user input text  |
| Input          | External Telegram message webhook                |
| Output         | Emits the message JSON to Trend Research Agent  |
| Version        | 1.2                                             |
| Failure Types  | Telegram API auth errors, webhook failures, malformed input text |
| Sub-workflow   | None                                            |

---

#### 2.2 Trend Research

- **Overview:** Queries Perplexity AI (Sonar Pro model) to find the top recent marketing trend related to the user’s prompt, focusing on trends from the last 14 days that are actionable for video marketing.
- **Nodes Involved:**  
  - Trend Research Agent (Perplexity API)
- **Node Details:**

| Node Name        | Trend Research Agent                             |
|------------------|-------------------------------------------------|
| Type             | Perplexity AI node                              |
| Role             | Fetches a relevant recent marketing trend based on user input |
| Configuration    | Uses the "sonar-pro" model; system prompt instructs to find one top trend with a source URL, concise and visual-oriented output; input is dynamic user message text |
| Variables Used   | Input prompt via `={{ $json.message.text }}`  |
| Input            | Output from Telegram Trigger                    |
| Output           | Trend research results JSON to Video Prompt Engineer |
| Version          | 1                                               |
| Failure Types    | API auth failure, timeout, unexpected response format, no relevant trends found |
| Sub-workflow     | None                                            |

---

#### 2.3 Video Prompt Engineering

- **Overview:** Uses OpenAI GPT-4o to convert the trend insight and user brief into a short, descriptive video prompt suitable for the Seedance API.
- **Nodes Involved:**  
  - Video Prompt Engineer (OpenAI GPT-4o)
- **Node Details:**

| Node Name         | Video Prompt Engineer                            |
|-------------------|-------------------------------------------------|
| Type              | OpenAI GPT-4o (Langchain node)                  |
| Role              | Generates a creative video prompt under 100 words based on user brief and trend insight |
| Configuration     | System prompt directs to produce a concise, visual, trend-integrated video prompt without extra commentary; uses chat model "chatgpt-4o-latest"; input is the trend research output |
| Variables Used    | Input prompt: `={{ $json.choices[0].message.content }}` |
| Input             | Output from Trend Research Agent                 |
| Output            | Video prompt string to Post Request - Wavespeed |
| Version           | 1.8                                              |
| Failure Types     | OpenAI API quota limits, malformed inputs, response parsing errors |
| Sub-workflow      | None                                             |

---

#### 2.4 Video Generation Request

- **Overview:** Sends a POST request to Wavespeed API, which interfaces with Seedance, to generate a 5-second video based on the crafted prompt.
- **Nodes Involved:**  
  - Post Request - Wavespeed
- **Node Details:**

| Node Name         | Post Request - Wavespeed                         |
|-------------------|-------------------------------------------------|
| Type              | HTTP Request node                               |
| Role              | Sends video generation request including prompt, duration, and seed to Wavespeed API (Seedance) |
| Configuration     | POST to `https://api.wavespeed.ai/api/v3/bytedance/seedance-v1-pro-i2v-1080p` with JSON body containing duration=5, prompt from previous node, image (empty), seed=-1; uses HTTP header authentication |
| Variables Used    | Prompt: `={{ $json.message.content }}`          |
| Input             | Output from Video Prompt Engineer                |
| Output            | JSON response with video generation status and ID to Wait node |
| Version           | 4.2                                              |
| Failure Types     | API authentication failure, HTTP errors, invalid prompt errors, rate limiting |
| Sub-workflow      | None                                             |

---

#### 2.5 Video Status Polling and Delivery

- **Overview:** Periodically polls the Wavespeed API to check if the video generation is complete. When the video is ready, sends the video URL back to the user on Telegram.
- **Nodes Involved:**  
  - Wait 30 Sec  
  - GET Request Wavespeed  
  - If  
  - Wait 30 sec (second wait node)  
  - Send a text message (Telegram output)
- **Node Details:**

| Node Name          | Wait 30 Sec                                     |
|--------------------|-------------------------------------------------|
| Type               | Wait node                                       |
| Role               | Pauses workflow for 30 seconds before polling video status |
| Configuration      | Wait time set to 30 seconds                      |
| Input              | Output from Post Request - Wavespeed            |
| Output             | Triggers GET Request Wavespeed                   |
| Version            | 1.1                                              |
| Failure Types      | Execution timeout if node is disabled or interrupted |
| Sub-workflow       | None                                             |

| Node Name          | GET Request Wavespeed                            |
|--------------------|-------------------------------------------------|
| Type               | HTTP Request node                               |
| Role               | Queries video generation result from Wavespeed API using job ID |
| Configuration      | GET request to `https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result` with HTTP header auth |
| Variables Used     | URL path parameter: `{{ $json.data.id }}` from previous node response |
| Input              | Output from Wait 30 Sec                          |
| Output             | JSON response containing video status and output URL |
| Version            | 4.2                                              |
| Failure Types      | API timeout, authentication failure, invalid ID, JSON parse errors |
| Sub-workflow       | None                                             |

| Node Name          | If                                              |
|--------------------|-------------------------------------------------|
| Type               | If node                                         |
| Role               | Checks if the video status is "processing" or completed |
| Configuration      | Condition: `$json.data.status == "processing"` |
| Input              | Output from GET Request Wavespeed                |
| Output             | If true: Wait 30 sec (retry polling)  
                      If false: Send a text message with video URL |
| Version            | 2.2                                              |
| Failure Types      | Expression evaluation errors                      |
| Sub-workflow       | None                                             |

| Node Name          | Wait 30 sec (second wait)                        |
|--------------------|-------------------------------------------------|
| Type               | Wait node                                       |
| Role               | Waits 30 seconds before retrying video status check |
| Configuration      | Wait time set to 30 seconds                      |
| Input              | True output from If node                         |
| Output             | Loops back to GET Request Wavespeed              |
| Version            | 1.1                                              |
| Failure Types      | Execution timeout                                 |
| Sub-workflow       | None                                             |

| Node Name          | Send a text message (Telegram output)            |
|--------------------|-------------------------------------------------|
| Type               | Telegram node                                   |
| Role               | Sends the final video URL back to Telegram user |
| Configuration      | Sends text: `={{ $json.data.outputs[0] }}` to chat ID from original Telegram message |
| Variables Used     | Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}` |
| Input              | False output from If node (video ready)          |
| Output             | None (end of workflow)                            |
| Version            | 1.2                                              |
| Failure Types      | Telegram API auth failure, message send errors   |
| Sub-workflow       | None                                             |

---

### 3. Summary Table

| Node Name              | Node Type                 | Functional Role                       | Input Node(s)           | Output Node(s)           | Sticky Note                             |
|------------------------|---------------------------|------------------------------------|------------------------|--------------------------|---------------------------------------|
| Telegram Trigger       | Telegram Trigger           | Receives user prompt from Telegram  | -                      | Trend Research Agent      | Telegram Trigger                      |
| Trend Research Agent   | Perplexity AI              | Finds top recent marketing trend    | Telegram Trigger       | Video Prompt Engineer     | Perplexity Research Agent             |
| Video Prompt Engineer  | OpenAI GPT-4o (Langchain) | Creates video prompt from trend     | Trend Research Agent   | Post Request - Wavespeed  | Video Prompt Agent                    |
| Post Request - Wavespeed | HTTP Request             | Sends prompt to Seedance API        | Video Prompt Engineer  | Wait 30 Sec              | POST Request to Wavespeed             |
| Wait 30 Sec           | Wait                      | Waits 30 seconds before polling     | Post Request - Wavespeed | GET Request Wavespeed    | GET Request Loop                     |
| GET Request Wavespeed  | HTTP Request              | Polls video generation status       | Wait 30 Sec            | If                       | GET Request Loop                     |
| If                     | If                        | Checks video generation status      | GET Request Wavespeed  | Wait 30 sec, Send a text message | GET Request Loop               |
| Wait 30 sec            | Wait                      | Waits 30 seconds before retry       | If (true path)         | GET Request Wavespeed     | GET Request Loop                     |
| Send a text message    | Telegram                  | Sends video URL back to user        | If (false path)        | -                        | Telegram Output                      |
| Sticky Note            | Sticky Note               | Annotation                        | -                      | -                        | 🎬 Seedance Video Marketing AI Agent (overview) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure to listen for “message” updates only.  
   - Use your Telegram API credentials (OAuth or Bot Token).  
   - Position: Start of the workflow.  

2. **Create Trend Research Agent Node**  
   - Type: Perplexity AI node  
   - Model: `sonar-pro`  
   - System prompt: Provide instructions to identify one top trend from past 14 days related to user input, including source URL and visual marketing angle.  
   - Input message: Use expression referencing the Telegram Trigger message text (`={{ $json.message.text }}`).  
   - Use your Perplexity API credentials.  
   - Connect Telegram Trigger output to this node’s input.  

3. **Add Video Prompt Engineer Node**  
   - Type: OpenAI GPT-4o Langchain node  
   - Model: `chatgpt-4o-latest`  
   - System message: Instruct to generate a concise, visual, trend-based video prompt under 100 words for Seedance API.  
   - Input message: Use the content of the trend research output (`={{ $json.choices[0].message.content }}`).  
   - Use your OpenAI API credentials.  
   - Connect Trend Research Agent output to this node’s input.  

4. **Add Post Request - Wavespeed Node**  
   - Type: HTTP Request (POST)  
   - URL: `https://api.wavespeed.ai/api/v3/bytedance/seedance-v1-pro-i2v-1080p`  
   - Method: POST  
   - Authentication: HTTP Header Auth with Wavespeed API key  
   - Body Parameters:  
     - `duration` = 5  
     - `prompt` = prompt from Video Prompt Engineer (`={{ $json.message.content }}`)  
     - `image` = (empty)  
     - `seed` = -1  
   - Connect Video Prompt Engineer output to this node.  

5. **Add Wait 30 Sec Node**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect output of Post Request - Wavespeed node to this node.  

6. **Add GET Request Wavespeed Node**  
   - Type: HTTP Request (GET)  
   - URL: `https://api.wavespeed.ai/api/v3/predictions/{{ $json.data.id }}/result`  
   - Authentication: HTTP Header Auth with Wavespeed API key (same as POST)  
   - Connect Wait 30 Sec node output to this node.  

7. **Add If Node**  
   - Type: If  
   - Condition: Check if `{{$json.data.status}}` equals `"processing"`  
   - Connect GET Request Wavespeed output to this node.  

8. **Add Wait 30 sec Node (retry wait)**  
   - Type: Wait  
   - Duration: 30 seconds  
   - Connect If node’s "true" output (status == processing) to this node.  

9. **Loop Back to GET Request**  
   - Connect Wait 30 sec (retry wait) output back to GET Request Wavespeed node to poll status repeatedly.  

10. **Add Send a text message Node**  
    - Type: Telegram  
    - Text: Use the video URL from Wavespeed output (`={{ $json.data.outputs[0] }}`)  
    - Chat ID: Use original Telegram message chat ID (`={{ $('Telegram Trigger').item.json.message.chat.id }}`)  
    - Use Telegram API credentials.  
    - Connect If node’s "false" output (status != processing) to this node.  

11. **Add Sticky Notes for Clarity** (optional)  
    - Add sticky notes annotating each block for easier maintenance and explanation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                              | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 🎬 Seedance Video Marketing AI Agent: Automates marketing video generation from Telegram prompts using Perplexity AI, GPT-4o, and Seedance API via Wavespeed. Designed for marketers to scale content creation with zero video editing.                                                                                                                                      | Full workflow description and use cases.                                                       |
| Step-by-step video tutorial on building this workflow is available at: https://youtu.be/2oZ5NhosKgo                                                                                                                                                                                                                                                                       | Video tutorial link for workflow construction by the author.                                   |
| Use cases include auto-generating TikTok/Instagram ads from fresh trends, scaling creative content, integrating with marketing chatbots, and using trends as visual inspiration.                                                                                                                                                                                          | Marketing and automation use case overview.                                                    |
| Tools used: Telegram Trigger & Telegram node, Perplexity AI (Sonar Pro), OpenAI GPT-4o, Wavespeed API (Seedance), HTTP Request, Wait, and If nodes.                                                                                                                                                                                                                       | Technology stack and API services summary.                                                     |
| Ensure all credentials (Telegram API, Perplexity API, OpenAI API, Wavespeed API) are properly configured with correct scopes and token validity to avoid auth issues.                                                                                                                                                                                                    | Credential setup best practice.                                                                |
| When polling video status, consider API rate limits and implement backoff or max retries if integrating or scaling.                                                                                                                                                                                                                                                      | Potential integration edge case and error handling note.                                       |
| Sticky notes in the workflow visually separate logical blocks and aid maintainability.                                                                                                                                                                                                                                                                                    | Workflow documentation and maintenance tip.                                                   |

---

_Disclaimer: The provided text is exclusively derived from an automated workflow designed with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected materials. All data handled is legal and public._