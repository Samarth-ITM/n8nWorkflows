Scrape Latest Github Trending Repositories

https://n8nworkflows.xyz/workflows/scrape-latest-github-trending-repositories-2866


# Scrape Latest Github Trending Repositories

### 1. Workflow Overview

This workflow automates the scraping of the latest trending repositories from GitHub’s trending page. It is designed for developers, researchers, and data analysts who want to stay updated on popular open-source projects without manually browsing GitHub. The workflow extracts key metadata such as repository names, authors, descriptions, programming languages, and repository URLs, then structures this data for easy consumption or further integration.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Data Retrieval:** HTTP request to fetch the GitHub trending page HTML.
- **1.3 HTML Content Extraction:** Extract the main container (box) holding trending repositories.
- **1.4 Repository Extraction:** Extract individual repository HTML blocks from the container.
- **1.5 Repository Data Parsing:** Extract detailed repository information (name, author, description, language) from each repository block.
- **1.6 Data Structuring:** Convert extracted data into a structured list and set clean output variables for downstream use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block initiates the workflow execution manually by the user.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger with no parameters; starts the workflow on user action.  
    - Inputs: None  
    - Outputs: Triggers the next node "Request to Github Trend"  
    - Edge Cases: None typical; workflow will not run unless manually triggered.  
    - Version: Compatible with n8n v1+  

#### 1.2 Data Retrieval

- **Overview:**  
  Sends an HTTP GET request to GitHub’s trending page to retrieve the raw HTML content.

- **Nodes Involved:**  
  - Request to Github Trend

- **Node Details:**  
  - **Request to Github Trend**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: GET (default)  
      - URL: https://github.com/trending  
      - No authentication or headers configured (public page)  
    - Inputs: Triggered by manual trigger node  
    - Outputs: Raw HTML content of the trending page passed to "Extract Box"  
    - Edge Cases:  
      - Network errors or GitHub downtime may cause request failure.  
      - Rate limiting by GitHub if run too frequently.  
      - Changes in GitHub page structure may affect downstream parsing.  
    - Version: HTTP Request node v4.2 or higher recommended for stability.

#### 1.3 HTML Content Extraction

- **Overview:**  
  Extracts the main container (`div.Box`) that holds the trending repositories from the full HTML response.

- **Nodes Involved:**  
  - Extract Box

- **Node Details:**  
  - **Extract Box**  
    - Type: HTML Extract  
    - Configuration:  
      - Operation: Extract HTML content  
      - Extraction key: "box"  
      - CSS Selector: `div.Box` (selects the main container div)  
      - Return value: HTML (full inner HTML of the container)  
    - Inputs: Receives raw HTML from HTTP Request node  
    - Outputs: Extracted HTML container passed to "Extract all repositories"  
    - Edge Cases:  
      - If GitHub changes the container class or structure, extraction will fail or return empty.  
      - Empty or malformed HTML input may cause extraction failure.  
    - Version: HTML node v1.2 or higher for stable extraction.

#### 1.4 Repository Extraction

- **Overview:**  
  Extracts each individual repository block (`article.Box-row`) as an HTML snippet from the container.

- **Nodes Involved:**  
  - Extract all repositories

- **Node Details:**  
  - **Extract all repositories**  
    - Type: HTML Extract  
    - Configuration:  
      - Operation: Extract HTML content  
      - Data property name: "box" (input property)  
      - Extraction key: "repositories"  
      - CSS Selector: `article.Box-row` (each repository block)  
      - Return array: true (returns list of repository HTML blocks)  
      - Return value: HTML  
      - Options: Trim and clean up text enabled for cleaner extraction  
    - Inputs: Receives "box" HTML from "Extract Box" node  
    - Outputs: Array of repository HTML blocks passed to "Turn to a list"  
    - Edge Cases:  
      - If no repository blocks found, output will be empty array.  
      - Changes in GitHub HTML structure may cause extraction failure.  
    - Version: HTML node v1.2 or higher.

#### 1.5 Repository Data Parsing

- **Overview:**  
  Parses each repository HTML block to extract detailed data fields: repository name (author/repo), programming language, and description.

- **Nodes Involved:**  
  - Turn to a list  
  - Extract repository data

- **Node Details:**  
  - **Turn to a list**  
    - Type: Split Out  
    - Configuration:  
      - Field to split out: "repositories" (array from previous node)  
    - Inputs: Receives array of repository HTML blocks  
    - Outputs: Emits each repository block as a separate item for detailed extraction  
    - Edge Cases: Empty input array results in no output items.  
    - Version: v1+  

  - **Extract repository data**  
    - Type: HTML Extract  
    - Configuration:  
      - Operation: Extract HTML content  
      - Data property name: "repositories" (input property)  
      - Extraction keys and CSS selectors:  
        - "repository": `a.Link` (anchor tag containing author/repo name)  
        - "language": `span.d-inline-block` (programming language label)  
        - "description": `p` (repository description paragraph)  
    - Inputs: Receives single repository HTML block from "Turn to a list"  
    - Outputs: Parsed fields passed to "Set Result Variables"  
    - Edge Cases:  
      - Missing language or description fields may result in empty values.  
      - Changes in GitHub HTML structure may cause extraction failure.  
    - Version: HTML node v1.2 or higher.

#### 1.6 Data Structuring

- **Overview:**  
  Cleans and structures the extracted repository data into well-defined variables including author, repository name, URL, description, and timestamp.

- **Nodes Involved:**  
  - Set Result Variables

- **Node Details:**  
  - **Set Result Variables**  
    - Type: Set  
    - Configuration:  
      - Assigns multiple variables using expressions:  
        - `author`: Extracted by splitting the "repository" string at '/' and trimming the first part  
        - `title`: Second part of the split repository string (repository name)  
        - `repository`: Original repository string (author/repo)  
        - `url`: Constructed GitHub URL by concatenating "https://github.com/" with repository string (spaces removed)  
        - `created_at`: Current timestamp (`$now`) at workflow execution  
        - `description`: Directly from extracted description field  
      - Includes other fields as is  
    - Inputs: Receives parsed repository data from "Extract repository data"  
    - Outputs: Final structured JSON object for each repository  
    - Edge Cases:  
      - If repository string format is unexpected, splitting may fail or produce incorrect author/title.  
      - Missing description fields will result in empty description.  
    - Version: Set node v3.4 or higher.

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                      | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                   |
|---------------------------|--------------------|------------------------------------|------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Initiates workflow manually        | None                         | Request to Github Trend      |                                                                                              |
| Request to Github Trend    | HTTP Request       | Fetches GitHub trending page HTML | When clicking ‘Test workflow’ | Extract Box                 |                                                                                              |
| Extract Box               | HTML Extract       | Extracts main container div.Box    | Request to Github Trend       | Extract all repositories     |                                                                                              |
| Extract all repositories   | HTML Extract       | Extracts individual repository blocks | Extract Box                  | Turn to a list              |                                                                                              |
| Turn to a list             | Split Out          | Splits repository array into items | Extract all repositories      | Extract repository data      |                                                                                              |
| Extract repository data    | HTML Extract       | Extracts detailed repo info        | Turn to a list                | Set Result Variables         |                                                                                              |
| Set Result Variables       | Set                | Structures and cleans extracted data | Extract repository data       | None                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Add an HTTP Request node**  
   - Name: `Request to Github Trend`  
   - Method: GET  
   - URL: `https://github.com/trending`  
   - Connect output of Manual Trigger to this node.

3. **Add an HTML Extract node**  
   - Name: `Extract Box`  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `box`  
     - CSS Selector: `div.Box`  
     - Return Value: HTML  
   - Connect output of HTTP Request node to this node.

4. **Add another HTML Extract node**  
   - Name: `Extract all repositories`  
   - Operation: Extract HTML content  
   - Data Property Name: `box` (matches output key from previous node)  
   - Extraction Values:  
     - Key: `repositories`  
     - CSS Selector: `article.Box-row`  
     - Return Array: true  
     - Return Value: HTML  
   - Enable options: Trim values, Clean up text  
   - Connect output of `Extract Box` to this node.

5. **Add a Split Out node**  
   - Name: `Turn to a list`  
   - Field to Split Out: `repositories`  
   - Connect output of `Extract all repositories` to this node.

6. **Add an HTML Extract node**  
   - Name: `Extract repository data`  
   - Operation: Extract HTML content  
   - Data Property Name: `repositories` (single item from split)  
   - Extraction Values:  
     - Key: `repository`, CSS Selector: `a.Link`  
     - Key: `language`, CSS Selector: `span.d-inline-block`  
     - Key: `description`, CSS Selector: `p`  
   - Connect output of `Turn to a list` to this node.

7. **Add a Set node**  
   - Name: `Set Result Variables`  
   - Assignments:  
     - `author`: `={{ $json.repository.split('/')[0].trim() }}`  
     - `title`: `={{ $json.repository.split('/')[1].trim() }}`  
     - `repository`: `={{ $json.repository }}`  
     - `url`: `=https://github.com/{{ $json.repository.replaceAll(' ','') }}`  
     - `created_at`: `={{$now}}`  
     - `description`: `={{ $json.description }}`  
   - Include other fields as is  
   - Connect output of `Extract repository data` to this node.

8. **Save and activate the workflow**  
   - Run manually via the manual trigger node or schedule as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Since GitHub’s trending page updates dynamically, run this workflow periodically to capture latest trends. | Workflow description and usage instructions                                                     |
| Customize HTTP Request node URL to target specific trending categories or languages on GitHub.      | Workflow customization suggestions                                                             |
| Integrate with Slack, email, or databases to notify or store trending repositories automatically.  | Workflow customization suggestions                                                             |
| n8n HTML Extract node uses CSS selectors to parse HTML content; changes in GitHub page structure may require updates to selectors. | Important for maintenance and troubleshooting                                                  |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling users and automation agents to reproduce, modify, or extend it confidently.