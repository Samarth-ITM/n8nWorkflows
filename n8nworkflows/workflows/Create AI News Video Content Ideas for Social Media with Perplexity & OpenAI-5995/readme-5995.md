Create AI News Video Content Ideas for Social Media with Perplexity & OpenAI

https://n8nworkflows.xyz/workflows/create-ai-news-video-content-ideas-for-social-media-with-perplexity---openai-5995


# Create AI News Video Content Ideas for Social Media with Perplexity & OpenAI

### 1. Workflow Overview

This workflow automates the generation of AI-driven short-form video content ideas focused on AI news and trends, tailored for social media marketing purposes. It targets tech founders and marketing agencies aiming to establish authority and attract inbound interest by regularly creating engaging content packages that include video scripts, captions, and text overlays.

The workflow logic is organized into these main blocks:

- **1.1 Scheduled Trigger & Content Retrieval:** Periodically triggers the workflow and fetches fresh AI-related stories from multiple topical areas using the Perplexity AI API.
- **1.2 Content Organization & Aggregation:** Processes and cleans the raw API responses, then merges them into a single combined content package.
- **1.3 AI Content Generation:** Uses OpenAI GPT-4.1-MINI to create structured video content packages (script, caption, overlay) based on the aggregated content and user profile.
- **1.4 Content Extraction & Formatting:** Parses the AI-generated raw text output into structured fields.
- **1.5 Data Persistence & Notifications:** Saves the final content packages to Google Sheets and notifies the user by email.
- **1.6 Auxiliary Setup & Notes:** Includes static information nodes (About Me, sticky notes) for configuration and instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Content Retrieval

**Overview:**  
This block initiates the workflow daily at 6 AM, then queries Perplexity AI to retrieve 5 recent, relevant AI news stories from three topical categories: general AI news, AI market trends, and AI business automation.

**Nodes Involved:**  
- Schedule Trigger  
- Topic 1 (eg-AI News)  
- Topic 2 (eg - AI Market Trends)  
- Topic 3 (eg- AI Business Automation)  

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts workflow daily at 06:00  
  - Configuration: Interval trigger set to trigger at hour 6 daily  
  - Input: None  
  - Output: Trigger event  
  - Edge cases: Missed triggers if n8n instance is down; time zone considerations  

- **Topic 1 (eg-AI News)**  
  - Type: HTTP Request  
  - Role: Queries Perplexity AI API for 5 latest AI news stories  
  - Configuration: POST request to Perplexity API with JSON body to fetch stories from today/yesterday; requires user’s Perplexity API key in Authorization header  
  - Inputs: Triggered from Schedule Trigger  
  - Outputs: Raw API response JSON  
  - Edge cases: API auth failure, rate limits, network timeouts, malformed responses  

- **Topic 2 (eg - AI Market Trends)**  
  - Type: HTTP Request  
  - Role: Queries Perplexity AI API for 5 new AI market and industry trend stories  
  - Configuration: Same as Topic 1 but with a different prompt in request body to focus on market trends  
  - Inputs: Triggered after Organise Content node (see below)  
  - Outputs: Raw API response JSON  
  - Edge cases: Same as Topic 1  

- **Topic 3 (eg- AI Business Automation)**  
  - Type: HTTP Request  
  - Role: Queries Perplexity AI API for 5 stories on AI-driven business automation  
  - Configuration: Similar POST request with specific prompt on business automation  
  - Inputs: Triggered after Organise Content1 node  
  - Outputs: Raw API response JSON  
  - Edge cases: Same as Topic 1  

---

#### 2.2 Content Organization & Aggregation

**Overview:**  
This block extracts relevant message content and citations from the Perplexity AI API responses, then merges all topical content into a single aggregated text and deduplicated citation list.

**Nodes Involved:**  
- Organise Content  
- Organise Content1  
- Organise Content2  
- Merge  
- Combine items  

**Node Details:**

- **Organise Content / Organise Content1 / Organise Content2**  
  - Type: Code (JavaScript)  
  - Role: Extracts the first choice message content and citations from Perplexity API response JSON, normalizes the structure for further processing  
  - Configuration: Reads `choices[0].message.content` and `citations` arrays, outputs simplified JSON with keys: `index`, `role`, `finish_reason`, `content`, `citations`  
  - Inputs: Raw JSON from respective Topic HTTP Request nodes  
  - Outputs: Cleaned content JSON objects  
  - Edge cases: Missing or malformed API response, empty messages, no citations  

- **Merge**  
  - Type: Merge  
  - Role: Combines three input streams from the three Organise Content nodes into one stream for unified processing  
  - Configuration: Number of inputs = 3, merging mode is default (append)  
  - Inputs: Outputs from Organise Content, Organise Content1, Organise Content2  
  - Outputs: Combined list of content items  
  - Edge cases: Input mismatch or missing inputs could cause merge failure  

- **Combine items**  
  - Type: Code (JavaScript)  
  - Role: Concatenates all aggregated content texts with double newlines and deduplicates all citation URLs into a single list  
  - Configuration: Maps items to join their `content` fields; flattens and filters citations starting with "http"; returns one object with `combinedContent` and `combinedCitations` keys  
  - Inputs: Merged items list  
  - Outputs: One JSON object with combined content and citations for AI generation input  
  - Edge cases: Empty input arrays, malformed citation URLs  

---

#### 2.3 AI Content Generation

**Overview:**  
Generates up to 10 AI-powered short-form video content packages from combined input, tailored with persona and business context, using OpenAI GPT-4.1-MINI.

**Nodes Involved:**  
- About me  
- Content Generation  

**Node Details:**

- **About me**  
  - Type: Set  
  - Role: Defines user/business profile variables for personalized AI content generation  
  - Configuration: Sets string variables: Name ("John Doe"), Niche ("a tech founder"), Business Name ("John Doe AI"), Business Type ("Marketing Agency")  
  - Inputs: Combined content from previous block  
  - Outputs: JSON with profile info to be merged with content text for prompt  
  - Edge cases: Missing or incorrect profile data will affect AI output relevance  

- **Content Generation**  
  - Type: OpenAI (LangChain integration)  
  - Role: Uses GPT-4.1-MINI to create video scripts, captions, and text overlays based on combined content and About Me profile  
  - Configuration:  
    - Model: GPT-4.1-MINI  
    - Messages: Single system + user prompt combining role description, user profile, content pillars, output format instructions, and injected combined content & citations from previous nodes  
  - Credentials: Requires an OpenAI API key credential (openAiApi)  
  - Inputs: JSON object with combined content and About Me variables  
  - Outputs: AI-generated text blocks following a strict format (Text Overlay, Video Script, Caption Text) for each content package  
  - Edge cases: API auth failure, rate limits, long prompt truncation, malformed output, unexpected formatting  

---

#### 2.4 Content Extraction & Formatting

**Overview:**  
Parses the raw AI-generated multi-package text into discrete structured fields for each content package, preparing for storage.

**Nodes Involved:**  
- Extract Data  

**Node Details:**

- **Extract Data**  
  - Type: Code (JavaScript)  
  - Role: Splits AI-generated text by lines, detects markers ("Text Overlay:", "Video Script:", "Caption Text:"), assembles multiple content packages into separate JSON items with three distinct fields: `text_overlay_output`, `video_script_output`, `caption_text_output`  
  - Inputs: AI raw output text from Content Generation  
  - Outputs: Array of objects, each representing one content package with clearly separated fields  
  - Edge cases: Missing markers, unexpected line breaks, triple quotes in scripts replaced with triple apostrophes, incomplete packages  

---

#### 2.5 Data Persistence & Notifications

**Overview:**  
Stores generated content into a Google Sheets document and sends an email notification confirming new content availability.

**Nodes Involved:**  
- Loop Over Items  
- Save Data  
- Notify user  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over each content package item for sequential processing  
  - Configuration: Default batch options (process items one by one)  
  - Inputs: Extracted structured content items from Extract Data  
  - Outputs: Single item per iteration sent to Save Data and Notify user nodes  

- **Save Data**  
  - Type: Google Sheets  
  - Role: Appends each content package as a new row into a specific Google Sheets tab  
  - Configuration:  
    - Operation: Append  
    - Document ID and Sheet ID predefined to a target Google Sheets document and specific sheet for content ideas  
    - Columns mapped to Caption, Text Overlay, Video Script fields from input JSON  
    - Approval and Published columns left blank for manual or downstream processing  
  - Credentials: Requires Google Sheets OAuth2 credentials  
  - Inputs: Single content package JSON from Loop Over Items  
  - Outputs: Confirmation of append operation  
  - Edge cases: Google API quota limits, auth errors, sheet not found, malformed input data  

- **Notify user**  
  - Type: Gmail  
  - Role: Sends email notification when content generation completes  
  - Configuration:  
    - Recipient hardcoded as example@domain.com (to be customized)  
    - Subject: "Content Generated"  
    - Body: "10 new articles are added in the google sheets."  
  - Credentials: Gmail OAuth2 credentials required  
  - Inputs: Triggered after Loop Over Items completes processing all batches  
  - Edge cases: Email send failure, auth errors  

---

#### 2.6 Auxiliary Setup & Notes

**Overview:**  
Contains static configuration and instructional notes for users setting up the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  

**Node Details:**

- **Sticky Note**  
  - Type: Sticky Note  
  - Role: Visual label titled "# Content Gen" indicating the content generation area in the workflow  

- **Sticky Note1**  
  - Type: Sticky Note  
  - Role: Detailed setup instructions including API key links, Google Sheet template, and support resources:  
    - Perplexity API key link: https://www.perplexity.ai/account/api/keys  
    - OpenAI API key link: https://platform.openai.com/  
    - Google Sheet template: https://docs.google.com/spreadsheets/d/1UcvTSCuKN_rXm6amblLyZ_Ogfk5tKuryYEBAlRoznpQ/edit?usp=sharing  
    - Setup guide PDF: https://drive.google.com/file/d/1XTDUf4iGE43d78duHkwS7fZuWieQh9un/view?usp=sharing  
    - Email support: info.gainflow@gmail.com  
    - Additional workflows doc: https://docs.google.com/document/d/1RACo90h-QwKA4hEZSlOQZsyw4iB5-43JM2l0s4lpuoY/edit?usp=sharing  

- **Sticky Note2**  
  - Type: Sticky Note  
  - Role: Empty or reserved space, no visible content  

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                                 | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                                                      |
|----------------------------|------------------------------|------------------------------------------------|------------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger             | Starts workflow daily at 6 AM                   | None                                     | Topic 1 (eg-AI News)                   | # Content Gen (covers content generation area)                                                                                  |
| Topic 1 (eg-AI News)        | HTTP Request                | Query AI news stories from Perplexity API      | Schedule Trigger                         | Organise Content                      |                                                                                                                                 |
| Organise Content            | Code                        | Extract message content & citations from Topic 1 output | Topic 1 (eg-AI News)                   | Topic 2 (eg - AI Market Trends), Merge |                                                                                                                                 |
| Topic 2 (eg - AI Market Trends) | HTTP Request             | Query AI market trends stories                  | Organise Content                        | Organise Content1                     |                                                                                                                                 |
| Organise Content1           | Code                        | Extract content from Topic 2                     | Topic 2 (eg - AI Market Trends)          | Topic 3 (eg- AI Business Automation), Merge |                                                                                                                                 |
| Topic 3 (eg- AI Business Automation) | HTTP Request          | Query AI business automation stories            | Organise Content1                       | Organise Content2                     |                                                                                                                                 |
| Organise Content2           | Code                        | Extract content from Topic 3                     | Topic 3 (eg- AI Business Automation)    | Merge                                |                                                                                                                                 |
| Merge                      | Merge                       | Combine all topical content streams              | Organise Content, Organise Content1, Organise Content2 | Combine items                        |                                                                                                                                 |
| Combine items              | Code                        | Aggregate combined content and deduplicate citations | Merge                                 | About me                            |                                                                                                                                 |
| About me                   | Set                         | Provide user/business profile variables          | Combine items                           | Content Generation                  |                                                                                                                                 |
| Content Generation         | OpenAI                      | Generate video scripts, captions, overlays using GPT | About me                              | Extract Data                       |                                                                                                                                 |
| Extract Data               | Code                        | Parse AI text output into structured content packages | Content Generation                     | Loop Over Items                   |                                                                                                                                 |
| Loop Over Items            | SplitInBatches              | Iterate over each content package for processing  | Extract Data                          | Save Data, Notify user             |                                                                                                                                 |
| Save Data                  | Google Sheets               | Append content packages to Google Sheets          | Loop Over Items                       | None                               |                                                                                                                                 |
| Notify user                | Gmail                       | Send notification email after content is saved    | Loop Over Items                       | None                               |                                                                                                                                 |
| Sticky Note                | Sticky Note                 | Workflow visual label "# Content Gen"              | None                                 | None                               | # Content Gen (covers main content generation block)                                                                            |
| Sticky Note1               | Sticky Note                 | Setup instructions, API key links, and help info  | None                                 | None                               | Setup guide and API key links with help resources: Perplexity, OpenAI, Google Sheets template, support email, workflow links    |
| Sticky Note2               | Sticky Note                 | Empty or reserved space                            | None                                 | None                               |                                                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set rule to trigger daily at 06:00 (hour 6)  

2. **Create three HTTP Request nodes for Perplexity API:**  
   - **Topic 1 (eg-AI News):**  
     - Method: POST  
     - URL: `https://api.perplexity.ai/chat/completions`  
     - Headers: `Authorization: Bearer <YOUR_API_KEY>`, `Content-Type: application/json`  
     - Body (JSON):  
       ```json
       {
         "model": "sonar-pro",
         "messages": [
           {"role": "system", "content": "Be precise and concise."},
           {"role": "user", "content": "Find me 5 new, interesting, and relevant stories and news related to artificial intelligence from today or yesterday. Label them 1 through 5"}
         ]
       }
       ```  
   - **Topic 2 (eg - AI Market Trends):** Same as Topic 1 but change user content to ask for 5 new AI market and industry trend stories.  
   - **Topic 3 (eg- AI Business Automation):** Same as above, user content asks for 5 new stories on AI business automation.

3. **Create three Code nodes to organize content:**  
   - For each Topic node output, create one Code node with JS to extract `choices[0].message.content` and `citations` arrays, mapping them to a simplified JSON structure.

4. **Create a Merge node:**  
   - Set number of inputs to 3  
   - Connect outputs of the three Organise Content nodes as inputs  
   - Combine all content streams into one

5. **Create a Code node (Combine items):**  
   - Combine all `.content` fields with double newlines between  
   - Deduplicate and flatten citation URLs starting with "http"  
   - Output one object with `combinedContent` and `combinedCitations` keys

6. **Create a Set node (About me):**  
   - Define user profile variables:  
     - Name: "John Doe"  
     - Niche: "a tech founder"  
     - Business Name: "John Doe AI"  
     - Business Type: "Marketing Agency"

7. **Create an OpenAI node (Content Generation):**  
   - Credentials: OpenAI API key  
   - Model: GPT-4.1-MINI  
   - Messages: Compose system and user messages using the About Me variables and combined content & citations from previous steps. Include detailed instructions for style, tone, and output format (Text Overlay, Video Script, Caption Text).  
   - Input: JSON with combined content and profile variables  

8. **Create a Code node (Extract Data):**  
   - Parse AI-generated text output by splitting lines  
   - Detect and extract "Text Overlay:", "Video Script:", and "Caption Text:" sections for multiple content packages  
   - Assemble and output structured JSON items with keys `text_overlay_output`, `video_script_output`, `caption_text_output`

9. **Create a SplitInBatches node (Loop Over Items):**  
   - Use default batch settings to process each content package one at a time

10. **Create a Google Sheets node (Save Data):**  
    - Operation: Append  
    - Document ID and Sheet ID: Set to target Google Sheet and sheet tab for content ideas  
    - Map columns: Caption → Caption Text, Text Overlay → Text Overlay, Video Script → Video Script  
    - Credentials: Google Sheets OAuth2

11. **Create a Gmail node (Notify user):**  
    - Credentials: Gmail OAuth2  
    - To: Your desired email address (replace example@domain.com)  
    - Subject: "Content Generated"  
    - Message: "10 new articles are added in the google sheets."  

12. **Connect nodes in order:**  
    - Schedule Trigger → Topic 1 → Organise Content → Topic 2 → Organise Content1 → Topic 3 → Organise Content2 → Merge → Combine items → About me → Content Generation → Extract Data → Loop Over Items → Save Data & Notify user (parallel outputs)

13. **Optionally, add Sticky Note nodes with setup instructions and titles as visual aids.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Get your API keys by clicking on the links below. Perplexity: https://www.perplexity.ai/account/api/keys; OpenAI: https://platform.openai.com/                                                                                                                                                                                                                                                                                                                                                   | Sticky Note1 in workflow                                                                                                   |
| Copy and use the Google Sheet template for storing content ideas: https://docs.google.com/spreadsheets/d/1UcvTSCuKN_rXm6amblLyZ_Ogfk5tKuryYEBAlRoznpQ/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                | Sticky Note1 in workflow                                                                                                   |
| Detailed setup guide PDF available at: https://drive.google.com/file/d/1XTDUf4iGE43d78duHkwS7fZuWieQh9un/view?usp=sharing                                                                                                                                                                                                                                                                                                                                                                       | Sticky Note1 in workflow                                                                                                   |
| For help, contact via email: info.gainflow@gmail.com                                                                                                                                                                                                                                                                                                                                                                                                                                         | Sticky Note1 in workflow                                                                                                   |
| Find more real-world use workflows here: https://docs.google.com/document/d/1RACo90h-QwKA4hEZSlOQZsyw4iB5-43JM2l0s4lpuoY/edit?usp=sharing                                                                                                                                                                                                                                                                                                                                                       | Sticky Note1 in workflow                                                                                                   |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automated workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.