Build Gmail Contact Database with GPT-5 Nano, Brave Search & Google Sheets

https://n8nworkflows.xyz/workflows/build-gmail-contact-database-with-gpt-5-nano--brave-search---google-sheets-10778


# Build Gmail Contact Database with GPT-5 Nano, Brave Search & Google Sheets

---

### 1. Workflow Overview

This workflow is designed to build an enriched Gmail contact database by extracting and aggregating contact information and social media profiles from Gmail sent emails, supplemented by web searches when email history is unavailable. It targets users who want to automate contact data collection and enrichment for CRM or outreach purposes.

The workflow operates in two main logical paths:

- **1.1 Email History Extraction & AI Processing:**  
  It fetches all sent emails, extracts contacts from email headers, searches Gmail inbox for existing conversation threads, and uses GPT-5 Nano to analyze email threads for detailed contact info like phone numbers, websites, and social media profiles.

- **1.2 Web Search & HTML Scraping:**  
  For contacts without email history, it extracts the domain from their email address, validates it (excluding generic domains like gmail.com), performs a Brave Search to find relevant contact pages on their website, and scrapes HTML content to extract social media and contact links.

Both paths converge to merge data and write enriched contact records into a Google Sheets database.

Logical blocks include:

- **1.1 Input Reception and Sent Emails Loading**  
- **1.2 Contact Extraction and Deduplication**  
- **1.3 Email Thread Retrieval and AI-based Information Extraction**  
- **1.4 Domain Validation and Brave Web Search**  
- **1.5 HTML Parsing for Social Profiles**  
- **1.6 Data Aggregation and Writing to Google Sheets**

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Sent Emails Loading

**Overview:**  
This block triggers the workflow manually and loads all "Sent" emails from Gmail to start processing contacts from outgoing messages.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (manual trigger)  
- LoadSentMessages (Gmail node)  
- SelectTo&CC (Set node)  
- Remove Duplicates (RemoveDuplicates node)  
- Loop Over Items (SplitInBatches node)  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command  
  - Config: Default, no parameters  
  - Inputs: None  
  - Outputs: To LoadSentMessages  
  - Failures: None expected unless manual interruption  

- **LoadSentMessages**  
  - Type: Gmail node (OAuth2)  
  - Role: Fetch all sent emails with label "SENT"  
  - Config: Operation getAll, returnAll true, filters labelIds: ["SENT"]  
  - Inputs: Manual trigger  
  - Outputs: To SelectTo&CC  
  - Failures: Auth errors, rate limits, Gmail API downtime  
  - Retry enabled with 5s wait  

- **SelectTo&CC**  
  - Type: Set node  
  - Role: Selects "To" and "Cc" fields from sent emails for further processing  
  - Config: Includes only "To" and "Cc" fields plus other fields  
  - Inputs: LoadSentMessages output  
  - Outputs: To Remove Duplicates  

- **Remove Duplicates**  
  - Type: RemoveDuplicates node  
  - Role: Removes duplicate contacts based on "To" field to avoid redundant processing  
  - Config: Compares selectedFields on "To"  
  - Inputs: SelectTo&CC output  
  - Outputs: To Loop Over Items  

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes contacts in batches for controlled execution  
  - Config: No reset between batches (stateful)  
  - Inputs: Remove Duplicates output  
  - Outputs: To downstream processing or wait node  
  - Failures: Batch processing errors, memory issues if batch size too large  

---

#### 1.2 Contact Extraction and Deduplication

**Overview:**  
Extracts basic contact info (name, email, website) from the "To" field of emails and validates extracted domains.

**Nodes Involved:**  
- MailParser (Code node)  
- ValidDomain? (IF node)  

**Node Details:**

- **MailParser**  
  - Type: Code node (JavaScript)  
  - Role: Parses the "To" field to extract contact name, email, and derives website domain from email if not generic  
  - Config: Uses regex to parse email formats, excludes common free mail domains, derives company website URL  
  - Input: Single or multiple items from "Loop Over Items"  
  - Output: Object with keys: name, email, phone (null), website, and social media placeholders (null)  
  - Failures: Parsing errors if malformed emails, empty inputs, unexpected string formats  

- **ValidDomain?**  
  - Type: IF node  
  - Role: Checks if the derived website domain is non-empty and valid (not excluded)  
  - Config: Checks if "website" field is not empty string  
  - Input: MailParser output  
  - Output True: Path to Web Search  
  - Output False: No operation node (skips web search)  
  - Failures: Logic errors if input data malformed  

---

#### 1.3 Email Thread Retrieval and AI-based Information Extraction

**Overview:**  
For contacts with email history, searches Gmail inbox for existing threads, extracts detailed contact info from conversations using GPT-5 Nano.

**Nodes Involved:**  
- GetInboxMessages (Gmail node)  
- InboxMessages? (IF node)  
- SelectImportantFields (Set node)  
- OpenAI Chat Model (Langchain OpenAI node)  
- Information Extractor (Langchain Information Extractor node)  
- Merge (Code node)  

**Node Details:**

- **GetInboxMessages**  
  - Type: Gmail node (OAuth2)  
  - Role: Searches Gmail inbox for threads involving the contact's email address  
  - Config: Operation getAll, filters by sender address equal to contact email, limit 1 message for efficiency  
  - Input: From "3s" wait node after batch loop  
  - Output: Email threads including body, signatures, quoted replies  
  - Failures: API limits, auth errors, no threads found  

- **InboxMessages?**  
  - Type: IF node  
  - Role: Checks if inbox messages were found (non-empty)  
  - Config: Checks if length of keys in first GetInboxMessages item JSON > 0  
  - Output True: Proceed to extract fields  
  - Output False: Use MailParser fallback  

- **SelectImportantFields**  
  - Type: Set node  
  - Role: Extracts key fields from email thread JSON for AI input: from address, from name, text body, recipient name and address  
  - Config: Assigns fields from JSON paths  
  - Input: InboxMessages output  
  - Output: To OpenAI Chat Model  

- **OpenAI Chat Model (GPT-5 Nano)**  
  - Type: Langchain OpenAI Chat Model  
  - Role: Processes email text with system prompt to extract structured contact info (phone, social links, website) while ignoring recipient data  
  - Config: Model "gpt-5-nano", no additional options  
  - Credentials: OpenAI API key required  
  - Input: Text and contact info from SelectImportantFields  
  - Output: JSON with extracted contact details  
  - Failures: API rate limits, timeouts, prompt errors  

- **Information Extractor**  
  - Type: Langchain Information Extractor node  
  - Role: Validates and extracts structured JSON from OpenAI output conforming to schema with phone, web, social links  
  - Config: System prompt enforces strict JSON output without extra text  
  - Input: OpenAI Chat Model output  
  - Output: Clean structured data for merging  

- **Merge**  
  - Type: Code node  
  - Role: Combines extracted data from AI with original email contact info (name, email) into unified contact record  
  - Config: Merges fields with fallback for any missing data  
  - Input: Information Extractor and SelectImportantFields outputs  
  - Output: Final enriched contact JSON for writing  

---

#### 1.4 Domain Validation and Brave Web Search

**Overview:**  
For contacts without email history, performs a Brave Search using the extracted website domain to find relevant contact pages and clusters.

**Nodes Involved:**  
- SearchWebsite (Brave Search node)  
- GetClusters1 (Code node)  
- HaveClusters (IF node)  
- Loop Over Items1 (SplitInBatches node)  

**Node Details:**

- **SearchWebsite**  
  - Type: Brave Search API node  
  - Role: Searches Brave with the contact's website domain to find relevant web pages (contact pages, about us, social, etc.)  
  - Config: Search count 3, safesearch moderate, spellcheck true, search language English, result filter "web"  
  - Credentials: Brave Search API key  
  - Input: ValidDomain? true output  
  - Output: Search results JSON  

- **GetClusters1**  
  - Type: Code node  
  - Role: Filters Brave Search clusters for URLs that likely contain contact info or social links by matching keywords in titles and URLs  
  - Config: Uses a list of keywords such as "kontakt", "info", "contact", "about-us", "social", etc.  
  - Input: SearchWebsite output  
  - Output: List of filtered cluster URLs  

- **HaveClusters**  
  - Type: IF node  
  - Role: Checks if any clusters were found (length > 0)  
  - Output True: Process clusters (Loop Over Items1)  
  - Output False: Return contact from MailParser fallback  

- **Loop Over Items1**  
  - Type: SplitInBatches  
  - Role: Iterates over cluster URLs to scrape HTML and extract social info  
  - Input: From HaveClusters true path  
  - Output: To HTMLParser and GetHTML nodes  

---

#### 1.5 HTML Parsing for Social Profiles

**Overview:**  
Scrapes HTML content from contact page URLs and extracts social media links and other contact details using regex patterns.

**Nodes Involved:**  
- GetHTML (HTTP Request node)  
- Aggregate (Aggregate node)  
- HTMLParser (Code node)  

**Node Details:**

- **GetHTML**  
  - Type: HTTP Request node  
  - Role: Requests the full HTML content of the contact page URL  
  - Config: URL from cluster URLs, timeout 10 seconds, allows unauthorized certs  
  - Input: Loop Over Items1 output (cluster URLs)  
  - Output: HTML content for aggregation  

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Aggregates all HTML responses into a single data object for parsing  
  - Input: GetHTML output  
  - Output: To Loop Over Items1 (further processing)  

- **HTMLParser**  
  - Type: Code node  
  - Role: Parses aggregated HTML content searching for social media URLs using regex patterns for Facebook, Instagram, Twitter, LinkedIn, YouTube, TikTok, Spotify, SoundCloud, Bandcamp, and Linktree  
  - Config: Extracts first unique link per platform, cleans URLs from query params and trailing slashes  
  - Input: Loop Over Items1 output  
  - Output: JSON with social media links combined with base contact info from MailParser  
  - Failures: HTML structure changes, no matches, incomplete data  

---

#### 1.6 Data Aggregation and Writing to Google Sheets

**Overview:**  
Final stage merges all collected data and appends new or updated contact records into a Google Sheets document.

**Nodes Involved:**  
- ReturnContactFromMailParser (Code node)  
- No Operation, do nothing (NoOp node)  
- No Operation, do nothing1 (NoOp node)  
- WriteToDB (Google Sheets node)  

**Node Details:**

- **ReturnContactFromMailParser**  
  - Type: Code node  
  - Role: Passes through contact data from MailParser fallback when no clusters or email history found  
  - Input: From HaveClusters false path  
  - Output: To WriteToDB  

- **No Operation, do nothing / No Operation, do nothing1**  
  - Type: NoOp node  
  - Role: Placeholder nodes used to handle paths where no further action is required  
  - Input: From Loop Over Items or ValidDomain? false path  
  - Output: To WriteToDB or ends chain  

- **WriteToDB**  
  - Type: Google Sheets node (OAuth2)  
  - Role: Appends the enriched contact info as a new row into a specified Google Sheet  
  - Config: Auto-maps input data fields to columns named: name, email, phone, website, facebook, instagram, spotify, youtube, linkedin, twitter, linktree, tiktok, soundcloud, bandcamp  
  - Sheet: Document ID and sheet gid=0 specified  
  - Input: Merged contact data or fallback outputs  
  - Failures: Credential errors, quota exceeded, sheet not found, mapping errors  

---

### 3. Summary Table

| Node Name                 | Node Type                   | Functional Role                                   | Input Node(s)                   | Output Node(s)               | Sticky Note                                                                                                                                                |
|---------------------------|-----------------------------|--------------------------------------------------|--------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger              | Manual start of workflow                          | None                           | LoadSentMessages             | ##  Load & Filter Sent Emails<br>Triggers workflow manually and loads all sent emails from Gmail.<br>Extracts basic contact info (name, email) for processing. |
| LoadSentMessages           | Gmail                       | Load all sent emails from Gmail                   | When clicking ‘Execute workflow’ | SelectTo&CC                 | See above                                                                                                                                                  |
| SelectTo&CC               | Set                         | Select "To" & "Cc" fields from emails             | LoadSentMessages               | Remove Duplicates            | See above                                                                                                                                                  |
| Remove Duplicates          | RemoveDuplicates            | Remove duplicate contacts by "To" field           | SelectTo&CC                   | Loop Over Items              | See above                                                                                                                                                  |
| Loop Over Items            | SplitInBatches              | Process contacts in batches                        | Remove Duplicates              | No Operation, do nothing / 3s | See above                                                                                                                                                  |
| No Operation, do nothing   | NoOp                        | Placeholder for no further action                 | Loop Over Items               | 3s                          |                                                                                                                                                            |
| 3s                        | Wait                        | Wait 3 seconds before next step                    | No Operation, do nothing      | GetInboxMessages             |                                                                                                                                                            |
| GetInboxMessages           | Gmail                       | Search inbox for contact's email threads          | 3s                            | InboxMessages?              | ## GetInboxMessages<br>Searches Gmail for last existing conversations with this contact.<br>Includes full email threads (sent & received).                |
| InboxMessages?             | IF                          | Check if inbox messages found                      | GetInboxMessages              | SelectImportantFields / MailParser | See above                                                                                                                                                |
| SelectImportantFields      | Set                         | Extract key fields for AI processing               | InboxMessages? (true)          | OpenAI Chat Model            | See above                                                                                                                                                  |
| OpenAI Chat Model          | Langchain OpenAI Chat Model | Extract structured contact info from email text  | SelectImportantFields          | Information Extractor        | ## AI Extraction from Email History<br>Uses GPT-5 Nano to extract phone, social media, websites from email threads.                                        |
| Information Extractor      | Langchain Info Extractor    | Validate and extract JSON contact data             | OpenAI Chat Model             | Merge                       | See above                                                                                                                                                  |
| Merge                     | Code                        | Combine AI output with email contact info          | Information Extractor, SelectImportantFields | WriteToDB                | ## Merge<br>Combines AI extracted data with info from email.<br>## HTML Parser<br>Extracts social profiles from web search.<br>## WriteToDB<br>Saves to Sheets. |
| MailParser                | Code                        | Parse "To" field for name, email, derive website  | InboxMessages? (false)         | ValidDomain?                 |                                                                                                                                                            |
| ValidDomain?               | IF                          | Check if website domain is valid                   | MailParser                    | SearchWebsite / No Operation, do nothing1 | ## Web Search (No Email History)<br>Extract domain, validate, search Brave API, scrape HTML.                                                               |
| SearchWebsite             | Brave Search API             | Search Brave Search for website contact pages     | ValidDomain? (true)            | GetClusters1                | See above                                                                                                                                                  |
| GetClusters1              | Code                        | Filter Brave results to contact/social clusters    | SearchWebsite                 | HaveClusters                | See above                                                                                                                                                  |
| HaveClusters              | IF                          | Check if clusters found                            | GetClusters1                  | Loop Over Items1 / ReturnContactFromMailParser | See above                                                                                                                                                  |
| Loop Over Items1          | SplitInBatches              | Iterate cluster URLs for scraping                  | HaveClusters (true)            | HTMLParser, GetHTML          | See above                                                                                                                                                  |
| GetHTML                   | HTTP Request                | Download HTML content of cluster URLs              | Loop Over Items1              | Aggregate                   | See above                                                                                                                                                  |
| Aggregate                 | Aggregate                   | Aggregate multiple HTML responses                   | GetHTML                      | Loop Over Items1             | See above                                                                                                                                                  |
| HTMLParser                | Code                        | Extract social media links from HTML               | Loop Over Items1              | WriteToDB                   | See above                                                                                                                                                  |
| ReturnContactFromMailParser | Code                        | Pass contact data fallback                          | HaveClusters (false)           | WriteToDB                   |                                                                                                                                                            |
| No Operation, do nothing1 | NoOp                        | Placeholder for no further action                   | ValidDomain? (false)           | WriteToDB                   |                                                                                                                                                            |
| WriteToDB                 | Google Sheets               | Append enriched contact data to Google Sheet       | Merge, ReturnContactFromMailParser, No Operation, do nothing1 | Loop Over Items             | See above                                                                                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node:**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  

2. **Add Gmail node to fetch all sent emails:**  
   - Type: Gmail (OAuth2)  
   - Name: "LoadSentMessages"  
   - Operation: getAll  
   - Filter: LabelIds = ["SENT"]  
   - Return All: true  
   - Connect: Manual Trigger → LoadSentMessages  
   - Credentials: Set Gmail OAuth2 credentials  

3. **Create Set node to select "To" and "Cc":**  
   - Type: Set  
   - Name: "SelectTo&CC"  
   - Include fields: "To", "Cc", plus other fields  
   - Connect: LoadSentMessages → SelectTo&CC  

4. **Add RemoveDuplicates node to remove duplicate contacts:**  
   - Type: RemoveDuplicates  
   - Name: "Remove Duplicates"  
   - Compare by field: "To"  
   - Connect: SelectTo&CC → Remove Duplicates  

5. **Add SplitInBatches node to process contacts:**  
   - Type: SplitInBatches  
   - Name: "Loop Over Items"  
   - Options: reset = false  
   - Connect: Remove Duplicates → Loop Over Items  

6. **Add No Operation node as placeholder:**  
   - Type: NoOp  
   - Name: "No Operation, do nothing"  
   - Connect: Loop Over Items → No Operation  

7. **Add Wait node for 3 seconds:**  
   - Type: Wait  
   - Name: "3s"  
   - Amount: 3 seconds  
   - Connect: No Operation → 3s  

8. **Add Gmail node to get inbox messages for contact:**  
   - Type: Gmail (OAuth2)  
   - Name: "GetInboxMessages"  
   - Operation: getAll  
   - Filters: Sender equals contact email (dynamic)  
   - Limit: 1  
   - Connect: 3s → GetInboxMessages  
   - Credentials: Gmail OAuth2  

9. **Add IF node to check if inbox messages exist:**  
   - Type: IF  
   - Name: "InboxMessages?"  
   - Condition: Check if first GetInboxMessages item JSON keys length > 0  
   - Connect: GetInboxMessages → InboxMessages?  

10. **Add Set node to extract important fields for AI:**  
    - Type: Set  
    - Name: "SelectImportantFields"  
    - Assign: from address, from name, text content, recipient name, recipient address  
    - Connect: InboxMessages? (true) → SelectImportantFields  

11. **Add Langchain OpenAI Chat Model node:**  
    - Type: Langchain OpenAI Chat Model  
    - Name: "OpenAI Chat Model"  
    - Model: gpt-5-nano  
    - Input: Text from SelectImportantFields with system prompt for contact extraction  
    - Credentials: OpenAI API key  
    - Connect: SelectImportantFields → OpenAI Chat Model  

12. **Add Langchain Information Extractor node:**  
    - Type: Langchain Information Extractor  
    - Name: "Information Extractor"  
    - Schema: Structured JSON for tel, web, social links  
    - Connect: OpenAI Chat Model → Information Extractor  

13. **Add Code node to merge AI output with contact info:**  
    - Type: Code  
    - Name: "Merge"  
    - Function: Merge AI output fields with name/email from email thread  
    - Connect: Information Extractor → Merge  

14. **Add Code node to parse "To" field for contacts without inbox messages:**  
    - Type: Code  
    - Name: "MailParser"  
    - Function: Extract name/email, derive website from email domain excluding generic domains  
    - Connect: InboxMessages? (false) → MailParser  

15. **Add IF node to validate extracted website domain:**  
    - Type: IF  
    - Name: "ValidDomain?"  
    - Condition: Check website field not empty  
    - Connect: MailParser → ValidDomain?  

16. **Add Brave Search node to query contact pages:**  
    - Type: Brave Search API  
    - Name: "SearchWebsite"  
    - Query: Website domain from MailParser  
    - Count: 3 results  
    - Options: safesearch moderate, spellcheck true, search_lang en, result_filter web  
    - Credentials: Brave Search API key  
    - Connect: ValidDomain? (true) → SearchWebsite  

17. **Add Code node to filter and extract contact clusters:**  
    - Type: Code  
    - Name: "GetClusters1"  
    - Function: Filter Brave Search results for URLs containing contact-related keywords  
    - Connect: SearchWebsite → GetClusters1  

18. **Add IF node to check if clusters exist:**  
    - Type: IF  
    - Name: "HaveClusters"  
    - Condition: Check if GetClusters1 output length > 0  
    - Connect: GetClusters1 → HaveClusters  

19. **Add SplitInBatches node to process cluster URLs:**  
    - Type: SplitInBatches  
    - Name: "Loop Over Items1"  
    - Connect: HaveClusters (true) → Loop Over Items1  

20. **Add HTTP Request node to get HTML content:**  
    - Type: HTTP Request  
    - Name: "GetHTML"  
    - URL: Dynamic from Loop Over Items1  
    - Timeout: 10 seconds  
    - Allow Unauthorized Certs: true  
    - Connect: Loop Over Items1 → GetHTML  

21. **Add Aggregate node to compile HTML responses:**  
    - Type: Aggregate  
    - Name: "Aggregate"  
    - Operation: aggregateAllItemData  
    - Connect: GetHTML → Aggregate  

22. **Add Code node to parse HTML and extract social links:**  
    - Type: Code  
    - Name: "HTMLParser"  
    - Function: Uses regex to extract social media URLs from aggregated HTML  
    - Connect: Loop Over Items1 → HTMLParser  

23. **Add Code node to return fallback contact from MailParser:**  
    - Type: Code  
    - Name: "ReturnContactFromMailParser"  
    - Connect: HaveClusters (false) → ReturnContactFromMailParser  

24. **Add No Operation nodes as needed for false branches:**  
    - Type: NoOp  
    - Names: "No Operation, do nothing" and "No Operation, do nothing1"  
    - Connect: Loop Over Items → No Operation, do nothing  
    - Connect: ValidDomain? (false) → No Operation, do nothing1  

25. **Add Google Sheets node to append contacts:**  
    - Type: Google Sheets (OAuth2)  
    - Name: "WriteToDB"  
    - Operation: append row  
    - Document ID: Your Google Sheets ID  
    - Sheet Name: "gid=0" (or your sheet tab)  
    - Columns: Map all contact fields (name, email, phone, website, facebook, instagram, etc.)  
    - Connect: Merge, ReturnContactFromMailParser, No Operation, do nothing1 → WriteToDB  
    - Connect: WriteToDB → Loop Over Items (to continue batch processing)  
    - Credentials: Google Sheets OAuth2  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| This workflow processes your Gmail sent folder to create an enriched contact database in Google Sheets. It uses a two-path approach: Path A uses email history with GPT-5 Nano for detailed extraction; Path B uses Brave Search API for web scraping when email history is missing. Setup steps include creating a Google Sheet from a template, connecting Gmail, OpenAI, Brave Search, and Google Sheets credentials, and customizing excluded domains. Run manually to process all sent emails. | Workflow description and setup instructions from Sticky Note5.  Template Sheet: [Make a copy here](https://docs.google.com/spreadsheets/d/1ox0cP_v8UuonAFr3eXkOFRlBb_P86NRedEaK4fss5cA/edit?usp=sharing) |
| The HTML parser extracts social media links using regex patterns for multiple platforms including Facebook, Instagram, Twitter (X), LinkedIn, YouTube, TikTok, Spotify, SoundCloud, Bandcamp, and Linktree. It cleans URLs by removing query parameters and trailing slashes to maintain consistency.                                                                                                                                                                                              | Explanation from Sticky Note2 and HTMLParser node code.                                                                           |
| Gmail nodes require OAuth2 credentials with appropriate scopes for reading sent emails and inbox messages. OpenAI node requires an API key with access to GPT-5 Nano. Brave Search API credentials must be set up with a valid API key. Google Sheets node requires OAuth2 credentials with write access to the specified spreadsheet.                                                                                                                                                                        | Credential requirements detailed in node configurations.                                                                          |
| The workflow uses batch processing (SplitInBatches) to handle multiple contacts sequentially, avoiding API rate limits and memory overload.                                                                                                                                                                                                                                                                                                                                                                | General design note from batch nodes Loop Over Items and Loop Over Items1.                                                        |
| The AI extraction includes strict rules to ignore recipient contact info and only extract data for the most recent sender, returning null for missing or unclear values, ensuring clean and reliable data output.                                                                                                                                                                                                                                                                                             | From system prompt template in Information Extractor node.                                                                        |

---

# Disclaimer

The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.

---