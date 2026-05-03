Create Dev.to Articles with OpenAI/Gemini - AI-Generated Content with Images

https://n8nworkflows.xyz/workflows/create-dev-to-articles-with-openai-gemini---ai-generated-content-with-images-7574


# Create Dev.to Articles with OpenAI/Gemini - AI-Generated Content with Images

### 1. Workflow Overview

This n8n workflow automates the creation and publication of Dev.to articles enriched with AI-generated content and images. It leverages OpenAI and Google Gemini AI models for content generation, structured parsing, and image creation, integrating these with Cloudinary for image hosting and the Dev.to API for publishing.

The workflow is logically divided into these blocks:

- **1.1 Input Reception & Initialization:** Starting the workflow manually and setting up required input data and credentials.
- **1.2 AI Content Generation:** Using AI agents and models to generate article content in a structured format.
- **1.3 Image Generation & Upload:** Creating images with Google Gemini AI, uploading them to Cloudinary, and preparing image data.
- **1.4 Data Aggregation & Article Writing:** Aggregating generated data and using AI to write the final article content.
- **1.5 Publishing:** Posting the completed article with images to Dev.to.
- **1.6 Looping & Batch Processing:** Managing multiple content pieces in batches and iterating over them.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

**Overview:**  
This block initiates the workflow manually and prepares input data and AI credentials necessary for downstream processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Set input data/credintenials

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Initiates the workflow manually by user action.  
  - *Configuration:* Default manual trigger without parameters.  
  - *Connections:* Outputs to `Set input data/credintenials`.  
  - *Edge Cases:* User may forget to trigger; no automatic scheduling here (disabled Schedule Trigger present but not connected).  

- **Set input data/credintenials**  
  - *Type:* Set  
  - *Role:* Prepares input data and sets required credentials or configuration variables for AI agents.  
  - *Configuration:* Contains static or expression-based values to configure AI nodes (e.g., API keys, prompts).  
  - *Connections:* Outputs to `AI Agent`.  
  - *Edge Cases:* Misconfiguration or missing credentials here will cause authorization failures downstream.

---

#### 2.2 AI Content Generation

**Overview:**  
This block uses an AI Agent powered by OpenAI Chat models to generate structured article data. It parses AI output into a usable format for further processing.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser1  
- OpenAI Chat Model4

**Node Details:**

- **AI Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Orchestrates AI model calls to generate article data based on input.  
  - *Configuration:* Uses OpenAI Chat Model as underlying LM; no explicit parameters shown but configured via upstream inputs.  
  - *Connections:* Receives from `Set input data/credintenials`; outputs to `Split Out`.  
  - *Edge Cases:* API rate limits, model timeouts, or parsing errors can occur.

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides natural language generation capabilities to AI Agent.  
  - *Configuration:* Default OpenAI Chat model settings, likely GPT-4 or similar.  
  - *Connections:* Input to AI Agent as language model.  
  - *Edge Cases:* OpenAI API auth failures, timeouts.

- **Structured Output Parser1**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses AI-generated text into structured JSON or defined formats for downstream processing.  
  - *Configuration:* Uses a predefined schema to extract article metadata and sections.  
  - *Connections:* Receives from `OpenAI Chat Model4`; outputs to `AI Agent`.  
  - *Edge Cases:* Parsing errors if AI output format changes.

- **OpenAI Chat Model4**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Secondary AI model used to assist structured parsing or further content refinement.  
  - *Connections:* Outputs to `Structured Output Parser1`.  
  - *Edge Cases:* Same as other OpenAI nodes.

---

#### 2.3 Image Generation & Upload

**Overview:**  
Generates images related to the article using Google Gemini AI, uploads them to Cloudinary, and sets data for image insertion into articles.

**Nodes Involved:**  
- Split Out  
- Loop Over Items  
- Set Input Image  
- Generate an image1 (Google Gemini)  
- Post image Cloudinary2  
- Set Output Image

**Node Details:**

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the AI Agent output into individual items for processing.  
  - *Connections:* Input from `AI Agent`, output to `Loop Over Items`.  
  - *Edge Cases:* Empty splits if no items present.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes items one-by-one or in batches.  
  - *Connections:* From `Split Out`, outputs to `Aggregate` and `Set Input Image`.  
  - *Edge Cases:* Batch size misconfiguration could cause performance issues.

- **Set Input Image**  
  - *Type:* Set  
  - *Role:* Prepares image generation input data (prompts, parameters) for Google Gemini.  
  - *Connections:* Input from `Loop Over Items`, output to `Generate an image1`.  
  - *Edge Cases:* Invalid prompt data causing poor image generation.

- **Generate an image1**  
  - *Type:* Langchain Google Gemini  
  - *Role:* Generates images based on input prompts.  
  - *Connections:* Input from `Set Input Image`, output to `Post image Cloudinary2`.  
  - *Edge Cases:* API limits, auth errors, image generation failures.

- **Post image Cloudinary2**  
  - *Type:* HTTP Request  
  - *Role:* Uploads generated image to Cloudinary for hosting.  
  - *Configuration:* Configured with Cloudinary API endpoint and credentials.  
  - *Connections:* Input from `Generate an image1`, output to `Set Output Image`.  
  - *Edge Cases:* Upload failures, network issues.

- **Set Output Image**  
  - *Type:* Set  
  - *Role:* Stores uploaded image URLs and metadata for article insertion.  
  - *Connections:* Input from `Post image Cloudinary2`, output loops back to `Loop Over Items`.  
  - *Edge Cases:* Missing or malformed upload response.

---

#### 2.4 Data Aggregation & Article Writing

**Overview:**  
Aggregates processed batch items, uses AI to write the full article, and parses the final structured output.

**Nodes Involved:**  
- Aggregate  
- Article wrtiter  
- OpenAI Chat Model1  
- OpenAI Chat Model2  
- Structured Output Parser2  
- OpenAI Chat Model4 (also part of content generation)

**Node Details:**

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Combines individual batch items into a single dataset for article writing.  
  - *Connections:* Inputs from `Loop Over Items`, output to `Article wrtiter`.  
  - *Edge Cases:* Incorrect aggregation could cause data loss.

- **Article wrtiter**  
  - *Type:* Langchain Chain LLM  
  - *Role:* Uses AI to write the full article text from aggregated data.  
  - *Connections:* Input from `Aggregate`, output to `Post article on dev.to`.  
  - *Edge Cases:* Generation failures, incomplete content.

- **OpenAI Chat Model1**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Language model used within `Article wrtiter` chain.  
  - *Connections:* Input to `Article wrtiter`.  
  - *Edge Cases:* API errors, timeouts.

- **OpenAI Chat Model2**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Supports structured output parsing of article content.  
  - *Connections:* Input to `Structured Output Parser2`.  
  - *Edge Cases:* Parsing or API failures.

- **Structured Output Parser2**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses article writing AI output into final structured format for posting.  
  - *Connections:* Input from `OpenAI Chat Model2`, output to `Article wrtiter`.  
  - *Edge Cases:* Parsing errors.

---

#### 2.5 Publishing

**Overview:**  
Posts the completed article, including text and images, to the Dev.to platform via its API.

**Nodes Involved:**  
- Post article on dev.to

**Node Details:**

- **Post article on dev.to**  
  - *Type:* HTTP Request  
  - *Role:* Sends HTTP POST request to Dev.to API to publish the article.  
  - *Configuration:* Includes API endpoint, authentication tokens (OAuth2 or API key), and request body with article content and image URLs.  
  - *Connections:* Input from `Article wrtiter`.  
  - *Edge Cases:* API errors, auth failures, rate limits, malformed requests.

---

#### 2.6 Looping & Batch Processing

**Overview:**  
Handles multiple generated articles or content pieces by looping through batches and processing each sequentially.

**Nodes Involved:**  
- Split Out  
- Loop Over Items  
- Aggregate  
- Set Input Image  
- Set Output Image

**Node Details:**

- These nodes work together to split the AI output into items, process each item (including image generation), then aggregate results before final article writing and publishing.  
- Proper batch size and error handling are critical to avoid partial updates or data loss.

---

### 3. Summary Table

| Node Name               | Node Type                        | Functional Role                  | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                     |
|-------------------------|---------------------------------|---------------------------------|----------------------------------|--------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Workflow start trigger           | -                                | Set input data/credintenials    |                                                                                                |
| Set input data/credintenials      | Set                            | Initializes input and credentials| When clicking ‘Execute workflow’ | AI Agent                       |                                                                                                |
| AI Agent                     | Langchain Agent                | Orchestrates AI content generation| Set input data/credintenials      | Split Out                      |                                                                                                |
| OpenAI Chat Model            | Langchain OpenAI Chat Model   | Language model for AI Agent      | - (used internally by AI Agent)  | AI Agent                      |                                                                                                |
| Structured Output Parser1    | Langchain Structured Output Parser | Parses AI output to structured format | OpenAI Chat Model4               | AI Agent                      |                                                                                                |
| OpenAI Chat Model4           | Langchain OpenAI Chat Model   | Secondary model for parsing      | -                                | Structured Output Parser1      |                                                                                                |
| Split Out                   | Split Out                     | Splits AI Agent output into items| AI Agent                        | Loop Over Items               |                                                                                                |
| Loop Over Items             | Split In Batches              | Processes items in batches       | Split Out                      | Aggregate, Set Input Image     |                                                                                                |
| Set Input Image             | Set                          | Prepares prompts for image generation| Loop Over Items               | Generate an image1            |                                                                                                |
| Generate an image1          | Langchain Google Gemini       | Generates images from prompts    | Set Input Image                | Post image Cloudinary2         |                                                                                                |
| Post image Cloudinary2      | HTTP Request                  | Uploads images to Cloudinary     | Generate an image1             | Set Output Image              |                                                                                                |
| Set Output Image            | Set                          | Stores uploaded image data       | Post image Cloudinary2          | Loop Over Items               |                                                                                                |
| Aggregate                  | Aggregate                     | Combines processed items         | Loop Over Items                | Article wrtiter              |                                                                                                |
| Article wrtiter            | Langchain Chain LLM           | AI writes final article          | Aggregate                     | Post article on dev.to        |                                                                                                |
| OpenAI Chat Model1         | Langchain OpenAI Chat Model   | Language model for article writing | - (used internally by Article wrtiter) | Article wrtiter          |                                                                                                |
| OpenAI Chat Model2         | Langchain OpenAI Chat Model   | Model for parsing article output | -                              | Structured Output Parser2      |                                                                                                |
| Structured Output Parser2  | Langchain Structured Output Parser | Parses article output structure  | OpenAI Chat Model2             | Article wrtiter              |                                                                                                |
| Post article on dev.to     | HTTP Request                  | Publishes article on Dev.to      | Article wrtiter               | -                            |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node for Input & Credentials**  
   - Name: `Set input data/credintenials`  
   - Type: Set  
   - Configure variables for AI access (e.g., API keys, prompts).  
   - Connect from `When clicking ‘Execute workflow’`.

3. **Create Langchain AI Agent Node**  
   - Name: `AI Agent`  
   - Type: Langchain Agent  
   - Configure to use `OpenAI Chat Model`.  
   - Connect from `Set input data/credintenials`.

4. **Create OpenAI Chat Model Node for AI Agent**  
   - Name: `OpenAI Chat Model`  
   - Type: Langchain OpenAI Chat Model  
   - Set model parameters (e.g., GPT-4, temperature).  
   - Connect as language model input to `AI Agent`.

5. **Create OpenAI Chat Model4 Node for Parsing**  
   - Name: `OpenAI Chat Model4`  
   - Type: Langchain OpenAI Chat Model  
   - Configure similarly to support output parsing.  
   - Connect output to `Structured Output Parser1`.

6. **Create Structured Output Parser1 Node**  
   - Name: `Structured Output Parser1`  
   - Type: Langchain Structured Output Parser  
   - Configure with output schema for article metadata and sections.  
   - Connect to `AI Agent` as output parser.

7. **Create Split Out Node**  
   - Name: `Split Out`  
   - Type: Split Out  
   - Connect input from `AI Agent`.  
   - Output to `Loop Over Items`.

8. **Create Loop Over Items Node**  
   - Name: `Loop Over Items`  
   - Type: Split In Batches  
   - Configure batch size appropriately (e.g., 1).  
   - Connect from `Split Out`.  
   - Outputs to `Aggregate` and `Set Input Image`.

9. **Create Set Input Image Node**  
   - Name: `Set Input Image`  
   - Type: Set  
   - Set image generation prompt data.  
   - Connect from `Loop Over Items`.  
   - Output to `Generate an image1`.

10. **Create Image Generation Node (Google Gemini)**  
    - Name: `Generate an image1`  
    - Type: Langchain Google Gemini  
    - Configure authentication and prompt inputs.  
    - Connect from `Set Input Image`.  
    - Output to `Post image Cloudinary2`.

11. **Create HTTP Request Node to Upload Images**  
    - Name: `Post image Cloudinary2`  
    - Type: HTTP Request  
    - Configure with Cloudinary API endpoint, credentials (API key/secret).  
    - POST method with image data.  
    - Connect from `Generate an image1`.  
    - Output to `Set Output Image`.

12. **Create Set Output Image Node**  
    - Name: `Set Output Image`  
    - Type: Set  
    - Extract and set image URL and metadata from Cloudinary response.  
    - Connect from `Post image Cloudinary2`.  
    - Loop output back to `Loop Over Items` for batch continuation.

13. **Create Aggregate Node**  
    - Name: `Aggregate`  
    - Type: Aggregate  
    - Aggregate batch item outputs into one dataset.  
    - Connect from `Loop Over Items`.  
    - Output to `Article wrtiter`.

14. **Create Article Writing Chain Node**  
    - Name: `Article wrtiter`  
    - Type: Langchain Chain LLM  
    - Configure with `OpenAI Chat Model1` and `Structured Output Parser2`.  
    - Connect from `Aggregate`.  
    - Output to `Post article on dev.to`.

15. **Create OpenAI Chat Model1 Node (Article Writing)**  
    - Name: `OpenAI Chat Model1`  
    - Type: Langchain OpenAI Chat Model  
    - Configure model parameters for article generation.  
    - Connect as language model input to `Article wrtiter`.

16. **Create OpenAI Chat Model2 Node (Article Parsing)**  
    - Name: `OpenAI Chat Model2`  
    - Type: Langchain OpenAI Chat Model  
    - Configure for output parsing.  
    - Connect to `Structured Output Parser2`.

17. **Create Structured Output Parser2 Node**  
    - Name: `Structured Output Parser2`  
    - Type: Langchain Structured Output Parser  
    - Configure schema for final article output.  
    - Connect output to `Article wrtiter`.

18. **Create HTTP Request Node to Post Article to Dev.to**  
    - Name: `Post article on dev.to`  
    - Type: HTTP Request  
    - Configure with Dev.to API endpoint: `https://dev.to/api/articles`  
    - Set authentication (API key in headers).  
    - POST method with JSON body including article text and image URLs.  
    - Connect from `Article wrtiter`.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Workflow uses Langchain n8n nodes for AI orchestration and OpenAI/Gemini models for text and image generation. | n8n Langchain integration documentation: https://docs.n8n.io/integrations/ai/langchain/                  |
| Cloudinary is used for image hosting; ensure Cloudinary account and API keys are set up properly.          | Cloudinary API docs: https://cloudinary.com/documentation/image_upload_api_reference                    |
| Dev.to API requires API key for publishing articles; get yours at https://dev.to/settings/account/api_tokens | Dev.to API docs: https://docs.dev.to/api/#operation/createArticle                                       |
| Batch processing ensures scalable handling of multiple articles or image generations.                      | Consider error handling for partial batch failures or API rate limits.                                 |
| Disabled nodes for Google Sheets and Schedule Trigger indicate potential for automation extension.         | These nodes can be enabled and configured for scheduled or spreadsheet-driven input.                    |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow created with n8n integration and automation tools. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.