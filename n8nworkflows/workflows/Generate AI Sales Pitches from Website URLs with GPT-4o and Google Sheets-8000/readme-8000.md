Generate AI Sales Pitches from Website URLs with GPT-4o and Google Sheets

https://n8nworkflows.xyz/workflows/generate-ai-sales-pitches-from-website-urls-with-gpt-4o-and-google-sheets-8000


# Generate AI Sales Pitches from Website URLs with GPT-4o and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of personalized AI-driven sales pitches from a list of website URLs maintained in a Google Sheet. It is designed for sales or marketing teams seeking tailored outreach content based on the actual content of potential client websites. The workflow comprises the following logical blocks:

- **1.1 Input Acquisition:** Fetches website URLs from a Google Sheet.
- **1.2 Iterative Processing:** Loops over each URL to process them one by one.
- **1.3 Website Content Extraction:** Scrapes website content using Firecrawl API, retrieving markdown and screenshot formats.
- **1.4 AI-Driven Personalization:** Sends scraped content to OpenAI's GPT-4o model with a carefully crafted prompt to generate a concise, consultative sales pitch.
- **1.5 Output Shaping and Rate Limiting:** Adjusts the AI response into a specific field format and enforces a brief wait to avoid rate limits or throttling.
- **1.6 Data Persistence:** Updates the original Google Sheet by appending or updating the personalized message alongside the corresponding website URL.
- **1.7 Completion Handling:** Marks the end of processing for each URL batch.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Acquisition

- **Overview:** Retrieves the list of website URLs from a Google Sheet. Each row should contain at least a website URL and optionally a personalized message field.
- **Nodes Involved:**  
  - Fetch website URL from sheet  
  - Manual Trigger  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Manual Trigger**  
    - Type: Trigger node activated manually  
    - Role: Starts the workflow manually  
    - Config: No parameters configured  
    - Connections: Outputs to "Fetch website URL from sheet"  
    - Edge cases: None inherent, but workflow depends on manual start  
  - **Fetch website URL from sheet**  
    - Type: Google Sheets node (read operation)  
    - Role: Reads all rows from the specified Google Sheet (Sheet1, gid=0) which holds website URLs  
    - Config: Reads entire sheet, with credentials configured via OAuth2  
    - Key expressions: None; uses static document and sheet IDs  
    - Inputs: From Manual Trigger  
    - Outputs: To "Loop over URLs"  
    - Edge cases: Sheet access errors, credential failures, empty or malformed rows  
  - **Sticky Note** (Instructional)  
    - Provides setup instructions and spreadsheet formatting requirements:  
      - Column 1 = Website URLs  
      - Column 2 = Personalized Message (initially blank)  
      - Requires Google credentials setup in n8n  

#### 2.2 Iterative Processing

- **Overview:** Processes each website URL one at a time by splitting the array of URLs into batches of one to sequentially scrape and generate personalized messages.
- **Nodes Involved:**  
  - Loop over URLs  
  - Done  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Loop over URLs**  
    - Type: SplitInBatches node  
    - Role: Controls processing flow to handle one URL per batch, enabling sequential and manageable API calls  
    - Config: batchSize = 1  
    - Inputs: From "Fetch website URL from sheet"  
    - Outputs:  
      - Main output to "Scrape website and get its content"  
      - Secondary output to "Done" (likely to mark completion or for control logic)  
    - Edge cases: Failure in batch iteration, empty input array  
  - **Done**  
    - Type: NoOp node (no operation)  
    - Role: Placeholder to signal completion or act as an endpoint in the workflow branch  
    - Inputs: From "Loop over URLs" secondary output  
    - Outputs: None  
  - **Sticky Note1** (Instructional)  
    - Explains the core operation: looping through each website, scraping content via Firecrawl, and personalizing the pitch using ChatGPT  
    - Prerequisites listed: Firecrawl API key and OpenAI credentials  

#### 2.3 Website Content Extraction

- **Overview:** Scrapes website content in markdown format to provide textual data for AI processing.
- **Nodes Involved:**  
  - Scrape website and get its content

- **Node Details:**  
  - **Scrape website and get its content**  
    - Type: Firecrawl API node (third-party web scraping)  
    - Role: Fetches website content (markdown and screenshot formats) for analysis  
    - Config: URL dynamically set from the current batch item’s "Website" field; scrape options set to markdown and screenshot; headers empty  
    - Inputs: From "Loop over URLs"  
    - Outputs: To "Personalize Message"  
    - Credentials: Firecrawl API key required  
    - Edge cases: Scraping failures (e.g., site unavailable, blocked requests), API rate limits, malformed URLs  

#### 2.4 AI-Driven Personalization

- **Overview:** Uses OpenAI GPT-4o to generate a personalized sales pitch based on the scraped website content, adhering to strict formatting and tone guidelines.
- **Nodes Involved:**  
  - Personalize Message

- **Node Details:**  
  - **Personalize Message**  
    - Type: OpenAI node (LangChain integration)  
    - Role: Sends website markdown content with a detailed prompt to GPT-4o to produce a customized sales pitch  
    - Config:  
      - Model: chatgpt-4o-latest  
      - Prompt instructs GPT to produce:  
        - Context bullets (2–4) about the organization’s activities and AI use cases  
        - A consultative, specific pitch paragraph (70–110 words) without buzzwords or hype  
        - Personalized greeting if a contact name is provided  
      - Output is plain text with a strict structure  
    - Inputs: Scraped content JSON (markdown)  
    - Outputs: To "Shape output (Edit Fields)"  
    - Credentials: OpenAI API key required  
    - Edge cases: API call failures, prompt misinterpretation, token limits  

#### 2.5 Output Shaping and Rate Limiting

- **Overview:** Extracts the AI-generated message content and reshapes it into a specific JSON field; then enforces a 2-second wait to manage rate limits before writing back to Google Sheets.
- **Nodes Involved:**  
  - Shape output (Edit Fields)  
  - Wait 2s

- **Node Details:**  
  - **Shape output (Edit Fields)**  
    - Type: Set node  
    - Role: Defines a new field "Personalized Message" using the "message.content" from the AI response JSON  
    - Config: Sets "Personalized Message" = `={{ $json.message.content }}`  
    - Inputs: From "Personalize Message"  
    - Outputs: To "Wait 2s"  
    - Edge cases: Expression failures if AI response has unexpected structure  
  - **Wait 2s**  
    - Type: Wait node  
    - Role: Pauses workflow for 2 seconds to avoid API throttling or rate limit issues  
    - Config: 2 seconds delay  
    - Inputs: From "Shape output (Edit Fields)"  
    - Outputs: To "Update sheet with personalized message"  

#### 2.6 Data Persistence

- **Overview:** Writes or updates the personalized sales pitch message back into the Google Sheet in the appropriate row.
- **Nodes Involved:**  
  - Update sheet with personalized message  
  - Sticky Note2 (instructional)

- **Node Details:**  
  - **Update sheet with personalized message**  
    - Type: Google Sheets node (append or update operation)  
    - Role: Adds or updates the "Personalized Message" column in the Google Sheet with the AI-generated pitch  
    - Config:  
      - Sheet: Sheet1 (gid=0) in the specified Google Sheet document  
      - Mapping: Matches on "Personalized Message" column to append or update  
      - Columns defined: Website and Personalized Message  
    - Inputs: From "Wait 2s"  
    - Outputs: Loops back to "Loop over URLs" for next batch  
    - Credentials: Google Sheets OAuth2 required  
    - Edge cases: Sheet write errors, credential issues, race conditions if multiple workflows write simultaneously  
  - **Sticky Note2**  
    - Explains that the message is mapped to the correct column in the spreadsheet  

---

### 3. Summary Table

| Node Name                      | Node Type                           | Functional Role                          | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                          |
|--------------------------------|-----------------------------------|----------------------------------------|-------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------------|
| Manual Trigger                 | Manual Trigger                    | Workflow start                         | None                          | Fetch website URL from sheet           | Upon being triggered, this first step is going to fetch all URLs from your spreadsheet...          |
| Fetch website URL from sheet   | Google Sheets (read)              | Reads URLs from spreadsheet            | Manual Trigger                | Loop over URLs                        | Same as above                                                                                       |
| Loop over URLs                 | SplitInBatches                   | Processes URLs one at a time           | Fetch website URL from sheet  | Scrape website and get its content, Done | This is where the meat of things happens. The automation will loop over each website...             |
| Done                          | NoOp                             | Marks completion or branch endpoint    | Loop over URLs (secondary)    | None                                  | Same as above                                                                                       |
| Scrape website and get its content | Firecrawl (web scraper)           | Scrapes website content                 | Loop over URLs                | Personalize Message                   | Same as above                                                                                       |
| Personalize Message            | OpenAI (LangChain integration)   | Generates AI personalized pitch        | Scrape website and get its content | Shape output (Edit Fields)          | Same as above                                                                                       |
| Shape output (Edit Fields)     | Set                             | Extracts and formats AI response       | Personalize Message           | Wait 2s                              |                                                                                                    |
| Wait 2s                       | Wait                            | Rate limit delay                        | Shape output (Edit Fields)    | Update sheet with personalized message |                                                                                                    |
| Update sheet with personalized message | Google Sheets (append/update)      | Writes personalized pitch back to sheet | Wait 2s                     | Loop over URLs                       | Your message will get mapped to the appropriate column in the spreadsheet                          |
| Sticky Note                   | Sticky Note                     | Setup instructions                      | None                         | None                                 | Upon being triggered, this first step is going to fetch all URLs from your spreadsheet...          |
| Sticky Note1                  | Sticky Note                     | Core process instructions               | None                         | None                                 | This is where the meat of things happens. The automation will loop over each website...             |
| Sticky Note2                  | Sticky Note                     | Output message mapping note             | None                         | None                                 | Your message will get mapped to the appropriate column in the spreadsheet                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed  
   - Purpose: To start the workflow on demand

2. **Add Google Sheets Node to Fetch URLs**  
   - Type: Google Sheets (read operation)  
   - Configure:  
     - Document ID: Your Google Sheet ID containing URLs  
     - Sheet Name: "gid=0" or appropriate sheet tab  
     - Operation: Read all rows  
   - Credentials: Setup Google Sheets OAuth2 credentials in n8n  
   - Connect Manual Trigger → Fetch website URL from sheet

3. **Add SplitInBatches Node to Loop Through URLs**  
   - Type: SplitInBatches  
   - Parameter: Batch size = 1 to process one URL at a time  
   - Connect Fetch website URL from sheet → Loop over URLs

4. **Add Firecrawl Node to Scrape Website Content**  
   - Type: Firecrawl (or any web scraper node supporting markdown/screenshot)  
   - Parameters:  
     - URL: Set dynamically to `{{ $json.Website }}` from current batch  
     - Scrape options: Request markdown and screenshot formats  
   - Credentials: Set Firecrawl API key credentials in n8n  
   - Connect Loop over URLs → Scrape website and get its content

5. **Add OpenAI Node (LangChain) for AI Personalization**  
   - Type: OpenAI (LangChain node)  
   - Model: Select "chatgpt-4o-latest"  
   - Messages:  
     - Add system/user prompt with instructions to analyze markdown content (from `$json.data.markdown`) and generate a structured sales pitch with specific tone and format (see prompt in workflow overview)  
   - Credentials: Setup OpenAI API key in n8n  
   - Connect Scrape website and get its content → Personalize Message

6. **Add Set Node to Shape Output**  
   - Type: Set  
   - Add field: "Personalized Message"  
   - Value: `={{ $json.message.content }}` (extract AI response text)  
   - Connect Personalize Message → Shape output (Edit Fields)

7. **Add Wait Node for Rate Limiting**  
   - Type: Wait  
   - Parameters: 2 seconds delay  
   - Connect Shape output (Edit Fields) → Wait 2s

8. **Add Google Sheets Node to Update Sheet**  
   - Type: Google Sheets (append or update operation)  
   - Configure:  
     - Document ID and Sheet Name same as fetch node  
     - Operation: Append or update  
     - Columns: Map "Website" and "Personalized Message" fields  
     - Matching columns: At least "Personalized Message" or as per your logic  
   - Credentials: Use same Google Sheets OAuth2 credential  
   - Connect Wait 2s → Update sheet with personalized message

9. **Loop Back to SplitInBatches Node**  
   - Connect Update sheet with personalized message → Loop over URLs (for next batch iteration)

10. **Add NoOp Node for Completion**  
    - Type: NoOp  
    - Connect Loop over URLs secondary output → Done (optional for clean termination)

11. **Add Sticky Notes (Optional)**  
    - Add notes around key nodes for instructions and prerequisites:
      - Spreadsheet format and credentials setup near start  
      - Explanation of loop and AI processing near main processing nodes  
      - Confirmation of message mapping near update node

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Spreadsheet setup requires two columns: "Website" and "Personalized Message".                                           | Ensure spreadsheet matches this structure exactly for proper mapping.                          |
| Firecrawl API used to scrape website content in markdown format.                                                       | Requires Firecrawl API key credential setup in n8n.                                            |
| OpenAI GPT-4o model used with a prompt enforcing no buzzwords and a consultative tone for sales pitches.               | Adjust prompt as needed to fit organizational style or outreach focus.                         |
| Rate limiting achieved by a 2-second wait node to avoid API throttling.                                                | Can be adjusted based on API quotas and limits.                                               |
| Google Sheets OAuth2 credentials must be configured with access to the relevant spreadsheet.                           | Proper permissions required for read and write operations.                                   |
| For detailed prompt and usage examples, refer to OpenAI and Firecrawl official documentation.                           | https://platform.openai.com/docs/api-reference, https://firecrawl.com/docs/api               |

---

**Disclaimer:**  
The provided content is generated exclusively from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.