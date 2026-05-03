Generate Viral LinkedIn Content with GPT and Images using Telegram Bot Interface

https://n8nworkflows.xyz/workflows/generate-viral-linkedin-content-with-gpt-and-images-using-telegram-bot-interface-8334


# Generate Viral LinkedIn Content with GPT and Images using Telegram Bot Interface

### 1. Workflow Overview

This workflow enables automated generation of viral LinkedIn content and corresponding AI-generated images through a Telegram bot interface. It serves content creators, social media managers, and marketers who want to efficiently create engaging LinkedIn posts combined with relevant visuals, leveraging advanced AI models and real-time research tools.

The workflow is logically divided into these key blocks:

- **1.1 Input Reception via Telegram:** Captures user requests sent through a Telegram chat.
- **1.2 Viral Content Analysis and Framework Creation:** Uses AI agents and the Tavily search tool to analyze trending LinkedIn posts, producing a structured framework and image prompt for viral content.
- **1.3 Post Generation:** Creates three LinkedIn post variants based on the framework.
- **1.4 Post Selection by Community Manager AI:** Evaluates and selects the most viral post among the three options.
- **1.5 Image Generation:** Generates an AI image based on the selected prompt and retrieves it.
- **1.6 Output Delivery to Telegram:** Sends the generated image and the selected post back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception via Telegram

- **Overview:**  
  Listens for incoming messages on Telegram and triggers the workflow when a message is received.

- **Nodes Involved:**  
  - Telegram Trigger

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger node  
    - Role: Starts workflow on new Telegram messages  
    - Configuration: Listens to "message" updates, uses Telegram API credentials  
    - Expressions: None  
    - Inputs: None (entry point)  
    - Outputs: Connects to expert_algo node  
    - Failures: Possible Telegram API connectivity or authentication issues  
    - Notes: Requires Telegram API credentials configured in n8n  

#### 2.2 Viral Content Analysis and Framework Creation

- **Overview:**  
  An AI agent analyzes the user's request text to research viral LinkedIn posts related to the topic, using the Tavily search tool, and outputs a structured framework and an image prompt.

- **Nodes Involved:**  
  - expert_algo  
  - tavily  
  - Structured Output Parser

- **Node Details:**

  - **expert_algo**  
    - Type: Langchain Agent node (AI agent)  
    - Role: Interprets user input, researches viral posts using Tavily, generates a viral post writing framework and image prompt  
    - Configuration:  
      - Input text: User's Telegram message text  
      - System message instructs the agent to act as a LinkedIn algorithm expert, use Tavily for research, and output a JSON with "framework" and "img_prompt" fields  
    - Expressions: `={{ $json.message.text }}` for input  
    - Inputs: Telegram Trigger  
    - Outputs: CM Junior, generate_img, Structured Output Parser  
    - Failures: Possible API timeouts, Tavily API failures, or parsing errors  
    - Notes: Requires OpenAI and Tavily credentials  

  - **tavily**  
    - Type: Tavily Tool node  
    - Role: Performs advanced research on trending LinkedIn posts for the agent  
    - Configuration: Searches last 3 days, topic "general", max 5 results, advanced search depth  
    - Inputs: Called internally by expert_algo agent via AI tool connection  
    - Failures: Tavily API errors, rate limits  

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses expert_algo’s JSON output into structured data  
    - Configuration: Expects JSON with `framework` (string) and `img_prompt` (string)  
    - Inputs: expert_algo output parser  
    - Outputs: CM Junior node  
    - Failures: JSON schema validation errors  

#### 2.3 Post Generation

- **Overview:**  
  Generates three viral LinkedIn post suggestions using the framework from the previous block.

- **Nodes Involved:**  
  - CM Junior  
  - Structured Output Parser1

- **Node Details:**

  - **CM Junior**  
    - Type: Langchain Chain LLM node  
    - Role: Writes three LinkedIn post variations based on the framework  
    - Configuration:  
      - Input text template includes the framework from expert_algo  
      - Instructions specify style, length (500-900 characters), and tone  
      - Output format is JSON with `post_1`, `post_2`, `post_3`  
    - Inputs: Output from Structured Output Parser  
    - Outputs: Community Manager, Structured Output Parser1  
    - Failures: AI model errors, parsing failures  

  - **Structured Output Parser1**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses the three post proposals JSON into structured data  
    - Configuration: Expects JSON with `post_1`, `post_2`, `post_3` fields  
    - Inputs: CM Junior output parser  
    - Outputs: Community Manager  
    - Failures: JSON schema validation errors  

#### 2.4 Post Selection by Community Manager AI

- **Overview:**  
  AI Community Manager evaluates the three proposed posts and selects the most viral one, leveraging Tavily for validation.

- **Nodes Involved:**  
  - Community Manager  
  - Structured Output Parser2

- **Node Details:**

  - **Community Manager**  
    - Type: Langchain Agent node  
    - Role: Chooses the best viral post among three options using Tavily for research  
    - Configuration:  
      - Input text: Lists the three posts from CM Junior  
      - System prompt instructs evaluation and output of a JSON with a single `post` field  
    - Inputs: Structured Output Parser1  
    - Outputs: Send a text message, Structured Output Parser2  
    - Failures: AI model or Tavily API errors, parsing issues  

  - **Structured Output Parser2**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses the selected post JSON  
    - Configuration: Expects JSON with single `post` string  
    - Inputs: Community Manager output parser  
    - Outputs: Send a text message node  
    - Failures: JSON parsing errors  

#### 2.5 Image Generation

- **Overview:**  
  Generates an AI image based on the image prompt from expert_algo output, downloads the generated image, and prepares it for sending.

- **Nodes Involved:**  
  - generate_img  
  - Split Out  
  - download_img  
  - Send a photo message

- **Node Details:**

  - **generate_img**  
    - Type: HTTP Request  
    - Role: Sends a POST request to a free AI text-to-image API with the image prompt  
    - Configuration:  
      - URL: AI image generator endpoint on RapidAPI  
      - Body: Includes prompt from expert_algo output (`img_prompt`), style_id=4, size=1-1  
      - Headers: Uses RapidAPI host and requires user’s RapidAPI key  
    - Inputs: expert_algo output  
    - Outputs: Split Out node  
    - Failures: API key errors, network issues, rate limits  

  - **Split Out**  
    - Type: Split Out node  
    - Role: Extracts image URLs (in `result.data.results`) from API response array to process individually  
    - Inputs: generate_img  
    - Outputs: download_img  
    - Failures: Empty or malformed response arrays  

  - **download_img**  
    - Type: HTTP Request  
    - Role: Downloads the thumbnail image from the URL extracted by Split Out  
    - Configuration: URL from field `thumb` in JSON  
    - Inputs: Split Out  
    - Outputs: Send a photo message  
    - Failures: 404 or broken links, network timeouts  

  - **Send a photo message**  
    - Type: Telegram  
    - Role: Sends the downloaded image as a photo message back to Telegram chat  
    - Configuration:  
      - Operation: sendPhoto  
      - Chat ID: User’s Telegram chat ID (placeholder)  
      - Sends binary data (image)  
    - Inputs: download_img  
    - Failures: Telegram API errors, chat ID misconfiguration  

#### 2.6 Output Delivery to Telegram

- **Overview:**  
  Sends the selected viral LinkedIn post text back to the user via Telegram.

- **Nodes Involved:**  
  - Send a text message

- **Node Details:**

  - **Send a text message**  
    - Type: Telegram  
    - Role: Sends text message containing the chosen LinkedIn post to Telegram  
    - Configuration:  
      - Text: Selected post from Community Manager output (`post`)  
      - Chat ID: User’s Telegram chat ID (placeholder)  
    - Inputs: Structured Output Parser2  
    - Failures: Telegram API errors, chat ID misconfiguration  

---

### 3. Summary Table

| Node Name              | Node Type                             | Functional Role                               | Input Node(s)                  | Output Node(s)              | Sticky Note                                                                                                                                            |
|------------------------|-------------------------------------|-----------------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger        | Telegram Trigger                    | Entry point: listens for Telegram messages    | None                          | expert_algo                 |                                                                                                                                                        |
| expert_algo             | Langchain Agent                    | Analyze user input, research viral posts, generate framework + image prompt | Telegram Trigger              | CM Junior, generate_img     |                                                                                                                                                        |
| tavily                  | Tavily Tool                       | Provides research data for expert_algo AI agent | Called internally by expert_algo | Community Manager            |                                                                                                                                                        |
| Structured Output Parser| Langchain Structured Output Parser | Parses expert_algo JSON output                  | expert_algo                   | CM Junior                   |                                                                                                                                                        |
| CM Junior               | Langchain Chain LLM               | Generates 3 viral LinkedIn post variations    | Structured Output Parser      | Community Manager           |                                                                                                                                                        |
| Structured Output Parser1| Langchain Structured Output Parser | Parses 3 post proposals JSON                    | CM Junior                    | Community Manager           |                                                                                                                                                        |
| Community Manager       | Langchain Agent                   | Selects best viral post among 3 proposals     | Structured Output Parser1     | Send a text message         |                                                                                                                                                        |
| Structured Output Parser2| Langchain Structured Output Parser | Parses selected post JSON                        | Community Manager             | Send a text message         |                                                                                                                                                        |
| generate_img            | HTTP Request                     | Generates AI image from image prompt           | expert_algo                   | Split Out                   | Requires RapidAPI key for AI image generation                                                                                                         |
| Split Out               | Split Out                        | Extracts individual image URLs from response  | generate_img                  | download_img                |                                                                                                                                                        |
| download_img            | HTTP Request                     | Downloads AI-generated image                    | Split Out                    | Send a photo message        |                                                                                                                                                        |
| Send a photo message    | Telegram                         | Sends generated image to Telegram user         | download_img                 | None                       |                                                                                                                                                        |
| Send a text message     | Telegram                         | Sends selected LinkedIn post text to Telegram  | Structured Output Parser2    | None                       |                                                                                                                                                        |
| Sticky Note             | Sticky Note                     | Documentation and setup notes                    | None                          | None                       | ## LinkedIn Viral Content Generator & Image Poster (Telegram Bot)  \n\nThis workflow helps you generate viral LinkedIn posts + AI images via Telegram.|

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates  
   - Credentials: Configure Telegram API credentials  
   - Position: Entry point  
   
2. **Create Langchain Agent node "expert_algo"**  
   - Type: Langchain Agent  
   - Input: `={{ $json.message.text }}` (Telegram message text)  
   - System prompt: Provide instructions to analyze viral LinkedIn posts, use Tavily, output JSON with `framework` and `img_prompt`  
   - Credentials: OpenAI API, Tavily API  
   - Connect output from Telegram Trigger to expert_algo input  

3. **Create Tavily Tool node "tavily"**  
   - Type: Tavily Tool  
   - Parameters: Search last 3 days, topic “general”, max results 5, advanced search depth  
   - Credentials: Tavily API  
   - Connect internally as AI tool for expert_algo (via AI tool input)  

4. **Add Structured Output Parser node for expert_algo output**  
   - Type: Langchain Structured Output Parser  
   - JSON schema example: `{ "framework": "string", "img_prompt": "string" }`  
   - Connect expert_algo output parser to this node  

5. **Create Langchain Chain LLM node "CM Junior"**  
   - Type: Chain LLM  
   - Input: Pass `framework` from previous output  
   - Prompt: Generate 3 LinkedIn posts (500-900 chars), friendly/instructive tone, using the framework  
   - Output format: JSON with `post_1`, `post_2`, `post_3`  
   - Connect from Structured Output Parser  

6. **Add Structured Output Parser node for CM Junior output**  
   - Schema: `{ "post_1": "string", "post_2": "string", "post_3": "string" }`  
   - Connect CM Junior output parser to this node  

7. **Create Langchain Agent node "Community Manager"**  
   - Input: The three posts from CM Junior  
   - System prompt: Choose the most viral post using Tavily research, output JSON with `post` field  
   - Credentials: OpenAI API, Tavily API  
   - Connect from Structured Output Parser1  

8. **Add Structured Output Parser node for Community Manager output**  
   - Schema: `{ "post": "string" }`  
   - Connect Community Manager output parser  

9. **Create HTTP Request node "generate_img"**  
   - Method: POST  
   - URL: `https://ai-text-to-image-generator-flux-free-api.p.rapidapi.com/aaaaaaaaaaaaaaaaaiimagegenerator/quick.php`  
   - Body parameters:  
     - prompt = `={{ $json.output.img_prompt }}` from expert_algo output  
     - style_id = 4  
     - size = "1-1"  
   - Headers:  
     - `x-rapidapi-host`: `ai-text-to-image-generator-flux-free-api.p.rapidapi.com`  
     - `x-rapidapi-key`: Your RapidAPI key  
   - Connect expert_algo output to generate_img  

10. **Add Split Out node**  
    - Field to split: `result.data.results` (array of image results)  
    - Connect generate_img output  

11. **Create HTTP Request node "download_img"**  
    - URL: `={{ $json.thumb }}` (thumbnail URL from split output)  
    - Connect Split Out output  

12. **Create Telegram node "Send a photo message"**  
    - Operation: sendPhoto  
    - Chat ID: Your Telegram chat ID (replace placeholder)  
    - Send binary data: enabled  
    - Connect download_img output  

13. **Create Telegram node "Send a text message"**  
    - Text: `={{ $json.output.post }}` (selected post from Community Manager)  
    - Chat ID: Your Telegram chat ID (replace placeholder)  
    - Connect Structured Output Parser2 output  

14. **Connect nodes as follows:**  
    - Telegram Trigger → expert_algo  
    - expert_algo → CM Junior & generate_img  
    - generate_img → Split Out → download_img → Send a photo message  
    - expert_algo → Structured Output Parser → CM Junior → Structured Output Parser1 → Community Manager → Structured Output Parser2 → Send a text message  

15. **Configure Credentials:**  
    - Telegram API credentials for Telegram nodes  
    - OpenAI API credentials for Langchain nodes  
    - Tavily API credentials for Tavily and Langchain agents  
    - RapidAPI key for AI image generator  

16. **Replace placeholders:**  
    - Telegram chat ID in Send a photo message and Send a text message nodes  
    - RapidAPI key in generate_img node  

17. **Test workflow:**  
    - Trigger via Telegram message  
    - Verify AI-generated framework, posts, selected post, image generation, and Telegram outputs  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow generates viral LinkedIn posts and AI images triggered by Telegram messages. It combines AI research tools (Tavily), OpenAI language models, and an external AI image generator API. Credentials and API keys must be configured securely in n8n’s credential manager. Replace all placeholders before activating. Test thoroughly to avoid exposing sensitive data. | Workflow description and setup instructions from sticky note within the workflow.                      |
| For AI image generation, obtain a RapidAPI key for the free AI text-to-image generator service: https://rapidapi.com/flux-free-api/api/ai-text-to-image-generator-flux-free-api                                                                                                                                                                                                       | AI Image Generator API info                                                                             |
| Tavily provides social media research capabilities and must be authorized with a valid API key configured in n8n.                                                                                                                                                                                                                                                                  | Tavily API documentation                                                                                |
| OpenAI GPT models require API keys; the workflow uses GPT-5 Nano for analysis and variant generation.                                                                                                                                                                                                                                                                              | OpenAI API documentation                                                                                |
| For Telegram integration, create a Telegram bot and retrieve its API token; configure chat IDs carefully to match user or group chat targets.                                                                                                                                                                                                                                       | Telegram Bot API documentation                                                                          |

---

This document enables precise understanding, reproduction, and maintenance of the "Generate Viral LinkedIn Content with GPT and Images using Telegram Bot Interface" workflow, ensuring robust operation and easy customization.