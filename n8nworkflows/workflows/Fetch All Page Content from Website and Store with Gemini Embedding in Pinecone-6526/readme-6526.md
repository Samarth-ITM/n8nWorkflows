Fetch All Page Content from Website and Store with Gemini Embedding in Pinecone

https://n8nworkflows.xyz/workflows/fetch-all-page-content-from-website-and-store-with-gemini-embedding-in-pinecone-6526


# Fetch All Page Content from Website and Store with Gemini Embedding in Pinecone

### 1. Workflow Overview

This workflow automates the creation of a Pinecone vector knowledge base by fetching website page content from either a sitemap XML or a list of direct URLs. It is designed to handle URL input, extract textual content from web pages, generate vector embeddings using Google Gemini's embedding model, and store the results in a Pinecone index for subsequent AI-driven retrieval or search.

The workflow is logically divided into three main blocks:

- **1.1 URL Input & Consolidation:** Accepts input via a form trigger for either a sitemap URL or a comma-separated list of page URLs. It processes and consolidates these URLs into a clean, unique list for further processing.

- **1.2 Content Extraction:** Iterates over the consolidated URLs in batches, fetching each page's HTML content, waiting to respect website server load, and extracting clean textual content while omitting images.

- **1.3 Embedding & Pinecone Storage:** Converts extracted page text into vector embeddings using the Google Gemini embedding model and inserts these embeddings with associated documents into a Pinecone vector store, clearing any existing namespace data beforehand to ensure data consistency.

---

### 2. Block-by-Block Analysis

#### 2.1 URL Input & Consolidation

**Overview:**  
This block receives input from a user form that accepts either a sitemap XML URL or a list of individual page URLs. It then routes the input accordingly, fetches and parses the sitemap if provided, or cleans and formats the list of page URLs. The final output is a merged, deduplicated list of URLs to process.

**Nodes Involved:**  
- Input Sitemap or page urls (Form Trigger)  
- Switch  
- Split Pages URL (Code)  
- Fetch Sitemap (HTTP Request)  
- XML Conversion (XML)  
- Extract Page URLs (Code)  
- Merge URLs (Merge)  
- Remove Duplicate URLs (Remove Duplicates)  

**Node Details:**  

- **Input Sitemap or page urls (Form Trigger)**  
  - *Type:* Form Trigger  
  - *Role:* Entry point to receive user input of sitemap URL and/or page URLs.  
  - *Config:* Contains two form fields: "Sitemap URL" (single-line input) and "Page URLs" (textarea, comma-separated).  
  - *Input/Output:* Outputs JSON with keys `"Sitemap URL"` and `"Page URLs"`.  
  - *Failure Modes:* Missing or malformed URLs; empty inputs.  

- **Switch**  
  - *Type:* Switch (Routing node)  
  - *Role:* Determines whether to process sitemap URL or page URLs based on input presence.  
  - *Config:*  
    - First output if `"Page URLs"` is not empty (string non-empty check).  
    - Second output if `"Sitemap URL"` ends with "xml" (valid sitemap).  
  - *Input/Output:* Routes data to either "Split Pages URL" or "Fetch Sitemap".  
  - *Edge Cases:* Both empty (no route); both present (routes to both outputs).  

- **Split Pages URL (Code)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses the comma-separated "Page URLs" string into individual URL objects, ensuring trailing slashes.  
  - *Key Code:* Adds trailing slash if missing; trims whitespace; splits by commas.  
  - *Input:* JSON with key `"Page URLs"` as string.  
  - *Output:* Array of objects `{ url: string }`.  
  - *Failure Modes:* Non-string input; empty or malformed URLs.  

- **Fetch Sitemap (HTTP Request)**  
  - *Type:* HTTP Request  
  - *Role:* Downloads sitemap XML from the provided URL.  
  - *Config:* URL set dynamically from `"Sitemap URL"` input.  
  - *Output:* Raw XML data.  
  - *Failure Modes:* Network errors, 404, invalid URL, timeout.  

- **XML Conversion (XML)**  
  - *Type:* XML to JSON converter  
  - *Role:* Converts downloaded sitemap XML into JSON format for easier processing.  
  - *Failure Modes:* Malformed XML, conversion errors.  

- **Extract Page URLs (Code)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Iterates over the JSON sitemap structure to extract page URLs from `<url><loc>` elements.  
  - *Key Code:* Loops over `$input.first().json.urlset.url` extracting `loc` values.  
  - *Output:* Array of objects `{ url: string }`.  
  - *Failure Modes:* Unexpected sitemap JSON structure, empty sitemap.  

- **Merge URLs (Merge)**  
  - *Type:* Merge  
  - *Role:* Combines URLs from both sitemap extraction and direct page URL input into single stream.  
  - *Failure Modes:* None typical, but empty inputs may produce empty output.  

- **Remove Duplicate URLs (Remove Duplicates)**  
  - *Type:* Remove Duplicates  
  - *Role:* Ensures uniqueness by removing duplicate URLs to avoid redundant processing.  
  - *Failure Modes:* None typical, but watch for case sensitivity or trailing slash differences causing missed duplicates.  

---

#### 2.2 Content Extraction

**Overview:**  
Processes each unique URL in batches, fetches the HTML content, waits 5 seconds between requests to avoid server overload, and extracts clean textual content from the page body excluding images.

**Nodes Involved:**  
- Loop Over Page URLs (Split In Batches)  
- Fetch Page HTML For content (HTTP Request)  
- Wait 5 sec (Wait)  
- Extract Content (HTML)  

**Node Details:**  

- **Loop Over Page URLs (Split In Batches)**  
  - *Type:* Split In Batches  
  - *Role:* Processes URLs in manageable batches to control request rate and memory usage.  
  - *Config:* Default batch size (not explicitly set in JSON, so default applies).  
  - *Failure Modes:* Large batch sizes may cause timeouts or memory issues.  

- **Fetch Page HTML For content (HTTP Request)**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the full HTML content of the current URL.  
  - *Config:* URL dynamically set to current item's `url` field.  
  - *Failure Modes:* Network errors, 404, redirects, timeouts.  

- **Wait 5 sec (Wait)**  
  - *Type:* Wait  
  - *Role:* Pauses workflow execution for 5 seconds after each page fetch to avoid rate limiting or server overload.  
  - *Failure Modes:* None typical, but long workflows may time out overall.  

- **Extract Content (HTML)**  
  - *Type:* HTML Extraction  
  - *Role:* Extracts main text content from the HTML body, explicitly skipping `<img>` tags. Cleans extracted text for downstream usage.  
  - *Config:* Extracts from CSS selector `body`, skips images.  
  - *Failure Modes:* Pages with dynamic content or heavy JavaScript may yield incomplete extraction; malformed HTML.  

---

#### 2.3 Embedding & Pinecone Storage

**Overview:**  
Transforms extracted textual content into vector embeddings using Google Gemini embedding model, prepares document data for vector storage, and inserts data into a Pinecone index named "supportbot". The namespace is cleared before insertion to avoid stale data.

**Nodes Involved:**  
- Gemini Embeddings (Embeddings)  
- Data Loader (Document Loader)  
- Pinecone KnowledgeBase (Vector Store)  

**Node Details:**  

- **Gemini Embeddings (Embeddings)**  
  - *Type:* Langchain Embeddings (Google Gemini)  
  - *Role:* Generates 3076-dimensional vector embeddings using the `models/gemini-embedding-001` model for the extracted text content.  
  - *Config:* Model name set explicitly; no parameters for batch size shown.  
  - *Failure Modes:* API authentication errors, model unavailability, rate limits, content too large to embed.  

- **Data Loader (Document Loader)**  
  - *Type:* Langchain Document Loader  
  - *Role:* Converts the extracted content into documents compatible with Langchain vector store ingestion.  
  - *Config:* Default options; no special loader settings.  
  - *Failure Modes:* Input format errors, empty content.  

- **Pinecone KnowledgeBase (Vector Store)**  
  - *Type:* Langchain Vector Store (Pinecone)  
  - *Role:* Inserts embeddings and associated documents into the Pinecone index named "supportbot". Clears the namespace beforehand for clean data insertion.  
  - *Config:* Mode set to "insert"; `clearNamespace` enabled.  
  - *Failure Modes:* Pinecone API errors, authentication failures, quota limits, network issues.  

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                                | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                                         |
|------------------------|----------------------------------|-----------------------------------------------|----------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Input Sitemap or page urls | Form Trigger                    | Entry point to get sitemap or page URLs       | —                                | Switch                         | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Switch                 | Switch                           | Routes input based on sitemap or page URLs    | Input Sitemap or page urls        | Split Pages URL, Fetch Sitemap  | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Split Pages URL         | Code                             | Parses and cleans individual page URLs        | Switch (Page URLs branch)         | Merge URLs                    | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Fetch Sitemap           | HTTP Request                     | Downloads sitemap XML                          | Switch (Sitemap URL branch)       | XML Conversion                | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| XML Conversion          | XML                              | Converts sitemap XML to JSON                   | Fetch Sitemap                    | Extract Page URLs             | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Extract Page URLs       | Code                             | Extracts URLs from sitemap JSON                | XML Conversion                  | Merge URLs                    | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Merge URLs              | Merge                            | Combines URLs from both inputs                 | Split Pages URL, Extract Page URLs | Remove Duplicate URLs          | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Remove Duplicate URLs   | Remove Duplicates                | Removes duplicate URLs                          | Merge URLs                      | Loop Over Page URLs           | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Loop Over Page URLs     | Split In Batches                 | Processes URLs in batches                       | Remove Duplicate URLs            | Extract Content, Fetch Page HTML For content | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Fetch Page HTML For content | HTTP Request                  | Downloads HTML content of each page            | Loop Over Page URLs             | Wait 5 sec                   | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Wait 5 sec             | Wait                             | Delays 5 seconds between requests              | Fetch Page HTML For content       | Loop Over Page URLs           | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Extract Content         | HTML                             | Extracts clean textual content from HTML       | Loop Over Page URLs             | Pinecone KnowledgeBase        | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Gemini Embeddings       | Langchain Embeddings (Google Gemini) | Converts text into vector embeddings           | Extract Content                 | Pinecone KnowledgeBase (ai_embedding) | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Data Loader            | Langchain Document Loader         | Prepares documents for vector store ingestion | Gemini Embeddings               | Pinecone KnowledgeBase (ai_document) | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Pinecone KnowledgeBase  | Langchain Vector Store (Pinecone) | Inserts embeddings and documents into Pinecone | Extract Content, Gemini Embeddings, Data Loader | —                             | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |
| Sticky Note            | Sticky Note                      | Workflow overview and instructions             | —                                | —                             | This n8n workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node** named "Input Sitemap or page urls":  
   - Add two fields to the form:  
     - "Sitemap URL" (Single line text, placeholder: `https://website.com/page-sitemap.xml`)  
     - "Page URLs" (Textarea, placeholder: `https://website.com/about, https://website.com/contact`)  
   - This node serves as the workflow entry point.

2. **Create a Switch node** named "Switch":  
   - Connect "Input Sitemap or page urls" → "Switch".  
   - Configure two rules:  
     - Rule 1: If "Page URLs" is NOT empty (string not empty).  
     - Rule 2: If "Sitemap URL" ends with "xml".  
   - This routes input based on which field is populated.

3. **Create a Code node** named "Split Pages URL":  
   - Connect Switch output 1 (Page URLs) → Split Pages URL.  
   - Paste JavaScript to split the comma-separated URLs, trim spaces, and add trailing slashes if missing:  
     ```javascript
     function addTrailingSlash(str) {
       if (typeof str !== 'string') {
         return str;
       }
       if (!str.endsWith('/')) {
         return str + '/';
       }
       return str;
     }
     const urls = [];
     for (const item of $input.first().json['Page URLs'].split(',')) {
       urls.push({ url: addTrailingSlash(item).trim()})
     }
     return urls;
     ```

4. **Create an HTTP Request node** named "Fetch Sitemap":  
   - Connect Switch output 2 (Sitemap URL) → Fetch Sitemap.  
   - Set URL to expression: `{{$json["Sitemap URL"]}}`  
   - Method: GET (default).

5. **Create an XML node** named "XML Conversion":  
   - Connect "Fetch Sitemap" → "XML Conversion".  
   - Use default XML to JSON conversion settings.

6. **Create a Code node** named "Extract Page URLs":  
   - Connect "XML Conversion" → "Extract Page URLs".  
   - Paste JavaScript to extract URLs from sitemap JSON:  
     ```javascript
     const items = [];
     for (const item of $input.first().json.urlset.url) {
       items.push({ url: item.loc });
     }
     return items;
     ```

7. **Create a Merge node** named "Merge URLs":  
   - Connect outputs from "Split Pages URL" and "Extract Page URLs" into "Merge URLs" (two inputs).  
   - Use default merge mode (append).

8. **Create a Remove Duplicates node** named "Remove Duplicate URLs":  
   - Connect "Merge URLs" → "Remove Duplicate URLs".  
   - Configure to remove duplicates based on `url` property.

9. **Create a Split In Batches node** named "Loop Over Page URLs":  
   - Connect "Remove Duplicate URLs" → "Loop Over Page URLs".  
   - Use default batch size (e.g., 10).

10. **Create an HTTP Request node** named "Fetch Page HTML For content":  
    - Connect "Loop Over Page URLs" → "Fetch Page HTML For content".  
    - Set URL to expression: `{{$json.url}}`  
    - Method: GET.

11. **Create a Wait node** named "Wait 5 sec":  
    - Connect "Fetch Page HTML For content" → "Wait 5 sec".  
    - Set wait time to 5 seconds.

12. **Connect "Wait 5 sec" back to "Loop Over Page URLs"** to continue batch processing.

13. **Create an HTML Extract node** named "Extract Content":  
    - Connect "Loop Over Page URLs" → "Extract Content" (parallel to Fetch Page HTML).  
    - Configure extraction:  
      - Operation: Extract HTML content  
      - CSS Selector: `body`  
      - Skip Selectors: `img`  
      - Enable text cleanup.

14. **Create a Langchain Embeddings node** named "Gemini Embeddings":  
    - Connect "Extract Content" → "Gemini Embeddings".  
    - Set model name to `models/gemini-embedding-001`.  
    - Configure OpenAI/Google credentials for Gemini embeddings accordingly.

15. **Create a Langchain Document Loader node** named "Data Loader":  
    - Connect "Gemini Embeddings" → "Data Loader".  
    - Use default document loader options.

16. **Create a Langchain Vector Store Pinecone node** named "Pinecone KnowledgeBase":  
    - Connect "Extract Content" → "Pinecone KnowledgeBase" (main input).  
    - Connect "Gemini Embeddings" → "Pinecone KnowledgeBase" (ai_embedding input).  
    - Connect "Data Loader" → "Pinecone KnowledgeBase" (ai_document input).  
    - Configure mode as "insert".  
    - Enable clearing namespace before insertion.  
    - Configure Pinecone credentials (API key, environment, index name "supportbot").  

17. **Create a Sticky Note node** with the content from section 1 overview and place it for documentation and user guidance.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow builds a Pinecone knowledge base from website content, handling both sitemap and direct URL inputs.                                                     | Sticky Note throughout the workflow                                                               |
| The Gemini embedding model used is `models/gemini-embedding-001`, a 3076-dimensional embedding from Google Gemini.                                                    | Gemini Embeddings node configuration                                                             |
| To avoid overwhelming target websites, the workflow inserts a 5-second wait between page requests.                                                                    | Wait 5 sec node                                                                                   |
| Pinecone namespace is cleared before inserting new vector data to prevent stale or duplicate entries.                                                                 | Pinecone KnowledgeBase node parameter                                                            |
| Form Trigger supports inputting either a sitemap URL ending with ".xml" or a comma-separated list of URLs with trailing slashes normalized automatically.             | Input Sitemap or page urls node and Split Pages URL code node                                     |
| More info on Pinecone: https://www.pinecone.io/                                                                                                                      | External resource                                                                                  |
| More info on Google Gemini embeddings: https://developers.google.com/vertex-ai/embeddings                                                                            | External resource                                                                                  |
| The workflow assumes website pages are largely static HTML with extractable body content; dynamic or heavily JavaScript-rendered pages may require additional handling. | General extraction limitation                                                                     |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.