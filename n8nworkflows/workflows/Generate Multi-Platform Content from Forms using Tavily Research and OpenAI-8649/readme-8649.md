Generate Multi-Platform Content from Forms using Tavily Research and OpenAI

https://n8nworkflows.xyz/workflows/generate-multi-platform-content-from-forms-using-tavily-research-and-openai-8649


# Generate Multi-Platform Content from Forms using Tavily Research and OpenAI

### 1. Workflow Overview

This workflow automates the creation of tailored content across multiple social media platforms from user-submitted form data. It targets content marketers, social media managers, and brands seeking efficient generation of relevant and engaging posts for LinkedIn, Facebook, and blogs.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Captures user inputs from a form specifying content subject and target audience.
- **1.2 Search Preparation**: Extracts and formats search parameters based on form data.
- **1.3 Web Research**: Queries the Tavily API to gather current, relevant information on the content subject.
- **1.4 Data Processing**: Splits and aggregates multiple search results into a structured format for AI consumption.
- **1.5 AI Content Generation**: Uses OpenAI-powered LangChain agents to generate platform-specific posts — blog, LinkedIn, and Facebook — tailored to the target audience.
- **1.6 Content Packaging**: Organizes generated content into a unified output structure.
- **1.7 Notification**: Sends a Slack notification containing all generated content for review.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures content topic and intended audience via a form submission to trigger the workflow.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point; receives "Content Subject" and "Target Audience" as required fields from a form titled "Content Details"  
    - Configuration: Form fields are mandatory; webhook automatically created for form submissions  
    - Input: None (trigger node)  
    - Output: JSON containing user-submitted fields  
    - Edge cases: Missing required fields will block trigger; network/webhook errors could prevent activation  

#### 2.2 Search Preparation

- **Overview:**  
  Extracts search query and target audience information from form data, preparing variables for web search.

- **Nodes Involved:**  
  - Set Search Fields

- **Node Details:**  
  - **Set Search Fields**  
    - Type: Set node  
    - Role: Assigns variables `query` (content subject) and `targetAudience` from the form submission JSON  
    - Configuration: Uses expressions to map form fields to workflow variables  
    - Input: Receives JSON from "On form submission"  
    - Output: JSON with `query` and `targetAudience` fields ready for API call  
    - Edge cases: If form data is malformed or missing expected keys, expressions may fail or produce empty values  

#### 2.3 Web Research

- **Overview:**  
  Performs an HTTP POST request to the Tavily API to fetch relevant news and content related to the query term.

- **Nodes Involved:**  
  - Search Web

- **Node Details:**  
  - **Search Web**  
    - Type: HTTP Request  
    - Role: Sends search request to Tavily API with parameters including API key, query, search depth, and max results  
    - Configuration:  
      - POST method to https://api.tavily.com/search  
      - JSON body includes dynamic `query` from previous node, fixed parameters for topic ("news"), and max results (3)  
      - Requires user to replace placeholder API key with actual Tavily API key  
    - Input: Receives search parameters from "Set Search Fields"  
    - Output: JSON search results including raw content and answers  
    - Edge cases:  
      - Missing or invalid API key results in authentication failure  
      - Network issues or API downtime may cause timeouts or errors  
      - Unexpected API response format could break downstream processing  

#### 2.4 Data Processing

- **Overview:**  
  Processes the multiple search results by splitting individual results and then aggregating specified fields into a single combined object for AI consumption.

- **Nodes Involved:**  
  - Split Out  
  - Aggregate

- **Node Details:**  
  - **Split Out**  
    - Type: Split Out  
    - Role: Extracts each search result from the `results` array into separate items for processing  
    - Configuration: Splits on field `results`  
    - Input: Receives raw API response from "Search Web"  
    - Output: Multiple items, one per search result  
    - Edge cases: If results array is empty or missing, no output items generated  
  - **Aggregate**  
    - Type: Aggregate  
    - Role: Combines all split items into a single JSON object including only `title` and `raw_content` fields for each result  
    - Configuration: Includes only specified fields (`title`, `raw_content`) from all items  
    - Input: Receives multiple items from "Split Out"  
    - Output: Single aggregated item with combined data array  
    - Edge cases: Empty input results in empty aggregation; malformed data fields could be omitted  

#### 2.5 AI Content Generation

- **Overview:**  
  Generates platform-specific content by feeding aggregated research data and audience parameters into three LangChain OpenAI agent nodes, each specialized for blog, LinkedIn, and Facebook content styles.

- **Nodes Involved:**  
  - Blog Writer  
  - LinkedIn  
  - Facebook  
  - OpenAI Chat Model (shared AI model credential)

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides the OpenAI API connection and AI model for all agent nodes  
    - Configuration: Uses user-configured OpenAI API credentials  
    - Input: AI prompt inputs from all three agent nodes  
    - Output: AI-generated text for each platform  
    - Edge cases: API quota limits, authentication failures, or model errors can cause failure in all AI nodes  
  - **Blog Writer**  
    - Type: LangChain Conversational Agent  
    - Role: Produces a two-paragraph professional blog article based on aggregated content and target audience  
    - Configuration:  
      - System message defines a skilled blog writer role with detailed writing instructions, tone, structure, and examples  
      - Input text includes aggregated research data and target audience variable  
    - Input: Aggregated data from "Aggregate" and audience from "Set Search Fields"  
    - Output: Blog post text  
    - Edge cases: AI hallucination or incoherent output; input data missing or empty may cause poor results  
  - **LinkedIn**  
    - Type: LangChain Conversational Agent  
    - Role: Converts article content into a LinkedIn post optimized for professional audiences with emojis, hashtags, and a call to action  
    - Configuration:  
      - System message defines LinkedIn specialist role with clear post requirements and examples  
      - Input text includes article content and target audience parameters  
    - Input: Receives output from "Aggregate" and audience data  
    - Output: LinkedIn post text  
    - Edge cases: Similar to Blog Writer; AI output length limits or formatting issues  
  - **Facebook**  
    - Type: LangChain Conversational Agent  
    - Role: Converts article content into concise, high-impact Facebook posts tailored for target audience with emojis and hashtags  
    - Configuration:  
      - System message defines Facebook content creator role with specific tone and formatting instructions  
      - Input text includes aggregated content and audience data  
    - Input: Receives aggregated data and audience info  
    - Output: Facebook post text  
    - Edge cases: Same as above; risk of AI output not matching intended style or length  

#### 2.6 Content Packaging

- **Overview:**  
  Collects all three generated content pieces and assigns them into a single JSON object for final delivery.

- **Nodes Involved:**  
  - Edit Fields

- **Node Details:**  
  - **Edit Fields**  
    - Type: Set node  
    - Role: Assigns the outputs from Blog Writer, Facebook, and LinkedIn nodes into named fields (`Blog`, `Facebook`, `Linkedin`)  
    - Configuration: Uses expressions to map each platform’s AI output into respective fields  
    - Input: Receives AI-generated content from Blog Writer, Facebook, and LinkedIn nodes  
    - Output: Single JSON object with all three content variants  
    - Edge cases: Missing or failed AI node outputs yield empty or null fields  

#### 2.7 Notification

- **Overview:**  
  Sends a Slack message containing all generated content to a designated channel for review and distribution.

- **Nodes Involved:**  
  - Completed Notification

- **Node Details:**  
  - **Completed Notification**  
    - Type: Slack node  
    - Role: Posts a formatted message with the content subject and all generated posts to a Slack channel  
    - Configuration:  
      - Uses Slack OAuth2 credentials  
      - Sends to a fixed channel by ID  
      - Message text dynamically inserts form submission subject and content fields (`Blog`, `Facebook`, `Linkedin`) from previous node  
      - Executes once per workflow run  
    - Input: Receives packaged content from "Edit Fields" and form data from "On form submission" for subject  
    - Output: Slack notification posted  
    - Edge cases: Slack API errors, invalid channel ID, or credential expiry can prevent notification delivery  

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                         | Input Node(s)              | Output Node(s)          | Sticky Note                                                                                  |
|---------------------|-----------------------------------|---------------------------------------|----------------------------|-------------------------|----------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                      | Entry point: receive form data        | None                       | Set Search Fields        |                                                                                              |
| Set Search Fields    | Set                              | Extract search query and audience     | On form submission          | Search Web               |                                                                                              |
| Search Web          | HTTP Request                     | Call Tavily API to get relevant info | Set Search Fields           | Split Out                | See instructions to configure Tavily API key in sticky note                                  |
| Split Out           | Split Out                        | Split API results array into items    | Search Web                  | Aggregate                |                                                                                              |
| Aggregate           | Aggregate                        | Aggregate selected fields for AI      | Split Out                   | LinkedIn                 |                                                                                              |
| LinkedIn            | LangChain Conversational Agent   | Generate LinkedIn post                 | Aggregate, OpenAI Chat Model| Facebook                 |                                                                                              |
| Facebook            | LangChain Conversational Agent   | Generate Facebook post                 | Blog Writer, OpenAI Chat Model | Blog Writer           |                                                                                              |
| Blog Writer         | LangChain Conversational Agent   | Generate blog article                  | Aggregate, OpenAI Chat Model| Edit Fields              |                                                                                              |
| Edit Fields         | Set                              | Package all generated content          | Blog Writer, Facebook, LinkedIn | Completed Notification |                                                                                              |
| Completed Notification| Slack                           | Send Slack notification with content  | Edit Fields                 | None                    |                                                                                              |
| OpenAI Chat Model   | LangChain OpenAI Chat Model       | Provide OpenAI API connection          | None (AI input from agents) | LinkedIn, Facebook, Blog Writer |                                                                                              |
| Sticky Note         | Sticky Note                      | Documentation and instructions         | None                       | None                    | ## Content Creation System from Form Input - See workflow description and Tavily API setup  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger**  
   - Add a **Form Trigger** node named `On form submission`.  
   - Configure form title as "Content Details".  
   - Add two required fields: "Content Subject" and "Target Audience".  
   - This node triggers the workflow on form submission.

2. **Set Search Parameters**  
   - Add a **Set** node named `Set Search Fields`.  
   - Connect from `On form submission`.  
   - Create two string fields:  
     - `query` set to `={{ $json["Content Subject"] }}`  
     - `targetAudience` set to `={{ $json["Target Audience"] }}`

3. **HTTP Request to Tavily API**  
   - Add an **HTTP Request** node named `Search Web`.  
   - Connect from `Set Search Fields`.  
   - Set method to POST and URL to `https://api.tavily.com/search`.  
   - Set the body type to JSON with this payload (replace the API key placeholder):  
     ```json
     {
       "api_key": "YOUR_TAVILY_API_KEY",
       "query": "{{ $json.query.replace(/\"/g, '\\\"') }}",
       "search_depth": "basic",
       "include_answer": true,
       "topic": "news",
       "include_raw_content": true,
       "max_results": 3
     }
     ```
   - Enable sending body as JSON.

4. **Split Results**  
   - Add a **Split Out** node named `Split Out`.  
   - Connect from `Search Web`.  
   - Set field to split out as `results`.

5. **Aggregate Selected Fields**  
   - Add an **Aggregate** node named `Aggregate`.  
   - Connect from `Split Out`.  
   - Configure to aggregate all item data including only fields `title` and `raw_content`.

6. **Configure OpenAI Chat Model Credential**  
   - Set up OpenAI API credentials in n8n with your API key.  
   - Add a **LangChain OpenAI Chat Model** node named `OpenAI Chat Model`.  
   - Configure to use your OpenAI API credentials.

7. **Create Blog Writer Agent**  
   - Add a **LangChain Conversational Agent** node named `Blog Writer`.  
   - Connect main input from `Aggregate`.  
   - Connect AI language model input from `OpenAI Chat Model`.  
   - Set prompt text to:  
     ```
     Article Content:
     {{ $('Aggregate').item.json.data.toJsonString() }}

     Target Audience:
     {{ $('Set Search Fields').item.json.targetAudience }}
     ```  
   - Set agent to "conversationalAgent".  
   - Paste the detailed system message defining the blog writer role and writing instructions.

8. **Create LinkedIn Agent**  
   - Add a **LangChain Conversational Agent** node named `LinkedIn`.  
   - Connect main input from `Aggregate`.  
   - Connect AI language model from `OpenAI Chat Model`.  
   - Use prompt text:  
     ```
     Article Content:
     {{ $json.data.toJsonString() }}

     Target Audience:
     {{ $('Set Search Fields').item.json.targetAudience }}
     ```  
   - Use system message specifying LinkedIn content creation guidelines, including emoji and hashtag requirements.

9. **Create Facebook Agent**  
   - Add a **LangChain Conversational Agent** node named `Facebook`.  
   - Connect main input from `Blog Writer`.  
   - Connect AI language model from `OpenAI Chat Model`.  
   - Use prompt text:  
     ```
     Article Content:
     {{ $('Aggregate').item.json.data.toJsonString() }}

     Target Audience:
     {{ $('Set Search Fields').item.json.targetAudience }}
     ```  
   - Use system message specifying Facebook post style and formatting.

10. **Edit Fields to Package Content**  
    - Add a **Set** node named `Edit Fields`.  
    - Connect from `Blog Writer`.  
    - Assign three string fields:  
      - `Blog` = `={{ $json.output }}` (Blog Writer output)  
      - `Facebook` = `={{ $('Facebook').item.json.output }}`  
      - `Linkedin` = `={{ $('LinkedIn').item.json.output }}`

11. **Slack Notification**  
    - Add a **Slack** node named `Completed Notification`.  
    - Connect from `Edit Fields`.  
    - Configure Slack OAuth2 credentials.  
    - Set channel ID to the desired Slack channel.  
    - Set message text with dynamic insertion of content subject and all three content fields:  
      ```
      ✅ The Content Generated on the Topic of: {{ $('On form submission').item.json["Content Subject"] }}

      Blog Post: 
      {{ $json.Blog }}

      Facebook Post:
      {{ $json.Facebook }}

      Linkedin Post:
      {{ $json.Linkedin }}
      ```

12. **Sticky Note (Optional Documentation Node)**  
    - Add a **Sticky Note** node with the provided content outlining workflow purpose, instructions, and API setup notes.

13. **Connect Node Flow**  
    - Ensure connections follow this order:  
      `On form submission` → `Set Search Fields` → `Search Web` → `Split Out` → `Aggregate` → (`LinkedIn`, `Facebook`, `Blog Writer`) → `Edit Fields` → `Completed Notification`  
    - Connect `OpenAI Chat Model` node’s AI language model input to all three agent nodes (`LinkedIn`, `Facebook`, `Blog Writer`).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                      | Context or Link                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow automates multi-platform content creation by transforming form submissions into tailored blog posts, LinkedIn posts, and Facebook posts using AI and web research.                                                                 | Workflow purpose overview              |
| To use the Tavily API node, obtain your API key by signing up at [tavily.com](https://tavily.com) and add your key in the "Search Web" node JSON body replacing the placeholder.                                                                     | Tavily API setup instructions         |
| Blog writing prompt includes detailed instructions and examples ensuring concise, engaging two-paragraph articles suitable for general audiences.                                                                                                | Blog Writer node system message        |
| LinkedIn post prompt emphasizes clarity, engagement, emojis, hashtags, and a call to action tailored to professional audiences.                                                                                                                  | LinkedIn node system message            |
| Facebook post prompt focuses on short, attention-grabbing content with emojis and hashtags optimized for mobile reading and interaction.                                                                                                         | Facebook node system message            |
| Slack notification posts all generated content into a designated channel for easy team review and distribution.                                                                                                                                    | Slack node notification setup          |

---

**Disclaimer:** The text provided is exclusively derived from an n8n workflow automation. It adheres strictly to current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.