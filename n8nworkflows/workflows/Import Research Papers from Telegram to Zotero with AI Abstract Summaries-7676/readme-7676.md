Import Research Papers from Telegram to Zotero with AI Abstract Summaries

https://n8nworkflows.xyz/workflows/import-research-papers-from-telegram-to-zotero-with-ai-abstract-summaries-7676


# Import Research Papers from Telegram to Zotero with AI Abstract Summaries

### 1. Workflow Overview

This workflow automates the import of research papers into a Zotero library by processing DOI or arXiv links sent via Telegram. It parses messages for identifiers, fetches enriched metadata from multiple scholarly APIs (Crossref, DataCite, Unpaywall), selects the best available full-text PDF, creates corresponding Zotero items with attachments, generates a concise AI-based summary of the abstract, and sends feedback to the user through Telegram.

The workflow is organized into logical blocks:

- **1.1 Input Reception & DOI Extraction:** Receive Telegram messages and extract DOI/arXiv identifiers.
- **1.2 Zotero API Key Preparation:** Fetch and validate Zotero API credentials.
- **1.3 Metadata Retrieval:** Query Crossref, DataCite, and Unpaywall APIs for bibliographic and access data.
- **1.4 Metadata Merging & Processing:** Combine API responses, pick best PDF URL, construct Zotero item metadata.
- **1.5 Zotero Item Creation & PDF Attachment:** Conditionally create Zotero parent item and attach PDF if available.
- **1.6 Abstract Summarization & Telegram Feedback:** Use an LLM to summarize the abstract and send results back to the Telegram user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & DOI Extraction

- **Overview:**  
  This block triggers the workflow on receiving a Telegram message, and extracts the DOI or arXiv identifier from the message text.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Code (Parse DOI)

- **Node Details:**  

  - **Telegram Trigger**  
    - *Type:* Telegram Trigger node  
    - *Role:* Listens for incoming Telegram messages (specifically message updates) to start the workflow.  
    - *Config:* Listens for "message" updates, no extra fields.  
    - *Input:* Telegram message event.  
    - *Output:* JSON with message details.  
    - *Failure Modes:* Bot token misconfiguration, webhook setup issues, Telegram API downtime.

  - **Code (Parse DOI)**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses message text to extract DOI or arXiv ID using regex patterns.  
    - *Config:*  
      - Extracts DOI from various URL patterns: doi.org links, raw DOI strings, arXiv URLs.  
      - Returns error JSON if no DOI found to stop or handle gracefully downstream.  
    - *Input:* Message text from Telegram Trigger.  
    - *Output:* JSON object with extracted `doi` or error message.  
    - *Failure Modes:* Message without valid DOI/arXiv link, regex mismatch, malformed message text.

---

#### 2.2 Zotero API Key Preparation

- **Overview:**  
  Stores the Zotero API key from workflow credentials and performs a simple API request to validate the key and retrieve user info.

- **Nodes Involved:**  
  - Edit Fields (Set Zotero API Key)  
  - HTTP Request (Validate Zotero Key)

- **Node Details:**  

  - **Edit Fields**  
    - *Type:* Set node  
    - *Role:* Assigns the Zotero API key from n8n credentials to workflow data for reuse.  
    - *Config:* Uses expression to fetch `zoteroApi.key` credential.  
    - *Input:* DOI parsing output.  
    - *Output:* JSON including `Zotero API Key`.  
    - *Failure Modes:* Missing credentials, incorrect API key.

  - **HTTP Request (Validate Zotero Key)**  
    - *Type:* HTTP Request node  
    - *Role:* Sends a GET request to Zotero API keys endpoint to verify the API key and retrieve user info.  
    - *Config:*  
      - URL: `https://api.zotero.org/keys/{{ $json['Zotero API Key'] }}`  
      - Header: `Zotero-API-Key` with the key value.  
    - *Input:* JSON with Zotero API key.  
    - *Output:* User info including userID and key validation.  
    - *Failure Modes:* Invalid API key, network errors, rate limits.

---

#### 2.3 Metadata Retrieval

- **Overview:**  
  Fetches detailed bibliographic and access metadata from Crossref, DataCite, and Unpaywall APIs using the extracted DOI.

- **Nodes Involved:**  
  - HTTP Request3 (Crossref API)  
  - HTTP Request2 (DataCite API)  
  - HTTP Request4 (Unpaywall API)  
  - Edit Fields1 (Set Crossref)  
  - Edit Fields2 (Set DataCite)  
  - Edit Fields3 (Set Unpaywall)

- **Node Details:**  

  - **HTTP Request3 (Crossref API)**  
    - *Type:* HTTP Request  
    - *Role:* Requests Crossref metadata for the DOI.  
    - *Config:* URL: `https://api.crossref.org/works/{{ doi }}`  
    - *Input:* DOI from Code node.  
    - *Output:* Crossref JSON response or error.  
    - *Failure Modes:* DOI not found, API downtime, rate limiting.

  - **HTTP Request2 (DataCite API)**  
    - *Type:* HTTP Request  
    - *Role:* Requests DataCite metadata for the DOI.  
    - *Config:* URL: `https://api.datacite.org/dois/{{ doi }}`  
    - *Input:* DOI.  
    - *Output:* DataCite JSON response or error.  
    - *Failure Modes:* Same as Crossref.

  - **HTTP Request4 (Unpaywall API)**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves open access and PDF availability data.  
    - *Config:* URL: `https://api.unpaywall.org/v2/{{ doi }}?email=your-email@example.com` (email parameter recommended)  
    - *Input:* DOI.  
    - *Output:* Unpaywall data JSON or error.  
    - *Failure Modes:* API quota limits, missing email param, DOI not found.

  - **Edit Fields1, Edit Fields2, Edit Fields3**  
    - *Type:* Set nodes  
    - *Role:* Store each API response in separate JSON fields (`crossref`, `datacite`, `unpay`) for downstream merging.  
    - *Input:* API responses.  
    - *Output:* JSON enriched with respective data fields.  
    - *Failure Modes:* None critical; node executes regardless of API errors.

---

#### 2.4 Metadata Merging & Processing

- **Overview:**  
  Merges all API metadata and constructs a comprehensive Zotero item metadata object. Selects the best PDF URL by priority. Determines if the item is a journal article or preprint.

- **Nodes Involved:**  
  - Merge  
  - Code4 (Build Zotero metadata + pick PDF)  
  - If3 (Check PDF availability)

- **Node Details:**  

  - **Merge**  
    - *Type:* Merge node  
    - *Role:* Combines Crossref, DataCite, and Unpaywall data streams into a single combined JSON object by position.  
    - *Config:* Combine mode, combining three inputs by position.  
    - *Input:* Edit Fields1, Edit Fields2, Edit Fields3 outputs.  
    - *Output:* Single merged JSON with `crossref`, `datacite`, `unpay`, and `doi`.  
    - *Failure Modes:* Missing inputs if any API failed.

  - **Code4 (Build Zotero metadata + pick PDF)**  
    - *Type:* Code node (JavaScript)  
    - *Role:*  
      - Extracts metadata fields such as title, authors, publication date, abstract, tags from merged data.  
      - Picks the best available PDF URL from Crossref, Unpaywall, DataCite, or arXiv fallback.  
      - Builds Zotero metadata object with proper fields including creators, tags, itemType.  
    - *Key Logic:*  
      - PDF priority: Crossref → Unpaywall → DataCite → arXiv.  
      - Handles multiple date formats, author name formats.  
      - Adds `"from-telegram"` tag.  
    - *Input:* Merged API JSON.  
    - *Output:* Object with `doi`, `title`, `meta` (Zotero item metadata), `pdfUrl`, `hasPdf` boolean.  
    - *Failure Modes:* Missing or malformed API data, null fields, empty abstracts.

  - **If3 (Check PDF availability)**  
    - *Type:* If node  
    - *Role:* Checks if a PDF URL is available (boolean flag `hasPdf`).  
    - *Input:* Output from Code4.  
    - *Output:* Routes workflow either to create the Zotero parent item with PDF attachment or to notify user PDF not found.  
    - *Failure Modes:* Unset or invalid boolean flags.

---

#### 2.5 Zotero Item Creation & PDF Attachment

- **Overview:**  
  Creates the Zotero item with metadata and conditionally attaches the linked PDF as an attachment item.

- **Nodes Involved:**  
  - HTTP Request9 (Create Zotero Parent Item)  
  - HTTP Request1 (Create Zotero Attachment Item)

- **Node Details:**  

  - **HTTP Request9 (Create Zotero Parent Item)**  
    - *Type:* HTTP Request  
    - *Role:* Sends POST request to Zotero API to create a bibliographic parent item with metadata.  
    - *Config:*  
      - URL: `https://api.zotero.org/users/{{ userID }}/items`  
      - Method: POST  
      - Headers: `ZOTERO-API-KEY`, `Content-Type: application/json`  
      - Body: JSON array containing the metadata object from Code4.  
    - *Input:* Metadata from Code4, userID and key from earlier Zotero key validation.  
    - *Output:* Zotero API response with created item key.  
    - *Failure Modes:* Invalid API key, malformed metadata, network errors.

  - **HTTP Request1 (Create Zotero Attachment Item)**  
    - *Type:* HTTP Request  
    - *Role:* Creates a linked_url attachment item referencing the full-text PDF under the parent item.  
    - *Config:*  
      - URL: `https://api.zotero.org/users/{{ userID }}/items`  
      - Method: POST  
      - Headers: `Zotero-API-Key`, `Content-Type: application/json`  
      - Body: JSON array with attachment item linking to the PDF URL using the parent item's `key`.  
    - *Input:* Parent item key from HTTP Request9, PDF URL from Code4 via If3.  
    - *Output:* Zotero API response for attachment item creation.  
    - *Failure Modes:* Missing parent key, invalid PDF URL, API errors.

---

#### 2.6 Abstract Summarization & Telegram Feedback

- **Overview:**  
  Summarizes the research paper abstract with a language model and sends a formatted message including title, URL, and summary back to the user on Telegram.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenRouter Chat Model (language model provider)  
  - Send a text message (summary)  
  - Send a text message1 (PDF not found notification)

- **Node Details:**  

  - **Basic LLM Chain**  
    - *Type:* LangChain LLM Chain node  
    - *Role:* Generates a concise summary of the abstract text.  
    - *Config:*  
      - Prompt instructs to generate a short 2-3 sentence plain English summary highlighting main contributions.  
      - Input abstract fetched from Zotero created item metadata.  
      - Uses OpenRouter Chat Model as language model backend.  
    - *Input:* Abstract from Zotero item.  
    - *Output:* Summary text.  
    - *Failure Modes:* API limits, empty or missing abstract, LLM errors.

  - **OpenRouter Chat Model**  
    - *Type:* LangChain language model node  
    - *Role:* Provides AI model interface with Google Gemini 2.0 Flash (free).  
    - *Config:* Model "google/gemini-2.0-flash-exp:free".  
    - *Input:* Prompt from Basic LLM Chain.  
    - *Output:* AI-generated text.  
    - *Failure Modes:* API quota, network issues.

  - **Send a text message**  
    - *Type:* Telegram node  
    - *Role:* Sends the formatted summary message with title, URL, and abstract summary to Telegram user.  
    - *Config:*  
      - Chat ID from Telegram Trigger.  
      - Message text includes paper title, URL, and LLM-generated summary.  
      - Attribution disabled.  
    - *Input:* Summary from Basic LLM Chain, created Zotero item URL.  
    - *Output:* Message sent confirmation.  
    - *Failure Modes:* Invalid chat ID, Telegram API limits.

  - **Send a text message1 (PDF not found notification)**  
    - *Type:* Telegram node  
    - *Role:* Sends a simple "PDF not found" message if no full-text PDF was located.  
    - *Config:* Same chat ID, simple text.  
    - *Failure Modes:* Same as above.

---

### 3. Summary Table

| Node Name           | Node Type                     | Functional Role                                  | Input Node(s)                     | Output Node(s)                 | Sticky Note                                                                                                         |
|---------------------|-------------------------------|-------------------------------------------------|----------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger               | Starts workflow on Telegram message              | -                                | Code                          |                                                                                                                     |
| Code                | Code                          | Parse DOI/arXiv identifier from Telegram message | Telegram Trigger                 | Edit Fields                   |                                                                                                                     |
| Edit Fields         | Set                           | Store Zotero API Key from credentials             | Code                            | HTTP Request                  | - Stores Zotero API Key for later steps; validates API key with HTTP request.                                        |
| HTTP Request        | HTTP Request                  | Validate Zotero API key, fetch user info          | Edit Fields                    | HTTP Request3, HTTP Request2, HTTP Request4 |                                                                                                                     |
| HTTP Request3       | HTTP Request                  | Fetch Crossref metadata                            | HTTP Request                   | Edit Fields1                  | - Fetch bibliographic data from Crossref using DOI.                                                                 |
| HTTP Request2       | HTTP Request                  | Fetch DataCite metadata                            | HTTP Request                   | Edit Fields2                  | - Retrieve additional metadata from DataCite.                                                                       |
| HTTP Request4       | HTTP Request                  | Fetch Unpaywall open access info                   | HTTP Request                   | Edit Fields3                  | - Check open-access availability and full-text links.                                                               |
| Edit Fields1        | Set                           | Store Crossref response                            | HTTP Request3                  | Merge                        | - Store API responses separately for merging.                                                                       |
| Edit Fields2        | Set                           | Store DataCite response                            | HTTP Request2                  | Merge                        |                                                                                                                     |
| Edit Fields3        | Set                           | Store Unpaywall response                           | HTTP Request4                  | Merge                        |                                                                                                                     |
| Merge               | Merge                         | Combine API responses                              | Edit Fields1, Edit Fields2, Edit Fields3 | Code4                        | - Combine metadata from Crossref, DataCite, and Unpaywall into one object.                                           |
| Code4               | Code                          | Build Zotero metadata and select PDF URL          | Merge                         | If3                         | - Prioritize PDF links; build Zotero item metadata; identify item type.                                              |
| If3                 | If                            | Check if PDF URL is available                      | Code4                         | HTTP Request9, Send a text message1 | - Branch: create Zotero item with or without PDF attachment; notify user if PDF missing.                              |
| HTTP Request9       | HTTP Request                  | Create Zotero parent item                           | If3 (true branch)             | HTTP Request1                | - Create bibliographic Zotero item with metadata.                                                                    |
| HTTP Request1       | HTTP Request                  | Create Zotero attachment item (linked PDF)         | HTTP Request9                 | Basic LLM Chain              | - Attach PDF URL as linked attachment to Zotero item.                                                                |
| Basic LLM Chain     | LangChain LLM Chain           | Summarize abstract using LLM                        | HTTP Request1                 | Send a text message          | - Use LLM to generate concise abstract summary.                                                                       |
| OpenRouter Chat Model | LangChain Language Model      | Provides AI model for summarization                 | Basic LLM Chain (ai_languageModel) | Basic LLM Chain              |                                                                                                                     |
| Send a text message  | Telegram                      | Send paper title, URL, and summary back to user    | Basic LLM Chain               | -                           | - Deliver summary and metadata back to Telegram user.                                                                |
| Send a text message1 | Telegram                      | Notify user PDF not found                           | If3 (false branch)            | -                           | - Alert user when no PDF could be found.                                                                              |
| Sticky Note          | Sticky Note                   | Workflow overview and description                   | -                            | -                           | Contains detailed workflow purpose, features, credentials needed, and benefits.                                       |
| Sticky Note1         | Sticky Note                   | Early step notes on Telegram trigger and Zotero key | -                            | -                           | Describes initial trigger and Zotero API key validation steps.                                                        |
| Sticky Note2         | Sticky Note                   | Notes on metadata API requests                      | -                            | -                           | Explains the metadata retrieval from Crossref, DataCite, and Unpaywall.                                               |
| Sticky Note3         | Sticky Note                   | Notes on metadata merging and PDF selection         | -                            | -                           | Describes combining metadata and selecting the PDF URL.                                                              |
| Sticky Note4         | Sticky Note                   | Notes on Zotero item creation and PDF attachment    | -                            | -                           | Explains conditional creation of Zotero items and PDF handling.                                                      |
| Sticky Note5         | Sticky Note                   | Notes on abstract summarization and Telegram reply  | -                            | -                           | Covers LLM summarization and user feedback messaging.                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates.  
   - Purpose: Start workflow when a user sends a message to your Telegram bot.

2. **Add Code node "Parse DOI"**  
   - Type: Code  
   - JavaScript: Extract DOI or arXiv ID from incoming message text using regex.  
   - Output JSON with `doi` or error message.  
   - Connect Telegram Trigger → Code.

3. **Add Set node "Edit Fields" to assign Zotero API Key**  
   - Type: Set  
   - Assign field `Zotero API Key` with expression: `{{$credentials.zoteroApi.key}}`  
   - Connect Code → Edit Fields.

4. **Add HTTP Request node "Validate Zotero Key"**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.zotero.org/keys/{{ $json["Zotero API Key"] }}`  
   - Header: `Zotero-API-Key` with the key value.  
   - Connect Edit Fields → HTTP Request.

5. **Add three HTTP Request nodes to fetch metadata:**  
   - **Crossref:**  
     - URL: `https://api.crossref.org/works/{{ $('Code').item.json.doi }}`  
     - Connect HTTP Request → HTTP Request3.  
   - **DataCite:**  
     - URL: `https://api.datacite.org/dois/{{ $('Code').item.json.doi }}`  
     - Connect HTTP Request → HTTP Request2.  
   - **Unpaywall:**  
     - URL: `https://api.unpaywall.org/v2/{{ $('Code').item.json.doi }}?email=your-email@example.com`  
     - Connect HTTP Request → HTTP Request4.

6. **Add three Set nodes to store API responses:**  
   - Set `crossref` field with Crossref response, connect HTTP Request3 → Edit Fields1.  
   - Set `datacite` field with DataCite response, connect HTTP Request2 → Edit Fields2.  
   - Set `unpay` field with Unpaywall response, connect HTTP Request4 → Edit Fields3.

7. **Add Merge node**  
   - Combine the three sets by position into one JSON object.  
   - Connect Edit Fields1, Edit Fields2, Edit Fields3 → Merge.

8. **Add Code node "Build Zotero metadata + pick PDF"**  
   - Use JavaScript to:  
     - Extract and unify metadata fields (title, authors, abstract, date).  
     - Select best PDF URL by priority: Crossref → Unpaywall → DataCite → arXiv.  
     - Construct Zotero item metadata object.  
     - Output fields: `doi`, `title`, `meta`, `pdfUrl`, `hasPdf` (boolean).  
   - Connect Merge → Code4.

9. **Add If node "Check PDF availability"**  
   - Condition: `hasPdf` is true.  
   - Connect Code4 → If3.

10. **Add HTTP Request node "Create Zotero Parent Item"**  
    - Method: POST  
    - URL: `https://api.zotero.org/users/{{ $('HTTP Request').item.json.userID }}/items`  
    - Headers: `ZOTERO-API-KEY` from earlier HTTP Request node user key, `Content-Type: application/json`  
    - Body: JSON array containing `meta` object from Code4.  
    - Connect If3 (true) → HTTP Request9.

11. **Add HTTP Request node "Create Zotero Attachment Item"**  
    - Method: POST  
    - URL: same as above  
    - Headers: as above  
    - Body: JSON array with an attachment item linking to `pdfUrl`, parent item key from HTTP Request9 response, type `linked_url`, contentType `application/pdf`.  
    - Connect HTTP Request9 → HTTP Request1.

12. **Add LangChain OpenRouter Chat Model node**  
    - Model: `google/gemini-2.0-flash-exp:free`  
    - Connect HTTP Request1 (as ai_languageModel) → Basic LLM Chain.

13. **Add LangChain Basic LLM Chain node**  
    - Prompt: Summarize the abstract text (fetched from the created Zotero item).  
    - Configure clear instructions for concise, plain English summary.  
    - Connect OpenRouter Chat Model → Basic LLM Chain.

14. **Add Telegram node "Send a text message"**  
    - Text: formatted message with paper title, URL, and summary from Basic LLM Chain.  
    - Chat ID: from Telegram Trigger.  
    - Connect Basic LLM Chain → Send a text message.

15. **Add Telegram node "Send a text message1"**  
    - Text: "PDF not found"  
    - Chat ID: from Telegram Trigger.  
    - Connect If3 (false) → Send a text message1.

16. **Set up required credentials in n8n:**  
    - Telegram API (bot token)  
    - Zotero API Key (in credentials, named `zoteroApi`)  
    - OpenRouter API Key (for LangChain OpenRouter node)  
    - Optional: Email for Unpaywall API requests.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                              | Context or Link                                                                                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| This workflow allows automatic import of research papers into Zotero by sending DOI/arXiv links via Telegram. It aggregates metadata from multiple sources and attaches PDFs when possible. Summaries are generated with a language model. | Workflow purpose and summary from main sticky note.                                                                |
| Required credentials include Telegram API token, Zotero API key, and OpenRouter API key for AI summarization. An email parameter is recommended for Unpaywall API stability.                                                                | Credential requirements from sticky note.                                                                          |
| The PDF selection prioritizes Crossref's direct links, then Unpaywall open access PDFs, then DataCite identifiers, and finally arXiv fallback based on DOI pattern.                                                                         | Metadata merging and PDF selection approach from sticky note.                                                      |
| The Zotero item type is heuristically determined: items with a publication title are classified as "journalArticle", others as "preprint".                                                                                                | Code4 logic for item type determination.                                                                            |
| The LLM prompt encourages clear, concise summaries avoiding jargon, ideal for quick comprehension in academic contexts.                                                                                                                  | Prompt details in Basic LLM Chain node.                                                                              |
| For best results, ensure the Telegram bot is properly set up with webhook URLs configured and the Zotero API key has write permissions to your Zotero library.                                                                            | Operational notes implicit in node configurations.                                                                  |
| Links to official API docs: Crossref (https://api.crossref.org), DataCite (https://api.datacite.org), Unpaywall (https://unpaywall.org/products/api), Zotero API (https://www.zotero.org/support/dev/web_api/v3/start)                      | Useful references for extending or debugging the workflow.                                                          |

---

This document fully describes the workflow "Import Research Papers from Telegram to Zotero with AI Abstract Summaries," enabling expert users and AI agents to understand, reproduce, and extend the solution effectively.