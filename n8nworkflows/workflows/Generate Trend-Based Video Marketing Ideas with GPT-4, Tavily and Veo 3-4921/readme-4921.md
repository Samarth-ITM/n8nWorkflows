Generate Trend-Based Video Marketing Ideas with GPT-4, Tavily and Veo 3

https://n8nworkflows.xyz/workflows/generate-trend-based-video-marketing-ideas-with-gpt-4--tavily-and-veo-3-4921


# Generate Trend-Based Video Marketing Ideas with GPT-4, Tavily and Veo 3

### 1. Workflow Overview

This workflow automates the daily generation and delivery of trend-based marketing video ideas for an e-commerce brand, Sally’s Closet, which sells women's wear and accessories online. It leverages AI (GPT-4) and real-time web trend research tools (Tavily), then formats creative video prompts tailored for Veo 3, an AI video generation platform, and finally emails the generated video link.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger:** Automatically starts the workflow daily at a set time.
- **1.2 Trend Research and Idea Generation:** Uses Tavily to gather recent trending topics, then GPT-4 generates a creative marketing video idea linking trends to Sally’s Closet products.
- **1.3 Video Prompt Creation:** GPT-4 converts the video idea into a cinematic-style video prompt optimized for Veo 3.
- **1.4 Video Request Submission:** Sends the formatted video prompt to Veo 3 via the FAL API to request video generation.
- **1.5 Video Generation Status Check Loop:** Waits and periodically checks the video generation status until the video is ready.
- **1.6 Deliver Video Link:** Once complete, sends the final video URL via Gmail to the marketing team.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow once daily at a specified hour to kick off the automated video marketing idea generation process.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - **Type:** Schedule Trigger node  
    - **Role:** Starts the workflow daily at 10:00 AM.  
    - **Configuration:** Configured to trigger at hour 10 every day.  
    - **Inputs:** None (start node)  
    - **Outputs:** Triggers "Idea Gen Agent (research)" node.  
    - **Edge Cases:** If n8n server is down or stopped, the trigger will not fire; no retries.  

---

#### 1.2 Trend Research and Idea Generation

- **Overview:**  
  This block collects recent trending topics related to the brand using Tavily’s web search, then generates a creative, trend-based marketing video idea using GPT-4.

- **Nodes Involved:**  
  - Idea Gen Agent (research)  
  - Tavily

- **Node Details:**  
  - **Tavily**  
    - **Type:** Tavily web search node  
    - **Role:** Searches for trending topics, viral moments, or news relevant to Sally’s Closet within the last 24-48 hours.  
    - **Configuration:** The query is dynamically set based on AI input (empty string override placeholder).  
    - **Inputs:** Receives AI-generated input (via ai_tool connection).  
    - **Outputs:** Passes search results to "Idea Gen Agent (research)" node via ai_tool output.  
    - **Edge Cases:**  
        - Web search may fail or return no relevant results.  
        - Network or API errors can cause failure or empty output.  
  - **Idea Gen Agent (research)**  
    - **Type:** Langchain OpenAI GPT-4 node  
    - **Role:** Acts as a creative marketing strategist generating daily marketing video ideas that tie trending topics to products.  
    - **Configuration:**  
      - Model: GPT-4  
      - System prompt instructs the system to: research trends using Tavily, creatively link trends to products, and output a structured video idea with title, inspiration, connection, hook, and CTA.  
    - **Inputs:** Receives Tavily search results via ai_tool input.  
    - **Outputs:** Sends generated marketing video idea to "Video Prompt Agent."  
    - **Key Expressions:** Uses a complex system prompt embedding brand details and output format instructions.  
    - **Edge Cases:**  
      - GPT-4 may return incomplete or off-topic ideas.  
      - Rate limits or API errors can cause failures.

---

#### 1.3 Video Prompt Creation

- **Overview:**  
  Converts the generated marketing video idea into a concise, cinematic video prompt tailored for Veo 3’s AI video generation engine.

- **Nodes Involved:**  
  - Video Prompt Agent

- **Node Details:**  
  - **Video Prompt Agent**  
    - **Type:** Langchain OpenAI GPT-4 node  
    - **Role:** Transforms a user’s short video concept into a vivid, under-100-words video prompt including scene description, sensory language, camera angles, and branding cues.  
    - **Configuration:**  
      - Model: GPT-4  
      - System prompt instructs to create cinematic-style prompts with specific formatting and tone styles.  
      - Input message: The content of the video idea from the previous node.  
    - **Inputs:** Receives creative video idea text from "Idea Gen Agent (research)".  
    - **Outputs:** Sends formatted prompt to "FAL Veo 3 Post Request."  
    - **Edge Cases:**  
      - GPT-4 might produce prompts that are too long or vague.  
      - API failures or timeouts possible.

---

#### 1.4 Video Request Submission

- **Overview:**  
  Sends the formatted video prompt to the FAL API Veo 3 endpoint to request video generation.

- **Nodes Involved:**  
  - FAL Veo 3 Post Request

- **Node Details:**  
  - **FAL Veo 3 Post Request**  
    - **Type:** HTTP Request node  
    - **Role:** Posts the Veo 3 video prompt to the external API to queue the video generation job.  
    - **Configuration:**  
      - Method: POST  
      - URL: https://queue.fal.run/fal-ai/veo3  
      - Body: JSON with key "prompt" containing the video prompt content.  
      - Authentication: Generic HTTP Header Auth (credentials needed)  
    - **Inputs:** Receives cinematic prompt from "Video Prompt Agent."  
    - **Outputs:** Passes response containing request ID to "Wait" node.  
    - **Edge Cases:**  
      - API authentication errors or invalid credentials.  
      - Network timeouts or API rate limits.  
      - Invalid prompt format causing rejection.

---

#### 1.5 Video Generation Status Check Loop

- **Overview:**  
  Implements a polling loop that waits then checks the video generation status repeatedly until the video is ready.

- **Nodes Involved:**  
  - Wait  
  - Get Video Status from FAL / Veo 3  
  - If

- **Node Details:**  
  - **Wait**  
    - **Type:** Wait node  
    - **Role:** Pauses workflow for 90 seconds to allow video rendering progress.  
    - **Configuration:** Fixed 90 seconds delay.  
    - **Inputs:** Receives video request ID from "FAL Veo 3 Post Request" or from the "If" node when status is not completed.  
    - **Outputs:** Triggers "Get Video Status from FAL / Veo 3."  
    - **Edge Cases:** Workflow may be delayed if wait time is too short or too long for average video generation time.  
  - **Get Video Status from FAL / Veo 3**  
    - **Type:** HTTP Request node  
    - **Role:** Queries the FAL API to fetch current status of the video generation request by request_id.  
    - **Configuration:**  
      - Method: GET  
      - URL: https://queue.fal.run/fal-ai/veo3/requests/{{request_id}}/status  
      - Authentication: Generic HTTP Header Auth (same credentials)  
    - **Inputs:** Receives request_id from "Wait."  
    - **Outputs:** Sends status JSON to "If" node.  
    - **Edge Cases:** API errors, invalid request_id, or network failures.  
  - **If**  
    - **Type:** If node  
    - **Role:** Checks if video generation status equals "COMPLETED."  
    - **Configuration:** Condition: `$json.status === "COMPLETED"`  
    - **Inputs:** Receives status from "Get Video Status from FAL / Veo 3."  
    - **Outputs:**  
      - True: Proceeds to "Get Video URL request."  
      - False: Loops back to "Wait" node to repeat polling.  
    - **Edge Cases:** Edge cases include status never reaching "COMPLETED" causing infinite loop or timeout.

---

#### 1.6 Deliver Video Link

- **Overview:**  
  Retrieves the finalized video URL and sends it via email using Gmail node.

- **Nodes Involved:**  
  - Get Video URL request  
  - Gmail

- **Node Details:**  
  - **Get Video URL request**  
    - **Type:** HTTP Request node  
    - **Role:** Fetches the full details of the completed video rendering request, including the final video URL.  
    - **Configuration:**  
      - Method: GET  
      - URL: https://queue.fal.run/fal-ai/veo3/requests/{{request_id}}  
      - Authentication: Generic HTTP Header Auth (same credentials)  
    - **Inputs:** Receives request_id from "If" node.  
    - **Outputs:** Sends video URL JSON to "Gmail."  
    - **Edge Cases:** API errors or missing URL field.  
  - **Gmail**  
    - **Type:** Gmail node  
    - **Role:** Sends an email containing the video URL to a predefined recipient.  
    - **Configuration:**  
      - SendTo: Configured email address (redacted)  
      - Subject: "Sally's Closet Marketing Video"  
      - Message: Video URL from previous node (plain text)  
      - Email Type: text  
      - Credentials: Gmail OAuth2 credentials required  
    - **Inputs:** Receives video URL JSON from "Get Video URL request."  
    - **Outputs:** None (end of workflow)  
    - **Edge Cases:** Gmail auth errors, quota limits, or invalid email address.

---

### 3. Summary Table

| Node Name                  | Node Type                               | Functional Role                           | Input Node(s)                    | Output Node(s)                    | Sticky Note                          |
|----------------------------|---------------------------------------|-----------------------------------------|---------------------------------|----------------------------------|------------------------------------|
| Schedule Trigger           | Schedule Trigger                      | Starts workflow daily                    | None                            | Idea Gen Agent (research)         | Scheduled Trigger                   |
| Tavily                    | Tavily Tool                           | Searches trending topics                 | Idea Gen Agent (research) (ai_tool) | Idea Gen Agent (research) (ai_tool) | Ideation & Prompt Generation        |
| Idea Gen Agent (research) | Langchain OpenAI GPT-4                | Generates marketing video ideas          | Schedule Trigger, Tavily         | Video Prompt Agent                | Ideation & Prompt Generation        |
| Video Prompt Agent        | Langchain OpenAI GPT-4                | Creates cinematic video prompt for Veo 3 | Idea Gen Agent (research)         | FAL Veo 3 Post Request           | Ideation & Prompt Generation        |
| FAL Veo 3 Post Request    | HTTP Request                         | Sends video prompt to Veo 3 API          | Video Prompt Agent              | Wait                            | Request Video                      |
| Wait                      | Wait                                | Pauses before checking video status      | FAL Veo 3 Post Request, If       | Get Video Status from FAL / Veo 3 | Check & Get Result Loop            |
| Get Video Status from FAL / Veo 3 | HTTP Request                         | Polls video generation status             | Wait                           | If                              | Check & Get Result Loop            |
| If                        | If                                  | Checks if video is completed              | Get Video Status from FAL / Veo 3 | Get Video URL request, Wait      | Check & Get Result Loop            |
| Get Video URL request      | HTTP Request                         | Retrieves final video URL                  | If                             | Gmail                           | Send to Email                     |
| Gmail                     | Gmail                               | Sends video URL via email                  | Get Video URL request           | None                           | Send to Email                     |
| Sticky Note               | Sticky Note                         | Ideation & Prompt Generation description  | None                           | None                           | Ideation & Prompt Generation        |
| Sticky Note1              | Sticky Note                         | Scheduled Trigger description              | None                           | None                           | Scheduled Trigger                   |
| Sticky Note2              | Sticky Note                         | Request Video description                   | None                           | None                           | Request Video                      |
| Sticky Note3              | Sticky Note                         | Check & Get Result Loop description         | None                           | None                           | Check & Get Result Loop            |
| Sticky Note4              | Sticky Note                         | Send to Email description                   | None                           | None                           | Send to Email                     |
| Sticky Note5              | Sticky Note                         | General description and usage info          | None                           | None                           | 🎬 AI Video Marketing Agent (Veo 3 + GPT-4 + Tavily + Gmail) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 10:00 AM.

2. **Add a Tavily node:**  
   - Type: Tavily Tool  
   - Configure the query parameter to accept dynamic input (for AI override) or fixed brand-related search.  
   - No credentials configured by default; add if required.

3. **Add 'Idea Gen Agent (research)' node:**  
   - Type: Langchain OpenAI (OpenAI GPT-4)  
   - Credentials: Configure OpenAI API key with GPT-4 access.  
   - Parameters:  
     - Model: gpt-4  
     - System prompt: Provide detailed instructions to generate marketing video ideas that incorporate trends and Sally’s Closet product themes (see overview section for prompt content).  
   - Connect Schedule Trigger main output to this node’s main input.  
   - Connect Tavily node’s ai_tool output to this node’s ai_tool input.

4. **Add 'Video Prompt Agent' node:**  
   - Type: Langchain OpenAI (OpenAI GPT-4)  
   - Credentials: Same OpenAI API key as above.  
   - Parameters:  
     - Model: gpt-4  
     - System prompt: Instruct to convert video idea text into cinematic video prompt for Veo 3.  
     - Message input: Pass the content from the "Idea Gen Agent (research)" node.  
   - Connect output of "Idea Gen Agent (research)" main to this node’s main input.

5. **Add 'FAL Veo 3 Post Request' node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://queue.fal.run/fal-ai/veo3  
   - Body Content-Type: JSON  
   - JSON Body: `{ "prompt": {{$json.message.content}} }`  
   - Authentication: Set up Generic HTTP Header Auth credentials (obtain API key or token from FAL/Veo 3 provider).  
   - Connect output of "Video Prompt Agent" main to this node’s main input.

6. **Add 'Wait' node:**  
   - Type: Wait  
   - Parameters: Wait for 90 seconds.  
   - Connect output of "FAL Veo 3 Post Request" main to this node’s main input.

7. **Add 'Get Video Status from FAL / Veo 3' node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}/status`  
   - Authentication: Use same Generic HTTP Header Auth credentials.  
   - Connect output of "Wait" node main to this node’s main input.

8. **Add 'If' node:**  
   - Type: If  
   - Condition: Check if `{{$json.status}}` equals `"COMPLETED"`.  
   - Connect output of "Get Video Status from FAL / Veo 3" main to this node’s main input.  
   - True branch: Connect to "Get Video URL request."  
   - False branch: Connect back to "Wait" node to continue polling.

9. **Add 'Get Video URL request' node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/veo3/requests/{{ $json.request_id }}`  
   - Authentication: Use same Generic HTTP Header Auth credentials.  
   - Connect True output of "If" node to this node’s main input.

10. **Add 'Gmail' node:**  
    - Type: Gmail  
    - Credentials: Configure with Gmail OAuth2 credentials.  
    - Parameters:  
      - Send To: Your marketing team email address.  
      - Subject: "Sally's Closet Marketing Video"  
      - Message: Pass `{{$json.video.url}}` or full URL field from the previous node.  
      - Email Type: Text  
    - Connect output of "Get Video URL request" node main to this node’s main input.

11. **Add Sticky Notes (optional):**  
    - Add descriptive sticky notes for each block as per the original workflow for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                          | Context or Link                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| 🎬 AI Video Marketing Agent (Veo 3 + GPT-4 + Tavily + Gmail) - Automate your entire video content creation pipeline with this AI-powered, no-code workflow built in n8n. This template connects a suite of smart tools to help generate scroll-stopping short video ideas based on daily trending topics and auto-deliver them via email—ready for production in Veo 3. | General workflow description                      |
| Watch full step-by-step Tutorial Build Video: https://youtu.be/x7nHpcggpX8                                                                                                                                                                                                                                                                                                                             | Tutorial video link                               |
| Use cases include e-commerce brands needing fresh daily content, marketing teams automating ideation, solopreneurs building lean video production, and anyone experimenting with Veo 3 prompt-based storytelling.                                                                                                                                                                                     | Use case information                             |
| Pro Tip: Hook this workflow up with Veo 3 generation API (FAL) to complete automation end-to-end.                                                                                                                                                                                                                                                                                                      | Implementation tip                               |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, adhering strictly to content policies without illegal or offensive elements. All data handled is legal and public.