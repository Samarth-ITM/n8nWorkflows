Turns Reddit Pain Points into Comic Ads using Dumpling AI and GPT-4o

https://n8nworkflows.xyz/workflows/turns-reddit-pain-points-into-comic-ads-using-dumpling-ai-and-gpt-4o-5394


# Turns Reddit Pain Points into Comic Ads using Dumpling AI and GPT-4o

### 1. Workflow Overview

This workflow automates the creation of comic-style advertisement images based on real user pain points sourced from Reddit discussions. It is designed for direct-to-consumer (DTC) brands seeking authentic marketing messages by mining Reddit for relevant pain points related to their product, crafting persuasive ad angles, and generating comic visuals that illustrate these messages.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Captures product details from a user-submitted form.
- **1.2 Reddit Keyword Generation**: Extracts a concise Reddit keyword/phrase reflecting pain points from the product description using GPT-4o.
- **1.3 Reddit Data Scraping and Filtering**: Searches Reddit posts by keyword, filters by popularity and content presence.
- **1.4 Post Relevance Classification**: Uses an AI model to classify whether each Reddit post is relevant to the product.
- **1.5 Relevant Data Aggregation**: Formats and aggregates relevant Reddit posts for further processing.
- **1.6 Ad Angle Generation and Ranking**: Generates 10 emotionally resonant ad angles from Reddit content and ranks the top 10.
- **1.7 Comic Prompt Creation**: Transforms top ad angles into AI image generator prompts for 4-panel comic ads.
- **1.8 Comic Image Generation and Upload**: Iteratively generates comic images using Dumpling AI, fetches the images, and uploads them to Google Drive.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects brand name, website, and product explanation via a form submission to start the workflow.
- **Nodes Involved:**  
  - `When a Form is Submitted`
- **Node Details:**
  - **Type:** Form Trigger  
  - **Purpose:** Entry point capturing product details from an interactive form titled "Add Details Here".
  - **Configuration:** Requires three fields: Brand Name, Website, Product Explanation (all mandatory).
  - **Input/Output:**  
    - No input, triggered on form submission.  
    - Outputs JSON with form fields.
  - **Edge Cases:**  
    - Missing required fields will prevent submission.  
    - Form trigger requires webhook availability and proper exposure.
  - **Version:** 2.2

#### 2.2 Reddit Keyword Generation

- **Overview:** Uses GPT-4o to generate a 1-3 word Reddit keyword or phrase summarizing pain points from the product explanation.
- **Nodes Involved:**  
  - `Generate Reddit Keyword from Product Description`
- **Node Details:**
  - **Type:** LangChain OpenAI Node (ChatGPT-4o)  
  - **Purpose:** NLP extraction of relevant Reddit search keyword from product description.  
  - **Configuration:**  
    - Model: chatgpt-4o-latest  
    - Prompt: Marketing strategist role; extracts a concise keyword/phrase only.  
    - Input: Product Explanation from form JSON.  
  - **Expressions:** Uses expression to inject product description: `{{ $json['Product Explanation'] }}`  
  - **Output:** Single keyword string to be used in Reddit search.  
  - **Credentials:** OpenAI API key needed.  
  - **Edge Cases:**  
    - Model might return unexpected text if prompt is misconfigured.  
    - API failure or rate limits possible.  
  - **Version:** 1.8

#### 2.3 Reddit Data Scraping and Filtering

- **Overview:** Searches Reddit for posts using the generated keyword, then filters posts to retain only those with at least 2 upvotes and non-empty text.
- **Nodes Involved:**  
  - `Search Reddit Posts Using Keyword`  
  - `Filter Posts With 2+ Upvotes and Text`
- **Node Details:**

  - **Search Reddit Posts Using Keyword**
    - **Type:** Reddit Node  
    - **Purpose:** Searches Reddit across all subreddits for posts matching the keyword.  
    - **Configuration:**  
      - Limit: 10 posts  
      - Keyword: Dynamic from previous node output.  
      - OAuth2 credentials for Reddit API access required.  
    - **Edge Cases:**  
      - Reddit API quota or auth errors.  
      - Keyword yielding no results.  
    - **Version:** 1

  - **Filter Posts With 2+ Upvotes and Text**
    - **Type:** If Node  
    - **Purpose:** Filters posts by conditions: at least 2 upvotes AND non-empty selftext.  
    - **Configuration:**  
      - Conditions combined with AND:  
        - `$json.ups >= 2`  
        - `$json.selftext` is not empty  
    - **Edge Cases:**  
      - Posts without selftext but with relevant titles may be excluded.  
    - **Version:** 2.2

#### 2.4 Post Relevance Classification

- **Overview:** Uses an AI agent to judge the relevance of each Reddit post to the product, returning a boolean and explanation.
- **Nodes Involved:**  
  - `Classify Post Relevance to Product` (agent)  
  - `GPT Model: Classify Relevance` (LM OpenAI)  
  - `Support Relevance Classification` (ToolThink)  
  - `Parse Relevance Classifier Output` (Output Parser)
- **Node Details:**

  - **Classify Post Relevance to Product**
    - **Type:** LangChain Agent Node  
    - **Purpose:** Defines classification task with prompt including product description and Reddit post details.  
    - **Prompt:** Returns JSON array with `isRelevant` and `reason`.  
    - **Input:** Product explanation and Reddit post fields.  
    - **Output:** Raw AI classification JSON.  
    - **Edge Cases:**  
      - Misclassification risk, prompt drift, or incomplete data.  
    - **Version:** 1.9

  - **GPT Model: Classify Relevance**
    - **Type:** LangChain LM Chat OpenAI  
    - **Purpose:** Executes the classification model with GPT-4o-mini.  
    - **Credentials:** OpenAI API  
    - **Version:** 1.2

  - **Support Relevance Classification**
    - **Type:** LangChain toolThink (supports chain-of-thought reasoning)  
    - **Purpose:** Assists agent with reasoning for classification.  
    - **Version:** 1

  - **Parse Relevance Classifier Output**
    - **Type:** LangChain Structured Output Parser  
    - **Purpose:** Validates and extracts structured JSON output from classification.  
    - **Schema:** Example JSON with `isRelevant` and `reason`.  
    - **Version:** 1.2

#### 2.5 Relevant Data Aggregation

- **Overview:** Extracts key Reddit post fields from relevant posts and aggregates all such posts into a single dataset.
- **Nodes Involved:**  
  - `Format Relevant Reddit Post Fields`  
  - `Aggregate Relevant Reddit Data`
- **Node Details:**

  - **Format Relevant Reddit Post Fields**
    - **Type:** Set Node  
    - **Purpose:** Maps relevant post fields to a simplified structure for downstream use.  
    - **Fields Extracted:** upvotes, subreddit, originalPost (selftext), title, url.  
    - **Input:** From classified relevant Reddit post.  
    - **Output:** Reformatted JSON per post.  
    - **Version:** 3.4

  - **Aggregate Relevant Reddit Data**
    - **Type:** Aggregate Node  
    - **Purpose:** Combines all relevant posts into a single data array under `data` key.  
    - **Operation:** `aggregateAllItemData` collects all items' data into one array.  
    - **Version:** 1

#### 2.6 Ad Angle Generation and Ranking

- **Overview:** Generates 10 unique, compelling ad angles from aggregated Reddit pain points and ranks the top 10 most persuasive ones.
- **Nodes Involved:**  
  - `Generate 10 Ad Angles From Reddit Posts` (agent)  
  - `Ad Angle Generation` (LM OpenAI)  
  - `Support Ad Angle Generation` (ToolThink)  
  - `Rank Top 10 Ad Angles` (OpenAI)
- **Node Details:**

  - **Generate 10 Ad Angles From Reddit Posts**
    - **Type:** LangChain Agent  
    - **Purpose:** Copywriter role to create 10 emotional, punchy ad variations framed as "hero" moments.  
    - **Input:** Aggregated Reddit posts and product explanation.  
    - **Output:** Numbered list of 10 ad variations.  
    - **Version:** 1.7

  - **Ad Angle Generation**
    - **Type:** LangChain LM Chat OpenAI (GPT-4o-mini)  
    - **Purpose:** Executes language model for generation.  
    - **Credentials:** OpenAI API  
    - **Version:** 1.2

  - **Support Ad Angle Generation**
    - **Type:** LangChain toolThink  
    - **Purpose:** Supports chain-of-thought for generation.  
    - **Version:** 1

  - **Rank Top 10 Ad Angles**
    - **Type:** LangChain OpenAI  
    - **Purpose:** Marketing strategist role to select, re-rank, and output top 10 most relevant and persuasive ad angles.  
    - **Input:** Raw ad variations from previous node.  
    - **Model:** chatgpt-4o-latest  
    - **Output:** Numbered list (1–10), no commentary.  
    - **Version:** 1.8

#### 2.7 Comic Prompt Creation

- **Overview:** Converts each ranked ad angle into a descriptive 4-panel comic image prompt for AI image generation.
- **Nodes Involved:**  
  - `Create Comic Prompts from Ad Angles` (agent)  
  - `Comic Prompt Creation` (LM OpenAI)  
  - `Support Comic Prompt Creation` (ToolThink)  
  - `Parse Comic Prompt JSON Output` (Output Parser)  
  - `Split Comic Prompt Objects`
- **Node Details:**

  - **Create Comic Prompts from Ad Angles**
    - **Type:** LangChain Agent  
    - **Purpose:** Creative director role to write AI image generator prompt describing 4-panel comic illustrating the ad message.  
    - **Input:** Product explanation and an ad angle.  
    - **Output:** JSON array of 10 objects with `prompt` property.  
    - **System Message:** Strictly returns JSON array only.  
    - **Version:** 1.9

  - **Comic Prompt Creation**
    - **Type:** LangChain LM Chat OpenAI (GPT-4o-mini)  
    - **Purpose:** Generates the comic prompts.  
    - **Credentials:** OpenAI API  
    - **Version:** 1.2

  - **Support Comic Prompt Creation**
    - **Type:** LangChain toolThink  
    - **Purpose:** Supports chain-of-thought reasoning for prompt creation.  
    - **Version:** 1

  - **Parse Comic Prompt JSON Output**
    - **Type:** LangChain Structured Output Parser  
    - **Purpose:** Parses and validates the JSON array of comic prompts.  
    - **Version:** 1.2

  - **Split Comic Prompt Objects**
    - **Type:** SplitOut Node  
    - **Purpose:** Splits JSON array of 10 prompts into individual items for batch processing.  
    - **Version:** 1

#### 2.8 Comic Image Generation and Upload

- **Overview:** Iteratively generates comic images from prompts using Dumpling AI, retrieves image URLs, and uploads images to Google Drive.
- **Nodes Involved:**  
  - `Loop Through Comic Prompts` (SplitInBatches)  
  - `Generate Comic Image via Dumpling AI` (HTTP Request)  
  - `Fetch Comic Image URL` (HTTP Request)  
  - `Upload Image to Google Drive`
- **Node Details:**

  - **Loop Through Comic Prompts**
    - **Type:** SplitInBatches  
    - **Purpose:** Processes comic prompts one-by-one in batches for API calls.  
    - **Version:** 3

  - **Generate Comic Image via Dumpling AI**
    - **Type:** HTTP Request  
    - **Purpose:** Sends prompt to Dumpling AI API to generate 512x512 JPG comic image.  
    - **Configuration:**  
      - POST to `https://app.dumplingai.com/api/v1/generate-ai-image`  
      - Model: FLUX.1-pro  
      - Prompt cleaned to remove newlines and extra spaces  
      - Fixed seed for reproducibility  
      - 25 steps, guidance 3, aspect ratio 1:1, quality 80  
      - Safety tolerance set to 2  
    - **Authentication:** HTTP Header with Dumpling AI API key.  
    - **Version:** 4.2  
    - **Edge Cases:**  
      - API errors, rate limiting, prompt rejection due to safety filters.

  - **Fetch Comic Image URL**
    - **Type:** HTTP Request  
    - **Purpose:** Retrieves the generated image URL from Dumpling AI response.  
    - **Method:** GET  
    - **Version:** 4.2

  - **Upload Image to Google Drive**
    - **Type:** Google Drive Node  
    - **Purpose:** Uploads the fetched comic image to a specified Google Drive folder.  
    - **Configuration:**  
      - Filename derived from image URL basename  
      - Fixed folder ID (configurable)  
      - Uses OAuth2 credentials for Google Drive access  
    - **Version:** 3  
    - **Edge Cases:**  
      - Auth failure, quota limits, invalid folder ID.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                          | Input Node(s)                        | Output Node(s)                       | Sticky Note                                                                                   |
|----------------------------------|----------------------------------|----------------------------------------|------------------------------------|------------------------------------|----------------------------------------------------------------------------------------------|
| When a Form is Submitted          | Form Trigger                     | Captures brand/product info via form   | -                                  | Generate Reddit Keyword from Product Description | ## Generate Reddit Keyword                                                                |
| Generate Reddit Keyword from Product Description | LangChain OpenAI (chat)            | Extracts Reddit keyword from product description | When a Form is Submitted            | Search Reddit Posts Using Keyword    | ## Generate Reddit Keyword                                                                |
| Search Reddit Posts Using Keyword | Reddit Node                      | Searches Reddit posts by keyword       | Generate Reddit Keyword from Product Description | Filter Posts With 2+ Upvotes and Text | ## Scrape Reddit                                                                          |
| Filter Posts With 2+ Upvotes and Text | If Node                         | Filters posts by upvotes and text      | Search Reddit Posts Using Keyword    | Classify Post Relevance to Product  | ## Filter Posts By Popularity                                                             |
| Classify Post Relevance to Product | LangChain Agent                 | Classifies relevance of Reddit posts   | Filter Posts With 2+ Upvotes and Text | Format Relevant Reddit Post Fields | ## Post Relevance Classifier                                                             |
| GPT Model: Classify Relevance    | LangChain LM Chat OpenAI         | Executes classification model          | Classify Post Relevance to Product  | Support Relevance Classification    | ## Post Relevance Classifier                                                             |
| Support Relevance Classification | LangChain toolThink              | Supports classification reasoning      | GPT Model: Classify Relevance       | Parse Relevance Classifier Output   | ## Post Relevance Classifier                                                             |
| Parse Relevance Classifier Output | LangChain Output Parser          | Parses classification JSON output      | Support Relevance Classification    | Format Relevant Reddit Post Fields  | ## Post Relevance Classifier                                                             |
| Format Relevant Reddit Post Fields | Set Node                       | Extracts key Reddit post fields        | Parse Relevance Classifier Output   | Aggregate Relevant Reddit Data      |                                                                                            |
| Aggregate Relevant Reddit Data   | Aggregate Node                   | Aggregates all relevant posts data     | Format Relevant Reddit Post Fields  | Generate 10 Ad Angles From Reddit Posts |                                                                                            |
| Generate 10 Ad Angles From Reddit Posts | LangChain Agent               | Creates 10 emotional ad angles         | Aggregate Relevant Reddit Data      | Rank Top 10 Ad Angles               | ## Turn Painpoints Into Messaging                                                        |
| Ad Angle Generation              | LangChain LM Chat OpenAI         | Executes ad angle generation model     | Generate 10 Ad Angles From Reddit Posts | Support Ad Angle Generation         | ## Turn Painpoints Into Messaging                                                        |
| Support Ad Angle Generation      | LangChain toolThink              | Supports ad angle generation reasoning | Ad Angle Generation                 | Generate 10 Ad Angles From Reddit Posts | ## Turn Painpoints Into Messaging                                                        |
| Rank Top 10 Ad Angles            | LangChain OpenAI                 | Ranks and selects top 10 ad angles     | Generate 10 Ad Angles From Reddit Posts | Create Comic Prompts from Ad Angles | ## Rank Top 10 Messaging                                                                |
| Create Comic Prompts from Ad Angles | LangChain Agent               | Creates 4-panel comic image prompts    | Rank Top 10 Ad Angles               | Split Comic Prompt Objects          | ## Create Image Prompt                                                                 |
| Comic Prompt Creation            | LangChain LM Chat OpenAI         | Executes comic prompt generation model | Create Comic Prompts from Ad Angles | Support Comic Prompt Creation       | ## Create Image Prompt                                                                 |
| Support Comic Prompt Creation    | LangChain toolThink              | Supports comic prompt generation       | Comic Prompt Creation               | Parse Comic Prompt JSON Output      | ## Create Image Prompt                                                                 |
| Parse Comic Prompt JSON Output   | LangChain Output Parser          | Parses JSON array of comic prompts     | Support Comic Prompt Creation       | Split Comic Prompt Objects          | ## Create Image Prompt                                                                 |
| Split Comic Prompt Objects       | SplitOut Node                   | Splits comic prompt array into items   | Parse Comic Prompt JSON Output      | Loop Through Comic Prompts          | ## Create Image Prompt                                                                 |
| Loop Through Comic Prompts       | SplitInBatches                  | Processes comic prompts iteratively    | Split Comic Prompt Objects          | Generate Comic Image via Dumpling AI | ## Create Images                                                                       |
| Generate Comic Image via Dumpling AI | HTTP Request                 | Calls Dumpling AI to generate images   | Loop Through Comic Prompts          | Fetch Comic Image URL               | ## Create Images                                                                       |
| Fetch Comic Image URL            | HTTP Request                   | Retrieves generated image URL           | Generate Comic Image via Dumpling AI | Upload Image to Google Drive         | ## Create Images                                                                       |
| Upload Image to Google Drive     | Google Drive Node               | Uploads comic images to Google Drive   | Fetch Comic Image URL               | Loop Through Comic Prompts          | ## Create Images                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "When a Form is Submitted"  
   - Configure form fields:  
     - Brand Name (required)  
     - Website (required)  
     - Product Explanation (required)  
   - Set form title: "Add Details Here"  
   - Save and activate webhook.

2. **Add LangChain OpenAI Node for Reddit Keyword Extraction**  
   - Type: LangChain OpenAI (ChatGPT)  
   - Name: "Generate Reddit Keyword from Product Description"  
   - Model: chatgpt-4o-latest  
   - Prompt:  
     ```
     You are a marketing strategist and Reddit researcher. 
     Given the following product description, extract a single keyword or phrase (1 to 3 words max) 
     that people on Reddit might use when discussing pain points related to this product or service.

     Return ONLY the keyword or phrase, no extra text.

     Product Description: {{ $json['Product Explanation'] }}
     ```  
   - Credential: OpenAI API  
   - Connect output of form trigger node to this node.

3. **Add Reddit Node to Search Posts**  
   - Type: Reddit  
   - Name: "Search Reddit Posts Using Keyword"  
   - Operation: Search  
   - Location: All Reddit  
   - Limit: 10  
   - Keyword: Use expression to reference previous node's output keyword: `={{ $json.message.content }}`  
   - Credentials: Reddit OAuth2  
   - Connect output of Reddit keyword node as input.

4. **Add If Node to Filter Posts**  
   - Type: If  
   - Name: "Filter Posts With 2+ Upvotes and Text"  
   - Conditions (AND):  
     - `$json.ups >= 2` (number greater or equal)  
     - `$json.selftext` is not empty (string not empty)  
   - Connect output of Reddit search node.

5. **Add LangChain Agent Node for Post Relevance Classification**  
   - Type: LangChain Agent  
   - Name: "Classify Post Relevance to Product"  
   - Prompt:  
     ```
     Important: always use the think2 tool

     You are a content relevance classifier.

     Given the following DTC product description:
     {{ $('When a Form is Submitted').item.json['Product Explanation'] }}

     And the details of a single Reddit post:
     • Title: {{ $('Search Reddit Posts Using Keyword').item.json.title }}
     • Content: {{ $('Search Reddit Posts Using Keyword').item.json.selftext }}
     • Subreddit: {{ $('Search Reddit Posts Using Keyword').item.json.subreddit }}
     • Upvotes: {{ $('Search Reddit Posts Using Keyword').item.json.ups }}
     • Upvote Ratio: {{ $('Search Reddit Posts Using Keyword').item.json.upvote_ratio }}
     • URL: {{ $('Search Reddit Posts Using Keyword').item.json.url }}

     Your task is to determine whether this post is relevant for marketing our product.
     Return **only** a JSON array with one object containing two fields:
     - `isRelevant`: true if the post is relevant, otherwise false
     - `reason`: short explanation of why

     Example output:
     [
       {
         "isRelevant": true,
         "reason": "The user describes the exact pain point our product solves."
       }
     ]
     ```  
   - Model: GPT-4o-mini via LangChain LM Chat OpenAI node  
   - Connect output from filter node.

6. **Add Supporting Nodes for Classification**  
   - LangChain toolThink node connected as AI tool to classification agent for chain-of-thought.  
   - LangChain Output Parser node with JSON schema for output validation.  
   - LangChain LM Chat OpenAI node for base model execution.

7. **Add Set Node to Extract Relevant Reddit Fields**  
   - Name: "Format Relevant Reddit Post Fields"  
   - Assign variables from Reddit post JSON: upvotes, subreddit, originalPost (selftext), title, url.  
   - Connect output of parsed classification node.

8. **Add Aggregate Node to Combine Relevant Posts**  
   - Name: "Aggregate Relevant Reddit Data"  
   - Operation: Aggregate all item data into a single array.  
   - Connect output of formatting node.

9. **Add LangChain Agent Node to Generate 10 Ad Angles**  
   - Name: "Generate 10 Ad Angles From Reddit Posts"  
   - Prompt:  
     ```
     You are a copywriter and strategist helping a DTC brand turn Reddit pain points into powerful marketing messages.

     Here are Reddit posts expressing problems/frustrations:

     {{ JSON.stringify($json.data) }}

     The brand's product is described as: {{ $('When a Form is Submitted').item.json['Product Explanation'] }}

     Create 10 unique, punchy ad angles (1-2 sentences) framed as hero moments where the product solves the issue.

     Output as numbered list 1 to 10.
     ```  
   - Model: GPT-4o-mini  
   - Connect output of aggregation node.

10. **Add Supporting ToolThink and LM Chat Nodes for Ad Angle Generation**  
    - ToolThink node connected as AI tool to generation agent.  
    - LM Chat OpenAI node executes the generation.

11. **Add LangChain OpenAI Node to Rank Top 10 Ad Angles**  
    - Name: "Rank Top 10 Ad Angles"  
    - Prompt:  
      ```
      You are a marketing strategist evaluating ad angles for a DTC product.

      Product description: {{ $('When a Form is Submitted').item.json['Product Explanation'] }}

      Variations: {{ $json.output }}

      Instructions:
      1. Select top 10 most relevant and persuasive.
      2. Re-rank from best to least.
      3. Return only numbered list 1 to 10, no commentary.
      4. Ignore vague or redundant lines.
      ```  
    - Model: chatgpt-4o-latest  
    - Connect output from ad angle generation.

12. **Add LangChain Agent Node for Comic Prompt Creation**  
    - Name: "Create Comic Prompts from Ad Angles"  
    - Prompt:  
      ```
      You are a creative director making 4-panel comic ads.

      Product: {{ $('When a Form is Submitted').item.json['Product Explanation'] }}

      Ad message: {{ $json.message.content }}

      Write an AI image prompt describing a 4-panel comic visual story:
      Panel 1: setup/pain point
      Panel 2: escalation
      Panel 3: product hero moment
      Panel 4: positive outcome

      Return a JSON array of 10 objects each with "prompt" property.
      ```  
    - Model: GPT-4o-mini  
    - Connect output of ranking node.

13. **Add Supporting Nodes for Comic Prompt Creation**  
    - ToolThink node connected as AI tool.  
    - Output Parser node with JSON schema for prompts.  
    - SplitOut node to split array into individual prompt objects.

14. **Add SplitInBatches Node to Process Prompts Iteratively**  
    - Name: "Loop Through Comic Prompts"  
    - Batch size: 1 (default)  
    - Connect output of SplitOut node.

15. **Add HTTP Request Node to Generate Comic Image via Dumpling AI**  
    - Name: "Generate Comic Image via Dumpling AI"  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`  
    - Body (JSON):  
      ```json
      {
        "model": "FLUX.1-pro",
        "input": {
          "prompt": "{{ $json.prompt.replace(/\\n/g, ' ').replace(/\\r/g, '').replace(/\\s+/g, ' ').trim() }}",
          "seed": 42,
          "steps": 25,
          "width": 512,
          "height": 512,
          "guidance": 3,
          "interval": 2,
          "aspect_ratio": "1:1",
          "output_format": "jpg",
          "output_quality": 80,
          "safety_tolerance": 2,
          "prompt_upsampling": false
        }
      }
      ```  
    - Authentication: HTTP Header with Dumpling AI API key credential.  
    - Connect output of batch node.

16. **Add HTTP Request Node to Fetch Generated Image URL**  
    - Name: "Fetch Comic Image URL"  
    - Method: GET  
    - URL: `={{ $json.images[0].url }}`  
    - Connect output from Dumpling AI generation node.

17. **Add Google Drive Node to Upload Images**  
    - Name: "Upload Image to Google Drive"  
    - Operation: Upload file by URL  
    - File name: `={{ $json.images[0].url.split('/').pop() }}`  
    - Destination folder: Set folder ID (e.g., "1R5bTxrKmi9NDMFJIh3aQgbNuZwmCybLV")  
    - Credentials: Google Drive OAuth2  
    - Connect output of fetch image URL node.

18. **Loopback:** Connect output of Google Drive node back to the batch node to process next image.

19. **Activate Workflow** and test end-to-end with sample data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                        |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The workflow uses chain-of-thought reasoning tools (`toolThink`) to improve AI classification and generation accuracy.         | Internal n8n LangChain feature                                        |
| Dumpling AI requires an API key and supports safety tolerance settings to filter inappropriate content in generated images.   | https://app.dumplingai.com                                            |
| Google Drive folder used for image storage must be pre-configured with write permissions for the linked OAuth2 credentials.   | Google Drive API documentation                                        |
| The workflow depends heavily on OpenAI API availability and quota; monitoring usage is recommended for production environments.| https://platform.openai.com/docs                                     |
| Reddit API OAuth2 credentials must have appropriate permissions to perform search operations.                                  | https://www.reddit.com/dev/api/                                      |
| The input form webhook URL must be publicly accessible to receive user submissions.                                            | n8n webhook configuration                                            |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly available.