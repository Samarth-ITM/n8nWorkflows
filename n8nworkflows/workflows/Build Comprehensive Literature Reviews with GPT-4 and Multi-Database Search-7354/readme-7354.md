Build Comprehensive Literature Reviews with GPT-4 and Multi-Database Search

https://n8nworkflows.xyz/workflows/build-comprehensive-literature-reviews-with-gpt-4-and-multi-database-search-7354


# Build Comprehensive Literature Reviews with GPT-4 and Multi-Database Search

### 1. Workflow Overview

This workflow automates the creation of a comprehensive literature review on a specified topic using GPT-4 and multi-database academic search. It is designed to retrieve, filter, and synthesize scientific papers from several academic databases, parse their content, and generate a structured review section for each paper. The workflow logically divides into the following blocks:

- **1.1 Input Reception and Parameter Setup:** Collects the literature review parameters such as topic, publication year range, and maximum number of papers to review.
- **1.2 Multi-Database Paper Search:** Queries multiple academic databases simultaneously to retrieve relevant papers matching the topic and year constraints.
- **1.3 Paper Sorting and Selection:** Sorts retrieved papers by citation count to prioritize influential works and limits the selection to the top N papers.
- **1.4 Paper Content Parsing:** Parses the PDFs of the selected papers to extract textual content suitable for summarization.
- **1.5 Literature Review Synthesis:** Uses GPT-4 to generate key literature review sections from the parsed paper content.
- **1.6 Review Compilation and Export:** Combines all synthesized sections into a single literature review document and exports it as a markdown file.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parameter Setup

- **Overview:**  
  This initial block captures and displays the parameters guiding the literature review process (topic, year range, max papers). These parameters feed the entire workflow.

- **Nodes Involved:**  
  - Start (Sticky Note)

- **Node Details:**  
  - **Start**  
    - Type: Sticky Note  
    - Role: Displays input parameters as a reference for users and downstream nodes.  
    - Configuration: Uses expressions to display dynamic values from the initial JSON input (`topic`, `startYear`, `endYear`, `maxPapers`).  
    - Inputs: None (workflow entry point)  
    - Outputs: Connects to the paper search node  
    - Edge Cases: If input parameters are missing or malformed, subsequent nodes may fail or return empty results.

#### 2.2 Multi-Database Paper Search

- **Overview:**  
  Searches multiple academic databases (PubMed, Semantic Scholar, arXiv, Google Scholar) for papers matching the topic and within the specified year range, returning up to 50 results.

- **Nodes Involved:**  
  - PDF Vector - Search Papers

- **Node Details:**  
  - **PDF Vector - Search Papers**  
    - Type: PDF Vector Search (custom node)  
    - Role: Queries academic databases simultaneously to retrieve papers metadata and PDFs.  
    - Configuration: 
      - Query set to the input topic  
      - Year filter applied via `yearFrom` and `yearTo` parameters  
      - Fields retrieved: title, abstract, authors, year, DOI, PDF URL, citation count  
      - Providers: pubmed, semantic_scholar, arxiv, google_scholar  
      - Result limit: 50  
    - Inputs: From Start node  
    - Outputs: Papers list to Sorting node  
    - Edge Cases:  
      - Network or API failures on one or more providers  
      - Missing PDF URLs for some papers  
      - Rate-limiting or quota restrictions on providers  

#### 2.3 Paper Sorting and Selection

- **Overview:**  
  Sorts the retrieved papers in descending order by citation count to prioritize influential studies, then limits the list to the maximum number of papers specified by the user.

- **Nodes Involved:**  
  - Sort by Citations  
  - Select Top Papers

- **Node Details:**  
  - **Sort by Citations**  
    - Type: Code Node  
    - Role: Sorts papers array by `totalCitations` field descending  
    - Configuration: Custom JavaScript function sorting items by citation number (missing values default to 0)  
    - Inputs: From Search Papers node  
    - Outputs: Sorted list to Select Top Papers node  
    - Edge Cases: Papers without citation data are treated as zero citations; sorting may not be meaningful if all zeros.

  - **Select Top Papers**  
    - Type: Code Node  
    - Role: Limits the sorted list to top N papers, where N is the `maxPapers` parameter (default 10 if undefined).  
    - Configuration: JavaScript slice method applied on sorted items  
    - Inputs: From Sort by Citations node  
    - Outputs: Top papers to Parse Papers node  
    - Edge Cases: If fewer than N papers are found, all are selected; if `maxPapers` is invalid, defaults to 10.

#### 2.4 Paper Content Parsing

- **Overview:**  
  Parses each selected paper’s PDF to extract the textual content necessary for GPT-4 summarization.

- **Nodes Involved:**  
  - PDF Vector - Parse Papers

- **Node Details:**  
  - **PDF Vector - Parse Papers**  
    - Type: PDF Vector (custom node)  
    - Role: Downloads and parses PDF documents to extract text.  
    - Configuration:  
      - `documentUrl` parameter set dynamically to each paper’s `pdfUrl`  
      - Uses automatic LLM parsing enabled  
      - Operation set to "parse" with resource set to "document"  
    - Inputs: Top papers from Select Top Papers node  
    - Outputs: Parsed content to Synthesize Review node  
    - Edge Cases:  
      - Missing or inaccessible PDF URLs cause parse failures  
      - Large PDFs may time out or exceed parsing limits  
      - Parsing errors if PDF format is unsupported or corrupted  

#### 2.5 Literature Review Synthesis

- **Overview:**  
  For each parsed paper, uses GPT-4 to generate a structured literature review section summarizing the key contributions, methodology, findings, and relevance to the original topic.

- **Nodes Involved:**  
  - Synthesize Review

- **Node Details:**  
  - **Synthesize Review**  
    - Type: OpenAI (GPT-4) node  
    - Role: Generates a textual summary for each paper using a custom prompt template.  
    - Configuration:  
      - Model: GPT-4  
      - Prompt includes paper metadata (title, authors, year) and parsed content  
      - Requests:  
        1. Key contribution summary (2-3 sentences)  
        2. Methodology overview  
        3. Main findings  
        4. Relevance to initial topic  
    - Inputs: Parsed paper content from PDF Vector Parse node  
    - Outputs: Review sections to Combine Sections node  
    - Edge Cases:  
      - OpenAI API rate limits or failures  
      - Incomplete or low-quality parsed content leads to shallow summaries  
      - Token limits exceeded if paper content is too large  

#### 2.6 Review Compilation and Export

- **Overview:**  
  Combines all individual review sections into a single markdown document and writes the output to a file named with the current date and topic.

- **Nodes Involved:**  
  - Combine Sections  
  - Export Review

- **Node Details:**  
  - **Combine Sections**  
    - Type: Code Node  
    - Role: Aggregates all synthesized review sections into one concatenated string separated by double newlines.  
    - Configuration: Maps each item’s `reviewSection` or falls back to raw content if missing, joins with line breaks.  
    - Inputs: GPT-4 generated sections from Synthesize Review node  
    - Outputs: Single combined review to Export Review node  
    - Edge Cases: If no sections are generated, output will be empty.

  - **Export Review**  
    - Type: Write Binary File  
    - Role: Saves the compiled literature review as a markdown (.md) file.  
    - Configuration:  
      - Filename includes `literature_review_` prefix and current date in `yyyy-MM-dd` format  
      - Content prepended with a header containing the topic and followed by combined review sections  
    - Inputs: Combined review from Combine Sections node  
    - Outputs: None (workflow end)  
    - Edge Cases:  
      - Filesystem write permission issues  
      - Filename conflicts or invalid characters are not explicitly handled

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                    | Input Node(s)            | Output Node(s)             | Sticky Note                                                                                         |
|-------------------------|---------------------------|----------------------------------|--------------------------|----------------------------|---------------------------------------------------------------------------------------------------|
| Start                   | Sticky Note               | Input parameter display           | None                     | PDF Vector - Search Papers  | ## Literature Review Parameters<br>Topic: {{ $json.topic }}<br>Year Range: {{ $json.startYear }}-{{ $json.endYear }}<br>Max Papers: {{ $json.maxPapers }} |
| PDF Vector - Search Papers | PDF Vector Search          | Multi-database paper search       | Start                    | Sort by Citations           | Search across multiple academic databases                                                         |
| Sort by Citations       | Code                      | Sort papers by citation count     | PDF Vector - Search Papers | Select Top Papers           |                                                                                                   |
| Select Top Papers       | Code                      | Limit to top N papers             | Sort by Citations        | PDF Vector - Parse Papers   |                                                                                                   |
| PDF Vector - Parse Papers | PDF Vector Parse           | Parse each paper’s PDF            | Select Top Papers        | Synthesize Review           | Parse each paper's PDF                                                                             |
| Synthesize Review       | OpenAI (GPT-4)            | Generate literature review sections | PDF Vector - Parse Papers | Combine Sections           |                                                                                                   |
| Combine Sections        | Code                      | Concatenate review sections       | Synthesize Review        | Export Review               |                                                                                                   |
| Export Review           | Write Binary File          | Export combined review as markdown | Combine Sections         | None                       |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Start Node (Sticky Note):**  
   - Create a Sticky Note node named `Start`.  
   - Set content with markdown showing parameters:  
     ```
     ## Literature Review Parameters

     Topic: {{ $json.topic }}
     Year Range: {{ $json.startYear }}-{{ $json.endYear }}
     Max Papers: {{ $json.maxPapers }}
     ```  
   - This node acts as the workflow entry point.

2. **Create PDF Vector - Search Papers Node:**  
   - Add a `PDF Vector` node, name it `PDF Vector - Search Papers`.  
   - Set operation to `search` and resource to `academic`.  
   - Configure:  
     - `query` parameter: `={{ $json.topic }}`  
     - `yearFrom`: `={{ $json.startYear }}`  
     - `yearTo`: `={{ $json.endYear }}`  
     - `limit`: 50  
     - `fields`: `["title","abstract","authors","year","doi","pdfUrl","totalCitations"]`  
     - `providers`: `["pubmed","semantic_scholar","arxiv","google_scholar"]`  
   - Connect output of `Start` node to this node.

3. **Create Sort by Citations Node:**  
   - Add a `Code` node named `Sort by Citations`.  
   - Use this JavaScript code to sort papers descending by citation count:  
     ```javascript
     return items.sort((a, b) => (b.json.totalCitations || 0) - (a.json.totalCitations || 0));
     ```  
   - Connect output of `PDF Vector - Search Papers` to this node.

4. **Create Select Top Papers Node:**  
   - Add a `Code` node named `Select Top Papers`.  
   - Use this code to slice the sorted list to max papers:  
     ```javascript
     const maxPapers = $node['Start'].json.maxPapers || 10;
     return items.slice(0, maxPapers);
     ```  
   - Connect output of `Sort by Citations` to this node.

5. **Create PDF Vector - Parse Papers Node:**  
   - Add a `PDF Vector` node named `PDF Vector - Parse Papers`.  
   - Set operation to `parse` and resource to `document`.  
   - Set `documentUrl` to `={{ $json.pdfUrl }}` to dynamically parse each paper’s PDF.  
   - Enable `useLlm` to `auto` to allow for enhanced parsing.  
   - Connect output of `Select Top Papers` to this node.

6. **Create Synthesize Review Node:**  
   - Add an `OpenAI` node named `Synthesize Review`.  
   - Set model to `gpt-4`.  
   - Configure messages with the following prompt template:  
     ```
     Create a literature review section for this paper:

     Title: {{ $json.title }}
     Authors: {{ $json.authors }}
     Year: {{ $json.year }}

     Content: {{ $json.content }}

     Generate:
     1. Key contribution summary (2-3 sentences)
     2. Methodology overview
     3. Main findings
     4. Relevance to topic: {{ $node['Start'].json.topic }}
     ```  
   - Connect output of `PDF Vector - Parse Papers` to this node.  
   - Configure OpenAI credentials for API access.

7. **Create Combine Sections Node:**  
   - Add a `Code` node named `Combine Sections`.  
   - Use this code to concatenate all review sections:  
     ```javascript
     const reviewSections = items.map(item => item.json.reviewSection || item.json.content || '').filter(section => section);
     return [{ json: { reviewSections: reviewSections.join('\n\n') } }];
     ```  
   - Connect output of `Synthesize Review` to this node.

8. **Create Export Review Node:**  
   - Add a `Write Binary File` node named `Export Review`.  
   - Set filename to:  
     `literature_review_{{ $now.format('yyyy-MM-dd') }}.md`  
   - Set file content to:  
     ```
     # Literature Review: {{ $node['Start'].json.topic }}

     {{ $json.reviewSections }}
     ```  
   - Connect output of `Combine Sections` to this node.

9. **Validate and Test:**  
   - Provide input JSON with properties:  
     - `topic` (string)  
     - `startYear` (number)  
     - `endYear` (number)  
     - `maxPapers` (number)  
   - Run the workflow and verify the markdown output file is generated with synthesized literature review content.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                               |
|------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| Multi-database academic search leverages PubMed, Semantic Scholar, arXiv, and Google Scholar providers. | Ensures broad coverage of scientific literature.                                            |
| Parsing PDFs with automatic LLM assistance improves text extraction quality for downstream summarization. | PDF Vector node supports enhanced parsing with AI assistance.                                |
| GPT-4 is used to generate structured literature review sections, ensuring consistency and depth.     | Requires valid OpenAI API credentials and may incur associated costs.                        |
| Exported literature review is saved as Markdown file with timestamped filename for version tracking.  | Markdown format allows easy editing and integration with documentation tools.                |
| Input validation on parameters (e.g., year range, max papers) is recommended to avoid empty or invalid queries. | Can be implemented upstream or via additional nodes for robustness.                          |

---

**Disclaimer:** The text provided is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.