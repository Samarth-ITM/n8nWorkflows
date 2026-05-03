E-commerce Product Fine-Tuning With Bright Data and OpenAI

https://n8nworkflows.xyz/workflows/e-commerce-product-fine-tuning-with-bright-data-and-openai-5199


# E-commerce Product Fine-Tuning With Bright Data and OpenAI

### 1. Workflow Overview

This workflow automates the collection, processing, and fine-tuning of a custom OpenAI language model specialized in writing compelling e-commerce product descriptions. It is designed for marketers, product managers, or developers seeking to generate fine-tuned AI models based on real product data scraped from Amazon.

The workflow is logically divided into two main functional blocks:

- **1.1 Data Collection and Fine-Tuning Preparation:**  
  Starts with manual execution, scrapes product data from Amazon using Bright Data, processes this data into a fine-tuning dataset formatted as `.jsonl`, uploads the dataset to OpenAI, and initiates a fine-tuning job for a GPT model.

- **1.2 AI Agent Interaction With Fine-Tuned Model:**  
  A chat-triggered sub-workflow that uses the fine-tuned OpenAI model to generate product descriptions or marketing copy interactively via an AI agent.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection and Fine-Tuning Preparation

**Overview:**  
This block gathers product data from Amazon via Bright Data scraping, formats the data into training examples for OpenAI fine-tuning, uploads the training file, and triggers a fine-tuning job.

**Nodes Involved:**  
- When clicking 'Execute workflow' (Manual Trigger)  
- BrightData (Bright Data Web Scraper)  
- Code (JS Code Processing)  
- OpenAI (File Upload for Fine-Tuning)  
- HTTP Request (Initiate Fine-Tuning Job)

##### Node Details:

- **When clicking 'Execute workflow'**  
  - Type: Manual Trigger  
  - Role: Entry point to start the workflow manually  
  - Configuration: Default manual trigger with no parameters  
  - Input: None  
  - Output: Triggers BrightData node  
  - Failures: None expected  
  - Notes: Used to explicitly start the scraping and fine-tuning preparation process.

- **BrightData**  
  - Type: Bright Data Web Scraper Integration  
  - Role: Scrapes product data from a predefined list of Amazon product URLs  
  - Configuration:  
    - Resource: `webScrapper`  
    - Dataset ID: Uses a stored Bright Data dataset with Amazon best-seller product URLs  
    - URLs: List of Amazon product links (13 items) hardcoded in parameters  
  - Input: Trigger from manual node  
  - Output: Sends scraped product JSON data to Code node  
  - Failures: Possible auth errors (invalid Bright Data API key), scraping failures due to site changes or IP bans, empty data sets  
  - Credentials: Requires valid Bright Data API credentials  
  - Notes: Scrapes structured product info (title, brand, features, description, rating, availability)

- **Code**  
  - Type: JavaScript Code Node  
  - Role: Transforms scraped product data into OpenAI fine-tuning `.jsonl` formatted training examples  
  - Configuration:  
    - Iterates over all input items  
    - Extracts relevant fields: title, brand, features (up to 5), description snippet, rating, availability  
    - Constructs training prompts with system, user, and assistant messages  
    - Aggregates all training examples into a single JSONL string  
    - Converts string to binary buffer suitable for file upload  
  - Key Expressions: Template strings for prompts and ideal assistant replies  
  - Input: Product JSON data from BrightData  
  - Output: Binary file property `data.jsonl` and metadata for OpenAI upload  
  - Failures: Handling of missing or malformed product data; warns and skips such items  
  - Version: Uses `this.helpers.prepareBinaryData` (n8n v0.188+ recommended)  
  - Notes: Ensures output file is valid for OpenAI fine-tuning file upload

- **OpenAI**  
  - Type: OpenAI File Upload Node (Langchain)  
  - Role: Uploads the generated `.jsonl` fine-tuning dataset file to OpenAI  
  - Configuration:  
    - Resource: `file`  
    - Purpose: `fine-tune`  
    - Binary Property Name: `data.jsonl` (from Code node output)  
  - Input: Binary file from Code node  
  - Output: File metadata including file ID for fine-tuning  
  - Failures: Auth errors, file size limits, invalid file format errors  
  - Credentials: Requires valid OpenAI API key  
  - Notes: Prepares the training file for the subsequent fine-tuning API call

- **HTTP Request**  
  - Type: HTTP Request Node  
  - Role: Initiates the OpenAI fine-tuning job using the uploaded training file ID  
  - Configuration:  
    - Method: POST  
    - URL: `https://api.openai.com/v1/fine_tuning/jobs`  
    - Body (JSON):  
      - `training_file`: ID from OpenAI file upload node output (`{{ $json.id }}`)  
      - `model`: `gpt-4o-mini-2024-07-18` (preset fine-tuning base model)  
    - Authentication: Uses OpenAI OAuth2 credentials  
  - Input: File ID from OpenAI node  
  - Output: Fine-tuning job metadata response  
  - Failures: API errors (invalid file ID, model selection errors, quota limits), network issues  
  - Credentials: OpenAI API key required

---

#### 2.2 AI Agent Interaction With Fine-Tuned Model

**Overview:**  
This block enables interactive chat functionality using the fine-tuned OpenAI model. It triggers on chat messages, routes them to an AI agent node configured with the fine-tuned model, and responds accordingly.

**Nodes Involved:**  
- When chat message received (Chat Trigger)  
- AI Agent (Langchain Agent)  
- OpenAI Chat Model (Fine-Tuned model integration)

##### Node Details:

- **When chat message received**  
  - Type: Langchain Chat Trigger  
  - Role: Webhook-based entry point for incoming chat messages  
  - Configuration: Default, listens for chat webhook events  
  - Input: Incoming chat messages from external source (webhook)  
  - Output: Passes chat message data to AI Agent  
  - Failures: Webhook communication failures, invalid message formats

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Core AI reasoning and response generation node  
  - Configuration: Uses the OpenAI Chat Model node as its language model  
  - Input: Chat messages from trigger  
  - Output: Generated AI responses back to chat system  
  - Failures: AI model invocation errors, rate limits, timeouts

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Provides the fine-tuned GPT model interface for language generation  
  - Configuration:  
    - Model ID: `YOUR_FINE_TUNED_MODEL_ID` (placeholder; must be replaced)  
    - Response format: text  
  - Input: Queries from AI Agent  
  - Output: Text completions to AI Agent  
  - Failures: Invalid model ID, authentication failure, API errors  
  - Credentials: OpenAI API key required

---

### 3. Summary Table

| Node Name                   | Node Type                         | Functional Role                             | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                                                                                                                                                                              |
|-----------------------------|----------------------------------|---------------------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking 'Execute workflow' | Manual Trigger                   | Starts the workflow manually                 | —                          | BrightData               | This n8n workflow automates scraping Amazon product data and uses it for fine-tuning a custom OpenAI model for marketing copy.                                                                                                                           |
| BrightData                  | Bright Data Web Scraper           | Scrapes product data from Amazon URLs       | When clicking 'Execute workflow' | Code                     |                                                                                                                                                                                                                                                          |
| Code                        | JavaScript Code                   | Processes scraped data into OpenAI fine-tuning JSONL training data | BrightData                 | OpenAI                   |                                                                                                                                                                                                                                                          |
| OpenAI                      | OpenAI File Upload (Langchain)   | Uploads JSONL training file to OpenAI       | Code                       | HTTP Request             |                                                                                                                                                                                                                                                          |
| HTTP Request                | HTTP Request                     | Initiates fine-tuning job on OpenAI          | OpenAI                     | —                        |                                                                                                                                                                                                                                                          |
| When chat message received  | Langchain Chat Trigger           | Receives chat input for AI agent             | —                          | AI Agent                 |                                                                                                                                                                                                                                                          |
| AI Agent                   | Langchain Agent                  | Generates responses using fine-tuned model  | When chat message received  | —                        |                                                                                                                                                                                                                                                          |
| OpenAI Chat Model           | Langchain OpenAI Chat Model      | Provides fine-tuned GPT model interface      | AI Agent                   | AI Agent                 |                                                                                                                                                                                                                                                          |
| Sticky Note                 | Sticky Note                      | Documentation and overview                    | —                          | —                        | This n8n workflow automates scraping Amazon product data and uses it for fine-tuning a custom OpenAI model for marketing copy. <br><br>**Steps Overview**<br>1. Manual Trigger<br>2. Bright Data scraping<br>3. Code processing<br>4. OpenAI upload<br>5. Fine-tuning job start<br>6. Chat/Agent interaction<br><br>**Important:** Replace placeholder API credentials and IDs with your own valid keys and model identifiers. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node:**  
   - Node Type: Manual Trigger  
   - Purpose: To start the workflow manually  
   - No parameters needed  

2. **Add BrightData Node for Web Scraping:**  
   - Node Type: Bright Data Web Scraper  
   - Configure credentials with your Bright Data API key  
   - Set Resource to `webScrapper`  
   - Enter a list of Amazon product URLs to scrape (example URLs given in original workflow)  
   - Set Dataset ID to your Bright Data dataset for Amazon products or use the URLs directly  
   - Connect Manual Trigger node output to BrightData node input  

3. **Add a Code Node to Process Scraped Data:**  
   - Node Type: JavaScript Code  
   - Paste the provided JS code that:  
     - Iterates over all scraped items  
     - Extracts product title, brand, features, description snippet, rating, availability  
     - Creates a prompt with system, user, and assistant messages  
     - Aggregates into `.jsonl` string and converts to binary buffer  
   - Connect BrightData node output to Code node input  

4. **Add OpenAI File Upload Node:**  
   - Node Type: OpenAI File Upload (Langchain)  
   - Configure OpenAI API credentials  
   - Set Resource to `file`  
   - Set Purpose to `fine-tune`  
   - Set Binary Property to `data.jsonl` (matching Code node output)  
   - Connect Code node output to OpenAI node input  

5. **Add HTTP Request Node to Start Fine-Tuning Job:**  
   - Node Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.openai.com/v1/fine_tuning/jobs`  
   - Authentication: Use OpenAI API credentials  
   - Body Type: JSON  
   - Body Content:  
     ```json
     {
       "training_file": "{{ $json.id }}",
       "model": "gpt-4o-mini-2024-07-18"
     }
     ```  
   - Connect OpenAI node output to HTTP Request node input  

6. **Add Chat Trigger Node for Incoming Messages:**  
   - Node Type: Langchain Chat Trigger  
   - Configure webhook as needed for your chat platform  
   - No special parameters  

7. **Add AI Agent Node:**  
   - Node Type: Langchain Agent  
   - No special parameters (default)  
   - Connect Chat Trigger node output to AI Agent input  

8. **Add OpenAI Chat Model Node:**  
   - Node Type: Langchain OpenAI Chat Model  
   - Configure OpenAI credentials  
   - Set Model ID to your fine-tuned model ID (replace `YOUR_FINE_TUNED_MODEL_ID`)  
   - Set response format to `text`  
   - Connect OpenAI Chat Model node output to AI Agent language model input  

9. **Connect AI Agent output to wherever you handle chat responses in your system.**

10. **Add Sticky Note (optional):**  
    - Add a Sticky Note node anywhere convenient  
    - Paste the overview and instructions content for documentation purposes

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates scraping Amazon product data and uses it for fine-tuning a custom OpenAI model specialized in marketing product descriptions. Replace placeholder API credentials for Bright Data and OpenAI with your own valid keys. Be sure to update the fine-tuned model ID in the chat model node to your actual model. | Workflow overview and critical setup instructions                                                  |
| Bright Data integration requires a valid Bright Data account and appropriate dataset or URLs for scraping Amazon products.                                                                                                                                                                                             | https://brightdata.com/                                                                             |
| OpenAI fine-tuning requires preparation of `.jsonl` files with system-user-assistant message triplets formatted per OpenAI fine-tuning guidelines.                                                                                                                                                                      | https://platform.openai.com/docs/guides/fine-tuning                                                 |
| The fine-tuning base model used here is `gpt-4o-mini-2024-07-18`. You can update this to newer or different models as OpenAI updates their offerings.                                                                                                                                                                  | https://platform.openai.com/docs/models                                                             |
| The AI Agent node uses Langchain integration to simplify AI workflows within n8n, enabling conversational agents to use custom fine-tuned models.                                                                                                                                                                      | https://docs.n8n.io/integrations/ai/                                                              |

---

_Disclaimer: The provided text is generated exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public._