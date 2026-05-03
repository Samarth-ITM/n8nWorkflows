Scrape LinkedIn Jobs with Gemini AI and Store in Google Sheets using RSS

https://n8nworkflows.xyz/workflows/scrape-linkedin-jobs-with-gemini-ai-and-store-in-google-sheets-using-rss-8219


# Scrape LinkedIn Jobs with Gemini AI and Store in Google Sheets using RSS

### 1. Workflow Overview

This workflow automates the process of scraping job listings from LinkedIn via an RSS feed, extracting detailed job information using AI (Google Gemini and Langchain), cleaning and structuring the data, and finally storing the results in a Google Sheets document. It is designed for users who want to maintain an up-to-date database of LinkedIn job postings with enriched, structured data to facilitate job market analysis or recruitment efforts.

The workflow is logically divided into these functional blocks:

- **1.1 Input Reception and Job Listing Retrieval:** Triggered manually, reads RSS feed URLs, and loops through job postings.
- **1.2 Job Page Scraping and HTML Extraction:** Fetches full HTML content of job detail pages and extracts embedded JSON data.
- **1.3 Data Parsing and Preprocessing:** Parses JSON content, cleans descriptions from HTML tags and excess spaces.
- **1.4 AI Processing and Structured Data Generation:** Uses Google Gemini AI with Langchain agent to extract specific job fields in JSON format.
- **1.5 Data Storage:** Updates or appends the structured job data into a Google Sheet for persistence.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Job Listing Retrieval

**Overview:**  
This block initiates the workflow manually, reads an RSS feed URL containing LinkedIn job posts, and iteratively processes each job post.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Read RSS Feed.  
- Loop for Job Posts  
- Sticky Note2  
- Sticky Note3  

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point, starts workflow on manual execution.  
  - Configuration: Default manual trigger, no parameters.  
  - Inputs: None  
  - Outputs: Initiates "Read RSS Feed." node  
  - Edge Cases: None typical; user must manually trigger.  

- **Read RSS Feed.**  
  - Type: RSS Feed Read  
  - Role: Reads RSS feed containing LinkedIn job listings.  
  - Configuration: URL parameterized as `{{ $json.rssFeedUrl }}` to allow dynamic input.  
  - Inputs: Trigger from manual node  
  - Outputs: Emits list of job posts (title, link, pubDate, etc.)  
  - Edge Cases: RSS feed URL invalid or unreachable; empty feed returns no items.  

- **Loop for Job Posts**  
  - Type: Split In Batches  
  - Role: Iterates through each job post item individually for sequential processing.  
  - Configuration: Default batch size (implicitly 1).  
  - Inputs: Output from RSS Feed  
  - Outputs: Single job post per iteration forwarded to "Scrape Job HTML." node  
  - Edge Cases: Empty input leads to no iterations; batch handling errors possible with malformed input.  

- **Sticky Note2**  
  - Content: "Retrieves information such as the job title, link, and publication date."  
  - Context: Covers "Read RSS Feed." node describing its function.  

- **Sticky Note3**  
  - Content: "Processes each job post one by one to ensure the data is handled correctly."  
  - Context: Covers "Loop for Job Posts" node explaining batch-wise processing.  

---

#### 2.2 Job Page Scraping and HTML Extraction

**Overview:**  
Fetches the full HTML content of each job posting page from LinkedIn and extracts embedded JSON data containing job details.

**Nodes Involved:**  
- Scrape Job HTML.  
- Extract HTML  
- Sticky Note4  
- Sticky Note5  

**Node Details:**

- **Scrape Job HTML.**  
  - Type: HTTP Request  
  - Role: Performs HTTP GET request to the job post URL to retrieve HTML page content.  
  - Configuration: URL dynamically set to `={{ $json.link }}` from RSS feed item.  
  - Inputs: Single job post from Loop for Job Posts  
  - Outputs: Full HTML content of job page as raw text.  
  - Edge Cases: HTTP errors (404, 500), rate limiting by LinkedIn, connection timeouts.  

- **Extract HTML**  
  - Type: HTML Extract  
  - Role: Extracts specific embedded JSON-LD script element from HTML content.  
  - Configuration: CSS selector targeting `script[type="application/ld+json"]`, extracting its content under key `body`. Text is cleaned up to remove extraneous characters.  
  - Inputs: Raw HTML from HTTP Request node  
  - Outputs: Extracted JSON-LD string containing structured job data.  
  - Edge Cases: Missing or malformed JSON-LD script element causes parsing failure.  

- **Sticky Note4**  
  - Content: "Scrapes the full HTML content of the job page."  
  - Context: Related to "Scrape Job HTML." node.  

- **Sticky Note5**  
  - Content: "Extracts a specific JSON script embedded in the HTML."  
  - Context: Related to "Extract HTML" node.  

---

#### 2.3 Data Parsing and Preprocessing

**Overview:**  
Parses the extracted JSON job data, normalizes and cleans the job description by removing HTML tags and extra spaces, and structures core job fields for AI processing.

**Nodes Involved:**  
- Parse Job Details.  
- Pre-clean Before AI  
- Sticky Note6  

**Node Details:**

- **Parse Job Details.**  
  - Type: Code (JavaScript)  
  - Role: Parses JSON string from the extracted script and maps relevant fields (title, company, description, location, country, employment type, valid through).  
  - Configuration: Custom JS code parsing `item.json.body` with error handling implicit in code.  
  - Inputs: Extracted JSON string from "Extract HTML"  
  - Outputs: Standardized JSON object per job post  
  - Edge Cases: JSON parse errors if data is corrupted or incomplete.  

- **Pre-clean Before AI**  
  - Type: Code (JavaScript)  
  - Role: Cleans job description text by removing all HTML tags and normalizing whitespace. Also extracts other metadata fields for AI consumption.  
  - Configuration: Custom JS code with regex to strip tags and trim spaces.  
  - Inputs: Parsed job details from previous node  
  - Outputs: Cleaned text and other fields ready for AI input  
  - Edge Cases: If description missing or null, defaults to empty string to avoid errors.  

- **Sticky Note6**  
  - Content: "Removes HTML tags and extra spaces to prepare the data for the AI model."  
  - Context: Covers the "Pre-clean Before AI" node.  

---

#### 2.4 AI Processing and Structured Data Generation

**Overview:**  
Uses a Langchain AI agent powered by Google Gemini Chat Model to analyze cleaned job descriptions and extract specific structured fields (benefits, location_remote, job_description) in simplified JSON format.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model  
- Structured Output Parser  

**Node Details:**

- **Google Gemini Chat Model**  
  - Type: Langchain Google Gemini LM Chat  
  - Role: Provides the underlying AI language model for text processing.  
  - Configuration: Default options; integrated with Langchain agent node as language model.  
  - Inputs: None (used as resource node)  
  - Outputs: AI-generated text response forwarded to AI Agent  
  - Edge Cases: API authentication errors, rate limits, or unexpected AI output format.  

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI response text into JSON format based on provided schema example.  
  - Configuration: JSON schema example includes keys `benefits`, `location_state`, `location_remote`, `job_description`.  
  - Inputs: Text output from Google Gemini Chat Model  
  - Outputs: Parsed JSON object to AI Agent  
  - Edge Cases: Parsing failures if AI output does not conform to schema.  

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Coordinates AI prompt, model interaction, and output parsing for extracting job data.  
  - Configuration:  
    - Prompt defines the task: extract benefits, location_remote, and short job description (200 characters or less) from job description text.  
    - Uses expressions to inject cleaned description: `{{ $json.description }}`  
    - Output is simplified JSON format.  
  - Inputs: Cleaned job data from Pre-clean Before AI  
  - Outputs: Structured AI-extracted job data  
  - Edge Cases: Expression evaluation errors, AI service failures, malformed outputs.  

---

#### 2.5 Data Storage

**Overview:**  
Updates a Google Sheets document with the enriched job data, appending new rows or updating existing ones based on job description links for deduplication.

**Nodes Involved:**  
- Update Google Sheet.  
- Sticky Note7  

**Node Details:**

- **Update Google Sheet.**  
  - Type: Google Sheets  
  - Role: Appends or updates rows in a specified Google Sheet with job data.  
  - Configuration:  
    - Operation: Append or Update  
    - Sheet Name and Document ID dynamically set via `{{ $json.sheetName }}` and `{{ $json.googleSheetsId }}`  
    - Columns mapped include Date, Title, Benefits, Company Name, Job Description, Job Description Link.  
    - Uses "Job Description Link" as matching column to avoid duplicates.  
    - Credentials: uses OAuth2 for Google Sheets with configured account.  
  - Inputs: Structured AI output and original job metadata combined.  
  - Outputs: None (end of workflow)  
  - Edge Cases: Authentication failures, quota limits on Google Sheets API, schema mismatch, network errors.  

- **Sticky Note7**  
  - Content: "Adds the extracted data to a Google Sheet."  
  - Context: Related to "Update Google Sheet." node.  

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                | Input Node(s)             | Output Node(s)          | Sticky Note                                       |
|---------------------------|----------------------------------|------------------------------------------------|---------------------------|-------------------------|--------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Manual workflow start                          | None                      | Read RSS Feed.           |                                                  |
| Read RSS Feed.             | RSS Feed Read                    | Reads LinkedIn jobs RSS feed                   | When clicking ‘Execute workflow’ | Loop for Job Posts    | Retrieves information such as the job title, link, and publication date. |
| Loop for Job Posts         | Split In Batches                 | Processes each job post individually           | Read RSS Feed.             | Scrape Job HTML.         | Processes each job post one by one to ensure the data is handled correctly. |
| Scrape Job HTML.           | HTTP Request                    | Fetches full HTML page for each job            | Loop for Job Posts         | Extract HTML             | Scrapes the full HTML content of the job page.   |
| Extract HTML              | HTML Extract                    | Extracts JSON-LD script from HTML              | Scrape Job HTML.           | Parse Job Details.       | Extracts a specific JSON script embedded in the HTML. |
| Parse Job Details.         | Code (JavaScript)               | Parses JSON-LD content into fields             | Extract HTML               | Pre-clean Before AI      |                                                  |
| Pre-clean Before AI        | Code (JavaScript)               | Cleans description text and normalizes data    | Parse Job Details.         | AI Agent                 | Removes HTML tags and extra spaces to prepare the data for the AI model. |
| Google Gemini Chat Model   | Langchain LM Chat (Google Gemini) | Provides AI language model                      | None                      | AI Agent (ai_languageModel) |                                                  |
| Structured Output Parser   | Langchain Output Parser         | Parses AI output into structured JSON          | Google Gemini Chat Model   | AI Agent (ai_outputParser)|                                                  |
| AI Agent                  | Langchain Agent                 | Orchestrates AI prompt, model, and output parsing | Pre-clean Before AI, Google Gemini Chat Model, Structured Output Parser | Update Google Sheet.    |                                                  |
| Update Google Sheet.       | Google Sheets                   | Stores structured job data into Google Sheets | AI Agent                  | None                    | Adds the extracted data to a Google Sheet.       |
| Sticky Note2               | Sticky Note                    | Comment on RSS feed retrieval                   | None                      | None                    | Retrieves information such as the job title, link, and publication date. |
| Sticky Note3               | Sticky Note                    | Comment on job post batch processing            | None                      | None                    | Processes each job post one by one to ensure the data is handled correctly. |
| Sticky Note4               | Sticky Note                    | Comment on job page scraping                     | None                      | None                    | Scrapes the full HTML content of the job page.   |
| Sticky Note5               | Sticky Note                    | Comment on JSON extraction from HTML            | None                      | None                    | Extracts a specific JSON script embedded in the HTML. |
| Sticky Note6               | Sticky Note                    | Comment on data cleaning before AI               | None                      | None                    | Removes HTML tags and extra spaces to prepare the data for the AI model. |
| Sticky Note7               | Sticky Note                    | Comment on storing data to Google Sheets         | None                      | None                    | Adds the extracted data to a Google Sheet.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node** named `When clicking ‘Execute workflow’`.  
   - No special configuration.

3. **Add an RSS Feed Read node** named `Read RSS Feed.`  
   - Set the feed URL parameter to `{{ $json.rssFeedUrl }}` to allow dynamic URLs.  
   - Connect output of Manual Trigger to this node.

4. **Add a Split In Batches node** named `Loop for Job Posts`.  
   - Default batch size (1) to process one job post at a time.  
   - Connect output of RSS Feed Read node to this node.

5. **Add an HTTP Request node** named `Scrape Job HTML.`  
   - Set the URL parameter to `={{ $json.link }}` to fetch the job page URL from current item.  
   - Connect output of Split In Batches node to this node.

6. **Add an HTML Extract node** named `Extract HTML`.  
   - Configure to extract content using CSS selector: `script[type="application/ld+json"]`.  
   - Extracted content key: `body`.  
   - Enable text cleanup option.  
   - Connect output of HTTP Request to this node.

7. **Add a Code node** named `Parse Job Details.`  
   - Use JavaScript code to parse the JSON in `item.json.body` and map fields: title, company, description, location, country, employmentType, validThrough.  
   - Connect output of HTML Extract to this node.

8. **Add a Code node** named `Pre-clean Before AI`.  
   - Use JavaScript code to remove all HTML tags from description and normalize spaces.  
   - Extract title, company, location, country, employmentType, validThrough as is.  
   - Connect output of Parse Job Details to this node.

9. **Add a Langchain Google Gemini LM Chat node** named `Google Gemini Chat Model`.  
   - Use default configuration.  
   - No input connections (acts as a resource node).

10. **Add a Langchain Structured Output Parser node** named `Structured Output Parser`.  
    - Provide JSON schema example with keys: `benefits`, `location_state`, `location_remote`, `job_description`.  
    - No input connections (acts as resource node).

11. **Add a Langchain Agent node** named `AI Agent`.  
    - Configure prompt text to instruct extraction of benefits, location_remote, and a short job description (max 200 chars) from the cleaned description field using expression `{{ $json.description }}`.  
    - Set prompt type to "define" and enable output parser.  
    - Connect:  
      - Input from `Pre-clean Before AI` node.  
      - AI language model set to `Google Gemini Chat Model`.  
      - AI output parser set to `Structured Output Parser`.  

12. **Add a Google Sheets node** named `Update Google Sheet.`  
    - Operation: Append or Update.  
    - Configure with Google Sheets OAuth2 credentials.  
    - Set Document ID and Sheet Name dynamically using `{{ $json.googleSheetsId }}` and `{{ $json.sheetName }}`.  
    - Map columns: Date (from RSS pubDate), Title, Benefits (from AI output), Company Name, Job Description (from AI output), Job Description Link (from RSS feed).  
    - Use "Job Description Link" as the matching column for update to avoid duplicates.  
    - Connect output of `AI Agent` to this node.

13. **Connect outputs accordingly:**  
    - Manual Trigger → Read RSS Feed.  
    - Read RSS Feed. → Loop for Job Posts.  
    - Loop for Job Posts → Scrape Job HTML.  
    - Scrape Job HTML. → Extract HTML.  
    - Extract HTML → Parse Job Details.  
    - Parse Job Details. → Pre-clean Before AI.  
    - Pre-clean Before AI → AI Agent.  
    - Google Gemini Chat Model → AI Agent (as languageModel).  
    - Structured Output Parser → AI Agent (as outputParser).  
    - AI Agent → Update Google Sheet.

14. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                                             |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses Google Gemini AI via Langchain integration for natural language processing and data extraction. | Requires Google Gemini API access and valid Langchain n8n nodes.                                                                             |
| Google Sheets OAuth2 credentials must be configured prior to use and authorized with access to the target sheet. | See n8n documentation for Google Sheets OAuth2 setup: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleSheets/          |
| The workflow handles job posts one at a time to avoid rate limits and improve error isolation.             |                                                                                                                                            |
| The AI prompt limits job description output to 200 characters to maintain concise summaries.               |                                                                                                                                            |
| RSS feed URL must be supplied dynamically in the starting JSON input with key `rssFeedUrl`.                 |                                                                                                                                            |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.