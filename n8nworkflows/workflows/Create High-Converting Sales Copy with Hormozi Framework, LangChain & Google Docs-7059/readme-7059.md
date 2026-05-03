Create High-Converting Sales Copy with Hormozi Framework, LangChain & Google Docs

https://n8nworkflows.xyz/workflows/create-high-converting-sales-copy-with-hormozi-framework--langchain---google-docs-7059


# Create High-Converting Sales Copy with Hormozi Framework, LangChain & Google Docs

### 1. Workflow Overview

This workflow automates the creation of high-converting sales copy in the style of Alex Hormozi, utilizing LangChain AI agents, Google Docs integration, and structured customer insights. It is designed to generate multiple unique advertorial documents optimized for different customer personas, leveraging customer reviews, product features, and proven psychological frameworks such as Maslow’s Hierarchy of Needs and Hormozi’s 4 Value Levers.

The workflow logic is organized into these main blocks:

- **1.1 Input Reception & Data Acquisition:** Captures form inputs, fetches base copy, product features, and customer reviews.
- **1.2 Customer Insight Extraction & Persona Development:** Processes raw reviews to create marketing insights and define customer personas mapped to Maslow’s hierarchy.
- **1.3 Headline Generation:** Produces multiple headline and subheadline combinations using Hormozi’s value levers and customer language.
- **1.4 Advertorial Copywriting:** Generates tailored advertorial content for each persona-headline pair, strictly following a reference structure.
- **1.5 Document Creation & Insertion:** Creates Google Docs files for customer analysis, headline options, and advertorial copies, inserting generated content accordingly.
- **1.6 Workflow Orchestration & Control:** Manages data flow, batching, and conditional logic to ensure smooth execution and scalable output quantity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Acquisition

**Overview:**  
This block gathers all necessary inputs from a user submission form, fetches the base advertorial draft copy, product features, and Amazon product reviews to prepare raw data for analysis.

**Nodes Involved:**  
- On form submission  
- Get Base Copy  
- Get Product Features  
- Get Amazon Reviews  
- If Reviews Exist  
- Randomize Reviews  
- Set Limit of Reviews to use  
- Edit Fields (review formatting)  
- Aggregate Reviews  

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; collects user inputs like folder IDs, file names, product name, number of copies, URLs for base copy, reviews sheet, and product features doc.  
  - *Key Parameters:* Form fields including Customer Analysis Folder ID, Advertorial Copy Folder ID, Base copy Google Docs URL, Product Feature Doc URL, Number of reviews, Product Name, Target City, Number of Copies.  
  - *Input/Output:* Outputs form data to downstream nodes.  
  - *Edge Cases:* Missing required fields will block execution.  

- **Get Base Copy**  
  - *Type:* Google Docs (get operation)  
  - *Role:* Retrieves the base advertorial text document using URL from form input.  
  - *Input:* Document URL from form submission.  
  - *Output:* Full content of base copy for AI processing.  
  - *Failure Types:* Auth errors, invalid URL, network issues.  

- **Get Product Features**  
  - *Type:* Google Docs (get operation)  
  - *Role:* Fetches product features and USPs document for AI to reference.  
  - *Input:* Product Feature/USPs Doc URL from form.  
  - *Output:* Raw text content of product features.  
  - *Failure Types:* Same as above (auth, URL, network).  

- **Get Amazon Reviews**  
  - *Type:* Google Sheets (read)  
  - *Role:* Pulls raw review data from specified Google Sheets URL.  
  - *Input:* Sheet URL from form submission, fixed document ID.  
  - *Output:* Raw review rows with fields like review title, comment, star rating, date, and URL.  
  - *Failure Types:* Auth failures, invalid sheet URL, empty sheet.  

- **If Reviews Exist**  
  - *Type:* Condition (If)  
  - *Role:* Checks if review data is present before proceeding.  
  - *Branches:* Proceeds with data or halts workflow.  

- **Randomize Reviews**  
  - *Type:* Sort (random)  
  - *Role:* Randomizes review order for unbiased sampling.  

- **Set Limit of Reviews to Use**  
  - *Type:* Limit  
  - *Role:* Truncates number of reviews to user-defined maximum from form or defaults to 10.  

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Formats individual review fields into structured strings (e.g., concatenates title and body).  
  - *Output:* Cleaned review data for AI input.  

- **Aggregate Reviews**  
  - *Type:* Aggregate  
  - *Role:* Combines review text fields into a single aggregated string for analysis.  

---

#### 2.2 Customer Insight Extraction & Persona Development

**Overview:**  
Analyzes aggregated reviews via AI agents to extract marketing angles, customer motivations, barriers, product opportunities, VOC snippets, and develops detailed customer personas mapped to Maslow’s hierarchy.

**Nodes Involved:**  
- Customer Reviews Analysis Agent  
- Review Insights (Code)  
- Create Customer Analysis Docs  
- Insert Customer Analysis Text  
- Maslow Hierarchy Analysis  
- Structured Output Parser3  

**Node Details:**  

- **Customer Reviews Analysis Agent**  
  - *Type:* LangChain AI Agent  
  - *Role:* Processes raw reviews to output structured marketing insights in five categories.  
  - *Input:* Aggregated review text.  
  - *Output:* Text block with marketing insights and VOC snippets.  
  - *Failure Types:* API limits, parsing errors, incomplete input.  

- **Review Insights**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parses AI output into separate arrays for marketing angles, motivations, frictions, opportunities, VOC snippets.  
  - *Output:* JSON with named arrays for downstream nodes.  

- **Create Customer Analysis Docs**  
  - *Type:* Google Docs (create)  
  - *Role:* Creates a new Google Doc for storing customer insights.  
  - *Input:* Folder ID and file name from form.  
  - *Output:* Document ID for update.  

- **Insert Customer Analysis Text**  
  - *Type:* Google Docs (update)  
  - *Role:* Inserts parsed marketing insights into the newly created customer analysis doc.  

- **Maslow Hierarchy Analysis**  
  - *Type:* LangChain AI Agent  
  - *Role:* Analyzes reviews and customer analysis output to segment personas aligned to Maslow’s hierarchy, generating rich personas with emotional context and testimonial quotes.  
  - *Input:* Raw reviews and customer analysis JSON.  
  - *Output:* JSON array of personas.  
  - *Failure Types:* Complex parsing errors, API timeouts.  

- **Structured Output Parser3**  
  - *Type:* LangChain Output Parser  
  - *Role:* Ensures Maslow Analysis output matches expected JSON schema and auto-fixes minor issues.  

---

#### 2.3 Headline Generation

**Overview:**  
Generates 10–15 headline and subheadline packages that appeal to fundamental human desires and product features, using Hormozi’s value levers and customer voice.

**Nodes Involved:**  
- Headline Writer (LangChain Agent)  
- Structured Output Parser1  
- Edit Fields2  
- Create Headline File  
- Insert Headlines  

**Node Details:**  

- **Headline Writer**  
  - *Type:* LangChain AI Agent  
  - *Role:* Produces headline packages by combining customer personas, product features, VOC, and base copy tone.  
  - *Input:* Base copy content, product features, customer analysis, personas JSON.  
  - *Output:* JSON object containing array of headline packages and voice notes.  
  - *Failure Types:* API timeout, malformed input.  

- **Structured Output Parser1**  
  - *Type:* LangChain Output Parser  
  - *Role:* Validates and structures headline output JSON.  

- **Edit Fields2**  
  - *Type:* Set  
  - *Role:* Assigns static test values for the target city and product name (may be for testing or defaulting).  

- **Create Headline File**  
  - *Type:* Google Docs (create)  
  - *Role:* Creates a Google Doc for headline options.  

- **Insert Headlines**  
  - *Type:* Google Docs (update)  
  - *Role:* Inserts the generated headline packages JSON into the headline options doc.  

---

#### 2.4 Advertorial Copywriting

**Overview:**  
Creates multiple unique advertorial copies tailored to different persona-headline pairs, strictly preserving the structure and tone of the reference copy using Hormozi’s framework.

**Nodes Involved:**  
- Sales Page Copywriter (LangChain Agent)  
- Structured Output Parser4  
- Split Out  
- Loop Over Items  
- Create Advertorial Docs  
- Insert Advertorial  

**Node Details:**  

- **Sales Page Copywriter**  
  - *Type:* LangChain AI Agent  
  - *Role:* Plans and writes multiple advertorial copies, matching personas with headlines, applying Hormozi’s 4 value levers, and maintaining schema fidelity.  
  - *Input:* Advertorial JSON output, headline packages, personas, number of copies, product name, target demographics, product features, customer testimonials.  
  - *Output:* Array of advertorial variants with copy number, target persona, headline, and full copy JSON.  
  - *Failure Types:* Overload from too many copies, API limits.  

- **Structured Output Parser4**  
  - *Type:* LangChain Output Parser  
  - *Role:* Validates and auto-fixes advertorial copies array output to conform to a strict JSON schema.  

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the array of advertorial copies into individual items for sequential processing.  

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes each advertorial copy separately, enabling batch limits for Google Docs creation.  

- **Create Advertorial Docs**  
  - *Type:* Google Docs (create)  
  - *Role:* Creates a new Google Doc file for each advertorial copy, using the copy number and file name pattern.  

- **Insert Advertorial**  
  - *Type:* Google Docs (update)  
  - *Role:* Inserts the full JSON advertorial copy content as text into the respective Google Doc.  

---

#### 2.5 Workflow Orchestration & Control

**Overview:**  
Manages data flow between nodes, controls branching based on input presence, and ensures orderly execution of the multi-step process.

**Nodes Involved:**  
- Think (LangChain Tool)  
- Think5 (LangChain Tool)  
- Sticky Notes (various for documentation)  

**Node Details:**  

- **Think** & **Think5**  
  - *Type:* LangChain Tool (Think)  
  - *Role:* Used internally by AI agents for complex reasoning and planning before text generation.  
  - *Connections:* Used by multiple AI agents for enhanced output quality.  

- **Sticky Notes**  
  - *Type:* n8n Sticky Note  
  - *Role:* Provide visual organizational comments and instructions within the workflow editor for ease of understanding and maintenance.  

---

### 3. Summary Table

| Node Name                   | Node Type                           | Functional Role                                   | Input Node(s)                          | Output Node(s)                         | Sticky Note                                                        |
|-----------------------------|-----------------------------------|--------------------------------------------------|--------------------------------------|--------------------------------------|------------------------------------------------------------------|
| On form submission           | Form Trigger                      | Entry point, collect user inputs                  | -                                    | Get Base Copy                        | Start here (required settings)                                   |
| Get Base Copy               | Google Docs                      | Retrieve base advertorial draft                   | On form submission                   | Get Product Features                 | Get Base Copy                                                   |
| Get Product Features        | Google Docs                      | Retrieve product features document                 | Get Base Copy                       | Get Amazon Reviews                  | Get Product Features                                            |
| Get Amazon Reviews          | Google Sheets                    | Fetch product reviews spreadsheet                  | Get Product Features                | If Reviews Exist                    | Get Amazon Customer Reviews                                     |
| If Reviews Exist            | If                              | Check reviews presence                              | Get Amazon Reviews                  | Randomize Reviews / End             |                                                                  |
| Randomize Reviews           | Sort (random)                   | Randomize review records                            | If Reviews Exist                    | Set Limit of Reviews to use         |                                                                  |
| Set Limit of Reviews to use | Limit                          | Limit number of reviews to user-defined max       | Randomize Reviews                   | Edit Fields                       |                                                                  |
| Edit Fields                | Set                            | Format review fields                                | Set Limit of Reviews to use         | Aggregate Reviews                  |                                                                  |
| Aggregate Reviews          | Aggregate                      | Aggregate review text for AI input                 | Edit Fields                       | Customer Reviews Analysis Agent    |                                                                  |
| Customer Reviews Analysis Agent | LangChain Agent              | Extract marketing insights from reviews            | Aggregate Reviews                  | Review Insights                   | Analyze VOC                                                     |
| Review Insights            | Code                           | Parse AI insight output into categories            | Customer Reviews Analysis Agent    | Create Customer Analysis Docs      |                                                                  |
| Create Customer Analysis Docs | Google Docs                   | Create Google Doc for customer insights            | Review Insights                   | Insert Customer Analysis Text      |                                                                  |
| Insert Customer Analysis Text | Google Docs                  | Insert marketing insights text into doc            | Create Customer Analysis Docs      | Advertorial Writer                |                                                                  |
| Maslow Hierarchy Analysis   | LangChain Agent                | Segment personas aligned to Maslow hierarchy       | Advertorial Writer / Review Insights | Structured Output Parser3          |                                                                  |
| Structured Output Parser3   | LangChain Output Parser        | Validate persona JSON output                         | Maslow Hierarchy Analysis          | Headline Writer                  |                                                                  |
| Headline Writer             | LangChain Agent                | Generate headline & subheadline packages            | Structured Output Parser3 / Review Insights | Edit Fields2 / Create Headline File |                                                                  |
| Edit Fields2                | Set                            | Assign static or default values for variables       | Headline Writer                   | Sales Page Copywriter            |                                                                  |
| Create Headline File        | Google Docs                   | Create Google Doc for headline options              | Edit Fields2                     | Insert Headlines                |                                                                  |
| Insert Headlines            | Google Docs                   | Insert generated headlines JSON into doc            | Create Headline File             | -                                |                                                                  |
| Sales Page Copywriter       | LangChain Agent                | Write multiple advertorial copies per persona-headline pair | Edit Fields2 / Headline Writer / Maslow Hierarchy Analysis | Structured Output Parser4          |                                                                  |
| Structured Output Parser4   | LangChain Output Parser        | Validate advertorial copies JSON output              | Sales Page Copywriter             | Split Out                     |                                                                  |
| Split Out                  | Split Out                      | Split array of advertorial copies                    | Structured Output Parser4          | Loop Over Items                 |                                                                  |
| Loop Over Items             | Split In Batches              | Process each advertorial copy separately             | Split Out                       | Create Advertorial Docs / End      |                                                                  |
| Create Advertorial Docs     | Google Docs                   | Create Google Doc for each advertorial copy          | Loop Over Items                   | Insert Advertorial              |                                                                  |
| Insert Advertorial          | Google Docs                   | Insert advertorial JSON text into respective doc     | Create Advertorial Docs           | Loop Over Items (continue)      |                                                                  |
| Think                      | LangChain Tool (Think)          | AI internal reasoning tool                           | -                              | Customer Reviews Analysis Agent     |                                                                  |
| Think5                     | LangChain Tool (Think)          | AI internal reasoning tool for multiple agents       | -                              | Headline Writer, Maslow Hierarchy Analysis, Sales Page Copywriter, Advertorial Writer |                                                                  |
| Sticky Note(s)             | Sticky Note                    | Visual documentation and instructions                | -                              | -                              | Multiple notes for organizational context (e.g., "Write Advertorial") |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form fields:  
     - Customer Analysis Folder ID (string, required)  
     - 10x Sales Copy Folder ID (string, required)  
     - File Name (string, required)  
     - Base copy (Google Docs URL) (string, required)  
     - Target City (string, required)  
     - Number of Copies (number, required)  
     - Product Name (string, required)  
     - URL of Product Reviews Tab (Google Sheets URL, required)  
     - Number of reviews to use (number, required)  
     - Product Feature/USPs Doc URL (string, required)  

2. **Add Google Docs Node "Get Base Copy"**  
   - Operation: Get  
   - Document URL: Use expression from form field "Base copy (Google Docs URL)"  
   - Credential: Google Docs OAuth2  

3. **Add Google Docs Node "Get Product Features"**  
   - Operation: Get  
   - Document URL: From form field "Product Feature/USPs Doc"  
   - Credential: Google Docs OAuth2  
   - Connect output of Get Base Copy to this node.  

4. **Add Google Sheets Node "Get Amazon Reviews"**  
   - Configure to read sheet from form field "URL of Product Reviews Tab (Google Sheets)"  
   - Use appropriate Document ID and Sheet Name extracted from URL  
   - Credential: Google Sheets OAuth2  
   - Connect output of Get Product Features to this node.  

5. **Add If Node "If Reviews Exist"**  
   - Condition: Check if review data is non-empty  
   - Connect output of Get Amazon Reviews to this node.  

6. **Add Sort Node "Randomize Reviews"**  
   - Sort Type: Random  
   - Connect If Reviews Exist True branch here.  

7. **Add Limit Node "Set Limit of Reviews to use"**  
   - Max Items: Expression from form field "Number of reviews to use" or default 10  
   - Connect Randomize Reviews output here.  

8. **Add Set Node "Edit Fields"**  
   - Create fields: Review ID, Review (concatenate Title and Body), Rating Score, Review URL, Date  
   - Connect output of Limit node here.  

9. **Add Aggregate Node "Aggregate Reviews"**  
   - Aggregate the Review field into a single string for AI input.  
   - Connect Edit Fields output here.  

10. **Add LangChain Agent "Customer Reviews Analysis Agent"**  
    - Purpose: Analyze aggregated reviews for marketing insights.  
    - Use system message instructing extraction of marketing angles, motivations, barriers, opportunities, VOC snippets.  
    - Connect Aggregate Reviews output here.  
    - Use OpenRouter or OpenAI credentials as appropriate.  

11. **Add Code Node "Review Insights"**  
    - Parse AI output into arrays for each insight category.  
    - Connect Customer Reviews Analysis Agent output here.  

12. **Add Google Docs Node "Create Customer Analysis Docs"**  
    - Operation: Create  
    - Folder ID and Title from form fields.  
    - Connect Review Insights output here.  

13. **Add Google Docs Node "Insert Customer Analysis Text"**  
    - Operation: Update  
    - Document URL from Create Customer Analysis Docs output  
    - Insert marketing insights text from Review Insights node.  

14. **Add LangChain Agent "Maslow Hierarchy Analysis"**  
    - Input: Raw reviews + customer analysis JSON  
    - Task: Generate personas aligned to Maslow’s hierarchy with emotional context.  
    - Connect Insert Customer Analysis Text output here.  

15. **Add LangChain Output Parser Node**  
    - Validate Maslow Hierarchy Analysis output JSON.  

16. **Add LangChain Agent "Headline Writer"**  
    - Input: Base copy content, product features, customer analysis, personas JSON.  
    - Task: Generate 10-15 headline + subheadline packages using Hormozi levers.  
    - Connect Maslow Hierarchy Analysis output here.  

17. **Add LangChain Output Parser Node**  
    - Validate Headline Writer output.  

18. **Add Set Node "Edit Fields2"**  
    - Assign default or test values for "target city" and "product name".  
    - Connect Headline Writer output here.  

19. **Add Google Docs Node "Create Headline File"**  
    - Operation: Create  
    - Folder ID and Title from form fields.  
    - Connect Edit Fields2 output here.  

20. **Add Google Docs Node "Insert Headlines"**  
    - Operation: Update  
    - Document URL from Create Headline File output.  
    - Insert headline packages JSON from Headline Writer output.  

21. **Add LangChain Agent "Sales Page Copywriter"**  
    - Inputs: Advertorial JSON, headline packages, personas, number of copies, product name, demographics, features, testimonials  
    - Task: Generate multiple unique advertorial copies applying Hormozi framework per persona-headline pair.  
    - Connect Edit Fields2 output here.  

22. **Add LangChain Output Parser Node**  
    - Validate Sales Page Copywriter output JSON array.  

23. **Add Split Out Node**  
    - Split array of advertorial copies into individual items.  
    - Connect Structured Output Parser4 output here.  

24. **Add Split In Batches Node "Loop Over Items"**  
    - Process each advertorial copy sequentially or in batches.  
    - Connect Split Out output here.  

25. **Add Google Docs Node "Create Advertorial Docs"**  
    - Operation: Create  
    - Title: Combine file name + copy number  
    - Folder ID from form field "Advertorial Copy Folder ID"  
    - Connect Loop Over Items output here.  

26. **Add Google Docs Node "Insert Advertorial"**  
    - Operation: Update  
    - Document URL: from Create Advertorial Docs output  
    - Insert advertorial JSON copy as text.  
    - Connect Create Advertorial Docs output here.  

27. **Add Think Tool Nodes as required**  
    - Add LangChain Tool Think nodes to support complex reasoning by AI agents.  
    - Connect them accordingly to AI agent nodes for enhanced performance.  

28. **Add Sticky Notes**  
    - Add sticky notes in the editor for documentation and workflow clarity (optional but recommended).  

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                           |
|------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow strictly follows the Hormozi 4 Value Levers framework combined with Maslow's Hierarchy of Needs for persuasive copywriting. | Core methodology of the workflow.                          |
| The workflow uses LangChain agents connected via the OpenRouter API, specifically the "anthropic/claude-sonnet-4" model for generation.  | AI model info: https://openrouter.ai/models/anthropic/claude-sonnet-4 |
| Ensure Google Docs and Sheets folders have correct sharing permissions for the OAuth2 credentials used.                                 | Important for successful file creation and access.         |
| The workflow produces JSON-structured advertorials to maintain strict schema fidelity and allow easy downstream automation or editing. | Workflow output format best practice.                      |
| Sticky notes within the workflow provide guidance for each logical section and recommended customization points.                        | See sticky notes in the n8n editor for details.             |

---

**Disclaimer:** The provided text and workflow components are derived exclusively from an automated n8n workflow. All processing complies with applicable content policies, excludes illegal or offensive content, and utilizes only public and lawful data.