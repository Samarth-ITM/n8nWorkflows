Create SEO Outlines from Top Google Results with SerpAPI and GPT-4o

https://n8nworkflows.xyz/workflows/create-seo-outlines-from-top-google-results-with-serpapi-and-gpt-4o-9056


# Create SEO Outlines from Top Google Results with SerpAPI and GPT-4o

### 1. Workflow Overview

This workflow automates the creation of SEO-optimized article outlines by analyzing top-ranking Google search results on a specified keyword. It is designed for content creators and SEO specialists who want to generate well-structured, MECE (Mutually Exclusive, Collectively Exhaustive) outlines based on real competitive content, ensuring relevance and search intent alignment.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception:** Captures the target keyword from a user-submitted form.
- **1.2 Search and Data Retrieval:** Uses SerpAPI to query Google for the keyword, excluding certain sites, and extracts the URLs of the top results.
- **1.3 Content Scraping and Cleaning:** Scrapes the HTML content of the URLs, extracts the main body content, converts HTML to Markdown, and aggregates multiple articles into a list.
- **1.4 AI-Powered Outline Generation:** Uses GPT-4o via LangChain to analyze the aggregated content and generate a structured SEO-optimized outline.
- **1.5 Output Handling:** Placeholder node for further processing or exporting the generated outline.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block provides the entry point for the workflow where users submit a keyword via a form. The keyword will be used to search Google and generate the outline.

**Nodes Involved:**  
- On form submission

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Captures user input from a web form with a single required field "Keyword".  
  - Configuration: Form titled "SEO optimized outline generator" with a description explaining the workflow's purpose.  
  - Expressions: Captures the `Keyword` field value as JSON output for downstream nodes.  
  - Inputs: None (trigger node)  
  - Outputs: Triggers "Get search results" node.  
  - Edge Cases: No submission or empty keyword (form enforces required field).  
  - Version: 2.3  

---

#### 2.2 Search and Data Retrieval

**Overview:**  
This block queries Google via SerpAPI for the submitted keyword, excluding Reddit and Quora URLs, extracts the top 5 organic result URLs, and prepares them for scraping.

**Nodes Involved:**  
- Get search results  
- Extract URLs from JSON  
- Split out URLs

**Node Details:**  
- **Get search results**  
  - Type: HTTP Request  
  - Role: Queries SerpAPI for Google search results using the keyword.  
  - Configuration:  
    - URL: `https://serpapi.com/search`  
    - Query Parameters:  
      - `q`: Keyword with exclusions (`-inurl:reddit.com -inurl:quora.com`)  
      - `location`: United States  
    - Authentication: Uses SerpAPI credential (API key)  
  - Input: Keyword from form trigger  
  - Output: JSON response with organic search results  
  - Edge Cases: API rate limits, network errors  
  - Retry enabled  
  - Version: 4.2  

- **Extract URLs from JSON**  
  - Type: Set  
  - Role: Extracts keyword and top 5 organic result URLs from SerpAPI JSON response.  
  - Configuration: Custom JSON output with fields:  
    - `Keyword`: passes through original keyword  
    - `URLs`: array of first five organic result links  
  - Input: JSON from "Get search results"  
  - Output: JSON with keyword and URLs array  
  - Edge Cases: Missing or fewer than five results (may cause missing URLs)  
  - Version: 3.4  

- **Split out URLs**  
  - Type: SplitOut  
  - Role: Splits the array of URLs into separate workflow items to scrape each URL individually.  
  - Configuration: Splits on field "URLs"  
  - Input: JSON with URLs array  
  - Output: Multiple items each containing a single URL  
  - Edge Cases: Empty or invalid URL entries  
  - Version: 1  

---

#### 2.3 Content Scraping and Cleaning

**Overview:**  
This block scrapes each URL's HTML content, extracts the main `<body>` content excluding certain tags, converts HTML to Markdown, aggregates the cleaned articles, and limits the number of articles to 3 to fit the AI model’s context window.

**Nodes Involved:**  
- Scrape Content  
- Limit  
- Extract body content  
- Clean up Text  
- Put all articles into one item

**Node Details:**  
- **Scrape Content**  
  - Type: HTTP Request  
  - Role: Scrapes raw HTML content from each URL.  
  - Configuration:  
    - URL from each split URL item  
    - Headers include User-Agent and accept headers to mimic a browser request  
    - Batching: max 5 URLs per batch to avoid overloading sites or IP blocking  
    - Retry: 5 tries, on error output continues workflow  
  - Input: Individual URL items  
  - Output: Raw HTML content of each page  
  - Edge Cases: Site down, anti-scraping measures, timeouts, HTTP errors  
  - Version: 4.2  

- **Limit**  
  - Type: Limit  
  - Role: Restricts number of scraped articles to 3 for AI context size constraints.  
  - Configuration: maxItems = 3  
  - Input: Scraped content items  
  - Output: Up to 3 items forwarded  
  - Edge Cases: Fewer than 3 scraped articles still processed  
  - Version: 1  

- **Extract body content**  
  - Type: HTML  
  - Role: Extracts only the `<body>` content from the scraped HTML, excluding images, meta, and links.  
  - Configuration:  
    - Operation: extractHtmlContent  
    - CSS selector: `body`  
    - Skip selectors: `img, meta, a`  
  - Input: Raw HTML from "Scrape Content"  
  - Output: Cleaned HTML with main textual content  
  - Edge Cases: Malformed HTML, missing body tag  
  - Version: 1.2  

- **Clean up Text**  
  - Type: Markdown  
  - Role: Converts extracted HTML body content to Markdown to simplify text and reduce noise.  
  - Configuration: Input is the extracted HTML field `html`, output stored in `markdown` key.  
  - Input: Extracted body HTML  
  - Output: Markdown text of article body  
  - Edge Cases: HTML to Markdown conversion errors (rare)  
  - Version: 1  

- **Put all articles into one item**  
  - Type: Aggregate  
  - Role: Combines the multiple Markdown article items into a single list for AI processing.  
  - Configuration: Aggregates field `markdown` from all items into an array.  
  - Input: Markdown items from cleaning step  
  - Output: Single item with array of Markdown articles  
  - Edge Cases: Empty article list (should not happen due to limit)  
  - Version: 1  

---

#### 2.4 AI-Powered Outline Generation

**Overview:**  
This block sends the aggregated article content and keyword to an OpenAI GPT-4o model via LangChain. The AI analyzes the content for common themes and structures, then generates a MECE, SEO-focused article outline.

**Nodes Involved:**  
- Create outline

**Node Details:**  
- **Create outline**  
  - Type: LangChain OpenAI  
  - Role: Utilizes GPT-4o model to generate a structured SEO article outline based on provided articles and keyword.  
  - Configuration:  
    - Model: GPT-4o  
    - Temperature: 0.7 (balance of creativity and focus)  
    - Messages:  
      - System role instructs AI as SEO Content Strategist to analyze 3 articles strictly without hallucination and generate MECE outline  
      - User content dynamically inserts Keyword and up to 3 articles' Markdown content, handles missing articles gracefully  
    - Input: Keyword and aggregated Markdown list  
    - Output: Generated outline text  
  - Edge Cases: API errors, model rate limits, incomplete article data causing less precise outlines  
  - Credentials: OpenAI API key configured in n8n  
  - Version: 1.8  

---

#### 2.5 Output Handling

**Overview:**  
This node is a placeholder to allow further processing of the generated outline, such as exporting to documents, generating content, or enrichment.

**Nodes Involved:**  
- Do with the outline what you want

**Node Details:**  
- **Do with the outline what you want**  
  - Type: NoOp (No Operation)  
  - Role: Placeholder for user-defined next steps after outline creation.  
  - Configuration: None  
  - Input: Outline text from AI node  
  - Output: None  
  - Edge Cases: None  
  - Version: 1  

---

### 3. Summary Table

| Node Name                    | Node Type                  | Functional Role                                 | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                                         |
|------------------------------|----------------------------|------------------------------------------------|--------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------|
| On form submission            | Form Trigger               | Capture keyword input                           | None                     | Get search results          | Add your keyword to the form and watch the magic happen ⚡                                                           |
| Get search results            | HTTP Request               | Search Google via SerpAPI excluding forums     | On form submission       | Extract URLs from JSON      | Scrape Google using SerpAPI; exclude Reddit, Quora, optionally own domain                                           |
| Extract URLs from JSON        | Set                        | Extract top 5 URLs and keyword from JSON       | Get search results       | Split out URLs              | Extract URLs from JSON; clean and select only needed URLs                                                           |
| Split out URLs                | SplitOut                   | Split URL array into individual URLs           | Extract URLs from JSON   | Scrape Content             | Split out URLs to scrape each separately                                                                             |
| Scrape Content                | HTTP Request               | Scrape HTML content of each URL                 | Split out URLs           | Limit                      | Scrape URLs with batching and polite headers; avoid hammering                                                        |
| Limit                        | Limit                      | Limit to 3 articles to fit AI context window   | Scrape Content           | Extract body content        | Limit amount of articles; handle failed scrapes                                                                     |
| Extract body content          | HTML                       | Extract main textual body content from HTML    | Limit                    | Clean up Text              | Extract body content without images, meta, links                                                                    |
| Clean up Text                | Markdown                   | Convert HTML to Markdown for cleaner text      | Extract body content     | Put all articles into one item | Clean up text by converting HTML to Markdown                                                                         |
| Put all articles into one item| Aggregate                  | Combine Markdown articles into one list         | Clean up Text            | Create outline              | Put all articles into one item                                                                                        |
| Create outline               | LangChain OpenAI           | Generate SEO-optimized MECE outline via GPT-4o | Put all articles into one item | Do with the outline what you want | Create outline using AI; analyze articles and generate MECE SEO structure                                           |
| Do with the outline what you want | NoOp                   | Placeholder for further processing              | Create outline           | None                       | Continue the workflow... write to Google Docs, generate content, enrich with search data                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** ("On form submission"):  
   - Set form title: "SEO optimized outline generator"  
   - Add one required text field: "Keyword" with placeholder "Your keyword here"  
   - This node triggers the workflow on form submission.

2. **Add HTTP Request node** ("Get search results"):  
   - URL: `https://serpapi.com/search`  
   - Query Parameters:  
     - `q`: Expression `={{ $json.Keyword }} -inurl:reddit.com -inurl:quora.com`  
     - `location`: "United States"  
   - Authentication: Use SerpAPI credential configured with your API key  
   - Connect output of form trigger to this node.

3. **Add Set node** ("Extract URLs from JSON"):  
   - Mode: Raw  
   - JSON Output:  
     ```json
     {
       "Keyword": "{{ $('On form submission').item.json.Keyword }}",
       "URLs": [
         "{{ $json.organic_results[0].link }}",
         "{{ $json.organic_results[1].link }}",
         "{{ $json.organic_results[2].link }}",
         "{{ $json.organic_results[3].link }}",
         "{{ $json.organic_results[4].link }}"
       ]
     }
     ```  
   - Connect output of "Get search results" to this node.

4. **Add SplitOut node** ("Split out URLs"):  
   - Field to split out: "URLs"  
   - Connect output of "Extract URLs from JSON" to this node.

5. **Add HTTP Request node** ("Scrape Content"):  
   - URL: Expression `={{ $json.URLs }}`  
   - Headers:  
     - User-Agent: `Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/138.0.0.0 Safari/537.36`  
     - Accept: `text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8`  
     - Accept-Encoding: `gzip, deflate, br`  
     - Accept-Language: `en-US,en;q=0.5`  
   - Enable batching with batch size 5  
   - Max Tries: 5  
   - On error: continue error output  
   - Connect output of "Split out URLs" to this node.

6. **Add Limit node** ("Limit"):  
   - Max items: 3  
   - Connect output of "Scrape Content" to this node.

7. **Add HTML node** ("Extract body content"):  
   - Operation: extractHtmlContent  
   - Extraction values:  
     - key: "html"  
     - CSS Selector: "body"  
     - Skip selectors: "img, meta, a"  
   - Connect output of "Limit" to this node.

8. **Add Markdown node** ("Clean up Text"):  
   - Input: field `html` from "Extract body content"  
   - Destination key: "markdown"  
   - Connect output of "Extract body content" to this node.

9. **Add Aggregate node** ("Put all articles into one item"):  
   - Aggregate field: "markdown"  
   - Connect output of "Clean up Text" to this node.

10. **Add LangChain OpenAI node** ("Create outline"):  
    - Model: GPT-4o  
    - Temperature: 0.7  
    - Messages:  
      - System message: instruct AI as SEO Content Strategist with detailed instructions to analyze 3 articles and keyword to create MECE SEO outline, no hallucination, no summaries, output format specified.  
      - User message: pass keyword and up to 3 articles' markdown content using expressions, handle nulls gracefully.  
    - Credentials: OpenAI API key configured in n8n  
    - Connect output of "Put all articles into one item" to this node.

11. **Add NoOp node** ("Do with the outline what you want"):  
    - Connect output of "Create outline" to this node.  
    - This node is a placeholder for downstream actions like exporting or further processing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Use this workflow to create SEO-friendly outlines from high-ranking Google articles. Enter a keyword, scrape top results, analyze with AI, and build a MECE SEO outline.       | Sticky Note overview in workflow                                                                       |
| Sign up for a SerpAPI account (free tier available) and create an OpenAI account to get API keys. Configure credentials in n8n before running the workflow.                   | https://serpapi.com/ and https://openai.com/api/                                                      |
| Exclude forums and UGC sites like Reddit and Quora using `-inurl:` filters in the search query to avoid noisy or low-quality content.                                        | Sticky Note on search query setup                                                                      |
| Limit scraping to 5 URLs in batch with polite User-Agent headers to avoid IP blocking and respect target sites.                                                                | Sticky Note on scraping best practices                                                                |
| Limit articles to 3 for AI context window constraints.                                                                                                                        | Sticky Note on Limit node                                                                              |
| Extract only the `<body>` content from HTML, skipping images, meta tags, and links to reduce noise before converting to Markdown.                                             | Sticky Note on HTML extraction and markdown conversion                                                |
| The AI node uses GPT-4o with instructions to produce a MECE, SEO-optimized outline strictly based on provided article content and keyword, avoiding hallucinations or summaries.| Sticky Note on AI usage                                                                                 |
| The placeholder NoOp node invites further customization such as exporting outlines to Google Docs or generating full articles with additional AI nodes.                      | Sticky Note on extending workflow                                                                     |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a no-code automation tool. The processing strictly respects content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.