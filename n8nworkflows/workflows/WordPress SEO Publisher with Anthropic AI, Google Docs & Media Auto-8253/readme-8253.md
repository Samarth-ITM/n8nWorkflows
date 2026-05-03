WordPress SEO Publisher with Anthropic AI, Google Docs & Media Auto

https://n8nworkflows.xyz/workflows/wordpress-seo-publisher-with-anthropic-ai--google-docs---media-auto-8253


# WordPress SEO Publisher with Anthropic AI, Google Docs & Media Auto

---

### 1. Workflow Overview

This workflow automates the creation and publishing of SEO-optimized WordPress content using AI-generated outlines and text, Google Docs for content drafts, and Google Sheets for data management. It integrates multiple AI language models, including Anthropic AI and Google Gemini, to generate content outlines, descriptions, internal links, and HTML versions. The workflow also manages content status and metadata updates in Google Sheets and handles media cover images via HTTP requests.

Logical blocks included:

- **1.1 Input Reception & Initialization:** Receives external triggers via webhook and retrieves initial information from Google Sheets.
- **1.2 Content Outline Generation:** Uses Anthropic AI and auxiliary tools to generate structured content outlines from the input.
- **1.3 Content Drafting & Description:** Produces short descriptions and drafts content in Google Docs leveraging Google Gemini AI.
- **1.4 Content Enrichment:** Adds internal links and generates HTML versions for SEO optimization, again using Google Gemini AI.
- **1.5 Media and Image Handling:** Fetches and manages cover images for the posts.
- **1.6 Data Aggregation & Status Updates:** Aggregates various data, updates Google Sheets with new content statuses and metadata.
- **1.7 Error Handling & Retry Logic:** Includes nodes configured with retry and error continuation strategies.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Initialization

- **Overview:**  
  This block handles the start of the workflow by receiving webhook requests, fetching initial data from Google Sheets, and updating status fields to track progress.

- **Nodes Involved:**  
  - Webhook1  
  - Get Info  
  - Update status

- **Node Details:**

  - **Webhook1**  
    - *Type:* Webhook (Trigger)  
    - *Role:* Entry point to receive external HTTP requests initiating the workflow.  
    - *Configuration:* Uses a unique webhook ID for secure reception.  
    - *Connections:* Outputs to Get Info.  
    - *Failure Modes:* Network issues, invalid payloads, or unauthorized access could prevent trigger.  
    - *Version:* 2

  - **Get Info**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves relevant data from a Google Sheet based on webhook input parameters.  
    - *Configuration:* Connects to a specific Google Sheet and worksheet, likely filtering rows.  
    - *Connections:* Outputs to Update status.  
    - *Failure Modes:* Authentication errors with Google Sheets API, missing data, or rate limits.  
    - *Version:* 4.5

  - **Update status**  
    - *Type:* Google Sheets  
    - *Role:* Updates the status of the content row to reflect it is being processed or initialized.  
    - *Configuration:* Updates specific fields in the sheet to track workflow progress.  
    - *Connections:* Outputs to Fetch Outline.  
    - *Failure Modes:* Same as Get Info with API authentication and data integrity concerns.  
    - *Version:* 4.5

---

#### 1.2 Content Outline Generation

- **Overview:**  
  This block generates a detailed content outline and on-page SEO structure using AI models and auxiliary tools, laying the foundation for content drafting.

- **Nodes Involved:**  
  - Fetch Outline  
  - Anthropic Chat Model1  
  - Perplexity_Sonar_Pro1  
  - Content Outline Generator & On-page

- **Node Details:**

  - **Fetch Outline**  
    - *Type:* Google Docs  
    - *Role:* Retrieves the existing outline document from Google Docs to be enhanced or used as input.  
    - *Configuration:* Accesses specified Google Docs document by ID.  
    - *Connections:* Outputs to Content Outline Generator & On-page.  
    - *Failure Modes:* Document not found, permission issues.  
    - *Version:* 2

  - **Anthropic Chat Model1**  
    - *Type:* AI Language Model (Anthropic)  
    - *Role:* Provides AI-generated suggestions or completions for outline content.  
    - *Configuration:* Uses Anthropic chat model with default or customized prompt settings.  
    - *Connections:* Feeds into Content Outline Generator & On-page as language model resource.  
    - *Failure Modes:* API authentication errors, rate limits, timeouts.  
    - *Version:* 1.3

  - **Perplexity_Sonar_Pro1**  
    - *Type:* HTTP Request Tool  
    - *Role:* Likely used to query an external SEO or content analysis API to enrich outline generation.  
    - *Configuration:* Custom HTTP requests with authentication headers or tokens.  
    - *Connections:* Feeds into Content Outline Generator & On-page as auxiliary AI tool.  
    - *Failure Modes:* HTTP errors, authentication failures, API downtime.  
    - *Version:* 4.2

  - **Content Outline Generator & On-page**  
    - *Type:* Langchain Agent Node  
    - *Role:* Central AI agent combining inputs from Anthropic and Perplexity to generate a structured content outline and on-page SEO elements.  
    - *Configuration:* Uses Langchain agent framework to orchestrate multiple AI models and tools.  
    - *Connections:* Outputs to Short Description.  
    - *Retry Logic:* Retries on failure with 5-second delay, continues on error by default.  
    - *Failure Modes:* AI response errors, timeout, data formatting issues.  
    - *Version:* 1.7

---

#### 1.3 Content Drafting & Description

- **Overview:**  
  Generates short descriptions and drafts full content in Google Docs using Google Gemini AI models, preparing content for publishing.

- **Nodes Involved:**  
  - Short Description  
  - Google Gemini Chat Model4  
  - Create Docs: Save Content

- **Node Details:**

  - **Short Description**  
    - *Type:* Chain LLM (Google Gemini)  
    - *Role:* Produces a concise content summary or meta description from the outline.  
    - *Configuration:* Uses chain of prompts to refine output.  
    - *Connections:* Outputs to Create Docs: Save Content.  
    - *Failure Modes:* AI generation errors, prompt misconfigurations.  
    - *Version:* 1.6

  - **Google Gemini Chat Model4**  
    - *Type:* AI Language Model (Google Gemini)  
    - *Role:* Language model supporting the Short Description node with chat-based completions.  
    - *Connections:* Feeds Short Description node as language model.  
    - *Failure Modes:* API issues similar to Anthropic node.  
    - *Version:* 1

  - **Create Docs: Save Content**  
    - *Type:* Google Docs  
    - *Role:* Creates or updates a Google Docs document with the drafted content.  
    - *Configuration:* Writes content into Google Docs, possibly creating new documents or updating existing ones.  
    - *Connections:* Outputs to Update Google Doc.  
    - *Failure Modes:* Permissions, document lock or concurrency issues.  
    - *Version:* 2

---

#### 1.4 Content Enrichment

- **Overview:**  
  Enhances content by adding internal links and generating SEO-friendly HTML versions, driven by Google Gemini AI models.

- **Nodes Involved:**  
  - Previous Posts1  
  - Aggregate4  
  - Add Internal Links  
  - Google Gemini Chat Model8  
  - HTML Version  
  - Google Gemini Chat Model7  
  - HTTP Request9  
  - Image Covers1  
  - Merge1  
  - Aggregate6  
  - Merge2  
  - Update Status to Sheet

- **Node Details:**

  - **Previous Posts1**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves data about previous posts for internal linking context.  
    - *Connections:* Outputs to Aggregate4.  
    - *Failure Modes:* API or data consistency issues.

  - **Aggregate4**  
    - *Type:* Aggregate  
    - *Role:* Aggregates previous posts data to prepare for linking.  
    - *Connections:* Outputs to Add Internal Links.

  - **Add Internal Links**  
    - *Type:* Chain LLM (Google Gemini)  
    - *Role:* Uses AI to insert internal links into the drafted content based on aggregated data.  
    - *Connections:* Outputs to HTML Version.  
    - *Failure Modes:* AI misinterpretation, output formatting errors.

  - **Google Gemini Chat Model8**  
    - *Type:* AI Language Model (Google Gemini)  
    - *Role:* Language model supporting Add Internal Links node.

  - **HTML Version**  
    - *Type:* Chain LLM (Google Gemini)  
    - *Role:* Converts enriched content into an SEO-optimized HTML format.  
    - *Connections:* Outputs to HTTP Request9 and Image Covers1.

  - **Google Gemini Chat Model7**  
    - *Type:* AI Language Model (Google Gemini)  
    - *Role:* Supports HTML Version node.

  - **HTTP Request9**  
    - *Type:* HTTP Request  
    - *Role:* Possibly sends HTML content to an external API or service for validation or further processing.

  - **Image Covers1**  
    - *Type:* HTTP Request  
    - *Role:* Requests or uploads cover images related to the content.  
    - *Connections:* Outputs to Merge1 and Google Sheets3.

  - **Merge1**  
    - *Type:* Merge  
    - *Role:* Combines outputs from Image Covers1 and Code4/Code5 nodes for final aggregation.

  - **Aggregate6**  
    - *Type:* Aggregate  
    - *Role:* Consolidates merged data before final update.

  - **Merge2**  
    - *Type:* Merge  
    - *Role:* Final merge before updating Google Sheets.

  - **Update Status to Sheet**  
    - *Type:* Google Sheets  
    - *Role:* Updates content status and metadata in the Google Sheet to reflect completion.

---

#### 1.5 Data Aggregation & Transformation

- **Overview:**  
  This block processes and transforms data for internal linking, media handling, and final content composition through code and aggregation nodes.

- **Nodes Involved:**  
  - Code3  
  - Split Out2  
  - HTTP Request6  
  - HTTP Request4  
  - Aggregate5  
  - Code4  
  - Code5  
  - Split Out3  
  - HTTP Request8  
  - HTTP Request7  
  - Aggregate7  
  - Edit Fields1  
  - Google Sheets3

- **Node Details:**

  - **Code3, Code4, Code5**  
    - *Type:* Code  
    - *Role:* Custom JavaScript code nodes for data manipulation, formatting, or filtering.  
    - *Connections:* Chain linked to split, HTTP requests, merges, and aggregates.  
    - *Failure Modes:* Code errors, exceptions, or unexpected input structure.

  - **Split Out2, Split Out3**  
    - *Type:* Split Out  
    - *Role:* Splits data streams for parallel processing in HTTP requests.

  - **HTTP Request4, HTTP Request6, HTTP Request7, HTTP Request8**  
    - *Type:* HTTP Request  
    - *Role:* Calls to external APIs for media, SEO, or content enrichment.

  - **Aggregate5, Aggregate7**  
    - *Type:* Aggregate  
    - *Role:* Collects and consolidates results from parallel HTTP requests or processing nodes.

  - **Edit Fields1**  
    - *Type:* Set  
    - *Role:* Edits or sets data fields before final aggregation.

  - **Google Sheets3**  
    - *Type:* Google Sheets  
    - *Role:* Reads or writes additional metadata or logs related to the content.

---

#### 1.6 Finalization & Status Update

- **Overview:**  
  Completes the workflow by merging all data, updating Google Sheets with final content status, and closing the loop.

- **Nodes Involved:**  
  - Merge2  
  - Update Status to Sheet

- **Node Details:**

  - **Merge2**  
    - *Type:* Merge  
    - *Role:* Final merge point for all processed data streams before updating the sheet.  
    - *Connections:* Outputs to Update Status to Sheet.

  - **Update Status to Sheet**  
    - *Type:* Google Sheets  
    - *Role:* Writes final statuses and metadata to track the publishing process as completed or updated.  
    - *Failure Modes:* API errors, write conflicts.

---

#### 1.7 Error Handling & Retry Logic

- **Overview:**  
  Nodes such as the Content Outline Generator & On-page include retry configurations to handle transient AI or network errors gracefully.

- **Details:**  
  - Retry attempts with 5-second intervals.  
  - On error, continue with regular output to avoid workflow stoppage.

---

### 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                                  | Input Node(s)             | Output Node(s)                    | Sticky Note                          |
|---------------------------------|-------------------------------|-------------------------------------------------|---------------------------|----------------------------------|------------------------------------|
| Webhook1                        | Webhook                       | Entry point for external trigger                |                           | Get Info                         |                                    |
| Get Info                       | Google Sheets                 | Retrieves initial data                           | Webhook1                  | Update status                   |                                    |
| Update status                  | Google Sheets                 | Updates processing status                        | Get Info                  | Fetch Outline                   |                                    |
| Fetch Outline                  | Google Docs                  | Retrieves content outline document               | Update status             | Content Outline Generator & On-page |                                    |
| Anthropic Chat Model1          | AI Language Model (Anthropic) | Provides AI-generated outline support           |                           | Content Outline Generator & On-page |                                    |
| Perplexity_Sonar_Pro1          | HTTP Request Tool            | External SEO tool integration                     |                           | Content Outline Generator & On-page |                                    |
| Content Outline Generator & On-page | Langchain Agent Node          | Generates content outline and SEO structure      | Fetch Outline, Anthropic Chat Model1, Perplexity_Sonar_Pro1 | Short Description             |                                    |
| Short Description              | Chain LLM (Google Gemini)    | Produces short content description               | Content Outline Generator & On-page | Create Docs: Save Content       |                                    |
| Google Gemini Chat Model4      | AI Language Model (Google Gemini) | Supports short description generation            |                           | Short Description               |                                    |
| Create Docs: Save Content      | Google Docs                  | Creates/updates Google Docs with drafted content | Short Description         | Update Google Doc               |                                    |
| Update Google Doc              | Google Docs                  | Updates Google Sheet with document info          | Create Docs: Save Content | Update to Sheet                |                                    |
| Update to Sheet                | Google Sheets                 | Updates sheet with new content info               | Update Google Doc         | Previous Posts1                |                                    |
| Previous Posts1                | Google Sheets                 | Retrieves previous posts data                      | Update to Sheet           | Aggregate4                    |                                    |
| Aggregate4                    | Aggregate                    | Aggregates previous posts data                     | Previous Posts1           | Add Internal Links            |                                    |
| Add Internal Links            | Chain LLM (Google Gemini)    | Inserts internal links into content                | Aggregate4                | HTML Version                 |                                    |
| Google Gemini Chat Model8      | AI Language Model (Google Gemini) | Supports internal linking AI processing           |                           | Add Internal Links            |                                    |
| HTML Version                  | Chain LLM (Google Gemini)    | Generates SEO-optimized HTML content               | Add Internal Links         | HTTP Request9, Image Covers1  |                                    |
| Google Gemini Chat Model7      | AI Language Model (Google Gemini) | Supports HTML version generation                   |                           | HTML Version                 |                                    |
| HTTP Request9                 | HTTP Request                 | Sends or processes HTML content externally        | HTML Version              | Merge2                       |                                    |
| Image Covers1                | HTTP Request                 | Handles cover image API requests                    | HTML Version              | Merge1, Google Sheets3        |                                    |
| Merge1                       | Merge                       | Combines media and code outputs                     | Image Covers1, Code4, Code5 | Aggregate6                   |                                    |
| Aggregate6                   | Aggregate                   | Aggregates merged data before final update          | Merge1                    | Merge2                       |                                    |
| Merge2                       | Merge                       | Final merge before writing back to sheet            | Aggregate6, HTTP Request9 | Update Status to Sheet        |                                    |
| Update Status to Sheet        | Google Sheets                | Updates final content status                         | Merge2                    |                              |                                    |
| Code3                        | Code                        | Custom data processing and splitting                | Google Sheets3            | Split Out2, Split Out3        |                                    |
| Split Out2                   | Split Out                   | Splits data stream for parallel HTTP requests       | Code3                     | HTTP Request6                 |                                    |
| HTTP Request6                | HTTP Request                | Calls external API for content enrichment            | Split Out2                | HTTP Request4                |                                    |
| HTTP Request4                | HTTP Request                | Calls external API for further processing             | HTTP Request6             | Aggregate5                   |                                    |
| Aggregate5                  | Aggregate                  | Aggregates parallel HTTP request results              | HTTP Request4             | Code4                       |                                    |
| Code4                      | Code                      | Processes aggregated data                               | Aggregate5                | Merge1                      |                                    |
| Split Out3                 | Split Out                 | Splits data stream for second parallel HTTP requests | Code3                     | HTTP Request8                |                                    |
| HTTP Request8              | HTTP Request              | Calls another external API                              | Split Out3                | HTTP Request7                |                                    |
| HTTP Request7              | HTTP Request              | External API call for content or media processing       | HTTP Request8             | Edit Fields1                |                                    |
| Edit Fields1               | Set                       | Edits fields before final aggregation                    | HTTP Request7             | Aggregate7                  |                                    |
| Aggregate7                 | Aggregate                 | Aggregates edited data                                   | Edit Fields1              | Code5                       |                                    |
| Code5                     | Code                      | Final data transformations                               | Aggregate7                | Merge1                      |                                    |
| Google Sheets3             | Google Sheets              | Reads or writes metadata or logs                         | Image Covers1             | Code3                       |                                    |
| Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3 | Sticky Note                | Visual annotations in the workflow editor                |                           |                              | No content in notes; no comments |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - Configure unique webhook endpoint for external triggers.

2. **Add Google Sheets Node (Get Info)**  
   - Type: Google Sheets (Read)  
   - Configure credentials and select the sheet and worksheet for retrieving initial content data.  
   - Connect Webhook output to this node.

3. **Add Google Sheets Node (Update status)**  
   - Type: Google Sheets (Update)  
   - Set to update status fields indicating workflow start.  
   - Connect Get Info output to this node.

4. **Add Google Docs Node (Fetch Outline)**  
   - Type: Google Docs (Read)  
   - Configure to fetch existing content outline document by ID.  
   - Connect Update status output to this node.

5. **Add Anthropic Chat Model Node**  
   - Type: Langchain AI Language Model (Anthropic)  
   - Configure Anthropic API credentials.  
   - Connect as AI language model input to Content Outline Generator node.

6. **Add HTTP Request Node (Perplexity_Sonar_Pro1)**  
   - Type: HTTP Request  
   - Configure for external SEO analysis API with necessary authentication.  
   - Connect as AI tool input to Content Outline Generator node.

7. **Add Langchain Agent Node (Content Outline Generator & On-page)**  
   - Type: Langchain Agent  
   - Use Anthropic Chat Model and Perplexity_Sonar_Pro1 as inputs.  
   - Configure retry policy: retry on failure with 5 seconds delay, continue on error.  
   - Connect Fetch Outline output to this node.  
   - Output connects to Short Description node.

8. **Add Google Gemini Chat Model Node (Google Gemini Chat Model4)**  
   - Type: AI Language Model (Google Gemini)  
   - Configure Google Gemini API credentials.  
   - Connect as language model input to Short Description node.

9. **Add Chain LLM Node (Short Description)**  
   - Type: Chain LLM  
   - Configure prompts to generate concise content descriptions from outline.  
   - Connect Content Outline Generator output as input.  
   - Output connects to Create Docs: Save Content node.

10. **Add Google Docs Node (Create Docs: Save Content)**  
    - Type: Google Docs (Write)  
    - Configure to create or update document with generated content.  
    - Connect Short Description output to this node.  
    - Output connects to Update Google Doc.

11. **Add Google Docs Node (Update Google Doc)**  
    - Type: Google Docs (Update)  
    - Update Google Sheets with document references or metadata.  
    - Connect Create Docs output to this node.  
    - Output connects to Update to Sheet.

12. **Add Google Sheets Node (Update to Sheet)**  
    - Type: Google Sheets (Update)  
    - Update relevant rows with content metadata and status.  
    - Connect Update Google Doc output to this node.  
    - Output connects to Previous Posts1.

13. **Add Google Sheets Node (Previous Posts1)**  
    - Type: Google Sheets (Read)  
    - Retrieve data on previous posts for internal linking.  
    - Connect Update to Sheet output to this node.  
    - Output connects to Aggregate4.

14. **Add Aggregate Node (Aggregate4)**  
    - Type: Aggregate  
    - Aggregate previous posts data for linking.  
    - Connect Previous Posts1 output here.  
    - Output connects to Add Internal Links.

15. **Add Google Gemini Chat Model Node (Google Gemini Chat Model8)**  
    - Type: AI Language Model (Google Gemini)  
    - Configure credentials.  
    - Connect as language model input to Add Internal Links.

16. **Add Chain LLM Node (Add Internal Links)**  
    - Type: Chain LLM  
    - Configure prompts to add internal links to content.  
    - Connect Aggregate4 output as input.  
    - Output connects to HTML Version.

17. **Add Google Gemini Chat Model Node (Google Gemini Chat Model7)**  
    - Type: AI Language Model (Google Gemini)  
    - Configure credentials.  
    - Connect as language model input to HTML Version.

18. **Add Chain LLM Node (HTML Version)**  
    - Type: Chain LLM  
    - Configure prompts to generate HTML SEO optimized content version.  
    - Connect Add Internal Links output as input.  
    - Output connects to HTTP Request9 and Image Covers1.

19. **Add HTTP Request Node (HTTP Request9)**  
    - Type: HTTP Request  
    - Configure to send or validate HTML content externally.  
    - Connect HTML Version output here.  
    - Output connects to Merge2.

20. **Add HTTP Request Node (Image Covers1)**  
    - Type: HTTP Request  
    - Configure to fetch or upload cover images for posts.  
    - Connect HTML Version output here.  
    - Output connects to Merge1 and Google Sheets3.

21. **Add Google Sheets Node (Google Sheets3)**  
    - Type: Google Sheets  
    - Configure for metadata or media logs.  
    - Connect Image Covers1 output here.  
    - Output connects to Code3.

22. **Add Code Node (Code3)**  
    - Type: Code  
    - Add custom JavaScript logic to process image and metadata data streams.  
    - Connect Google Sheets3 output here.  
    - Output connects to Split Out2 and Split Out3.

23. **Add Split Out Nodes (Split Out2 and Split Out3)**  
    - Type: Split Out  
    - Split data for parallel HTTP requests.  
    - Connect Code3 outputs accordingly.

24. **Add HTTP Request Nodes (HTTP Request6, HTTP Request8)**  
    - Configure for external API calls related to content enrichment or media.  
    - Connect Split Out nodes as inputs.

25. **Add HTTP Request Nodes (HTTP Request4, HTTP Request7)**  
    - Further HTTP calls chained from previous requests.  
    - Connect HTTP Request6 output to HTTP Request4.  
    - Connect HTTP Request8 output to HTTP Request7.

26. **Add Aggregate Nodes (Aggregate5, Aggregate7)**  
    - Aggregate results of HTTP calls.  
    - Connect HTTP Request4 output to Aggregate5.  
    - Connect Edit Fields1 output to Aggregate7.

27. **Add Code Nodes (Code4, Code5)**  
    - Process aggregated data for final composition.  
    - Connect Aggregate5 to Code4, Aggregate7 to Code5.

28. **Add Set Node (Edit Fields1)**  
    - Modify or set fields in data before final aggregation.  
    - Connect HTTP Request7 output here.  
    - Output connects to Aggregate7.

29. **Add Merge Node (Merge1)**  
    - Merge outputs from Image Covers1, Code4, and Code5.  
    - Connect Image Covers1, Code4, and Code5 outputs here.  
    - Output connects to Aggregate6.

30. **Add Aggregate Node (Aggregate6)**  
    - Aggregate merged data for final processing.  
    - Connect Merge1 output here.  
    - Output connects to Merge2.

31. **Add Merge Node (Merge2)**  
    - Final merge combining Aggregate6 and HTTP Request9 outputs.  
    - Connect Aggregate6 and HTTP Request9 outputs here.  
    - Output connects to Update Status to Sheet.

32. **Add Google Sheets Node (Update Status to Sheet)**  
    - Final update of content status and metadata in Google Sheets.  
    - Connect Merge2 output here.

33. **Configure Credentials**  
    - Set up API credentials for:  
      - Google Sheets (OAuth2)  
      - Google Docs (OAuth2)  
      - Anthropic AI (API Key)  
      - Google Gemini AI (API Key)  
      - Any external HTTP APIs used (API keys or tokens).

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                          |
|-------------------------------------------------------------------------------------------------|----------------------------------------|
| Workflow automates SEO content creation leveraging Anthropic and Google Gemini AI models.       | Workflow purpose                       |
| Uses Google Workspace (Sheets & Docs) for data management and content drafting.                  | Integration details                    |
| Includes retry and error continuation in AI nodes to ensure robustness.                         | Reliability design                     |
| For Anthropic AI and Google Gemini, ensure API keys are valid and have sufficient quotas.       | Credential management                  |
| External SEO tools (Perplexity Sonar Pro) and media APIs require stable internet connectivity.  | Integration requirements               |
| Sticky notes in the workflow have no content and can be ignored.                                | Workflow annotations                   |

---

Disclaimer: The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data are legal and public.

---