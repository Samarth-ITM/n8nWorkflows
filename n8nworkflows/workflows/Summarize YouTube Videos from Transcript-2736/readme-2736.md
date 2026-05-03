Summarize YouTube Videos from Transcript

https://n8nworkflows.xyz/workflows/summarize-youtube-videos-from-transcript-2736


# Summarize YouTube Videos from Transcript

### 1. Workflow Overview

This workflow automates the summarization of YouTube videos by extracting their transcripts and generating concise summaries using AI. It is designed for content creators, researchers, educators, and professionals who want to quickly obtain key insights from lengthy video content without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures the full YouTube video URL from a form trigger.
- **1.2 Transcript Retrieval:** Sends a POST request to the Apify Actor API to fetch the video transcript.
- **1.3 AI Summarization:** Processes the transcript text using LangChain summarization and OpenAI language models to generate a concise summary.
- **1.4 Output Handling:** Finalizes the workflow with a no-operation node, allowing for easy extension or integration with other services.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives the YouTube video URL input from the user via a form trigger node, enabling dynamic workflow execution based on user input.

- **Nodes Involved:**  
  - YouTube video URL (Form Trigger)  
  - Sticky Note1 (Instructional note)

- **Node Details:**

  - **YouTube video URL**  
    - Type: Form Trigger  
    - Role: Entry point capturing user input (full YouTube URL) via a web form.  
    - Configuration:  
      - Webhook path: `ddd`  
      - Form title: "Summarize YouTube video's"  
      - Single form field labeled "Full URL"  
    - Input: HTTP POST request with form data  
    - Output: JSON containing the "Full URL" field  
    - Edge Cases:  
      - Missing or invalid URL input may cause downstream failures.  
      - Webhook availability and network connectivity are prerequisites.  
    - Notes: Can be replaced with other input methods such as webhook or manual input.

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Provides user instruction to add the full YouTube URL and mentions input flexibility.

#### 1.2 Transcript Retrieval

- **Overview:**  
  This block sends a POST request to the Apify Actor API to retrieve the transcript of the YouTube video specified by the user.

- **Nodes Involved:**  
  - Request YouTube Transcript (HTTP Request)  
  - Sticky Note2 (Instructional note)  
  - Sticky Note4 (Instructional note on summarization node)

- **Node Details:**

  - **Request YouTube Transcript**  
    - Type: HTTP Request  
    - Role: Calls Apify Actor API to fetch the transcript JSON for the given YouTube URL.  
    - Configuration:  
      - Method: POST  
      - URL: Placeholder `"Apify API_KEY Here ???"` (must be replaced with actual endpoint including API key)  
      - Body (JSON): `{ "startUrls": [ "{{ $json['Full URL'] }}" ] }` — dynamically inserts the user-provided URL.  
      - Sends JSON body with the request.  
    - Input: JSON with "Full URL" from the form trigger node.  
    - Output: Transcript data in JSON format.  
    - Edge Cases:  
      - Invalid or expired API key causes authentication errors.  
      - Incorrect or malformed URL input leads to empty or failed transcript retrieval.  
      - API rate limits or network timeouts may cause request failures.  
    - Version Requirements: HTTP Request node version 4.2 or higher recommended for JSON body support.

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Guides user to insert the full API endpoint with the POST URL and API key after following setup instructions.

  - **Sticky Note4**  
    - Type: Sticky Note  
    - Role: Explains that the summarization node processes input text automatically without manual enhancements.

#### 1.3 AI Summarization

- **Overview:**  
  This block uses AI-powered nodes to generate a concise summary from the retrieved transcript, leveraging LangChain summarization and OpenAI language models.

- **Nodes Involved:**  
  - Summarization of a YouTube script (LangChain Summarization)  
  - Summarization Engine (OpenAI LM Chat)  
  - Sticky Note3 (Optional next steps note)

- **Node Details:**

  - **Summarization of a YouTube script**  
    - Type: LangChain Chain Summarization  
    - Role: Summarizes the transcript text using LangChain's summarization chain.  
    - Configuration: Default options (no custom prompt or parameters specified).  
    - Input: Transcript JSON from "Request YouTube Transcript" node.  
    - Output: Summarized text.  
    - Edge Cases:  
      - Input transcript missing or malformed may cause summarization failure.  
      - Large transcripts may hit token limits or cause timeouts.  
    - Version: Requires LangChain integration in n8n (version 2 or higher).

  - **Summarization Engine**  
    - Type: LangChain LM Chat OpenAI  
    - Role: Provides the underlying OpenAI language model for the summarization chain.  
    - Configuration:  
      - Uses OpenAI credentials named "OpenAi" (OAuth2 or API key).  
      - Default model and parameters (not explicitly customized).  
    - Input: Receives prompt/input from the summarization chain node.  
    - Output: AI-generated summary text.  
    - Edge Cases:  
      - API key invalid or quota exceeded causes authentication or rate limit errors.  
      - Network issues may cause request failures.  
    - Credential Requirements: Valid OpenAI API credentials configured in n8n.

  - **Sticky Note3**  
    - Type: Sticky Note  
    - Role: Suggests optional workflow extension or alternative enrichment services if workflow ends here.

#### 1.4 Output Handling

- **Overview:**  
  This block finalizes the workflow with a no-operation node, serving as a placeholder for future extensions or integrations.

- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp (No Operation)  
    - Role: Acts as a terminal node that does not modify data, useful for workflow clarity or future expansion.  
    - Input: Summarized text from "Summarization of a YouTube script" node.  
    - Output: Passes data unchanged.  
    - Edge Cases: None.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                   | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                   |
|-----------------------------|---------------------------------|---------------------------------|-------------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note                     | Project overview and purpose     | —                       | —                               | **Summarize YouTube videos** This project automates the summarization of YouTube videos...   |
| YouTube video URL           | Form Trigger                   | Input reception of YouTube URL  | —                       | Request YouTube Transcript       | Add the full YouTube URL. ☝️ You can change this input to a webhook or anything else.        |
| Request YouTube Transcript  | HTTP Request                   | Fetch transcript from Apify API | YouTube video URL       | Summarization of a YouTube script | Once you follow the Setup Instructions..., insert full URL endpoint with POST and API key.   |
| Sticky Note2                | Sticky Note                    | Setup instructions for API key  | —                       | —                               | Once you follow the Setup Instructions..., insert full URL endpoint with POST and API key.   |
| Sticky Note4                | Sticky Note                    | Summarization node explanation  | —                       | —                               | The summarization node works automatically and professionally, recognizing input text...     |
| Summarization of a YouTube script | LangChain Chain Summarization | Summarize transcript text       | Request YouTube Transcript | No Operation, do nothing         |                                                                                              |
| Summarization Engine        | LangChain LM Chat OpenAI       | OpenAI language model for summary | —                       | Summarization of a YouTube script | ☝️ Optional If the workflow ends here, consider checking with another enrichment service.    |
| Sticky Note3                | Sticky Note                    | Optional next steps suggestion  | —                       | —                               | ☝️ Optional If the workflow ends here, consider checking with another enrichment service.    |
| No Operation, do nothing    | NoOp                          | Workflow termination placeholder | Summarization of a YouTube script | —                               |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Form Trigger node:**  
   - Name: `YouTube video URL`  
   - Set webhook path to `ddd` (or your preferred path).  
   - Form title: "Summarize YouTube video's"  
   - Add one form field: Label it "Full URL" (type: string).  
   - This node will receive the YouTube video URL input.

3. **Add an HTTP Request node:**  
   - Name: `Request YouTube Transcript`  
   - Method: POST  
   - URL: Insert your Apify Actor API endpoint URL including your API key (e.g., `https://api.apify.com/v2/actor-tasks/{taskId}/runs?token=YOUR_API_KEY`)  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "startUrls": [
         "{{ $json['Full URL'] }}"
       ]
     }
     ```  
   - Connect the output of `YouTube video URL` node to this node.  
   - This node fetches the transcript for the provided YouTube video.

4. **Add a LangChain Chain Summarization node:**  
   - Name: `Summarization of a YouTube script`  
   - Use default summarization options.  
   - Connect the output of `Request YouTube Transcript` node to this node.  
   - This node summarizes the transcript text.

5. **Add a LangChain LM Chat OpenAI node:**  
   - Name: `Summarization Engine`  
   - Configure credentials: Select or create OpenAI API credentials (API key or OAuth2).  
   - Use default model settings (e.g., GPT-4 or GPT-3.5).  
   - Connect this node as the language model for the `Summarization of a YouTube script` node (set as the AI language model).  
   - This node provides the AI model for summarization.

6. **Add a No Operation node:**  
   - Name: `No Operation, do nothing`  
   - Connect the output of `Summarization of a YouTube script` node to this node.  
   - This node acts as a workflow endpoint and can be replaced or extended later.

7. **Add Sticky Notes as needed:**  
   - Add notes for user instructions, setup guidance, and optional next steps as per the original workflow.

8. **Test the workflow:**  
   - Trigger the webhook with a valid YouTube video URL.  
   - Verify that the transcript is fetched and a summary is generated.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This project automates the summarization of YouTube videos, transforming lengthy content into concise insights. | Workflow purpose and high-level description.                                                       |
| Setup requires an Apify account and API key to access the YouTube Transcript Actor.                            | Apify platform: https://apify.com/                                                                 |
| OpenAI credentials must be configured in n8n for the summarization engine to work.                             | OpenAI API: https://platform.openai.com/account/api-keys                                            |
| The YouTube video URL input can be replaced with other triggers such as webhooks or manual inputs.            | Flexibility in input methods.                                                                       |
| Consider alternative enrichment services if the workflow ends without sufficient summarization.                | Suggested in Sticky Note3 for workflow extension.                                                  |

---

This documentation fully describes the workflow’s structure, node configurations, and setup instructions, enabling users and automation agents to understand, reproduce, and modify the workflow effectively.