Extract and Verify Book Titles from Bookshelf Photos using GPT-4o and Google Books

https://n8nworkflows.xyz/workflows/extract-and-verify-book-titles-from-bookshelf-photos-using-gpt-4o-and-google-books-7350


# Extract and Verify Book Titles from Bookshelf Photos using GPT-4o and Google Books

### 1. Workflow Overview

This workflow processes bookshelf photos submitted via a webhook to extract and verify book titles and authors using AI and external book data sources. It targets applications needing automated book cataloging from images, such as personal library management or inventory systems.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception and Normalization:** Receives incoming HTTP POST requests containing image URLs and normalizes the input JSON structure.
- **1.2 AI Image Analysis:** Uses GPT-4o to strictly analyze the bookshelf image, extracting readable book titles and authors in a normalized JSON format.
- **1.3 Data Splitting and Preparation:** Splits the parsed list of books into individual items and prepares author name snippets for verification queries.
- **1.4 Title Verification via Google Books API:** Queries Google Books API to confirm and enrich each extracted book title and author information.
- **1.5 Data Normalization and Deduplication:** Cleans and normalizes verified data, then reaggregates and deduplicates the final list of books.
- **1.6 Response to Frontend:** Sends back the final verified and normalized list of books as JSON response to the initial webhook caller.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Normalization

**Overview:**  
This block handles incoming requests via a webhook and extracts the image URL from various possible JSON fields, trimming and normalizing it for consistent downstream processing.

**Nodes Involved:**  
- Webhook  
- Input normalized  
- Sticky Note (Webhook context)

**Node Details:**

- **Webhook**  
  - Type: HTTP Webhook (Trigger)  
  - Role: Receives POST requests with JSON payload containing image URL  
  - Configuration: Path set to a unique webhook ID, accepts POST, responds via response node  
  - Input/Output: No input, outputs raw request JSON  
  - Edge cases: Missing or malformed JSON, unsupported HTTP methods  
  - Sticky Note: "Webhook connects to front end that passes on JSON with imageURL (string)"

- **Input normalized**  
  - Type: Set node  
  - Role: Extracts and normalizes image URL from various possible JSON keys (`body.imageUrl`, `body.image`, `imageUrl`, `image`), trimming whitespace  
  - Configuration: Single string assignment with expression to check multiple fallback keys  
  - Input: Webhook output JSON  
  - Output: JSON with normalized `image` field  
  - Edge cases: Missing or empty image URL, invalid URL format  
  - Sticky Note: "Input is normalized"

---

#### 1.2 AI Image Analysis

**Overview:**  
Uses GPT-4o-mini model to perform strict image analysis on the bookshelf photo URL, extracting only clearly readable book titles and authors as a structured JSON array.

**Nodes Involved:**  
- Analyze image  
- Sticky Note (Image analysis context)

**Node Details:**

- **Analyze image**  
  - Type: OpenAI (LangChain OpenAI node)  
  - Role: Receives normalized image URL, sends to GPT-4o-mini with prompt instructing strict extraction of book titles/authors, no guessing, normalized capitalization, deduplication at output level  
  - Configuration: Model: GPT-4o-mini, operation: analyze image resource, prompt enforces strict JSON output format with books array  
  - Input: Normalized image URL  
  - Output: JSON containing `books` array with objects `{title, author|null}`  
  - Credentials: OpenAI API key configured  
  - Edge cases: API errors, rate limits, unclear images causing empty or incomplete data  
  - Sticky Note: "Image is analyzed and transformed."

---

#### 1.3 Data Splitting and Preparation

**Overview:**  
Splits the single JSON array of books into individual items. Each item carries extracted title and author info, and prepares a simplified searchAuthor field (first author only) for better Google Books API queries.

**Nodes Involved:**  
- Item list split (Code node)  
- Sticky Note (Splitting explanation)

**Node Details:**

- **Item list split**  
  - Type: Code node (JavaScript)  
  - Role: Parses the JSON from previous node, strips code fences if present, extracts each book into separate items with fields: `title`, `author`, and `searchAuthor` (first author name only or null)  
  - Configuration: Custom JS code carefully handling parsing and edge cases like missing fields, fallback JSON parsing  
  - Input: JSON containing `books` array or raw content string  
  - Output: Multiple items, each a single book record ready for validation  
  - Edge cases: Malformed JSON, empty books array, missing authors, multiple authors handled by splitting on commas or conjunctions  
  - Sticky Note: "Splits the output (in this case, books) into individual items in preparation for the next step which is to verify the book against a known source to confirm the title and author."

---

#### 1.4 Title Verification via Google Books API

**Overview:**  
Confirms and enriches each extracted book title and author by querying Google Books API using a composed search query with title and optionally first author.

**Nodes Involved:**  
- Title validation  
- Sticky Note (Verification context)

**Node Details:**

- **Title validation**  
  - Type: HTTP Request  
  - Role: Sends GET requests to Google Books API volumes endpoint with query parameters:  
    - `q` = `intitle:"title"` plus optional `inauthor:"searchAuthor"`  
    - maxResults=5, printType=books, orderBy=relevance  
  - Configuration: Query parameters dynamically composed with expressions to escape quotes and handle missing authors  
  - Input: Single book items (title, searchAuthor)  
  - Output: Google Books API JSON response with matching volumes  
  - Edge cases: API rate limits, no matches found (empty items array), invalid special characters in queries, network errors  
  - Sticky Note: "Confirms each title against Google Books"

---

#### 1.5 Data Normalization and Deduplication

**Overview:**  
Processes API responses to extract the most relevant title and author (first in lists), falls back to original data if API returns no results, and finally reaggregates the list of books deduplicating by lowercase title.

**Nodes Involved:**  
- Data normalized  
- Reaggregates list (Code node)  
- Sticky Notes (Normalization and deduplication)

**Node Details:**

- **Data normalized**  
  - Type: Set node  
  - Role: Extracts normalized title and author from Google Books API response's first item or falls back to original extracted values  
  - Configuration: Assignments use expressions to access nested JSON fields safely  
  - Input: Google Books API response  
  - Output: Single book item with normalized `title` and `author` fields  
  - Edge cases: Missing or empty API response, missing authors in API data, fallback to original values  
  - Sticky Note: "Normalizes data"

- **Reaggregates list**  
  - Type: Code node (JavaScript)  
  - Role: Collects all normalized books, removes duplicates by comparing lowercase trimmed titles, outputs a single array under `books` key  
  - Configuration: Uses a Set to track unique titles, outputs single JSON object with `books` array  
  - Input: Multiple normalized book items  
  - Output: Single JSON with deduplicated books array  
  - Edge cases: Titles differing only by case or whitespace, empty input arrays  
  - Sticky Note: "Reaggregates book list and dedupes"

---

#### 1.6 Response to Frontend

**Overview:**  
Sends the final verified, normalized, and deduplicated list of books back as JSON response to the original webhook caller.

**Nodes Involved:**  
- Respond to Webhook  
- Sticky Note (Response context)

**Node Details:**

- **Respond to Webhook**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP 200 JSON response containing the final books list JSON  
  - Configuration: Response code 200, content-type application/json, response body set to the workflow's final JSON output  
  - Input: Aggregated book list JSON  
  - Output: HTTP response to client  
  - Edge cases: Client disconnects, invalid JSON data  
  - Sticky Note: "Returns list back to frontend."

---

### 3. Summary Table

| Node Name          | Node Type              | Functional Role                             | Input Node(s)          | Output Node(s)       | Sticky Note                                         |
|--------------------|------------------------|---------------------------------------------|-----------------------|----------------------|-----------------------------------------------------|
| Webhook            | HTTP Webhook           | Receives image URL from frontend             | —                     | Input normalized     | Webhook connects to front end that passes on JSON with imageURL (string) |
| Input normalized   | Set                    | Normalizes input image URL field             | Webhook               | Analyze image        | Input is normalized                                 |
| Analyze image      | OpenAI (LangChain)     | Extracts book titles/authors from image     | Input normalized      | Item list split      | Image is analyzed and transformed.                  |
| Item list split    | Code                   | Splits books array into individual items    | Analyze image         | Title validation     | Splits the output (in this case, books) into individual items in preparation for the next step which is to verify the book against a known source to confirm the title and author. |
| Title validation   | HTTP Request           | Queries Google Books API for title verification | Item list split       | Data normalized      | Confirms each title against Google Books            |
| Data normalized    | Set                    | Normalizes API response data                 | Title validation      | Reaggregates list    | Normalizes data                                     |
| Reaggregates list  | Code                   | Deduplicates and aggregates verified books  | Data normalized       | Respond to Webhook   | Reaggregates book list and dedupes                  |
| Respond to Webhook | Respond to Webhook     | Sends final JSON response to frontend        | Reaggregates list     | —                    | Returns list back to frontend.                       |
| Sticky Note        | Sticky Note            | —                                           | —                     | —                    | Webhook connects to front end that passes on JSON with imageURL (string) |
| Sticky Note1       | Sticky Note            | —                                           | —                     | —                    | Input is normalized                                 |
| Sticky Note2       | Sticky Note            | —                                           | —                     | —                    | Image is analyzed and transformed.                  |
| Sticky Note3       | Sticky Note            | —                                           | —                     | —                    | Splits the output (in this case, books) into individual items in preparation for the next step which is to verify the book against a known source to confirm the title and author. |
| Sticky Note4       | Sticky Note            | —                                           | —                     | —                    | Confirms each title against Google Books            |
| Sticky Note5       | Sticky Note            | —                                           | —                     | —                    | Normalizes data                                     |
| Sticky Note6       | Sticky Note            | —                                           | —                     | —                    | Reaggregates book list and dedupes                  |
| Sticky Note7       | Sticky Note            | —                                           | —                     | —                    | Returns list back to frontend.                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use unique identifier (e.g., "365ea003-fe66-4211-ae03-69f1456d768e")  
   - Response Mode: Response Node  
   - Purpose: Receive JSON with image URL from frontend  

2. **Create Set Node "Input normalized"**  
   - Connect from Webhook node  
   - Add assignment:  
     - Field: `image` (string)  
     - Value (Expression):  
       ```
       {{$json.body?.imageUrl || $json.body?.image || $json.imageUrl || $json.image || ''}.trim()}
       ```  
   - Purpose: Normalize image URL input field  

3. **Create OpenAI Node "Analyze image"**  
   - Connect from "Input normalized"  
   - Resource: Image  
   - Operation: Analyze  
   - Model: GPT-4o-mini  
   - Text prompt (copy exactly):  
     ```
     You are a STRICT transformer. Analyze the image of book spines and return only clearly readable titles and authors. 
     Do NOT guess. If the author isn't clearly visible, set "author": null.
     Normalize capitalization. Deduplicate by title. 
     Output STRICT JSON only:
     {"books":[{"title":"string","author":"string|null"}]}
     ```  
   - Image URLs field: `={{$json.image}}`  
   - Credentials: Configure OpenAI API credentials with valid API key  

4. **Create Code Node "Item list split"**  
   - Connect from "Analyze image"  
   - Paste JS code to parse and split books array, prepare `searchAuthor` field:  
     ```js
     const items = await $input.all();
     const out = [];

     function stripCodeFence(s) {
       return String(s || '')
         .replace(/^```json\s*/i, '')
         .replace(/^```\s*/i, '')
         .replace(/```$/, '')
         .trim();
     }

     function firstAuthor(a) {
       if (!a) return null;
       const s = String(a);
       const parts = s.split(/\s*(?:,| and |&)\s*/i);
       return (parts[0] || '').trim() || null;
     }

     for (const item of items) {
       let books = null;

       if (Array.isArray(item.json?.books)) {
         books = item.json.books;
       }

       if (!books && typeof item.json?.content === 'string') {
         const cleaned = stripCodeFence(item.json.content);
         try {
           const parsed = JSON.parse(cleaned);
           if (Array.isArray(parsed.books)) books = parsed.books;
         } catch (e) {
           // ignore
         }
       }

       if (!books) continue;

       for (const b of books) {
         out.push({
           json: {
             title: b.title,
             author: b.author ?? null,
             searchAuthor: firstAuthor(b.author),
           },
         });
       }
     }

     return out;
     ```  
   - Purpose: Split array to individual book items, prepare for validation  

5. **Create HTTP Request Node "Title validation"**  
   - Connect from "Item list split"  
   - Method: GET  
   - URL: `https://www.googleapis.com/books/v1/volumes`  
   - Query Parameters:  
     - `q` (expression):  
       ```
       = 'intitle:"' + $json.title.replace(/"/g,'') + '"' + ($json.searchAuthor ? ' inauthor:"' + $json.searchAuthor.replace(/"/g,'') + '"' : '')
       ```  
     - `maxResults`: 5  
     - `printType`: books  
     - `orderBy`: relevance  
   - Response Format: JSON  
   - Purpose: Validate and enrich book info against Google Books  

6. **Create Set Node "Data normalized"**  
   - Connect from "Title validation"  
   - Assign fields:  
     - `title` (string):  
       ```
       = $json.items?.[0]?.volumeInfo?.title || $prevNode('Code').json.title
       ```  
     - `author` (string):  
       ```
       = $json.items?.[0]?.volumeInfo?.authors?.[0] || $prevNode('Code').json.author || null
       ```  
   - Purpose: Extract normalized title and author from API or fallback  

7. **Create Code Node "Reaggregates list"**  
   - Connect from "Data normalized"  
   - Paste JS code:  
     ```js
     const items = await $input.all();
     const seen = new Set();
     const books = [];

     for (const it of items) {
       const t = (it.json.title || '').toLowerCase().trim();
       if (t && !seen.has(t)) {
         seen.add(t);
         books.push({ title: it.json.title, author: it.json.author ?? null });
       }
     }

     return [{ json: { books } }];
     ```  
   - Purpose: Deduplicate and reaggregate final verified book list  

8. **Create Respond to Webhook Node**  
   - Connect from "Reaggregates list"  
   - Response Code: 200  
   - Response Headers: Content-Type: application/json  
   - Respond With: JSON  
   - Response Body: `={{$json}}`  
   - Purpose: Return final book list to caller  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                           |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| The prompt used in the AI node strictly forbids guessing, requiring only clearly visible text. | Ensures data quality and reduces false positives in extraction. |
| Google Books API query uses both title and first author to improve matching accuracy.          | [Google Books API Documentation](https://developers.google.com/books/docs/v1/getting_started) |
| The workflow uses deduplication logic both at AI extraction and final aggregation stages.      | Prevents duplicate book entries in the output list.         |
| This workflow requires valid OpenAI API credentials with access to GPT-4o-mini model.          | Setup needed in n8n credentials prior to execution.         |
| Webhook node path must be kept secret or protected to prevent unauthorized requests.           | Security best practice for HTTP endpoints.                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.