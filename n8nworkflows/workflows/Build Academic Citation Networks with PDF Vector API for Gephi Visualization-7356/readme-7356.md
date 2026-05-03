Build Academic Citation Networks with PDF Vector API for Gephi Visualization

https://n8nworkflows.xyz/workflows/build-academic-citation-networks-with-pdf-vector-api-for-gephi-visualization-7356


# Build Academic Citation Networks with PDF Vector API for Gephi Visualization

### 1. Workflow Overview

This workflow, titled **Build Academic Citation Networks with PDF Vector API for Gephi Visualization**, automates the creation of academic citation networks from given seed paper identifiers (such as DOIs or PubMed IDs). It systematically fetches metadata about the seed papers, explores citation relationships up to a user-defined depth, constructs a network graph data structure with nodes (papers) and edges (citation links), and finally outputs this network data both as a JSON file and as a Gephi-compatible GEXF format for visualization.

The workflow logic is organized into these main blocks:

- **1.1 Input Reception and Initialization**: Receives seed paper IDs and parameters controlling the depth of citation exploration.
- **1.2 Paper Data Fetching**: Retrieves detailed metadata for each seed paper using the PDF Vector API.
- **1.3 Citation Retrieval**: Searches for papers that cite the seed papers.
- **1.4 Network Construction**: Builds nodes and edges representing papers and citation relationships.
- **1.5 Network Aggregation**: Combines individual paper networks into a unified citation network graph.
- **1.6 Output Generation**: Exports the combined network as JSON and generates a GEXF file suitable for Gephi visualization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:**  
  This block collects the initial user input: a list of paper identifiers and the depth of citation levels to explore. It prepares the data for iterative processing.

- **Nodes Involved:**  
  - Configuration (Sticky Note)  
  - Set Parameters  
  - Split Paper IDs  

- **Node Details:**  

  - **Configuration (Sticky Note)**  
    - Type: Sticky Note (Documentation)  
    - Role: Provides a concise summary of workflow purpose and input/output expectations.  
    - Position: Top-left for user reference.  
    - Contains a brief description of the workflow inputs (Paper IDs, Depth) and output (Network graph data).  
    - No inputs or outputs.  

  - **Set Parameters**  
    - Type: Set  
    - Role: Defines input parameters explicitly within the workflow.  
    - Configuration: Sets two string parameters:  
      - `seedPapers`: Comma-separated paper IDs (example IDs given).  
      - `depth`: Number of citation levels to explore (default "2").  
    - Input: None (starting node).  
    - Output: Passes parameters downstream.  
    - Edge cases: If empty or malformed IDs are input, downstream nodes may fail or return incomplete data. Input validation is manual.  

  - **Split Paper IDs**  
    - Type: Code (JavaScript)  
    - Role: Parses the comma-separated string of paper IDs into an array of objects with trimmed IDs for iteration.  
    - Key Expression: Splits `seedPapers` string by commas, trims whitespace, and returns array of `{id: string}` objects.  
    - Input: Receives parameter object from “Set Parameters”.  
    - Output: Emits multiple items, each with a single paper ID for parallel processing.  
    - Edge cases: Empty strings produce empty arrays; malformed IDs propagate downstream without validation.

---

#### 1.2 Paper Data Fetching

- **Overview:**  
  For each seed paper ID, fetch detailed metadata including title, authors, year, DOI, abstract, citations, and references via the PDF Vector API.

- **Nodes Involved:**  
  - PDF Vector - Fetch Papers  

- **Node Details:**  

  - **PDF Vector - Fetch Papers**  
    - Type: PDF Vector API Integration  
    - Role: Fetches detailed academic paper metadata based on paper ID.  
    - Configuration:  
      - Operation: `fetch`  
      - Resource: `academic`  
      - IDs: Dynamically set to the current paper's `id`.  
      - Fields requested: title, authors, year, doi, abstract, totalCitations, totalReferences.  
    - Input: Receives each paper ID from “Split Paper IDs”.  
    - Output: Paper metadata JSON, passed downstream.  
    - Edge cases:  
      - API rate limits or network errors may cause failures.  
      - Invalid IDs may return empty or error responses.  
      - Missing fields in API response could cause downstream processing issues.  
    - Version-specific: Requires PDF Vector node version supporting `fetch` on `academic` resource.

---

#### 1.3 Citation Retrieval

- **Overview:**  
  Searches for papers citing each fetched seed paper, limited to 20 results, retrieving essential metadata for each citing paper.

- **Nodes Involved:**  
  - Fetch Citing Papers  

- **Node Details:**  

  - **Fetch Citing Papers**  
    - Type: PDF Vector API Integration  
    - Role: Retrieves a list of papers that cite the given seed paper.  
    - Configuration:  
      - Operation: `search`  
      - Resource: `academic`  
      - Query: `references:<doi>` where `<doi>` is dynamically taken from the seed paper metadata.  
      - Limit: 20 citing papers max.  
      - Fields: title, authors, year, doi, totalCitations.  
    - Input: Receives paper metadata from “PDF Vector - Fetch Papers”.  
    - Output: List of citing papers attached to each seed paper JSON under e.g. `citingPapers`.  
    - Edge cases:  
      - If DOI is missing or invalid, search query fails or returns empty.  
      - API limits or errors may result in no citing papers.  

---

#### 1.4 Network Construction

- **Overview:**  
  For each seed paper and its citing papers, constructs nodes and edges representing the citation network graph components.

- **Nodes Involved:**  
  - Build Network Data  

- **Node Details:**  

  - **Build Network Data**  
    - Type: Code (JavaScript)  
    - Role: Builds two arrays — `nodes` and `edges` — representing papers and citation links.  
    - Logic:  
      - Creates a node for the seed paper with properties: id (DOI or ID), label (title), size (log-scaled citations), citations count, year, and type ‘seed’.  
      - Iterates citing papers to create nodes and edges pointing from each citing paper to the seed paper.  
      - Node size for citing papers scaled differently (half of seed paper’s scale).  
    - Input: Receives combined JSON with seed paper and citing papers.  
    - Output: JSON with `nodes` and `edges` arrays.  
    - Edge cases:  
      - Missing DOIs or titles may produce incomplete nodes.  
      - Citation counts of zero handled by `log(x+1)` to avoid math errors.  
      - If no citing papers, returns only the seed paper node.  

---

#### 1.5 Network Aggregation

- **Overview:**  
  Aggregates all individual networks from multiple seed papers into a single unified network graph, removing duplicate nodes.

- **Nodes Involved:**  
  - Combine Network  

- **Node Details:**  

  - **Combine Network**  
    - Type: Code (JavaScript)  
    - Role: Merges arrays of nodes and edges from all processed papers.  
    - Logic:  
      - Concatenates all nodes and edges from input items.  
      - Removes duplicate nodes based on unique node `id` using a Map keyed by `id`.  
      - Returns combined unique nodes and all edges (edges are not deduplicated).  
    - Input: Receives multiple items each with `nodes` and `edges`.  
    - Output: Single JSON containing combined `nodes` and `edges`.  
    - Edge cases:  
      - Potential duplicate edges are not removed, which may cause redundancy in visualization.  
      - Assumes node IDs are unique identifiers.  

---

#### 1.6 Output Generation

- **Overview:**  
  Exports the combined citation network as a JSON file and generates a GEXF format file compatible with Gephi for network visualization.

- **Nodes Involved:**  
  - Export Network JSON  
  - Generate GEXF  

- **Node Details:**  

  - **Export Network JSON**  
    - Type: Write Binary File  
    - Role: Saves the combined network data as a formatted JSON file.  
    - Configuration:  
      - Filename: `citation_network_YYYY-MM-DD.json` (date dynamically inserted).  
      - File content: Stringified JSON of the combined `nodes` and `edges` with indentation.  
    - Input: Receives combined network JSON from “Combine Network”.  
    - Output: Written file on disk or configured storage.  
    - Edge cases:  
      - File system permissions could cause write errors.  
      - Large networks might cause memory issues.  

  - **Generate GEXF**  
    - Type: Code (JavaScript)  
    - Role: Converts combined network JSON into GEXF XML format for Gephi.  
    - Logic:  
      - Constructs XML header and graph tags.  
      - Iterates nodes to create `<node>` elements with attributes for citations and year.  
      - Iterates edges to create `<edge>` elements with IDs, source, target, and weight.  
      - Returns GEXF string.  
    - Input: Same combined network JSON from “Combine Network”.  
    - Output: JSON object with `gexf` string property.  
    - Edge cases:  
      - Special characters in labels may require escaping (not implemented explicitly).  
      - Large networks produce large XML files which may be slow to process.

---

### 3. Summary Table

| Node Name              | Node Type                    | Functional Role                         | Input Node(s)         | Output Node(s)                     | Sticky Note                                                                                   |
|------------------------|------------------------------|---------------------------------------|-----------------------|-----------------------------------|-----------------------------------------------------------------------------------------------|
| Configuration          | Sticky Note                  | Documentation / Workflow overview     | —                     | —                                 | ## Citation Network Builder\n\nInput: Paper IDs (DOI, PubMed ID, etc.)\nDepth: How many citation levels to explore\nOutput: Network graph data |
| Set Parameters          | Set                          | Define input parameters                | —                     | Split Paper IDs                   |                                                                                               |
| Split Paper IDs         | Code                         | Parse seed paper IDs into array        | Set Parameters         | PDF Vector - Fetch Papers          |                                                                                               |
| PDF Vector - Fetch Papers | PDF Vector API Integration  | Fetch metadata for each seed paper     | Split Paper IDs        | Fetch Citing Papers                | Fetch details for each paper                                                                  |
| Fetch Citing Papers     | PDF Vector API Integration   | Retrieve papers citing the seed paper | PDF Vector - Fetch Papers | Build Network Data                |                                                                                               |
| Build Network Data      | Code                         | Construct nodes and edges for citation network | Fetch Citing Papers | Combine Network                  |                                                                                               |
| Combine Network         | Code                         | Aggregate all nodes and edges, remove duplicate nodes | Build Network Data | Export Network JSON, Generate GEXF |                                                                                               |
| Export Network JSON     | Write Binary File            | Export combined network as JSON file  | Combine Network        | —                                 |                                                                                               |
| Generate GEXF           | Code                         | Convert network data to Gephi GEXF format | Combine Network     | —                                 |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note node named "Configuration"**  
   - Content:  
     ```
     ## Citation Network Builder

     Input: Paper IDs (DOI, PubMed ID, etc.)
     Depth: How many citation levels to explore
     Output: Network graph data
     ```
   - Position top-left for reference.

2. **Add a Set node named "Set Parameters"**  
   - Define two string parameters:  
     - `seedPapers`: Example value `"10.1038/nature12373,12345678,2301.12345"`  
     - `depth`: Example value `"2"`  
   - This node serves as the input source.

3. **Add a Code node named "Split Paper IDs"**  
   - Use JavaScript code:  
     ```js
     const papers = $json.seedPapers.split(',').map(id => ({ id: id.trim() }));
     return papers;
     ```  
   - Connect "Set Parameters" → "Split Paper IDs".

4. **Add a PDF Vector node named "PDF Vector - Fetch Papers"**  
   - Credentials: Configure with valid PDF Vector API credentials.  
   - Resource: `academic`  
   - Operation: `fetch`  
   - IDs parameter: Use expression `{{$json.id}}` to fetch one paper per item.  
   - Fields: Request `title`, `authors`, `year`, `doi`, `abstract`, `totalCitations`, `totalReferences`.  
   - Connect "Split Paper IDs" → "PDF Vector - Fetch Papers".

5. **Add a PDF Vector node named "Fetch Citing Papers"**  
   - Credentials: Use same PDF Vector API credentials.  
   - Resource: `academic`  
   - Operation: `search`  
   - Limit: 20  
   - Query parameter: Use expression `=references:{{$json.doi}}` to find papers citing the current DOI.  
   - Fields: `title`, `authors`, `year`, `doi`, `totalCitations`.  
   - Connect "PDF Vector - Fetch Papers" → "Fetch Citing Papers".

6. **Add a Code node named "Build Network Data"**  
   - JavaScript code:  
     ```js
     const nodes = [];
     const edges = [];

     nodes.push({
       id: $json.doi || $json.id,
       label: $json.title,
       size: Math.log($json.totalCitations + 1) * 10,
       citations: $json.totalCitations,
       year: $json.year,
       type: 'seed'
     });

     if ($json.citingPapers) {
       $json.citingPapers.forEach(paper => {
         nodes.push({
           id: paper.doi,
           label: paper.title,
           size: Math.log(paper.totalCitations + 1) * 5,
           citations: paper.totalCitations,
           year: paper.year,
           type: 'citing'
         });

         edges.push({
           source: paper.doi,
           target: $json.doi || $json.id,
           weight: 1
         });
       });
     }

     return { nodes, edges };
     ```  
   - Connect "Fetch Citing Papers" → "Build Network Data".

7. **Add a Code node named "Combine Network"**  
   - JavaScript code:  
     ```js
     const allNodes = [];
     const allEdges = [];

     items.forEach(item => {
       if (item.json.nodes) {
         allNodes.push(...item.json.nodes);
       }
       if (item.json.edges) {
         allEdges.push(...item.json.edges);
       }
     });

     const uniqueNodes = Array.from(new Map(allNodes.map(node => [node.id, node])).values());

     return [{ json: { nodes: uniqueNodes, edges: allEdges } }];
     ```  
   - Connect "Build Network Data" → "Combine Network".

8. **Add a Write Binary File node named "Export Network JSON"**  
   - Filename: Use expression `citation_network_{{$now.format("yyyy-MM-dd")}}.json`  
   - File Content: Use expression `={{ JSON.stringify({ nodes: $json.nodes, edges: $json.edges }, null, 2) }}`  
   - Connect "Combine Network" → "Export Network JSON".

9. **Add a Code node named "Generate GEXF"**  
   - JavaScript code:  
     ```js
     const nodes = $json.nodes;
     const edges = $json.edges;

     let gexf = `<?xml version="1.0" encoding="UTF-8"?>
     <gexf xmlns="http://www.gexf.net/1.2draft" version="1.2">
       <graph mode="static" defaultedgetype="directed">
         <nodes>\n`;

     nodes.forEach(node => {
       gexf += `      <node id="${node.id}" label="${node.label}">
           <attvalues>
             <attvalue for="citations" value="${node.citations}"/>
             <attvalue for="year" value="${node.year}"/>
           </attvalues>
         </node>\n`;
     });

     gexf += `    </nodes>
         <edges>\n`;

     edges.forEach((edge, i) => {
       gexf += `      <edge id="${i}" source="${edge.source}" target="${edge.target}" weight="${edge.weight}"/>\n`;
     });

     gexf += `    </edges>
       </graph>
     </gexf>`;

     return { gexf };
     ```  
   - Connect "Combine Network" → "Generate GEXF".

10. **Credentials Setup:**  
    - Configure PDF Vector API with valid credentials.  
    - No other credentials required.

11. **Final Checks:**  
    - Validate that seed paper IDs are correctly formatted.  
    - Confirm API limits and handle possible rate limits externally.  
    - Ensure file system write permissions for JSON output.  

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                            |
|------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| The workflow leverages the PDF Vector API to retrieve academic metadata and citation relations. | PDF Vector API documentation and credential setup required |
| Output GEXF files are compatible with Gephi (https://gephi.org/) for network visualization.    | Gephi official site                                        |
| Citation counts are log-scaled for better node size visualization in graphs.                    | Node size scaling rationale                                |
| The workflow currently limits citing papers to 20 per seed paper to manage data size.          | Limitation to ensure performance                            |
| Ensure API rate limits are respected to avoid failures during data fetching.                    | PDF Vector API rate limiting considerations                |

---

This completes the comprehensive reference for the "Build Academic Citation Networks with PDF Vector API for Gephi Visualization" workflow. It is structured for advanced users and automation agents to understand, reproduce, and extend the citation network construction process.