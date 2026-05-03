Extract Marketing Insights & Generate Content from TikTok Videos with Dumpling AI & GPT-4

https://n8nworkflows.xyz/workflows/extract-marketing-insights---generate-content-from-tiktok-videos-with-dumpling-ai---gpt-4-6154


# Extract Marketing Insights & Generate Content from TikTok Videos with Dumpling AI & GPT-4

### 1. Workflow Overview

This n8n workflow automates the extraction of marketing insights and content generation from TikTok videos using Dumpling AI for transcription and GPT-4 via LangChain agents for natural language processing. It is designed for marketers, content creators, and product teams who want to leverage user-generated TikTok content to identify customer pain points, desired outcomes, triggers, and generate tailored social media posts promoting their product.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Capture user input of TikTok video URL, a keyword related to the product, and the product name via a web form.
- **1.2 Transcript Retrieval & Formatting:** Use Dumpling AI API to fetch the TikTok video transcript in VTT format and clean it into plain text.
- **1.3 AI-Driven Insight Extraction:** Analyze the cleaned transcript with GPT-4 to extract categorized marketing insights such as pain points and motivating events.
- **1.4 Insight Parsing:** Programmatically separate the AI-generated text into structured fields.
- **1.5 Marketing Content Generation:** Rewrite the transcript into a persuasive, conversational marketing post positioning the specified product as the solution.
- **1.6 Data Logging:** Append or update a Google Sheet with the extracted insights, original transcript, and generated marketing post for record-keeping and further use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block handles receiving the end-user input via a web form, which includes the TikTok video URL, a keyword related to the product or problem domain, and the product name itself.

- **Nodes Involved:**  
  - `Form: Submit TikTok URL + Product Info`

- **Node Details:**  

  - **Node Name:** Form: Submit TikTok URL + Product Info  
    - Type: `n8n-nodes-base.formTrigger` (Form Trigger Node)  
    - Role: Entry point capturing user input through a web form.  
    - Configuration:  
      - Form Title: "TikTok Scraper"  
      - Fields:  
        - TikTok Video URL (text)  
        - Keyword (text)  
        - Product (text)  
    - Input/Output:  
      - No input nodes (trigger)  
      - Outputs to `Dumpling AI: Get TikTok Transcript`  
    - Edge Cases:  
      - Missing or invalid URLs or empty keyword/product fields may cause downstream API requests or AI prompts to fail or behave unexpectedly. Validation should be considered.  
      - Form submission errors or webhook issues could prevent workflow trigger.  
    - Version: 2.2

#### 1.2 Transcript Retrieval & Formatting

- **Overview:**  
  Retrieves the TikTok video transcript from Dumpling AI’s API and cleans the VTT subtitle format into plain text suitable for NLP analysis.

- **Nodes Involved:**  
  - `Dumpling AI: Get TikTok Transcript`  
  - `Format: Clean VTT Captions`

- **Node Details:**  

  - **Node Name:** Dumpling AI: Get TikTok Transcript  
    - Type: `n8n-nodes-base.httpRequest`  
    - Role: Calls Dumpling AI API to get the VTT transcript of the submitted TikTok video URL.  
    - Configuration:  
      - Method: POST  
      - URL: `https://app.dumplingai.com/api/v1/get-tiktok-transcript`  
      - Body Parameter: `videoUrl` set dynamically from form field `Tiktok Video URL`  
      - Authentication: HTTP Header Auth using stored Dumpling AI credential  
      - Response Format: Text, output saved as `body`  
    - Input: from Form node  
    - Output: raw VTT transcript text  
    - Edge Cases:  
      - API authentication failure or rate limits  
      - Invalid or inaccessible TikTok URLs leading to empty or error responses  
      - Network timeouts  
    - Version: 4.2

  - **Node Name:** Format: Clean VTT Captions  
    - Type: `n8n-nodes-base.code` (Code Node)  
    - Role: Cleans VTT subtitle text to generate a consolidated transcript string for AI processing.  
    - Configuration:  
      - Runs once per item  
      - JavaScript code that filters out VTT metadata lines (timestamps, header info) and joins the remaining lines into a single transcript string under `transcript` property  
      - Returns `transcript` or a note if no captions found  
    - Input: raw VTT from Dumpling AI node  
    - Output: cleaned transcript as plain text in `.transcript`  
    - Edge Cases:  
      - Empty or malformed VTT input leads to `transcript: null` and a note  
      - Unexpected VTT formatting variations may cause incomplete filtering  
    - Version: 2

#### 1.3 AI-Driven Insight Extraction

- **Overview:**  
  Uses GPT-4 through a LangChain Agent node to analyze the cleaned transcript and extract marketing insights centered on the input keyword, including pain points, desired outcomes, triggers, and direct quotes.

- **Nodes Involved:**  
  - `GPT-4: Extract Pain Points & Insights`  
  - `🧠 LangChain Tools (for Agents)`

- **Node Details:**  

  - **Node Name:** GPT-4: Extract Pain Points & Insights  
    - Type: `@n8n/n8n-nodes-langchain.agent` (LangChain Agent)  
    - Role: Sends a prompt with the transcript and keyword to GPT-4, instructing it to:  
      1. Classify speaker’s focus (problem, solution, or both)  
      2. Extract pain points, desired outcomes, triggers, and quotes  
    - Configuration:  
      - Prompt uses dynamic expressions to insert keyword and transcript  
      - System message frames the AI as a customer research analyst  
      - Prompt Type: Define (custom prompt)  
    - Input: cleaned transcript from code node  
    - Output: text output with categorized insights  
    - Edge Cases:  
      - AI generation variability or hallucination  
      - Transcript quality affecting accuracy  
      - API rate limits or authentication failures  
    - Version: 1.7

  - **Node Name:** 🧠 LangChain Tools (for Agents)  
    - Type: `@n8n/n8n-nodes-langchain.toolThink`  
    - Role: Provides additional LangChain toolset support for the agent node  
    - Input/Output: connected to the GPT-4 agent node  
    - Version: 1

#### 1.4 Insight Parsing

- **Overview:**  
  Parses the AI-generated text output to extract and separate the individual insight categories into structured JSON fields for easier downstream use and logging.

- **Nodes Involved:**  
  - `Parse: Separate Insights (pain points, outcomes, etc)`

- **Node Details:**  

  - **Node Name:** Parse: Separate Insights (pain points, outcomes, etc)  
    - Type: `n8n-nodes-base.aiTransform` (Code/Transform Node)  
    - Role: Uses JavaScript to locate labeled sections in the AI output text and slice them into four distinct fields: painPoints, desiredOutcomes, triggers, and quotes.  
    - Configuration:  
      - Inline JS code reads `output` string and slices by indices of section headers  
      - Returns an object with the four fields per item  
    - Input: AI output text from GPT-4 insight node  
    - Output: JSON object with separate insight fields  
    - Edge Cases:  
      - If AI output formatting changes, parsing may fail or produce incomplete fields  
      - Missing sections cause empty strings  
    - Version: 1

#### 1.5 Marketing Content Generation

- **Overview:**  
  Uses GPT-4 to rewrite the original transcript as a persuasive marketing post that positions the provided product as the solution to the pain points, preserving the speaker’s tone and style.

- **Nodes Involved:**  
  - `GPT-4: Rewrite Transcript as Marketing Post`  
  - `GPT-4: Used in Rewrite Agent`  
  - `🧠 LangChain Tools (for Agents)1`

- **Node Details:**  

  - **Node Name:** GPT-4: Rewrite Transcript as Marketing Post  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Prompts GPT-4 to analyze the transcript and rewrite it as a conversational, authentic social media post framing the specified product as the solution.  
    - Configuration:  
      - Dynamic prompt inserts Keyword, Product, and cleaned transcript  
      - System message sets AI as a persuasive copywriter preserving tone and voice  
      - Output: marketing post text  
    - Input: cleaned transcript from caption formatting node  
    - Output: rewritten post text  
    - Edge Cases:  
      - AI content generation variability  
      - Missing or unclear transcript input could reduce quality  
      - API limits or errors  
    - Version: 1.7

  - **Node Name:** GPT-4: Used in Rewrite Agent  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Role: Provides the GPT-4 language model for the rewrite agent node  
    - Configuration: Model set to `gpt-4o-mini`  
    - Credentials: OpenAI API key  
    - Connected as `ai_languageModel` to the rewrite agent  
    - Version: 1.2

  - **Node Name:** 🧠 LangChain Tools (for Agents)1  
    - Type: `@n8n/n8n-nodes-langchain.toolThink`  
    - Role: Supporting LangChain tools for the rewrite agent  
    - Connected as `ai_tool` to the rewrite agent  
    - Version: 1

#### 1.6 Data Logging

- **Overview:**  
  Appends or updates a Google Sheets document with the extracted insights, original transcript, TikTok URL, and the rewritten marketing post for record-keeping and further analysis.

- **Nodes Involved:**  
  - `Google Sheets: Log Insights + Post Copy`

- **Node Details:**  

  - **Node Name:** Google Sheets: Log Insights + Post Copy  
    - Type: `n8n-nodes-base.googleSheets`  
    - Role: Logs all relevant extracted data and generated content into a Google Sheet row.  
    - Configuration:  
      - Operation: Append or update by matching on "Video URL" column  
      - Sheet: Sheet1 (gid=0) in a specified Google Sheets document  
      - Columns mapped:  
        - Video URL from form input  
        - New Script (rewritten post) from GPT-4 rewrite node output  
        - Pain points, Desired outcomes, Triggers, Interesting direct quotes from parsed insight node  
        - Original transcription from cleaned caption node  
      - Credentials: Google Sheets OAuth2 API setup  
    - Input: multiple dynamic data sources merged from prior nodes  
    - Output: confirmation of sheet update  
    - Edge Cases:  
      - Google API authentication issues or quota limits  
      - Mismatched column headers or missing sheet permissions  
      - Errors if data fields are unexpectedly null or malformed  
    - Version: 4.5

---

### 3. Summary Table

| Node Name                              | Node Type                        | Functional Role                                | Input Node(s)                              | Output Node(s)                                   | Sticky Note                                                                                                  |
|--------------------------------------|---------------------------------|------------------------------------------------|--------------------------------------------|-------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Form: Submit TikTok URL + Product Info | n8n-nodes-base.formTrigger       | Capture TikTok URL, Keyword, Product inputs     | -                                          | Dumpling AI: Get TikTok Transcript               |                                                                                                              |
| Dumpling AI: Get TikTok Transcript   | n8n-nodes-base.httpRequest       | Retrieve TikTok transcript from Dumpling AI API | Form: Submit TikTok URL + Product Info     | Format: Clean VTT Captions                        |                                                                                                              |
| Format: Clean VTT Captions           | n8n-nodes-base.code              | Clean VTT subtitle text to plain transcript     | Dumpling AI: Get TikTok Transcript          | GPT-4: Extract Pain Points & Insights, GPT-4: Rewrite Transcript as Marketing Post |                                                                                                              |
| GPT-4: Extract Pain Points & Insights| @n8n/n8n-nodes-langchain.agent  | Analyze transcript for pain points, outcomes, etc | Format: Clean VTT Captions                   | Parse: Separate Insights (pain points, outcomes, etc) |                                                                                                              |
| Parse: Separate Insights (pain points, outcomes, etc) | n8n-nodes-base.aiTransform       | Parse AI output into structured insight fields  | GPT-4: Extract Pain Points & Insights       | Google Sheets: Log Insights + Post Copy            |                                                                                                              |
| GPT-4: Rewrite Transcript as Marketing Post | @n8n/n8n-nodes-langchain.agent  | Rewrite transcript as persuasive marketing post | Format: Clean VTT Captions, GPT-4: Used in Rewrite Agent | Google Sheets: Log Insights + Post Copy            |                                                                                                              |
| GPT-4: Used in Research Agent        | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4 model for insight extraction      | -                                          | GPT-4: Extract Pain Points & Insights              |                                                                                                              |
| GPT-4: Used in Rewrite Agent         | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provide GPT-4 model for rewrite agent            | -                                          | GPT-4: Rewrite Transcript as Marketing Post        |                                                                                                              |
| 🧠 LangChain Tools (for Agents)      | @n8n/n8n-nodes-langchain.toolThink | LangChain tool support for insight agent         | -                                          | GPT-4: Extract Pain Points & Insights              |                                                                                                              |
| 🧠 LangChain Tools (for Agents)1     | @n8n/n8n-nodes-langchain.toolThink | LangChain tool support for rewrite agent         | -                                          | GPT-4: Rewrite Transcript as Marketing Post        |                                                                                                              |
| Google Sheets: Log Insights + Post Copy | n8n-nodes-base.googleSheets     | Log extracted insights and marketing post to sheet | Parse: Separate Insights (pain points, outcomes, etc), GPT-4: Rewrite Transcript as Marketing Post | -                                                 |                                                                                                              |
| Sticky Note                         | n8n-nodes-base.stickyNote         | Informational summary of workflow purpose        | -                                          | -                                                 | 🎯 Workflow Summary: This workflow takes a TikTok link and product keyword, extracts the transcript, and uses GPT-4 to generate: key pain points, desired outcomes, motivating events, direct quotes, and a rewritten post for marketing. All results are saved to Google Sheets. Tools: Dumpling AI, GPT-4, LangChain, Google Sheets |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: `Form Trigger`  
   - Title: "TikTok Scraper"  
   - Add fields:  
     - "Tiktok Video URL" (text)  
     - "Keyword" (text)  
     - "Product" (text)  
   - This node acts as the webhook entry point for user input.

2. **Add HTTP Request Node to Get Transcript**  
   - Name: "Dumpling AI: Get TikTok Transcript"  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-tiktok-transcript`  
   - Authentication: HTTP Header Auth using Dumpling AI API credentials (create and configure credential with your API key)  
   - Body Parameters: JSON with key `videoUrl` set to expression `{{$json["Tiktok Video URL"]}}` from form node  
   - Response format: Text, output in property `body`  
   - Connect form trigger’s output to this node.

3. **Add Code Node to Clean VTT Captions**  
   - Name: "Format: Clean VTT Captions"  
   - Type: Code (JavaScript)  
   - Run Mode: Run once for each item  
   - Paste the following code:  
     ```js
     const vtt = $json.body;
     if (!vtt) {
       return { transcript: null, note: "no TikTok captions" };
     }
     const lines = vtt
       .split(/\r?\n/)
       .filter(l => l && !l.match(/^(\d+|WEBVTT|X-TIMESTAMP|[:\d+.\-> ]+$)/));
     return { transcript: lines.join(" ") };
     ```
   - Connect HTTP Request node output to this node.

4. **Set up GPT-4 Agent Node for Insight Extraction**  
   - Name: "GPT-4: Extract Pain Points & Insights"  
   - Type: LangChain Agent  
   - System Message: "You are a customer research analyst. Your job is to extract actionable marketing insights from user-generated content like TikTok transcripts."  
   - Prompt:  
     ```
     Here is a TikTok transcript. Analyze it for the following:

     1. Determine whether the speaker is describing a **problem**, a **solution**, or **both** regarding the keyword: "{{ $('Form: Submit TikTok URL + Product Info').item.json['Keyword '] }}".
     2. Extract:
        - Pain points
        - Desired outcomes
        - Triggers or motivating events
        - Interesting direct quotes or phrases

     Transcript: {{ $json.transcript }}
     ```
   - Connect cleaned transcript node output to this node.

5. **Add LangChain Tools Node for Insight Agent Support**  
   - Name: "🧠 LangChain Tools (for Agents)"  
   - Type: LangChain Tool Think  
   - Connect as AI tool input to the GPT-4 Insight Agent node.

6. **Add AI Transform Node to Parse Insight Output**  
   - Name: "Parse: Separate Insights (pain points, outcomes, etc)"  
   - Type: AI Transform  
   - JavaScript code:  
     ```js
     const items = $input.all();
     const outputItems = items.map((item) => {
       const output = item?.json?.output;

       const painPointsIndex = output.indexOf("Pain Points") + "Pain Points".length;
       const desiredOutcomesIndex = output.indexOf("Desired Outcomes");
       const triggersIndex = output.indexOf("Triggers or Motivating Events");
       const quotesIndex = output.indexOf("Interesting Direct Quotes or Phrases");

       const painPoints = output.slice(painPointsIndex, desiredOutcomesIndex).trim();
       const desiredOutcomes = output
         .slice(desiredOutcomesIndex, triggersIndex)
         .trim();
       const triggers = output.slice(triggersIndex, quotesIndex).trim();
       const quotes = output.slice(quotesIndex).trim();

       return {
         painPoints,
         desiredOutcomes,
         triggers,
         quotes,
       };
     });

     return outputItems;
     ```
   - Connect GPT-4 Insight Agent output to this node.

7. **Set up GPT-4 Agent Node for Marketing Post Rewrite**  
   - Name: "GPT-4: Rewrite Transcript as Marketing Post"  
   - Type: LangChain Agent  
   - System Message: "You are a persuasive copywriter skilled at rewriting user-generated content to position a product or service as the solution to customer problems or goals, while preserving the original voice and tone."  
   - Prompt:  
     ```
     Analyze the following TikTok transcript and determine whether the speaker is primarily describing:

     - a **problem** related to the keyword "{{ $('Form: Submit TikTok URL + Product Info').item.json['Keyword '] }}".
     - a **solution** they've found, or
     - **both** a problem and a solution.

     Then, rewrite the transcript with the following goals:

     1. Frame our product or service "{{ $('Form: Submit TikTok URL + Product Info').item.json['Product '] }}" as the ideal solution to the problem, or a key part of the solution being described.
     2. Preserve the speaker’s tone, voice, and informal/social style.
     3. Make it sound like a natural social media post or spoken monologue.
     4. Ensure it's conversational and authentic.

     Transcript: {{ $json.transcript }}
     ```
   - Connect cleaned transcript node output to this node.

8. **Add GPT-4 Model Node for Insight Agent**  
   - Name: "GPT-4: Used in Research Agent"  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o-mini`  
   - Credentials: Set up OpenAI API credentials  
   - Connect as AI language model to "GPT-4: Extract Pain Points & Insights" agent node.

9. **Add GPT-4 Model Node for Rewrite Agent**  
   - Name: "GPT-4: Used in Rewrite Agent"  
   - Type: LangChain LM Chat OpenAI  
   - Model: `gpt-4o-mini`  
   - Credentials: OpenAI API credentials  
   - Connect as AI language model to "GPT-4: Rewrite Transcript as Marketing Post" agent node.

10. **Add LangChain Tools Node for Rewrite Agent**  
    - Name: "🧠 LangChain Tools (for Agents)1"  
    - Type: LangChain Tool Think  
    - Connect as AI tool input to the rewrite agent node.

11. **Add Google Sheets Node to Log Results**  
    - Name: "Google Sheets: Log Insights + Post Copy"  
    - Type: Google Sheets  
    - Operation: Append or update row based on matching "Video URL" column  
    - Document ID: Your target Google Sheets document ID  
    - Sheet Name: "Sheet1" (gid=0)  
    - Columns Mapping:  
      - Video URL: from form input  
      - New Script: from marketing post rewrite node output  
      - Pain points, Desired outcomes, Triggers, Interesting direct quotes: from parsed insight node output  
      - Original Transcription: from cleaned transcript node output  
    - Credentials: Configure Google Sheets OAuth2 credentials  
    - Connect outputs of parsed insight node and marketing post rewrite node as inputs.

12. **Connect Nodes in Execution Order:**  
    Form → Dumpling AI → Format Captions →  
    → GPT-4 Insight Agent (with LangChain Tools) → Parse Insights → Google Sheets  
    Format Captions → GPT-4 Rewrite Agent (with GPT-4 model and LangChain Tools) → Google Sheets

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                              | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow leverages Dumpling AI's TikTok transcript extraction API, which requires an API key configured as HTTP Header Auth in n8n credentials.                     | Dumpling AI API: https://app.dumplingai.com/api/v1/get-tiktok-transcript                            |
| GPT-4 is accessed via LangChain agent nodes in n8n, enabling complex prompt engineering and multi-tool orchestration.                                                  | LangChain integration docs: https://docs.n8n.io/integrations/ai/langchain/                          |
| Google Sheets node requires OAuth2 credentials with edit access to the target spreadsheet for appending and updating rows.                                              | Google Sheets API setup: https://developers.google.com/sheets/api/guides/authorizing                 |
| The workflow assumes TikTok video URLs are publicly accessible and that Dumpling AI successfully returns VTT captions; videos without captions will yield no transcript.   | Edge cases: Missing captions or private videos can cause empty transcript outputs                    |
| The prompts use dynamic expressions to inject user inputs, so form field names must match exactly, including spaces ("Keyword ").                                       | Careful matching of form field labels and expressions is essential for proper variable substitution |
| Sticky Note Summary: This workflow takes a TikTok link and product keyword, extracts the transcript, and uses GPT-4 to generate key pain points, desired outcomes, motivating events, direct quotes, and a rewritten post for marketing. All results are saved to Google Sheets. Tools: Dumpling AI, GPT-4, LangChain, Google Sheets. |                                                                                            |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.