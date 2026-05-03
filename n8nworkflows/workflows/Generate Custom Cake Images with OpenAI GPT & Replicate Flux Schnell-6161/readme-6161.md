Generate Custom Cake Images with OpenAI GPT & Replicate Flux Schnell

https://n8nworkflows.xyz/workflows/generate-custom-cake-images-with-openai-gpt---replicate-flux-schnell-6161


# Generate Custom Cake Images with OpenAI GPT & Replicate Flux Schnell

### 1. Workflow Overview

This workflow, titled **"Cake Image Generator"**, automates the process of generating custom cake images using AI models. It is designed to receive user input specifying a name to be featured on a cake, generate a detailed image prompt via OpenAI's GPT model, create the corresponding cake image through the Replicate Flux Schnell model, and finally return the generated image to the requester.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Handles incoming user requests with parameters such as the cake name.
- **1.2 AI Prompt Generation:** Utilizes OpenAI GPT to craft a detailed and specific image prompt based on the user input.
- **1.3 Image Creation:** Sends the generated prompt to the Replicate Flux Schnell API to generate the cake image.
- **1.4 Image Retrieval & Response:** Downloads the generated image and responds back to the user with the image URL.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block receives HTTP POST requests from the frontend (e.g., Bolt) containing the name to be placed on the cake. It triggers the workflow execution.

- **Nodes Involved:**  
  - User Sends Request  
  - Sticky Note1 (comment)

- **Node Details:**

  - **User Sends Request**  
    - **Type:** Webhook node  
    - **Role:** Entry point for receiving user data.  
    - **Configuration:**  
      - HTTP Method: POST  
      - Webhook Path: `5cfa7429-5637-4c1a-a81c-a53dfb154a5d`  
      - Response Mode: Uses downstream `Respond to Requests` node for response  
    - **Inputs:** External HTTP POST request  
    - **Outputs:** Passes the user JSON body (expects at least a `name` field) downstream  
    - **Potential Failures:**  
      - Missing or malformed input data  
      - Network connectivity issues  
      - Webhook path conflicts if reused elsewhere  
    - **Sticky Note1:** "Receives user input (e.g., name/occasion) from the Bolt frontend."

---

#### 1.2 AI Prompt Generation

- **Overview:**  
  This block uses OpenAI GPT (via LangChain n8n integration) to generate a highly detailed text prompt for cake image generation, embedding the user's specified name directly into the prompt.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Prompt Generator  
  - Sticky Note (comment)

- **Node Details:**

  - **OpenAI Chat Model**  
    - **Type:** LangChain OpenAI Chat Model node  
    - **Role:** Provides access to GPT-4.1-mini for generating creative text prompts  
    - **Configuration:**  
      - Model: `gpt-4.1-mini`  
      - No additional options set  
      - Credentials: OpenAI API key configured  
    - **Inputs:** Triggered by the user request node  
    - **Outputs:** Sends chat completion results to the `Prompt Generator` node  
    - **Failures:**  
      - API authentication errors  
      - Rate limits or quota exceeded  
      - Network timeouts  
      - Invalid prompt or malformed input expressions  

  - **Prompt Generator**  
    - **Type:** LangChain Agent node  
    - **Role:** Constructs a detailed AI prompt for the cake image generation, embedding the user’s cake name inside the prompt string.  
    - **Configuration:**  
      - Text prompt instructs to create a highly detailed cake image prompt, embedding `{{ $json.body.name }}` exactly as a quoted string inside the generated prompt.  
      - Restricts output to plain text prompt only, no additional quotes or metadata.  
      - Example given: `"black forest gateau cake spelling out the words \"FLUX SCHNELL\", tasty, food photography, dynamic shot"`  
    - **Inputs:** Receives output from the OpenAI Chat Model node  
    - **Outputs:** Outputs the final prompt text for image generation  
    - **Expressions Used:**  
      - `{{ $json.body.name }}` dynamically inserts the user's provided name  
      - Escapes quotes carefully to ensure prompt formatting correctness  
    - **Failures:**  
      - Expression syntax errors  
      - Empty or undefined `name` input  
      - Misinterpretation of prompt instructions  
    - **Sticky Note:** "Uses OpenAI GPT to generate a creative cake prompt."

---

#### 1.3 Image Creation

- **Overview:**  
  This block takes the generated text prompt and sends it to the Replicate API using the Flux Schnell model to create the cake image.

- **Nodes Involved:**  
  - Generate Image  
  - Sticky Note2 (comment)

- **Node Details:**

  - **Generate Image**  
    - **Type:** HTTP Request node  
    - **Role:** Calls Replicate API to trigger image generation with the provided prompt  
    - **Configuration:**  
      - Method: POST  
      - URL: `https://api.replicate.com/v1/models/black-forest-labs/flux-schnell/predictions`  
      - JSON Body:  
        ```json
        {
          "version": "latest",
          "input": {
            "prompt": "{{$json['prompt']}}"
          }
        }
        ```  
      - Authentication: HTTP Header Auth using Replicate API token credential  
    - **Inputs:** Receives the prompt text from `Prompt Generator` node  
    - **Outputs:** Receives JSON response from Replicate API containing image generation URLs and metadata  
    - **Failures:**  
      - Authentication failure (invalid or expired token)  
      - API rate limits or downtime  
      - Malformed prompt causing generation errors  
      - HTTP request timeouts  
    - **Sticky Note2:** "Sends the generated prompt to Replicate Flux Schnell to create the cake image."

---

#### 1.4 Image Retrieval & Response

- **Overview:**  
  This block downloads the generated cake image from the URL provided by Replicate and responds back to the original requester with the final image URL.

- **Nodes Involved:**  
  - Download Image  
  - Respond to Requests  
  - Sticky Note3 (Download Image)  
  - Sticky Note4 (Respond to Requests)

- **Node Details:**

  - **Download Image**  
    - **Type:** HTTP Request node  
    - **Role:** Downloads the image file or resource from the URL returned by the Replicate API  
    - **Configuration:**  
      - URL: Extracted dynamically from `{{$json.output[0]}}` (first output URL from Replicate response)  
      - Default HTTP GET method  
    - **Inputs:** Receives image URL from `Generate Image` node  
    - **Outputs:** Passes downloaded image data or metadata to the next node  
    - **Failures:**  
      - Invalid or expired image URL  
      - HTTP errors like 404 or 403  
      - Network timeouts  
    - **Sticky Note3:** "Fetches the generated cake image from Replicate."

  - **Respond to Requests**  
    - **Type:** Respond to Webhook node  
    - **Role:** Sends the final response back to the original HTTP POST request  
    - **Configuration:**  
      - Uses default response options, returning the data passed from the `Download Image` node  
    - **Inputs:** Receives image data or URL from `Download Image` node  
    - **Outputs:** HTTP response to the user  
    - **Failures:**  
      - Response timeout if downstream nodes fail  
      - Improper data format causing client errors  
    - **Sticky Note4:** "Returns the final cake image URL to the frontend."

---

### 3. Summary Table

| Node Name           | Node Type                        | Functional Role                           | Input Node(s)          | Output Node(s)         | Sticky Note                                                                 |
|---------------------|---------------------------------|-----------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------|
| User Sends Request   | Webhook                         | Receives user input from frontend       | -                      | Prompt Generator       | Receives user input (e.g., name/occasion) from the Bolt frontend.           |
| OpenAI Chat Model    | LangChain OpenAI Chat Model     | Generates base text for prompt          | User Sends Request      | Prompt Generator       | Uses OpenAI GPT to generate a creative cake prompt.                         |
| Prompt Generator     | LangChain Agent                 | Creates detailed cake image prompt      | OpenAI Chat Model       | Generate Image         | Uses OpenAI GPT to generate a creative cake prompt.                         |
| Generate Image       | HTTP Request                   | Sends prompt to Replicate API           | Prompt Generator        | Download Image         | Sends the generated prompt to Replicate Flux Schnell to create the cake image. |
| Download Image       | HTTP Request                   | Downloads generated cake image           | Generate Image          | Respond to Requests    | Fetches the generated cake image from Replicate.                           |
| Respond to Requests  | Respond to Webhook             | Sends final response back to user       | Download Image          | -                      | Returns the final cake image URL to the frontend.                          |
| Sticky Note1         | Sticky Note                    | Comment                                | -                      | -                      | Receives user input (e.g., name/occasion) from the Bolt frontend.           |
| Sticky Note          | Sticky Note                    | Comment                                | -                      | -                      | Uses OpenAI GPT to generate a creative cake prompt.                         |
| Sticky Note2         | Sticky Note                    | Comment                                | -                      | -                      | Sends the generated prompt to Replicate Flux Schnell to create the cake image. |
| Sticky Note3         | Sticky Note                    | Comment                                | -                      | -                      | Fetches the generated cake image from Replicate.                           |
| Sticky Note4         | Sticky Note                    | Comment                                | -                      | -                      | Returns the final cake image URL to the frontend.                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node**  
   - Name: `User Sends Request`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `5cfa7429-5637-4c1a-a81c-a53dfb154a5d` (or custom)  
   - Response Mode: Set to `Respond to Webhook` node downstream

2. **Create LangChain OpenAI Chat Model Node**  
   - Name: `OpenAI Chat Model`  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4.1-mini`  
   - Credentials: Link your OpenAI API credentials  
   - Connect `User Sends Request` main output to this node

3. **Create LangChain Agent Node for Prompt Generation**  
   - Name: `Prompt Generator`  
   - Type: LangChain Agent  
   - Prompt Type: Define  
   - Text:  
     ```
     You are an expert in creating ai cake images.
     I want you to randomise the cake And put a very specific  {{ $json.body.name }} On the cake.

     But remember one thing I don't want you to add anything else other than the prompt In the output. I dont want you to add any quotes Or any other information other than the plain text of the prompt.

     Also wherever you mention to write the name
     Instead of "{{ $json.body.name }}" write it \"{{ $json.body.name }}\"
     the name \nabin\ is {{ $json.body.name }}

     eg : black forest gateau cake spelling out the words \"FLUX SCHNELL\", tasty, food photography, dynamic shot

     give a very detailed prompt so that it can perfectly create the images of the cake
     ```
   - Connect `OpenAI Chat Model` node's AI output to this node

4. **Create HTTP Request Node to Call Replicate API**  
   - Name: `Generate Image`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.replicate.com/v1/models/black-forest-labs/flux-schnell/predictions`  
   - Authentication: HTTP Header Auth using your Replicate API Token credential  
   - Body Content Type: JSON  
   - Body Parameters:  
     ```json
     {
       "version": "latest",
       "input": {
         "prompt": "{{$json['prompt']}}"
       }
     }
     ```  
   - Connect `Prompt Generator` node output to this node

5. **Create HTTP Request Node to Download Image**  
   - Name: `Download Image`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{ $json.output[0] }}` (extract first output URL from Replicate response)  
   - Connect `Generate Image` node output to this node

6. **Create Respond to Webhook Node**  
   - Name: `Respond to Requests`  
   - Type: Respond to Webhook  
   - Default options to return the data received  
   - Connect `Download Image` node output to this node

7. **Link connections in order:**  
   `User Sends Request` → `OpenAI Chat Model` → `Prompt Generator` → `Generate Image` → `Download Image` → `Respond to Requests`

8. **Configure Credentials:**  
   - OpenAI API key for the `OpenAI Chat Model` node  
   - Replicate API token for the `Generate Image` HTTP Request node

9. **Test with example POST JSON:**  
   ```json
   {
     "name": "John"
   }
   ```
   to verify that the returned image URL corresponds to a cake image spelling out "John".

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| OpenAI GPT is used here via the LangChain integration for advanced prompt generation.                    | n8n LangChain Integration Documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/ |
| Replicate Flux Schnell model is specialized for AI-generated cake images.                               | Replicate Model: https://replicate.com/black-forest-labs/flux-schnell                                     |
| Careful escaping of quotes in prompt generation is crucial to ensure the AI models interpret the input as intended. | -                                                                                                        |
| Webhook node expects a POST request with JSON body containing at least a `name` field for the cake.     | -                                                                                                        |

---

**Disclaimer:** The workflow content is fully derived from an automated n8n workflow export. It complies with all applicable content policies and contains no illegal or offensive material. All data processed is public and lawful.