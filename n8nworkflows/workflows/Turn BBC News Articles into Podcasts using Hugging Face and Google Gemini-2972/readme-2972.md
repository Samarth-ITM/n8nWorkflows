Turn BBC News Articles into Podcasts using Hugging Face and Google Gemini

https://n8nworkflows.xyz/workflows/turn-bbc-news-articles-into-podcasts-using-hugging-face-and-google-gemini-2972


# Turn BBC News Articles into Podcasts using Hugging Face and Google Gemini

### 1. Workflow Overview

This n8n workflow automates the transformation of BBC News articles into engaging podcast audio files using Hugging Face's text-to-speech service and Google Gemini's large language model (LLM). It is designed for content creators, students, and individuals seeking to quickly generate audio content from current news without technical expertise.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 News Fetching and Extraction:** Fetches the BBC homepage, extracts news article blocks, and parses titles, links, and descriptions.
- **1.3 News Filtering and Classification:** Limits the number of articles, then uses Google Gemini LLM to classify articles for podcast suitability.
- **1.4 Detailed Content Retrieval:** Fetches full article content for suitable news items and filters out empty results.
- **1.5 Podcast Script Generation:** Aggregates detailed news content and uses Gemini LLM to generate a podcast script in a conversational style.
- **1.6 Text-to-Speech Conversion:** Converts the generated podcast script into audio using Hugging Face's text-to-speech API.
- **1.7 Conditional Execution:** Checks if the podcast script exists before proceeding to audio generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Initiates the workflow manually.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**
  - **Type:** Manual Trigger  
  - **Role:** Starts the workflow on user command.  
  - **Configuration:** Default manual trigger with no parameters.  
  - **Connections:** Outputs to "Fetch BBC News Page".  
  - **Edge Cases:** None; manual trigger ensures controlled execution.

#### 2.2 News Fetching and Extraction

- **Overview:** Retrieves the BBC homepage HTML and extracts news article blocks, including titles, links, and descriptions.
- **Nodes Involved:**  
  - Fetch BBC News Page  
  - Extract News Block  
  - Split Out  
  - Extract News Content  
  - Limit 10 Items
- **Node Details:**

  - **Fetch BBC News Page**  
    - **Type:** HTTP Request  
    - **Role:** Fetches the main BBC News homepage as raw HTML string.  
    - **Configuration:** URL set to `https://www.bbc.com/`, response format as string.  
    - **Connections:** Outputs to "Extract News Block".  
    - **Edge Cases:** Network errors, site structure changes affecting HTML content.

  - **Extract News Block**  
    - **Type:** HTML Extract  
    - **Role:** Extracts HTML blocks containing news titles using CSS selector `.eGcloy`.  
    - **Configuration:** Extracts multiple elements as HTML array under key `newsTitle`.  
    - **Connections:** Outputs to "Split Out".  
    - **Edge Cases:** Selector changes on BBC site, empty extraction results.

  - **Split Out**  
    - **Type:** Split Out  
    - **Role:** Splits the array of news blocks into individual items for processing.  
    - **Configuration:** Splits on field `newsTitle`.  
    - **Connections:** Outputs to "Extract News Content".  
    - **Edge Cases:** Empty input array.

  - **Extract News Content**  
    - **Type:** HTML Extract  
    - **Role:** Parses each news block to extract title (`h2`), link (`a[href]`), and description (`.kYtujW`).  
    - **Configuration:** Extracts these fields per news item.  
    - **Connections:** Outputs to "Limit 10 Items".  
    - **Edge Cases:** Missing elements, malformed HTML.

  - **Limit 10 Items**  
    - **Type:** Limit  
    - **Role:** Restricts processing to the first 10 news articles to optimize performance.  
    - **Configuration:** Max items set to 10.  
    - **Connections:** Outputs to "News Classifier".  
    - **Edge Cases:** Less than 10 items available.

#### 2.3 News Filtering and Classification

- **Overview:** Uses Google Gemini LLM to classify news articles as suitable or not for podcast storytelling based on headline and description.
- **Nodes Involved:**  
  - News Classifier
- **Node Details:**

  - **News Classifier**  
    - **Type:** LangChain Text Classifier (Google Gemini)  
    - **Role:** Classifies each news article's headline and description for podcast suitability.  
    - **Configuration:**  
      - Input text: Concatenates title and description.  
      - Categories: "Suitable" and "Not Suitable" with detailed role descriptions and criteria emphasizing positive, engaging, and broadly interesting news.  
    - **Connections:** Outputs to "Fetch BBC News Detail".  
    - **Edge Cases:** Misclassification, API errors, rate limits.

#### 2.4 Detailed Content Retrieval

- **Overview:** Fetches full article content for news items classified as suitable and filters out any empty content.
- **Nodes Involved:**  
  - Fetch BBC News Detail  
  - Extract Detail  
  - Filter Empty Detail  
  - Aggregate
- **Node Details:**

  - **Fetch BBC News Detail**  
    - **Type:** HTTP Request  
    - **Role:** Retrieves full HTML content of each selected news article by appending the relative link to `https://www.bbc.com`.  
    - **Configuration:** URL dynamically constructed from article link.  
    - **Connections:** Outputs to "Extract Detail".  
    - **Edge Cases:** Broken links, HTTP errors, site structure changes.

  - **Extract Detail**  
    - **Type:** HTML Extract  
    - **Role:** Extracts detailed news content paragraphs using CSS selector `.dlWCEZ .fYAfXe`.  
    - **Configuration:** Returns multiple paragraphs as an array under `newsDetail`.  
    - **Connections:** Outputs to "Filter Empty Detail".  
    - **Edge Cases:** Selector changes, empty content.

  - **Filter Empty Detail**  
    - **Type:** Filter  
    - **Role:** Filters out news items where `newsDetail` is empty or missing.  
    - **Configuration:** Condition checks that `newsDetail` array is not empty.  
    - **Connections:** Outputs to "Aggregate".  
    - **Edge Cases:** All items filtered out, resulting in empty dataset.

  - **Aggregate**  
    - **Type:** Aggregate  
    - **Role:** Combines all filtered news details into a single data array for script generation.  
    - **Configuration:** Aggregates all item data.  
    - **Connections:** Outputs to "Basic Podcast LLM Chain".  
    - **Edge Cases:** Empty input array.

#### 2.5 Podcast Script Generation

- **Overview:** Uses Google Gemini LLM to convert aggregated news articles into a single podcast script formatted for direct speech synthesis.
- **Nodes Involved:**  
  - Basic Podcast LLM Chain  
  - Output Parser  
  - If script exists
- **Node Details:**

  - **Basic Podcast LLM Chain**  
    - **Type:** LangChain LLM Chain (Google Gemini)  
    - **Role:** Generates a podcast script from the news articles content.  
    - **Configuration:**  
      - Input text: Maps aggregated news details to an array of article contents.  
      - Prompt: Detailed instructions to create a warm, conversational podcast script with smooth transitions, formatted as a single JSON string under key `podcast_script`.  
      - Output parser enabled to parse JSON response.  
    - **Connections:**  
      - AI output parser connected to "Output Parser".  
      - Main output connected to "If script exists".  
    - **Edge Cases:** LLM response errors, malformed JSON output, API errors.

  - **Output Parser**  
    - **Type:** LangChain Structured Output Parser  
    - **Role:** Parses the JSON output from the LLM to extract the podcast script string.  
    - **Configuration:** Example JSON schema with key `podcast_script`.  
    - **Connections:** Outputs to "Basic Podcast LLM Chain" (for chaining).  
    - **Edge Cases:** Parsing failures if LLM output is invalid JSON.

  - **If script exists**  
    - **Type:** If Condition  
    - **Role:** Checks if the `podcast_script` exists and is non-empty before proceeding.  
    - **Configuration:** Condition tests existence of `output.podcast_script`.  
    - **Connections:**  
      - True branch outputs to "Hugging Face Text-to-Speech."  
      - False branch does nothing (ends workflow).  
    - **Edge Cases:** Missing or empty script, causing workflow to halt.

#### 2.6 Text-to-Speech Conversion

- **Overview:** Converts the podcast script text into audio using Hugging Face's text-to-speech API.
- **Nodes Involved:**  
  - Hugging Face Text-to-Speech.
- **Node Details:**

  - **Hugging Face Text-to-Speech.**  
    - **Type:** HTTP Request  
    - **Role:** Sends the podcast script to Hugging Face's TTS model endpoint and receives audio data.  
    - **Configuration:**  
      - URL: `https://router.huggingface.co/hf-inference/models/facebook/mms-tts-eng`  
      - Method: POST  
      - Body: JSON with `inputs` set to the podcast script text.  
      - Authentication: Uses predefined Hugging Face API credentials.  
    - **Connections:** None (end node).  
    - **Edge Cases:** API authentication errors, rate limits, TTS model failures, malformed input.

---

### 3. Summary Table

| Node Name                  | Node Type                          | Functional Role                                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                     |
|----------------------------|----------------------------------|-------------------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts the workflow manually                     |                               | Fetch BBC News Page            |                                                                                                                |
| Fetch BBC News Page         | HTTP Request                     | Fetches BBC homepage HTML                        | When clicking ‘Test workflow’  | Extract News Block             | ## This node fetches the main BBC News page, which contains links to various news articles.                   |
| Extract News Block          | HTML Extract                     | Extracts news article blocks from homepage HTML | Fetch BBC News Page            | Split Out                     |                                                                                                                |
| Split Out                  | Split Out                        | Splits news blocks array into individual items  | Extract News Block             | Extract News Content           |                                                                                                                |
| Extract News Content        | HTML Extract                     | Extracts title, link, description per article   | Split Out                     | Limit 10 Items                |                                                                                                                |
| Limit 10 Items              | Limit                           | Limits processing to first 10 articles          | Extract News Content           | News Classifier               |                                                                                                                |
| News Classifier             | LangChain Text Classifier (Gemini) | Classifies articles for podcast suitability     | Limit 10 Items                | Fetch BBC News Detail          | ## This node uses a Gemini LLM to classify news articles based on their titles and descriptions. It determines if the content is suitable for a podcast. |
| Fetch BBC News Detail       | HTTP Request                    | Fetches full article content for suitable news  | News Classifier               | Extract Detail                | ## This node fetches the detailed content of the news articles that were classified as suitable for a podcast. |
| Extract Detail             | HTML Extract                    | Extracts detailed news content paragraphs        | Fetch BBC News Detail          | Filter Empty Detail            |                                                                                                                |
| Filter Empty Detail         | Filter                         | Filters out articles with empty content          | Extract Detail                | Aggregate                    |                                                                                                                |
| Aggregate                  | Aggregate                      | Aggregates all filtered news details             | Filter Empty Detail            | Basic Podcast LLM Chain       |                                                                                                                |
| Basic Podcast LLM Chain     | LangChain LLM Chain (Gemini)    | Generates podcast script from news articles      | Aggregate                    | If script exists, Output Parser | ## This node uses a Gemini LLM to convert the news articles into a podcast script.                            |
| Output Parser              | LangChain Output Parser          | Parses JSON output from LLM                        | Basic Podcast LLM Chain (AI output parser) | Basic Podcast LLM Chain (ai_outputParser) |                                                                                                                |
| If script exists            | If Condition                   | Checks if podcast script exists before TTS       | Basic Podcast LLM Chain       | Hugging Face Text-to-Speech.  |                                                                                                                |
| Hugging Face Text-to-Speech.| HTTP Request                   | Converts podcast script text to speech audio     | If script exists              |                               | ##  It structures the script for direct use with the Hugging Face text-to-speech model.                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters needed.

2. **Create HTTP Request Node to Fetch BBC News Page**  
   - Type: HTTP Request  
   - Name: "Fetch BBC News Page"  
   - URL: `https://www.bbc.com/`  
   - Response Format: String  
   - Connect output of manual trigger to this node.

3. **Create HTML Extract Node to Extract News Blocks**  
   - Type: HTML Extract  
   - Name: "Extract News Block"  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `newsTitle`  
     - CSS Selector: `.eGcloy`  
     - Return Array: True  
     - Return Value: HTML  
   - Connect output of "Fetch BBC News Page" to this node.

4. **Create Split Out Node to Split News Blocks**  
   - Type: Split Out  
   - Name: "Split Out"  
   - Field to Split Out: `newsTitle`  
   - Connect output of "Extract News Block" to this node.

5. **Create HTML Extract Node to Extract News Content**  
   - Type: HTML Extract  
   - Name: "Extract News Content"  
   - Operation: Extract HTML Content  
   - Data Property Name: `newsTitle`  
   - Extraction Values:  
     - Key: `title`, CSS Selector: `h2`  
     - Key: `link`, CSS Selector: `a`, Attribute: `href`, Return Value: Attribute  
     - Key: `description`, CSS Selector: `.kYtujW`  
   - Connect output of "Split Out" to this node.

6. **Create Limit Node to Limit to 10 Items**  
   - Type: Limit  
   - Name: "Limit 10 Items"  
   - Max Items: 10  
   - Connect output of "Extract News Content" to this node.

7. **Create LangChain Text Classifier Node (Google Gemini)**  
   - Type: LangChain Text Classifier  
   - Name: "News Classifier"  
   - Input Text:  
     ```
     I will only send the headline as input:
     {{ $json.title }} {{ $json.description }}
     ```  
   - Categories:  
     - Suitable (with detailed role and criteria focused on positive, engaging news)  
     - Not Suitable (with criteria for negative, routine, or divisive content)  
   - Connect output of "Limit 10 Items" to this node.  
   - Configure Google Gemini credentials (Google Palm API) with your account.

8. **Create HTTP Request Node to Fetch Full News Detail**  
   - Type: HTTP Request  
   - Name: "Fetch BBC News Detail"  
   - URL: `=https://www.bbc.com{{ $json.link }}` (expression to append relative link)  
   - Connect output of "News Classifier" to this node.

9. **Create HTML Extract Node to Extract Detailed Content**  
   - Type: HTML Extract  
   - Name: "Extract Detail"  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `newsDetail`  
     - CSS Selector: `.dlWCEZ .fYAfXe`  
     - Return Array: True  
   - Connect output of "Fetch BBC News Detail" to this node.

10. **Create Filter Node to Remove Empty Details**  
    - Type: Filter  
    - Name: "Filter Empty Detail"  
    - Condition: `newsDetail` array is not empty  
    - Connect output of "Extract Detail" to this node.

11. **Create Aggregate Node to Combine News Details**  
    - Type: Aggregate  
    - Name: "Aggregate"  
    - Aggregate: Aggregate All Item Data  
    - Connect output of "Filter Empty Detail" to this node.

12. **Create LangChain LLM Chain Node for Podcast Script Generation**  
    - Type: LangChain Chain LLM (Google Gemini)  
    - Name: "Basic Podcast LLM Chain"  
    - Text Input:  
      ```
      =News Articles:{{ $json.data.map(item => item.newsDetail) }}
      ```  
    - Messages: Use detailed prompt instructing to convert news articles into a warm, engaging podcast script formatted as a single JSON string with key `podcast_script`.  
    - Enable Output Parser with JSON schema example:  
      ```
      {
        "podcast_script": "Sample text"
      }
      ```  
    - Connect output of "Aggregate" to this node.  
    - Configure Google Gemini credentials.

13. **Create LangChain Output Parser Node**  
    - Type: LangChain Output Parser Structured  
    - Name: "Output Parser"  
    - JSON Schema Example:  
      ```
      {
        "podcast_script": "California"
      }
      ```  
    - Connect AI output parser of "Basic Podcast LLM Chain" to this node.

14. **Create If Node to Check Script Existence**  
    - Type: If  
    - Name: "If script exists"  
    - Condition: Check if `output.podcast_script` exists and is non-empty  
    - Connect main output of "Basic Podcast LLM Chain" to this node.

15. **Create HTTP Request Node for Hugging Face Text-to-Speech**  
    - Type: HTTP Request  
    - Name: "Hugging Face Text-to-Speech."  
    - URL: `https://router.huggingface.co/hf-inference/models/facebook/mms-tts-eng`  
    - Method: POST  
    - Body Parameters: JSON with key `inputs` set to `={{ $json.output.podcast_script }}`  
    - Authentication: Predefined Credential Type using Hugging Face API credentials (token required)  
    - Connect "If script exists" true branch to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow fetches the main BBC News page, which contains links to various news articles.    | Sticky Note on "Fetch BBC News Page" node                                                       |
| The News Classifier node uses a Gemini LLM to classify news articles based on their titles and descriptions for podcast suitability. | Sticky Note on "News Classifier" node                                                           |
| The "Fetch BBC News Detail" node retrieves detailed content of news articles classified as suitable. | Sticky Note on "Fetch BBC News Detail" node                                                     |
| The "Basic Podcast LLM Chain" node uses Gemini LLM to convert news articles into a podcast script. | Sticky Note on "Basic Podcast LLM Chain" node                                                   |
| The Hugging Face Text-to-Speech node structures the script for direct use with Hugging Face's TTS model. | Sticky Note on "Hugging Face Text-to-Speech." node                                              |
| 3rd Party Application Requirements: Gemini LLM requires no access token; Hugging Face requires an access token for TTS. | Sticky Note near workflow start                                                                 |
| For more advanced customization, explore n8n documentation and community resources.             | From workflow description                                                                        |
| Workflow benefits include time-saving, ease of use, customization, and cost-effectiveness.       | From workflow description                                                                        |
| Workflow prompt for podcast script generation emphasizes a warm, conversational, and engaging style, suitable for ElevenLabs or similar TTS services. | From "Basic Podcast LLM Chain" prompt content                                                   |

---

This document provides a detailed, structured reference for understanding, reproducing, and modifying the "Turn BBC News Articles into Podcasts using Hugging Face and Google Gemini" workflow in n8n.