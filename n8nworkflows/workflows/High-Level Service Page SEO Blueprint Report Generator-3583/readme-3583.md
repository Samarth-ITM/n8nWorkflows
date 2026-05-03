High-Level Service Page SEO Blueprint Report Generator

https://n8nworkflows.xyz/workflows/high-level-service-page-seo-blueprint-report-generator-3583


# High-Level Service Page SEO Blueprint Report Generator

### 1. Workflow Overview

The **High-Level Service Page SEO Blueprint Report Generator** is an advanced, AI-driven n8n workflow designed to automate the creation of comprehensive SEO content blueprints for service-based business pages. It targets digital marketers, SEO specialists, content strategists, and web developers who need to efficiently analyze competitors and user intent to produce data-driven, conversion-optimized service page strategies.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Collects user inputs via a form, including competitor URLs, target keyword, services offered, brand name, and homepage status.
- **1.2 Competitor Content Extraction & Processing:** Converts competitor URLs into individual items, fetches their HTML content using the JINA Reader API, and extracts structured data such as headings, meta tags, schema markup, and n-grams.
- **1.3 Competitor Analysis:** Uses Google Gemini AI to analyze competitor data, identifying patterns in meta titles, outlines, headings, and structural elements.
- **1.4 User Intent Analysis:** Analyzes the target keyword to determine user intent, personas, and buyer journey stage.
- **1.5 Synthesis & Gap Analysis:** Synthesizes competitor and user intent analyses to identify content overlaps, gaps, SEO priorities, and UX/conversion opportunities.
- **1.6 Page Outline Generation:** Generates an optimal, conversion-focused page outline (H1-H4) based on prior analyses and user inputs.
- **1.7 UX & Conversion Recommendations:** Enhances the outline with detailed recommendations for CTAs, trust signals, copywriting tone, visuals, and risk reversal.
- **1.8 Final Blueprint Compilation:** Compiles all analyses and recommendations into a single, well-structured Markdown document for download.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects essential inputs from the user via a web form to drive the entire workflow.
- **Nodes Involved:**  
  - `Start` (Form Trigger)  
  - `Edit Fields` (Set API keys and parameters)  
  - `Convert URLs to Items` (Parse competitor URLs into separate items)

- **Node Details:**

  - **Start**  
    - Type: Form Trigger  
    - Role: Entry point; collects user inputs: Competitor URLs (textarea), Target Keyword, Services Offered (textarea), Brand Name, and Homepage flag (dropdown).  
    - Key Config: Limits competitors to 5; form description guides user input.  
    - Outputs: JSON with all form fields.

  - **Edit Fields**  
    - Type: Set  
    - Role: Stores credentials and parameters such as JINA Reader API Key, Google Gemini model name, and waiting time between AI calls.  
    - Key Config: Default waiting time is 1 second; user advised to increase to 20 seconds for free Google Gemini tier.  
    - Inputs: From `Start` node.  
    - Outputs: JSON with API keys and parameters.

  - **Convert URLs to Items**  
    - Type: Code  
    - Role: Parses the multiline competitor URLs string into an array of individual competitor URL items for batch processing.  
    - Key Expression: Splits input string by line breaks, trims, filters empty lines, returns array of JSON objects with `competitor_url`.  
    - Inputs: From `Start` node.  
    - Outputs: Multiple items, each with a single competitor URL.

- **Potential Failures:**  
  - User input errors (empty or malformed URLs).  
  - Missing or invalid API keys if not set in `Edit Fields`.

---

#### 2.2 Competitor Content Extraction & Processing

- **Overview:** Iterates over competitor URLs, fetches HTML content via JINA Reader API, and extracts structured SEO-relevant data including headings, meta tags, schema, and n-grams.

- **Nodes Involved:**  
  - `Loop Over Items` (Split items for sequential processing)  
  - `Get URL HTML` (HTTP Request to JINA Reader API)  
  - `Extract HTML Elements` (Code node parsing HTML content)  
  - `Set URL Data` (Set node formatting extracted data)  
  - `Code` (Aggregate competitor data into XML-like string)  
  - `Edit Fields1` (Pass aggregated data forward)

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes competitor URLs one by one to avoid API rate limits.  
    - Inputs: From `Convert URLs to Items`.  
    - Outputs: Single item per batch.

  - **Get URL HTML**  
    - Type: HTTP Request  
    - Role: Calls JINA Reader API to retrieve HTML content of competitor URL.  
    - Config: Uses Bearer token from `Edit Fields` for authorization; requests HTML format.  
    - Error Handling: Continues on error, retries up to 5 times with 5s delay.  
    - Inputs: From `Loop Over Items`.  
    - Outputs: Raw HTML content in `data` field.

  - **Extract HTML Elements**  
    - Type: Code  
    - Role: Parses HTML to extract:  
      - Heading tags (H1-H6) with position and text, preserving order  
      - Meta tags (title, description, canonical, Open Graph, Twitter cards, etc.)  
      - JSON-LD schema markup  
      - N-grams (2, 3, 4-word phrases) from headings, filtered to those with >1 occurrence  
    - Key Logic: Uses regex for extraction, cleans HTML entities, counts n-grams.  
    - Inputs: From `Get URL HTML`.  
    - Outputs: JSON with `outline`, `meta`, `schema`, and `ngrams`.

  - **Set URL Data**  
    - Type: Set  
    - Role: Converts extracted arrays/objects into JSON strings for downstream processing.  
    - Inputs: From `Extract HTML Elements`.  
    - Outputs: JSON with stringified `Outline`, `Meta`, `Ngrams`, and original `Competitor URL`.

  - **Code**  
    - Type: Code  
    - Role: Aggregates all competitor data items into a single XML-like formatted string for AI input.  
    - Logic: Iterates over items, formats competitor URL, outline, meta, and ngrams with indentation and XML tags.  
    - Inputs: From `Loop Over Items` (after `Set URL Data`).  
    - Outputs: Single JSON with `competitors_data` string.

  - **Edit Fields1**  
    - Type: Set  
    - Role: Passes the aggregated competitor data string forward.  
    - Inputs: From `Code`.  
    - Outputs: JSON with `competitors_data`.

- **Potential Failures:**  
  - HTTP request failures or timeouts to JINA API.  
  - HTML parsing errors if competitor pages have unusual markup.  
  - Large competitor pages may cause processing delays.  
  - API rate limits if batch size or timing is not managed.

---

#### 2.3 Competitor Analysis

- **Overview:** Uses Google Gemini AI to analyze competitor data and extract insights on meta trends, outline sections, key headings, and structural elements.

- **Nodes Involved:**  
  - `Competitors Analysis` (Langchain Chain LLM node)  
  - `Set Competitor Analysis` (Set node extracting AI output)  
  - `Wait` (Delay node to respect API rate limits)

- **Node Details:**

  - **Competitors Analysis**  
    - Type: Langchain Chain LLM (Google Gemini Chat)  
    - Role: Analyzes competitor data string and target keyword to generate a detailed competitor analysis report.  
    - Prompt: Instructs AI to identify competitor codes, meta title/description trends, common outline sections, key heading n-grams, and structural observations.  
    - Temperature: 0.4 (balanced creativity/precision).  
    - Inputs: `target_query` from form, `competitors_data` from previous block.  
    - Outputs: XML-tagged competitor analysis report.

  - **Set Competitor Analysis**  
    - Type: Set  
    - Role: Extracts the competitor analysis report content from AI output by removing XML tags.  
    - Inputs: From `Competitors Analysis`.  
    - Outputs: Cleaned competitor analysis string.

  - **Wait**  
    - Type: Wait  
    - Role: Delays next step by configured seconds to avoid API rate limits.  
    - Inputs: From `Set Competitor Analysis`.  
    - Outputs: Passes data forward after delay.

- **Potential Failures:**  
  - AI API errors or timeouts.  
  - Incorrect prompt formatting causing incomplete or malformed output.  
  - Rate limiting if wait time is insufficient.

---

#### 2.4 User Intent Analysis

- **Overview:** Uses Google Gemini AI to analyze the target keyword independently to determine user intent, personas, buyer journey stage, and expected content.

- **Nodes Involved:**  
  - `User Intent Analysis` (Langchain Chain LLM node)  
  - `Set User Intent Analysis` (Set node extracting AI output)  
  - `Wait1` (Delay node)

- **Node Details:**

  - **User Intent Analysis**  
    - Type: Langchain Chain LLM (Google Gemini Chat)  
    - Role: Analyzes the target keyword to generate a user intent report covering primary/secondary intents, user persona, buyer journey stage, expected services, and problem/solution framing.  
    - Temperature: 0.4.  
    - Inputs: `target_query` from form.  
    - Outputs: XML-tagged user intent report.

  - **Set User Intent Analysis**  
    - Type: Set  
    - Role: Extracts user intent report content from AI output by removing XML tags.  
    - Inputs: From `User Intent Analysis`.  
    - Outputs: Cleaned user intent analysis string.

  - **Wait1**  
    - Type: Wait  
    - Role: Delays next step by configured seconds.  
    - Inputs: From `Set User Intent Analysis`.  
    - Outputs: Passes data forward after delay.

- **Potential Failures:**  
  - AI API errors or malformed responses.  
  - Insufficient wait time causing rate limit errors.

---

#### 2.5 Synthesis & Gap Analysis

- **Overview:** Combines competitor analysis and user intent reports to identify content overlaps, gaps, SEO priorities, and UX/conversion opportunities.

- **Nodes Involved:**  
  - `Synthesis & Gap Analysis` (Langchain Chain LLM node)  
  - `Set Synthesis & Gap Analysis` (Set node extracting AI output)  
  - `Wait2` (Delay node)

- **Node Details:**

  - **Synthesis & Gap Analysis**  
    - Type: Langchain Chain LLM (Google Gemini Chat)  
    - Role: Synthesizes competitor and user intent reports to produce a strategic analysis highlighting table stakes, gaps, SEO priorities, and UX/conversion advantages.  
    - Temperature: 0.4.  
    - Inputs: `target_query`, competitor analysis report, user intent report.  
    - Outputs: XML-tagged synthesis and gap analysis report.

  - **Set Synthesis & Gap Analysis**  
    - Type: Set  
    - Role: Extracts synthesis report content from AI output by removing XML tags.  
    - Inputs: From `Synthesis & Gap Analysis`.  
    - Outputs: Cleaned synthesis and gap analysis string.

  - **Wait2**  
    - Type: Wait  
    - Role: Delays next step by configured seconds.  
    - Inputs: From `Set Synthesis & Gap Analysis`.  
    - Outputs: Passes data forward after delay.

- **Potential Failures:**  
  - AI API errors or incomplete output.  
  - Rate limiting if wait time is insufficient.

---

#### 2.6 Page Outline Generation

- **Overview:** Generates a detailed, conversion-focused page outline (H1-H4) based on the synthesis report, user inputs, and homepage status.

- **Nodes Involved:**  
  - `Ideal Page Outline Generation` (Langchain Chain LLM node)  
  - `Set Page Outline` (Set node extracting AI output)  
  - `Wait3` (Delay node)

- **Node Details:**

  - **Ideal Page Outline Generation**  
    - Type: Langchain Chain LLM (Google Gemini Chat)  
    - Role: Creates an SEO-optimized, persuasive page outline incorporating table stakes, gaps, and conversion elements.  
    - Inputs: `target_query`, synthesis and gap analysis, homepage flag, brand name, services offered.  
    - Temperature: 0.4.  
    - Outputs: XML-tagged recommended page outline.

  - **Set Page Outline**  
    - Type: Set  
    - Role: Extracts page outline content from AI output by removing XML tags.  
    - Inputs: From `Ideal Page Outline Generation`.  
    - Outputs: Cleaned page outline string.

  - **Wait3**  
    - Type: Wait  
    - Role: Delays next step by configured seconds.  
    - Inputs: From `Set Page Outline`.  
    - Outputs: Passes data forward after delay.

- **Potential Failures:**  
  - AI output formatting issues.  
  - Rate limiting.

---

#### 2.7 UX & Conversion Recommendations

- **Overview:** Enhances the page outline with detailed, actionable recommendations for CTAs, trust signals, copywriting tone, visuals, risk reversal, and readability.

- **Nodes Involved:**  
  - `UX, Conversion & Copywriting Enhancement` (Langchain Chain LLM node)  
  - `Set UX & Conversions Enhancements` (Set node extracting AI output)

- **Node Details:**

  - **UX, Conversion & Copywriting Enhancement**  
    - Type: Langchain Chain LLM (Google Gemini Chat)  
    - Role: Provides detailed CRO and UX copywriting recommendations tailored to the target query, brand, and services.  
    - Inputs: `target_query`, recommended page outline, user intent report, brand name, services offered.  
    - Temperature: 0.4.  
    - Outputs: XML-tagged UX and conversion recommendations.

  - **Set UX & Conversions Enhancements**  
    - Type: Set  
    - Role: Extracts UX and conversion recommendations from AI output by removing XML tags.  
    - Inputs: From `UX, Conversion & Copywriting Enhancement`.  
    - Outputs: Cleaned UX & conversion recommendations string.

- **Potential Failures:**  
  - AI API errors or incomplete output.

---

#### 2.8 Final Blueprint Compilation & Output

- **Overview:** Compiles all prior analyses and recommendations into a single, comprehensive Markdown blueprint document, then converts it to a downloadable text file.

- **Nodes Involved:**  
  - `Google Gemini Chat Model5` (Langchain LLM node)  
  - `Final Service Page Blueprint` (Langchain Chain LLM node)  
  - `Edit Fields2` (Set node extracting final blueprint)  
  - `Convert to File` (File conversion node)

- **Node Details:**

  - **Google Gemini Chat Model5**  
    - Type: Langchain LLM (Google Gemini Chat)  
    - Role: (Intermediate AI call, possibly for final formatting or processing)  
    - Inputs: From `Set UX & Conversions Enhancements`.  
    - Outputs: Passes data forward.

  - **Final Service Page Blueprint**  
    - Type: Langchain Chain LLM (Google Gemini Chat)  
    - Role: Consolidates all inputs (target query, brand, services, homepage flag, competitor analysis, user intent, synthesis, outline, UX recommendations) into a single Markdown document enclosed in `<final_service_page_blueprint>` XML tag.  
    - Output Structure: Markdown with sections for Executive Summary, User Intent, Competitor Summary, Strategic Opportunities, Page Outline, UX & Conversion Plan, and Key Success Factors.  
    - Inputs: All prior analysis and recommendations.  
    - Temperature: 0.4.

  - **Edit Fields2**  
    - Type: Set  
    - Role: Extracts the Markdown content from the XML wrapper for file conversion.  
    - Inputs: From `Final Service Page Blueprint`.  
    - Outputs: Clean Markdown string.

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts the Markdown string into a downloadable `.txt` file named "Blueprint.txt".  
    - Inputs: From `Edit Fields2`.  
    - Outputs: File object for download.

- **Potential Failures:**  
  - AI output formatting errors.  
  - File conversion errors.

---

### 3. Summary Table

| Node Name                         | Node Type                              | Functional Role                                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                                         |
|----------------------------------|--------------------------------------|-------------------------------------------------|-------------------------------|-------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note                      | Sticky Note                          | Provides workflow overview and setup instructions |                               |                               | ## Generate High-Level Service Page Blueprint Report... Setup instructions with API key links and demo link.                      |
| Start                            | Form Trigger                        | Collects user inputs via form                    |                               | Edit Fields                   |                                                                                                                                    |
| Edit Fields                     | Set                                 | Stores API keys and parameters                   | Start                         | Convert URLs to Items          |                                                                                                                                    |
| Convert URLs to Items            | Code                                | Parses competitor URLs into individual items     | Edit Fields                   | Loop Over Items               |                                                                                                                                    |
| Loop Over Items                 | SplitInBatches                      | Processes competitor URLs sequentially           | Convert URLs to Items          | Get URL HTML, Code            |                                                                                                                                    |
| Get URL HTML                   | HTTP Request                       | Fetches competitor HTML content via JINA API    | Loop Over Items               | Extract HTML Elements         |                                                                                                                                    |
| Extract HTML Elements           | Code                                | Extracts headings, meta tags, schema, n-grams   | Get URL HTML                  | Set URL Data                  |                                                                                                                                    |
| Set URL Data                   | Set                                 | Formats extracted data as JSON strings           | Extract HTML Elements         | Loop Over Items               |                                                                                                                                    |
| Code                           | Code                                | Aggregates competitor data into XML-like string | Loop Over Items               | Edit Fields1                 |                                                                                                                                    |
| Edit Fields1                  | Set                                 | Passes aggregated competitor data forward        | Code                         | Competitors Analysis          |                                                                                                                                    |
| Competitors Analysis           | Langchain Chain LLM (Google Gemini) | Analyzes competitor data for SEO insights        | Edit Fields1                 | Set Competitor Analysis       |                                                                                                                                    |
| Set Competitor Analysis         | Set                                 | Extracts competitor analysis report               | Competitors Analysis          | Wait                         |                                                                                                                                    |
| Wait                          | Wait                                | Delays to respect API rate limits                 | Set Competitor Analysis       | User Intent Analysis          |                                                                                                                                    |
| User Intent Analysis           | Langchain Chain LLM (Google Gemini) | Analyzes target keyword for user intent           | Wait                         | Set User Intent Analysis      |                                                                                                                                    |
| Set User Intent Analysis        | Set                                 | Extracts user intent report                        | User Intent Analysis          | Wait1                        |                                                                                                                                    |
| Wait1                         | Wait                                | Delays to respect API rate limits                 | Set User Intent Analysis      | Synthesis & Gap Analysis      |                                                                                                                                    |
| Synthesis & Gap Analysis       | Langchain Chain LLM (Google Gemini) | Synthesizes competitor and user intent analyses   | Wait1                        | Set Synthesis & Gap Analysis  |                                                                                                                                    |
| Set Synthesis & Gap Analysis    | Set                                 | Extracts synthesis and gap analysis report        | Synthesis & Gap Analysis      | Wait2                        |                                                                                                                                    |
| Wait2                         | Wait                                | Delays to respect API rate limits                 | Set Synthesis & Gap Analysis  | Ideal Page Outline Generation |                                                                                                                                    |
| Ideal Page Outline Generation  | Langchain Chain LLM (Google Gemini) | Generates optimal page outline                     | Wait2                        | Set Page Outline             |                                                                                                                                    |
| Set Page Outline               | Set                                 | Extracts page outline                              | Ideal Page Outline Generation | Wait3                        |                                                                                                                                    |
| Wait3                         | Wait                                | Delays to respect API rate limits                 | Set Page Outline             | UX, Conversion & Copywriting Enhancement |                                                                                                                                    |
| UX, Conversion & Copywriting Enhancement | Langchain Chain LLM (Google Gemini) | Provides UX, CRO, and copywriting recommendations | Wait3                        | Set UX & Conversions Enhancements |                                                                                                                                    |
| Set UX & Conversions Enhancements | Set                                 | Extracts UX and conversion recommendations         | UX, Conversion & Copywriting Enhancement | Google Gemini Chat Model5     |                                                                                                                                    |
| Google Gemini Chat Model5      | Langchain LLM (Google Gemini)        | Intermediate AI processing step                    | Set UX & Conversions Enhancements | Final Service Page Blueprint |                                                                                                                                    |
| Final Service Page Blueprint   | Langchain Chain LLM (Google Gemini) | Compiles final Markdown blueprint document         | Google Gemini Chat Model5     | Edit Fields2                 |                                                                                                                                    |
| Edit Fields2                  | Set                                 | Extracts final Markdown content                     | Final Service Page Blueprint  | Convert to File              |                                                                                                                                    |
| Convert to File               | ConvertToFile                       | Converts Markdown text to downloadable .txt file  | Edit Fields2                 |                               |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node (`Start`):**  
   - Title: "Competitors Analysis for Service-Based Queries"  
   - Fields:  
     - Competitors (textarea, required)  
     - Target Keyword (text, required)  
     - Services Offered (textarea, required)  
     - Brand Name (text, required)  
     - Is Homepage? (dropdown: Yes/No, required)  
   - Configure webhook for form submission.

2. **Create a Set Node (`Edit Fields`):**  
   - Assign:  
     - `JINA Reader API Key` (string, default "YOUR_API_KEY")  
     - `Google Gemini Model` (string, default "gemini-2.5-pro-preview-03-25")  
     - `Waiting Time (Seconds)` (string, default "1")  
   - Connect `Start` → `Edit Fields`.

3. **Create a Code Node (`Convert URLs to Items`):**  
   - JavaScript to split competitor URLs by line breaks, trim, filter empty lines, output array of `{ competitor_url }` items.  
   - Connect `Edit Fields` → `Convert URLs to Items`.

4. **Create a SplitInBatches Node (`Loop Over Items`):**  
   - Default batch size 1 (process one competitor at a time).  
   - Connect `Convert URLs to Items` → `Loop Over Items`.

5. **Create an HTTP Request Node (`Get URL HTML`):**  
   - Method: GET  
   - URL: `https://r.jina.ai/{{ $json.competitor_url }}`  
   - Headers:  
     - Authorization: `Bearer {{ $('Edit Fields').first().json['JINA Reader API Key'] }}`  
     - X-Return-Format: `html`  
   - Error Handling: Continue on error, max 5 retries, 5s wait between tries.  
   - Connect `Loop Over Items` → `Get URL HTML`.

6. **Create a Code Node (`Extract HTML Elements`):**  
   - JavaScript to parse HTML content: extract H1-H6 headings with positions, meta tags (title, description, Open Graph, Twitter), JSON-LD schemas, and generate filtered 2-4 word n-grams from headings.  
   - Clean HTML entities and remove inner tags.  
   - Connect `Get URL HTML` → `Extract HTML Elements`.

7. **Create a Set Node (`Set URL Data`):**  
   - Assign stringified JSON fields: Competitor URL, Outline, Meta, Ngrams.  
   - Connect `Extract HTML Elements` → `Set URL Data`.

8. **Connect `Set URL Data` back to `Loop Over Items`** to continue batch processing.

9. **Create a Code Node (`Code`):**  
   - Aggregate all competitor data items into a single XML-like string with competitor codes, outlines, meta, and ngrams.  
   - Connect `Loop Over Items` → `Code`.

10. **Create a Set Node (`Edit Fields1`):**  
    - Pass aggregated competitor data string as `competitors_data`.  
    - Connect `Code` → `Edit Fields1`.

11. **Create a Langchain Chain LLM Node (`Competitors Analysis`):**  
    - Model: Google Gemini (use credential)  
    - Prompt: Analyze competitor data and target keyword to generate competitor analysis report in XML tag `<competitor_analysis_report>`.  
    - Temperature: 0.4  
    - Connect `Edit Fields1` → `Competitors Analysis`.

12. **Create a Set Node (`Set Competitor Analysis`):**  
    - Extract competitor analysis report content by removing XML tags.  
    - Connect `Competitors Analysis` → `Set Competitor Analysis`.

13. **Create a Wait Node (`Wait`):**  
    - Delay by `Waiting Time (Seconds)` from `Edit Fields`.  
    - Connect `Set Competitor Analysis` → `Wait`.

14. **Create a Langchain Chain LLM Node (`User Intent Analysis`):**  
    - Model: Google Gemini  
    - Prompt: Analyze target keyword for user intent, personas, buyer journey, etc., output in `<user_intent_report>`.  
    - Temperature: 0.4  
    - Connect `Wait` → `User Intent Analysis`.

15. **Create a Set Node (`Set User Intent Analysis`):**  
    - Extract user intent report content by removing XML tags.  
    - Connect `User Intent Analysis` → `Set User Intent Analysis`.

16. **Create a Wait Node (`Wait1`):**  
    - Delay by `Waiting Time (Seconds)`.  
    - Connect `Set User Intent Analysis` → `Wait1`.

17. **Create a Langchain Chain LLM Node (`Synthesis & Gap Analysis`):**  
    - Model: Google Gemini  
    - Prompt: Synthesize competitor and user intent reports to identify overlaps, gaps, SEO priorities, UX advantages in `<synthesis_and_gap_analysis>`.  
    - Temperature: 0.4  
    - Connect `Wait1` → `Synthesis & Gap Analysis`.

18. **Create a Set Node (`Set Synthesis & Gap Analysis`):**  
    - Extract synthesis report content by removing XML tags.  
    - Connect `Synthesis & Gap Analysis` → `Set Synthesis & Gap Analysis`.

19. **Create a Wait Node (`Wait2`):**  
    - Delay by `Waiting Time (Seconds)`.  
    - Connect `Set Synthesis & Gap Analysis` → `Wait2`.

20. **Create a Langchain Chain LLM Node (`Ideal Page Outline Generation`):**  
    - Model: Google Gemini  
    - Prompt: Generate recommended page outline (H1-H4) based on synthesis report, homepage flag, brand, services, output in `<recommended_page_outline>`.  
    - Temperature: 0.4  
    - Connect `Wait2` → `Ideal Page Outline Generation`.

21. **Create a Set Node (`Set Page Outline`):**  
    - Extract page outline content by removing XML tags.  
    - Connect `Ideal Page Outline Generation` → `Set Page Outline`.

22. **Create a Wait Node (`Wait3`):**  
    - Delay by `Waiting Time (Seconds)`.  
    - Connect `Set Page Outline` → `Wait3`.

23. **Create a Langchain Chain LLM Node (`UX, Conversion & Copywriting Enhancement`):**  
    - Model: Google Gemini  
    - Prompt: Provide detailed UX, CRO, copywriting recommendations based on page outline, user intent, brand, services, output in `<ux_conversion_copy_recommendations>`.  
    - Temperature: 0.4  
    - Connect `Wait3` → `UX, Conversion & Copywriting Enhancement`.

24. **Create a Set Node (`Set UX & Conversions Enhancements`):**  
    - Extract UX & conversion recommendations by removing XML tags.  
    - Connect `UX, Conversion & Copywriting Enhancement` → `Set UX & Conversions Enhancements`.

25. **Create a Langchain Chain LLM Node (`Final Service Page Blueprint`):**  
    - Model: Google Gemini  
    - Prompt: Compile all inputs into a single Markdown blueprint document enclosed in `<final_service_page_blueprint>`.  
    - Temperature: 0.4  
    - Connect `Set UX & Conversions Enhancements` → `Final Service Page Blueprint`.

26. **Create a Set Node (`Edit Fields2`):**  
    - Extract Markdown content from XML tag.  
    - Connect `Final Service Page Blueprint` → `Edit Fields2`.

27. **Create a ConvertToFile Node (`Convert to File`):**  
    - Operation: To Text  
    - File Name: "Blueprint.txt"  
    - Source Property: `Final Blueprint` (from `Edit Fields2`)  
    - Connect `Edit Fields2` → `Convert to File`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                         | Context or Link                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| Create a new Jina Reader API key at https://jina.ai/api-dashboard/key-manager. Free tier allows up to 1 million tokens.                                                                                                                                | JINA Reader API Key setup                                                                                                |
| Create Google Gemini (PaLM) credentials following https://docs.n8n.io/integrations/builtin/credentials/googleai/#using-geminipalm-api-key.                                                                                                          | Google Gemini API setup                                                                                                  |
| For free Google Gemini tier users, set "Waiting Time" to 20 seconds to avoid exceeding 5 requests per minute limit.                                                                                                                                  | API rate limit management                                                                                                |
| The workflow outputs a Markdown blueprint document enclosed in XML tags; copy-paste content into https://markdownlivepreview.com/ for rendering.                                                                                                    | Markdown preview tool                                                                                                    |
| Demo of the generated report available at https://docs.google.com/document/d/1XDJV3zNB7cLPBzaMXstzEl7ZvPrjiuBbet5C5ZlC4bo/edit                                                                                                                        | Example output                                                                                                           |
| The workflow is designed to be customizable: adjust AI temperature, extraction logic, prompts, add industry-specific guidance, or integrate with CMS/project management tools.                                                                       | Customization suggestions                                                                                                |
| Avoid entering more than 5 competitor URLs to maintain context quality and avoid diluting AI analysis.                                                                                                                                                 | Input constraint                                                                                                         |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and extend the "High-Level Service Page SEO Blueprint Report Generator" workflow, ensuring robust integration and error anticipation.