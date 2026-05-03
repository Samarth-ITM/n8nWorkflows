🧑‍🦯Improve your Website Accessibility with GPT-4o and Google Sheet

https://n8nworkflows.xyz/workflows/-----improve-your-website-accessibility-with-gpt-4o-and-google-sheet-3640


# 🧑‍🦯Improve your Website Accessibility with GPT-4o and Google Sheet

### 1. Workflow Overview

This workflow automates the process of improving website accessibility by auditing image alternative texts (alt texts) on a specified web page and enhancing them using AI. It targets bloggers, SEO professionals, web developers, and product teams who want to ensure their websites comply with accessibility standards (WCAG) and improve SEO performance.

The workflow is logically divided into two main blocks:

- **1.1 Audit and Extraction Block:**  
  Downloads the HTML content of a target web page, extracts all `<img>` tags with their `src` and `alt` attributes, calculates alt text length, and stores this data into a Google Sheet for record-keeping and further processing.

- **1.2 AI Enhancement Block:**  
  Filters images with missing or short alt texts (length < 50 characters), limits the batch size to 5 images at a time, sends their URLs to GPT-4o (an OpenAI model with vision capabilities) to generate improved alt texts, and updates the Google Sheet with these AI-generated descriptions.

---

### 2. Block-by-Block Analysis

#### 2.1 Audit and Extraction Block

- **Overview:**  
  This block initiates the workflow by setting the target page URL, downloading its HTML content, parsing all image tags to extract `src` and `alt` attributes, calculating alt text length, and appending these details into a Google Sheet for tracking.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Page Link (Set)  
  - Download HTML (HTTP Request)  
  - Get Images urls with altText (Code)  
  - Store Results (Google Sheets)  
  - Download Results (Google Sheets)  
  - altLength < 50 (IF)  

- **Node Details:**

  1. **When clicking ‘Test workflow’**  
     - *Type:* Manual Trigger  
     - *Role:* Entry point to manually start the workflow.  
     - *Configuration:* No parameters; triggers workflow execution on demand.  
     - *Connections:* Outputs to `Page Link` and `Download Results`.  
     - *Edge Cases:* None specific; user-triggered.

  2. **Page Link**  
     - *Type:* Set  
     - *Role:* Defines the target URL (`url`) of the web page to audit and the `baseUrl` for resolving relative image URLs.  
     - *Configuration:*  
       - `url`: "https://www.samirsaci.com/sustainable-business-strategy-with-data-analytics/"  
       - `baseUrl`: "https://www.samirsaci.com"  
     - *Connections:* Input from Manual Trigger; output to `Download HTML`.  
     - *Edge Cases:* Incorrect URL format or unreachable URL will cause downstream HTTP request failure.

  3. **Download HTML**  
     - *Type:* HTTP Request  
     - *Role:* Downloads the raw HTML content of the specified page URL.  
     - *Configuration:* Uses the `url` from `Page Link` node. No special headers or authentication.  
     - *Connections:* Input from `Page Link`; output to `Get Images urls with altText`.  
     - *Edge Cases:* Network errors, 404 or 500 HTTP responses, or pages requiring authentication may cause failure.

  4. **Get Images urls with altText**  
     - *Type:* Code (JavaScript)  
     - *Role:* Parses the downloaded HTML to extract all `<img>` tags, their `src` and `alt` attributes, normalizes relative URLs using `baseUrl`, and calculates the length of alt texts.  
     - *Configuration:*  
       - Uses regex to find `<img>` tags and extract `alt` and `src`.  
       - Handles relative `src` URLs by concatenating with `baseUrl`.  
       - Defaults missing `alt` to "[No alt text]" and missing `src` to "[No src]".  
       - Outputs an array of objects with `index`, `src`, `alt`, and `altLength`.  
     - *Connections:* Input from `Download HTML`; output to `Store Results`.  
     - *Edge Cases:*  
       - Malformed HTML may cause regex to miss images.  
       - Images without `src` or `alt` handled with placeholders.  
       - Relative URL resolution assumes consistent base URL format.

  5. **Store Results**  
     - *Type:* Google Sheets (Append)  
     - *Role:* Appends extracted image data (index, page URL, src, alt, altLength) into a Google Sheet for audit records.  
     - *Configuration:*  
       - Requires Google Sheets API credentials.  
       - Document and sheet selected via list or ID.  
       - Maps fields: `index`, `page`, `src`, `alt`, `altLength`, and leaves `newAlt` empty initially.  
     - *Connections:* Input from `Get Images urls with altText`; output to `Download Results`.  
     - *Edge Cases:*  
       - Authentication errors with Google Sheets API.  
       - Sheet or document not found or access denied.  
       - Rate limits on Google Sheets API.

  6. **Download Results**  
     - *Type:* Google Sheets (Read)  
     - *Role:* Reads all existing records from the Google Sheet to prepare for filtering images needing alt text improvement.  
     - *Configuration:*  
       - Uses same Google Sheets credentials and document/sheet as `Store Results`.  
     - *Connections:* Input from Manual Trigger; output to `altLength < 50`.  
     - *Edge Cases:* Same as `Store Results`.

  7. **altLength < 50**  
     - *Type:* IF  
     - *Role:* Filters images where the alt text length is less than 100 characters (note: condition uses `< 100` but logically targets short alt texts).  
     - *Configuration:*  
       - Condition: `$json.altLength < 100` (number comparison).  
     - *Connections:* Input from `Download Results`; output to `Limit records` if true.  
     - *Edge Cases:*  
       - If no records meet condition, downstream nodes receive no data.

---

#### 2.2 AI Enhancement Block

- **Overview:**  
  This block processes images with short or missing alt texts by sending their URLs to GPT-4o to generate improved alternative texts. It limits the batch size to 5 images per run, updates the Google Sheet with the new alt texts, and loops over all items.

- **Nodes Involved:**  
  - Limit records (Limit)  
  - Loop Over Items (SplitInBatches)  
  - Generate altText (OpenAI GPT-4o)  
  - Update Results (Google Sheets)  

- **Node Details:**

  1. **Limit records**  
     - *Type:* Limit  
     - *Role:* Restricts the number of images processed per run to 5 to avoid API overload or rate limits.  
     - *Configuration:* `maxItems` set to 5.  
     - *Connections:* Input from `altLength < 50`; output to `Loop Over Items`.  
     - *Edge Cases:* If fewer than 5 images, processes all available.

  2. **Loop Over Items**  
     - *Type:* SplitInBatches  
     - *Role:* Iterates over each image record individually to process them one by one.  
     - *Configuration:* Default batch size (1 item per batch).  
     - *Connections:* Input from `Limit records`; two outputs:  
       - Output 1 (empty) - unused  
       - Output 2 to `Generate altText`.  
     - *Edge Cases:* Empty input results in no iterations.

  3. **Generate altText**  
     - *Type:* OpenAI (LangChain)  
     - *Role:* Sends the image URL to GPT-4o to generate an alternative text description under 150 characters.  
     - *Configuration:*  
       - Model: `gpt-4o-2024-05-13` (GPT-4o with vision)  
       - Resource: `image`  
       - Operation: `analyze`  
       - Input: `imageUrls` set dynamically from the current item's `src` field.  
       - Prompt: "Please generate the alternative text (alt text) for this image under 150 characters."  
       - Max tokens: 150  
     - *Connections:* Input from `Loop Over Items`; output to `Update Results`.  
     - *Edge Cases:*  
       - API authentication or quota errors.  
       - Image URL inaccessible or invalid.  
       - Model response latency or timeout.  
       - Unexpected or empty AI output.

  4. **Update Results**  
     - *Type:* Google Sheets (Update)  
     - *Role:* Updates the Google Sheet row corresponding to the image index with the newly generated `newAlt` text.  
     - *Configuration:*  
       - Uses `index` as matching column to find the correct row.  
       - Updates fields: `page` (empty string), `index`, and `newAlt` (AI-generated alt text).  
       - Document and sheet same as previous Google Sheets nodes.  
     - *Connections:* Input from `Generate altText`; output loops back to `Loop Over Items` (for next batch).  
     - *Edge Cases:*  
       - Matching row not found or multiple matches.  
       - Google Sheets API errors or rate limits.

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                                   | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                      |
|-------------------------|----------------------------|-------------------------------------------------|-----------------------------|----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger             | Entry point to start the workflow manually       |                             | Page Link, Download Results |                                                                                                |
| Page Link               | Set                        | Defines target page URL and base URL              | When clicking ‘Test workflow’ | Download HTML              |                                                                                                |
| Download HTML           | HTTP Request               | Downloads HTML content of the target page         | Page Link                   | Get Images urls with altText |                                                                                                |
| Get Images urls with altText | Code                       | Extracts image src and alt attributes from HTML   | Download HTML               | Store Results              |                                                                                                |
| Store Results           | Google Sheets (Append)     | Saves extracted image data into Google Sheet      | Get Images urls with altText | Download Results           |                                                                                                |
| Download Results        | Google Sheets (Read)       | Reads existing image data from Google Sheet       | When clicking ‘Test workflow’ | altLength < 50             |                                                                                                |
| altLength < 50          | IF                         | Filters images with alt text length less than 100 | Download Results            | Limit records              |                                                                                                |
| Limit records           | Limit                      | Limits processing to 5 images per run             | altLength < 50              | Loop Over Items            |                                                                                                |
| Loop Over Items         | SplitInBatches             | Iterates over images one by one                    | Limit records, Update Results | Generate altText, (empty)  |                                                                                                |
| Generate altText        | OpenAI (LangChain)         | Generates improved alt text using GPT-4o          | Loop Over Items             | Update Results             |                                                                                                |
| Update Results          | Google Sheets (Update)     | Updates Google Sheet with AI-generated alt text   | Generate altText            | Loop Over Items            |                                                                                                |
| Sticky Note1            | Sticky Note                | Explains first block: audit and extract images    |                             |                            | ### 1. First Block: audit the page to extract all the images with their respective alternative text ... [Learn more about the Google Sheet Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Sticky Note             | Sticky Note                | Explains second block: generate alt text for images with altLength < 50 |                             |                            | ### 2. SecondBlock: generate alternative text for the image with altLength < 50 ... [Learn more about the Google Sheet Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Sticky Note4            | Sticky Note                | Provides tutorial video link                       |                             |                            | ![Tutorial](https://www.samirsaci.com/content/images/2025/04/temp-8.png) [🎥 Check My Tutorial](https://www.youtube.com/watch?v=LwTIro6Rapk) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named `When clicking ‘Test workflow’` to start the workflow manually.

2. **Add a Set node** named `Page Link`:  
   - Add two string fields:  
     - `url`: Set to the target page URL, e.g., `https://www.samirsaci.com/sustainable-business-strategy-with-data-analytics/`  
     - `baseUrl`: Set to the base URL for relative image paths, e.g., `https://www.samirsaci.com`  
   - Connect output from `When clicking ‘Test workflow’` to this node.

3. **Add an HTTP Request node** named `Download HTML`:  
   - Set the URL to `={{ $json.url }}` to use the URL from the `Page Link` node.  
   - No authentication or special headers needed.  
   - Connect output from `Page Link` to this node.

4. **Add a Code node** named `Get Images urls with altText`:  
   - Use JavaScript code to parse the HTML from the previous node:  
     - Extract all `<img>` tags.  
     - Extract `src` and `alt` attributes using regex.  
     - Normalize relative `src` URLs using `baseUrl`.  
     - Set default alt text to "[No alt text]" if missing.  
     - Calculate `altLength` as the length of the alt text string.  
     - Output an array of objects with fields: `index`, `src`, `alt`, `altLength`.  
   - Connect output from `Download HTML` to this node.

5. **Add a Google Sheets node** named `Store Results`:  
   - Operation: Append  
   - Configure Google Sheets credentials with access to your target spreadsheet.  
   - Select the document and sheet (e.g., `gid=0`).  
   - Map columns: `index`, `page` (from `Page Link.url`), `src`, `alt`, `altLength`, and leave `newAlt` empty.  
   - Connect output from `Get Images urls with altText` to this node.

6. **Add a Google Sheets node** named `Download Results`:  
   - Operation: Read (get all rows)  
   - Use the same Google Sheets credentials, document, and sheet as `Store Results`.  
   - Connect output from `When clicking ‘Test workflow’` to this node.

7. **Add an IF node** named `altLength < 50`:  
   - Condition: Numeric comparison  
   - Expression: `{{$json.altLength}} < 100` (to filter images with alt text shorter than 100 characters)  
   - Connect output from `Download Results` to this node.

8. **Add a Limit node** named `Limit records`:  
   - Set `maxItems` to 5 to process a maximum of 5 images per run.  
   - Connect the `true` output from `altLength < 50` to this node.

9. **Add a SplitInBatches node** named `Loop Over Items`:  
   - Default batch size (1) to process images one by one.  
   - Connect output from `Limit records` to this node.

10. **Add an OpenAI node** (LangChain) named `Generate altText`:  
    - Set resource to `image` and operation to `analyze`.  
    - Model: Select `gpt-4o-2024-05-13` or the latest GPT-4o model with vision.  
    - Text prompt: "Please generate the alternative text (alt text) for this image under 150 characters."  
    - Image URLs: Set to `={{ $json.src }}` from the current batch item.  
    - Max tokens: 150  
    - Connect output from `Loop Over Items` (second output) to this node.

11. **Add a Google Sheets node** named `Update Results`:  
    - Operation: Update  
    - Use the same Google Sheets credentials, document, and sheet.  
    - Matching column: `index` (to update the correct row).  
    - Map fields:  
      - `index`: `={{ $('Loop Over Items').item.json.index }}`  
      - `newAlt`: `={{ $json.content }}` (AI-generated alt text)  
      - `page`: empty string or keep existing  
    - Connect output from `Generate altText` to this node.

12. **Connect the output of `Update Results` back to the first output of `Loop Over Items`** to continue processing the next batch item until all are processed.

13. **Add Sticky Notes** for documentation and guidance:  
    - Add one explaining the audit block setup and Google Sheets configuration.  
    - Add one explaining the AI enhancement block and usage notes.  
    - Add one with a tutorial video link:  
      ![Tutorial](https://www.samirsaci.com/content/images/2025/04/temp-8.png)  
      [🎥 Check My Tutorial](https://www.youtube.com/watch?v=LwTIro6Rapk)

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow built using n8n version 1.85.4, submitted April 21, 2025                                    | Version info                                                                                     |
| GPT-generated alt texts are limited to ~125–150 characters for best results                          | Best practice for alt text length                                                               |
| Use this workflow to comply with WCAG and improve Google indexing                                    | Accessibility and SEO benefits                                                                  |
| Easily adaptable to audit multiple domains or e-commerce catalogs                                   | Scalability note                                                                                |
| Tutorial video available for detailed setup and usage                                                | [YouTube Tutorial](https://www.youtube.com/watch?v=LwTIro6Rapk)                                 |
| Workflow author: Samir Saci, Supply Chain Engineer and Data Scientist, founder of LogiGreen Consulting | [LogiGreen Consulting](https://logi-green.com), [LinkedIn](https://www.linkedin.com/in/samir-saci) |

---

This documentation provides a complete and detailed understanding of the workflow’s structure, logic, and configuration, enabling advanced users and AI agents to reproduce, modify, or extend it confidently.