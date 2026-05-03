Transform YouTube Videos into LinkedIn Posts with SearchAPI & OpenAi

https://n8nworkflows.xyz/workflows/transform-youtube-videos-into-linkedin-posts-with-searchapi---openai-6397


# Transform YouTube Videos into LinkedIn Posts with SearchAPI & OpenAi

### 1. Workflow Overview

This workflow automates the transformation of YouTube video content into LinkedIn posts by extracting the video's transcript, processing it with an AI language model, and returning a formatted LinkedIn post. It is designed for content marketers, social media managers, or automation enthusiasts who want to repurpose YouTube video content efficiently.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Accepts incoming HTTP requests to trigger the workflow.
- **1.2 Transcript Retrieval:** Fetches the transcript of the specified YouTube video.
- **1.3 Transcript Validation:** Checks if the transcript exists and is usable.
- **1.4 AI Processing:** Prepares the transcript data and generates a LinkedIn post using OpenAI.
- **1.5 Response Handling:** Sends back either the generated post or an error message via the webhook response.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP requests that trigger the workflow. It acts as the entry point, expecting a payload that identifies the YouTube video to process.

- **Nodes Involved:**  
  - Webhook Trigger

- **Node Details:**  

  - **Webhook Trigger**  
    - Type and Role: HTTP webhook node that receives external requests to start the workflow.  
    - Configuration: Default webhook with no specific parameters set; it listens for POST requests by default.  
    - Key Expressions/Variables: Expects incoming data containing YouTube video identification (e.g., video ID or URL).  
    - Connections: Output connects to "Get YouTube Transcript".  
    - Failure Cases: Missing or malformed request data could cause downstream errors. No direct validation here.  
    - Version Notes: Uses typeVersion 1, compatible with n8n v0.140+.  

#### 1.2 Transcript Retrieval

- **Overview:**  
  This block retrieves the transcript text of the specified YouTube video by making an HTTP request to an external API or service that provides transcript data.

- **Nodes Involved:**  
  - Get YouTube Transcript

- **Node Details:**  

  - **Get YouTube Transcript**  
    - Type and Role: HTTP Request node to fetch transcript data from a YouTube transcript API or similar endpoint.  
    - Configuration: Details are not explicitly provided but typically require:  
      - URL constructed with YouTube video ID extracted from webhook input.  
      - Method: GET  
      - Authentication: Possibly none or API key via headers depending on service.  
    - Variables: Uses incoming webhook data to build the request URL.  
    - Input: From "Webhook Trigger".  
    - Output: Passes response data (transcript) to "Check Transcript Exists".  
    - Failure Cases: Network errors, invalid video IDs, API rate limits, transcript not available.  
    - Version Notes: Uses typeVersion 4.1 for HTTP Request node.  

#### 1.3 Transcript Validation

- **Overview:**  
  Verifies whether a transcript was successfully retrieved. If transcript data is missing or empty, routes the workflow to an error response; otherwise, continues processing.

- **Nodes Involved:**  
  - Check Transcript Exists  
  - Error Response - No Transcript

- **Node Details:**  

  - **Check Transcript Exists**  
    - Type and Role: IF node that evaluates if the transcript data exists and is non-empty.  
    - Configuration: Condition likely checks if a specific field in previous node’s output (e.g., transcript text) is defined and not empty.  
    - Input: From "Get YouTube Transcript".  
    - Output:  
      - True path: to "Prepare Data for OpenAI".  
      - False path: to "Error Response - No Transcript".  
    - Failure Cases: Expression errors if input data structure changes.  

  - **Error Response - No Transcript**  
    - Type and Role: Respond to Webhook node that sends an error message back to the HTTP requestor indicating no transcript was found.  
    - Configuration: Sends an HTTP error response with a descriptive message (e.g., 404 or 400 with message "Transcript not found").  
    - Input: From false branch of "Check Transcript Exists".  
    - Output: Terminal node for error path (ends workflow).  
    - Failure Cases: None expected other than network issues when responding.  

#### 1.4 AI Processing

- **Overview:**  
  Prepares the transcript data into a format suitable for AI input and invokes an OpenAI-powered language model to generate a LinkedIn post summarizing or rephrasing the video content.

- **Nodes Involved:**  
  - Prepare Data for OpenAI  
  - Generate LinkedIn Post

- **Node Details:**  

  - **Prepare Data for OpenAI**  
    - Type and Role: Set node used to structure and augment data for AI consumption.  
    - Configuration: Constructs input parameters such as prompt text, model parameters, or context variables from the transcript data.  
    - Variables: Uses expressions to map and transform transcript text into prompt format.  
    - Input: From true branch of "Check Transcript Exists".  
    - Output: Passes the constructed data to "Generate LinkedIn Post".  
    - Failure Cases: Expression errors if transcript format changes.  

  - **Generate LinkedIn Post**  
    - Type and Role: OpenAI node (via LangChain integration) that generates text content.  
    - Configuration:  
      - Model: OpenAI GPT model (e.g., GPT-4 or GPT-3.5)  
      - Input: Prompt prepared in previous node.  
      - Parameters: Temperature, max tokens, possibly stop sequences.  
    - Input: From "Prepare Data for OpenAI".  
    - Output: Passes generated LinkedIn post to "Success Response".  
    - Failure Cases: API authentication errors, rate limits, timeouts, or malformed prompts.  
    - Credentials: Requires configured OpenAI API credentials.  

#### 1.5 Response Handling

- **Overview:**  
  Returns the final output back to the requester via the webhook, either the generated LinkedIn post or an error message.

- **Nodes Involved:**  
  - Success Response

- **Node Details:**  

  - **Success Response**  
    - Type and Role: Respond to Webhook node that sends the AI-generated LinkedIn post back to the HTTP requestor.  
    - Configuration: Sends HTTP 200 response with JSON containing the generated post text.  
    - Input: From "Generate LinkedIn Post".  
    - Output: Terminal node (ends workflow).  
    - Failure Cases: Network issues or malformed response data could cause failures.  

---

### 3. Summary Table

| Node Name                | Node Type                          | Functional Role                       | Input Node(s)           | Output Node(s)               | Sticky Note                     |
|--------------------------|----------------------------------|-------------------------------------|-------------------------|-----------------------------|--------------------------------|
| Webhook Trigger          | Webhook                          | Entry point, receives HTTP requests |                         | Get YouTube Transcript      |                                |
| Get YouTube Transcript   | HTTP Request                     | Fetches YouTube video transcript    | Webhook Trigger         | Check Transcript Exists      |                                |
| Check Transcript Exists  | IF Node                         | Validates transcript presence       | Get YouTube Transcript  | Prepare Data for OpenAI, Error Response - No Transcript |                                |
| Prepare Data for OpenAI  | Set                             | Formats data for AI input            | Check Transcript Exists | Generate LinkedIn Post       |                                |
| Generate LinkedIn Post   | OpenAI (LangChain integration)  | Generates LinkedIn post with AI     | Prepare Data for OpenAI | Success Response             |                                |
| Success Response         | Respond to Webhook               | Sends generated post to requester   | Generate LinkedIn Post  |                             |                                |
| Error Response - No Transcript | Respond to Webhook           | Sends error if no transcript found  | Check Transcript Exists |                             |                                |
| Sticky Note1             | Sticky Note                     | (No content)                        |                         |                             |                                |
| Sticky Note2             | Sticky Note                     | (No content)                        |                         |                             |                                |
| Sticky Note3             | Sticky Note                     | (No content)                        |                         |                             |                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger node**  
   - Type: Webhook  
   - Configuration: Leave default settings to accept POST requests.  
   - Purpose: To receive incoming requests containing YouTube video identifiers.  

2. **Add an HTTP Request node named "Get YouTube Transcript"**  
   - Type: HTTP Request  
   - Configuration:  
     - Method: GET  
     - URL: Construct using expression to include the YouTube video ID from webhook input, e.g., `https://api.example.com/youtube/transcript?videoId={{ $json["videoId"] }}` (replace with actual transcript API endpoint).  
     - Authentication: Set if required by the transcript API.  
   - Connect Webhook Trigger output to this node input.

3. **Add an IF node named "Check Transcript Exists"**  
   - Type: IF  
   - Configuration:  
     - Condition: Check if the transcript field exists and is not empty, e.g., expression `{{$json["transcript"] && $json["transcript"].length > 0}}`.  
   - Connect "Get YouTube Transcript" output to this node.  

4. **Add a Respond to Webhook node named "Error Response - No Transcript"**  
   - Type: Respond to Webhook  
   - Configuration:  
     - Status Code: 400 or 404  
     - Response Body: JSON with error message, e.g., `{ "error": "Transcript not found" }`  
   - Connect the false output of "Check Transcript Exists" to this node.

5. **Add a Set node named "Prepare Data for OpenAI"**  
   - Type: Set  
   - Configuration:  
     - Define fields to prepare the AI prompt; for example:  
       - Field: `prompt`  
       - Value: Expression combining a prompt template with the transcript, e.g., `Create a LinkedIn post based on this transcript: {{$json["transcript"]}}`  
   - Connect the true output of "Check Transcript Exists" to this node.  

6. **Add an OpenAI node named "Generate LinkedIn Post"**  
   - Type: OpenAI (via LangChain or native OpenAI node)  
   - Configuration:  
     - Model: Choose GPT-4 or GPT-3.5  
     - Input: Use `prompt` field from previous node.  
     - Parameters: Set temperature, max tokens as needed (e.g., temperature 0.7, max tokens 300).  
   - Credentials: Configure with valid OpenAI API key.  
   - Connect "Prepare Data for OpenAI" output to this node.

7. **Add a Respond to Webhook node named "Success Response"**  
   - Type: Respond to Webhook  
   - Configuration:  
     - Status Code: 200  
     - Response Body: Return generated post, e.g., `{ "linkedinPost": {{$json["choices"][0]["text"]}} }` (adjust based on OpenAI node output format).  
   - Connect "Generate LinkedIn Post" output to this node.

8. **Test the full workflow** by sending HTTP POST requests with YouTube video IDs to the webhook URL.

---

### 5. General Notes & Resources

| Note Content                                      | Context or Link                                 |
|--------------------------------------------------|------------------------------------------------|
| This workflow leverages OpenAI's GPT models via LangChain integration to transform video transcripts into social media posts. | OpenAI docs: https://platform.openai.com/docs  |
| Ensure the YouTube transcript API used allows programmatic access and returns well-structured transcript data. | YouTube transcript APIs vary; verify your endpoint. |
| Proper error handling for missing or empty transcripts helps maintain workflow robustness and user clarity. | Best practices in webhook response design.     |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.