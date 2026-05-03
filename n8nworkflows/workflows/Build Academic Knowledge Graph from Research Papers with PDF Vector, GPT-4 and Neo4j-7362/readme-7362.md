Build Academic Knowledge Graph from Research Papers with PDF Vector, GPT-4 and Neo4j

https://n8nworkflows.xyz/workflows/build-academic-knowledge-graph-from-research-papers-with-pdf-vector--gpt-4-and-neo4j-7362


# Build Academic Knowledge Graph from Research Papers with PDF Vector, GPT-4 and Neo4j

### 1. Workflow Overview

This workflow automates the construction and updating of an academic knowledge graph by processing research papers. It targets academic researchers, data scientists, and knowledge engineers who want to build a structured, searchable graph of academic knowledge extracted from recent research papers, focusing on concepts, authors, methods, datasets, and citations.

The workflow is logically grouped into the following blocks:

- **1.1 Scheduled Trigger & Paper Fetching:** Automatically triggers daily and fetches recent academic papers using PDF Vector integration with academic data providers.
- **1.2 Paper Parsing & Entity Extraction:** Parses the fetched PDFs and uses GPT-4 to extract structured knowledge graph entities and their relationships.
- **1.3 Graph Construction & Neo4j Insertion:** Builds graph node and relationship structures and inserts them into a Neo4j database.
- **1.4 Knowledge Base Statistics & Logging:** Calculates statistics on the knowledge base update and logs these into a Postgres database for audit and monitoring purposes.
- **1.5 Informational Sticky Note:** Provides high-level documentation of the workflow’s purpose and capabilities.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Paper Fetching

- **Overview:**  
  This block automatically triggers the workflow once per day and fetches up to 20 recent academic papers related to a specified domain (defaulting to "artificial intelligence") using external academic data providers.

- **Nodes Involved:**  
  - Daily KB Update  
  - PDF Vector - Fetch Papers

- **Node Details:**  

  **Daily KB Update**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily via a timer trigger.  
  - Configuration: Fires every 1 day using interval scheduling.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to "PDF Vector - Fetch Papers"  
  - Edge Cases: Workflow will not trigger if n8n instance is offline or scheduler misconfigured.

  **PDF Vector - Fetch Papers**  
  - Type: PDF Vector node (Academic Search)  
  - Role: Searches academic papers from Semantic Scholar and arXiv with metadata fields including title, authors, abstract, year, DOI, PDF URL, and citation count.  
  - Configuration:  
    - Limit: 20 papers  
    - Query: Dynamically set from JSON input domain or default "artificial intelligence"  
    - Year filter: Only current year papers  
    - Providers: semantic_scholar, arxiv  
  - Inputs: Triggered by schedule  
  - Outputs: Passes paper metadata to "PDF Vector - Parse Papers"  
  - Edge Cases:  
    - No papers found returns empty array  
    - API rate limits or downtime may cause errors or empty results  
    - Missing or incomplete metadata fields possible

#### 1.2 Paper Parsing & Entity Extraction

- **Overview:**  
  Parses the PDF of each fetched paper to extract its content, then uses GPT-4 to extract knowledge graph entities and relationships in structured JSON format.

- **Nodes Involved:**  
  - PDF Vector - Parse Papers  
  - Extract Entities

- **Node Details:**  

  **PDF Vector - Parse Papers**  
  - Type: PDF Vector (Document Parsing)  
  - Role: Extracts text content from the paper PDF URL for downstream processing.  
  - Configuration:  
    - Always uses LLM (Large Language Model) for parsing  
    - Input document URL from previous node’s JSON `pdfUrl` field  
  - Inputs: Paper metadata with PDF URL  
  - Outputs: Parsed content JSON to "Extract Entities"  
  - Edge Cases:  
    - Invalid or inaccessible PDF URLs  
    - Parsing errors or timeouts  
    - LLM quota or API errors

  **Extract Entities**  
  - Type: OpenAI (GPT-4)  
  - Role: Uses GPT-4 to extract key entities and relationships from paper title and content.  
  - Configuration:  
    - Model: GPT-4  
    - Response format: Structured JSON object  
    - Prompt includes instructions to extract concepts, methods, datasets, research questions, findings, future directions, plus relationships.  
  - Inputs: Parsed content and metadata from previous node  
  - Outputs: Structured JSON with entities and relationships to "Build Graph Structure"  
  - Edge Cases:  
    - Model rate limits, timeouts, or API errors  
    - Parsing or prompt interpretation failures  
    - Unexpected or malformed JSON responses

#### 1.3 Graph Construction & Neo4j Insertion

- **Overview:**  
  Converts extracted entities into graph nodes and relationships, then inserts or merges them into a Neo4j graph database.

- **Nodes Involved:**  
  - Build Graph Structure  
  - Create Graph Nodes  
  - Create Relationships

- **Node Details:**  

  **Build Graph Structure**  
  - Type: Code (JavaScript)  
  - Role: Transforms GPT output and paper metadata into node and relationship objects formatted for Neo4j ingestion.  
  - Configuration:  
    - Parses JSON content from extraction  
    - Creates nodes for Paper, Authors, Concepts, Methods with properties  
    - Creates relationships: AUTHORED_BY (Paper→Author), DISCUSSES (Paper→Concept), USES (Paper→Method)  
    - Uses DOI or sanitized title as unique paper ID  
  - Inputs: JSON with extracted entities and original paper metadata  
  - Outputs: Object with arrays of nodes and relationships  
  - Edge Cases:  
    - Missing DOI or title fallback may cause ID collisions  
    - Empty or missing entities arrays  
    - Property data types or formatting issues

  **Create Graph Nodes**  
  - Type: Neo4j  
  - Role: Creates or merges graph nodes in Neo4j using Cypher queries.  
  - Configuration:  
    - Uses UNWIND to iterate nodes array  
    - MERGE nodes on unique `id` property  
    - Sets properties and dynamic label from node data  
  - Inputs: Nodes array from previous code node  
  - Outputs: Connects to "KB Statistics"  
  - Edge Cases:  
    - Neo4j connection/authentication failures  
    - Cypher query syntax errors if node labels or IDs malformed  
    - Duplicate node insertion attempts

  **Create Relationships**  
  - Type: Neo4j  
  - Role: Creates or merges graph relationships between nodes.  
  - Configuration:  
    - UNWIND relationships array  
    - MATCH nodes by ID  
    - MERGE relationships with dynamic relationship type  
  - Inputs: Relationships array from "Build Graph Structure"  
  - Outputs: Connects to "KB Statistics"  
  - Edge Cases:  
    - Missing nodes causing MATCH to fail (no relationships created)  
    - Neo4j errors or connectivity issues  
    - Invalid or unsupported relationship types

#### 1.4 Knowledge Base Statistics & Logging

- **Overview:**  
  Computes statistics about the processed papers and entities, then logs this metadata into a Postgres table for tracking knowledge base updates.

- **Nodes Involved:**  
  - KB Statistics  
  - Log KB Update

- **Node Details:**  

  **KB Statistics**  
  - Type: Code (JavaScript)  
  - Role: Calculates counts of papers processed, concepts extracted, authors added, methods identified, and timestamp of update.  
  - Configuration:  
    - Counts nodes by label from combined nodes array in JSON  
    - Returns a stats object with counts and ISO timestamp  
  - Inputs: Output from either "Create Graph Nodes" or "Create Relationships" (both connected)  
  - Outputs: Statistical object to "Log KB Update"  
  - Edge Cases:  
    - Empty or missing nodes array  
    - JSON processing errors

  **Log KB Update**  
  - Type: Postgres  
  - Role: Inserts a new record into the `kb_updates` table with statistics collected.  
  - Configuration:  
    - Table: kb_updates  
    - Columns: papers_processed, concepts, authors, methods, updated_at  
    - Operation: Insert  
    - Values: Mapped from previous node’s stats object  
  - Inputs: Statistics object from "KB Statistics"  
  - Outputs: None (terminal node)  
  - Edge Cases:  
    - Database connection or credential issues  
    - Schema mismatches or missing table/columns  
    - Insert operation failures

#### 1.5 Informational Sticky Note

- **Overview:**  
  Provides a descriptive summary of the workflow’s purpose and core capabilities for user reference.

- **Nodes Involved:**  
  - Knowledge Base Info (Sticky Note)

- **Node Details:**  

  **Knowledge Base Info**  
  - Type: Sticky Note  
  - Role: Document the workflow’s goal to extract and connect concepts, authors, methods, datasets, citations into a searchable knowledge graph.  
  - Configuration: Markdown content describing entities extracted and overall function.  
  - Inputs/Outputs: None (informational only)  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                              | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                         |
|-------------------------|---------------------------|----------------------------------------------|-----------------------------|-----------------------------|---------------------------------------------------------------------------------------------------|
| Knowledge Base Info      | Sticky Note               | Documentation of workflow purpose             |                             |                             | ## Knowledge Base Builder Extracts and connects concepts, authors, methods, datasets, citations. Builds searchable knowledge graph. |
| Daily KB Update         | Schedule Trigger           | Triggers workflow daily                        |                             | PDF Vector - Fetch Papers    |                                                                                                   |
| PDF Vector - Fetch Papers | PDF Vector (Academic Search) | Fetches recent academic papers                 | Daily KB Update              | PDF Vector - Parse Papers    |                                                                                                   |
| PDF Vector - Parse Papers | PDF Vector (Document Parsing) | Parses PDF content for text extraction          | PDF Vector - Fetch Papers    | Extract Entities            |                                                                                                   |
| Extract Entities         | OpenAI (GPT-4)             | Extracts structured entities & relationships   | PDF Vector - Parse Papers    | Build Graph Structure       |                                                                                                   |
| Build Graph Structure    | Code                      | Constructs nodes and relationships for Neo4j  | Extract Entities             | Create Graph Nodes, Create Relationships |                                                                                                   |
| Create Graph Nodes       | Neo4j                     | Inserts/merges graph nodes                      | Build Graph Structure        | KB Statistics               |                                                                                                   |
| Create Relationships     | Neo4j                     | Inserts/merges graph relationships              | Build Graph Structure        | KB Statistics               |                                                                                                   |
| KB Statistics            | Code                      | Computes processing statistics                   | Create Graph Nodes, Create Relationships | Log KB Update              |                                                                                                   |
| Log KB Update            | Postgres                  | Logs update statistics into database            | KB Statistics                |                             |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Sticky Note node named "Knowledge Base Info":**  
   - Content: Markdown describing the workflow’s purpose: Extracts concepts, keywords, authors, institutions, methods, datasets, citations, references; builds a searchable knowledge graph.

2. **Create a Schedule Trigger node named "Daily KB Update":**  
   - Set to trigger every 1 day (interval schedule).  
   - Connect output to next node.

3. **Create a PDF Vector node named "PDF Vector - Fetch Papers":**  
   - Operation: Search academic papers.  
   - Limit: 20 results.  
   - Query: Expression `{{$json.domain || 'artificial intelligence'}}` (default domain).  
   - Fields: title, authors, abstract, year, doi, pdfUrl, totalCitations.  
   - Providers: semantic_scholar, arxiv.  
   - Year From: Current year dynamically (`{{ new Date().getFullYear() }}`).  
   - Connect input from "Daily KB Update".  
   - Output to next node.

4. **Create a PDF Vector node named "PDF Vector - Parse Papers":**  
   - Operation: Parse document.  
   - Use LLM: Always enabled.  
   - Document URL: Expression `{{$json.pdfUrl}}`.  
   - Connect input from "PDF Vector - Fetch Papers".  
   - Output to next node.

5. **Create an OpenAI node named "Extract Entities":**  
   - Model: GPT-4.  
   - Options: Set response format to JSON object.  
   - Prompt message:  
     ```
     Extract knowledge graph entities from this paper:

     Title: {{ $json.title }}
     Content: {{ $json.content }}

     Extract:
     1. Key concepts (5-10 main ideas)
     2. Methods used
     3. Datasets mentioned
     4. Research questions
     5. Key findings
     6. Future directions

     Also identify relationships between these entities.

     Return as structured JSON with entities and relationships arrays.
     ```  
   - Connect input from "PDF Vector - Parse Papers".  
   - Output to next node.

6. **Create a Code node named "Build Graph Structure":**  
   - Paste JavaScript function code which:  
     - Parses GPT-4 content JSON.  
     - Retrieves paper metadata from "PDF Vector - Fetch Papers" node.  
     - Creates arrays of nodes (Paper, Author, Concept, Method) with appropriate properties.  
     - Creates relationships arrays (AUTHORED_BY, DISCUSSES, USES).  
     - Returns `{ nodes, relationships }`.  
   - Connect input from "Extract Entities".  
   - Output to two nodes.

7. **Create a Neo4j node named "Create Graph Nodes":**  
   - Operation: Create (or merge).  
   - Cypher query:  
     ```
     UNWIND $nodes AS node
     MERGE (n:Node {id: node.properties.id})
     SET n += node.properties
     SET n:${node.label}
     ```  
   - Parameters: Pass `{ nodes: $json.nodes }`  
   - Connect input from "Build Graph Structure".  
   - Output to "KB Statistics".

8. **Create a Neo4j node named "Create Relationships":**  
   - Operation: Create (or merge).  
   - Cypher query:  
     ```
     UNWIND $relationships AS rel
     MATCH (a {id: rel.from})
     MATCH (b {id: rel.to})
     MERGE (a)-[r:${rel.type}]->(b)
     ```  
   - Parameters: Pass `{ relationships: $json.relationships }`  
   - Connect input from "Build Graph Structure".  
   - Output to "KB Statistics".

9. **Create a Code node named "KB Statistics":**  
   - JavaScript code to count:  
     - Number of papers processed (length of items).  
     - Number of concepts, authors, methods by filtering nodes by label.  
     - Current timestamp as ISO string.  
   - Connect inputs from both "Create Graph Nodes" and "Create Relationships".  
   - Output to next node.

10. **Create a Postgres node named "Log KB Update":**  
    - Operation: Insert into table `kb_updates`.  
    - Columns: `papers_processed, concepts, authors, methods, updated_at`.  
    - Values: Mapped from previous node’s output.  
    - Configure Postgres credentials.  
    - Connect input from "KB Statistics".  
    - Terminal node.

11. **Ensure credentials are configured:**  
    - PDF Vector API credentials if required.  
    - OpenAI credentials with GPT-4 access.  
    - Neo4j database credentials for graph insertion.  
    - Postgres database credentials for logging.

12. **Test workflow end-to-end:**  
    - Trigger manually or wait for scheduled trigger.  
    - Monitor logs for errors related to API limits, connectivity, or data formats.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                          |
|-----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow leverages Semantic Scholar and arXiv as data sources for academic papers.                                           | PDF Vector node providers configuration                   |
| GPT-4 model is used via OpenAI node to extract structured knowledge graph entities and relationships from paper contents.        | OpenAI GPT-4 prompt design                                |
| Neo4j is employed as the graph database backend for storing and querying the academic knowledge graph.                           | Neo4j Cypher queries for node and relationship creation  |
| Postgres is used for logging knowledge base update statistics for monitoring and audit purposes.                                  | Database logging node configuration                        |
| For best performance, ensure PDF URLs are accessible and API keys have sufficient quotas.                                         | Operational prerequisites                                 |
| Workflow designed for incremental daily updates but can be adapted for other frequencies or manual triggering.                   | Scheduling flexibility                                    |
| Sticky note node provides at-a-glance documentation within the n8n editor interface.                                              | Workflow maintainability and knowledge sharing            |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.