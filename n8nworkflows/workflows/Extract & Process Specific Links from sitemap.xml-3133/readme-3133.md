Extract & Process Specific Links from sitemap.xml

https://n8nworkflows.xyz/workflows/extract---process-specific-links-from-sitemap-xml-3133


# Extract & Process Specific Links from sitemap.xml

### 1. Workflow Overview

This workflow is designed to read a `sitemap.xml` file from a specified URL, extract all URLs listed within it, and filter these URLs based on user-defined criteria such as file extensions (e.g., PDFs). It targets SEO specialists, developers, and content managers who need to analyze or process specific types of links from sitemaps.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Sitemap URL Setup:** Define the sitemap URL to fetch.
- **1.3 Fetch and Convert Sitemap:** HTTP request to retrieve the sitemap XML and convert it to JSON.
- **1.4 URL Extraction:** Split the JSON array of URLs into individual items.
- **1.5 URL Filtering:** Filter URLs based on specified conditions (e.g., URLs ending with `.pdf`).
- **1.6 Output / Further Processing:** The filtered URLs can be exported, emailed, or used downstream (not implemented here but implied).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually, allowing the user to start the process on demand.

- **Nodes Involved:**  
  - ‘Test workflow’ trigger

- **Node Details:**

  - **‘Test workflow’ trigger**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to “Set sitemap URL” node.  
    - Edge Cases: None typical; user must manually trigger.  
    - Version: 1

---

#### 2.2 Sitemap URL Setup

- **Overview:**  
  Sets the URL of the sitemap.xml file to be fetched. This is the only user-editable parameter to specify which sitemap to process.

- **Nodes Involved:**  
  - Set sitemap URL

- **Node Details:**

  - **Set sitemap URL**  
    - Type: Set  
    - Role: Defines the `sitemapUrl` variable containing the sitemap XML URL.  
    - Configuration: Assigns a string variable `sitemapUrl` with default value `https://duckduckgo.com/sitemap.xml`.  
    - Key Expression: None, static string assignment.  
    - Inputs: Receives trigger from manual trigger node.  
    - Outputs: Passes data to “Get Sitemap” node.  
    - Edge Cases: User must update URL for different sitemaps; invalid URLs or unreachable URLs will cause HTTP request failure downstream.  
    - Version: 3.4  
    - Sticky Note: "**Set your sitemap.xml\nurl here.**"

---

#### 2.3 Fetch and Convert Sitemap

- **Overview:**  
  This block fetches the sitemap XML from the URL and converts it into a JSON structure for easier processing.

- **Nodes Involved:**  
  - Get Sitemap  
  - Convert Sitemap to JSON

- **Node Details:**

  - **Get Sitemap**  
    - Type: HTTP Request  
    - Role: Retrieves the sitemap.xml content from the URL stored in `sitemapUrl`.  
    - Configuration: URL parameter set dynamically via expression `={{ $json.sitemapUrl }}`; no additional options set.  
    - Inputs: Receives data from “Set sitemap URL”.  
    - Outputs: Passes raw XML data to “Convert Sitemap to JSON”.  
    - Edge Cases: HTTP errors (404, 500), network timeouts, invalid URLs, or non-XML responses may cause failure.  
    - Version: 4.2

  - **Convert Sitemap to JSON**  
    - Type: XML  
    - Role: Parses the XML sitemap content into normalized JSON.  
    - Configuration: Options enabled - trim whitespace, normalize tags, merge attributes, ignore attributes.  
    - Inputs: Receives raw XML from “Get Sitemap”.  
    - Outputs: JSON object representing the sitemap structure to “Split Out”.  
    - Edge Cases: Malformed XML or unexpected XML structure may cause parsing errors.  
    - Version: 1  
    - Sticky Note: "## Fetch and process the sitemap.xml file\nThis part fetches and process the sitemap.xml file from XML data to JSON that we can work with."

---

#### 2.4 URL Extraction

- **Overview:**  
  Splits the JSON array of URLs (`urlset.url`) into individual items so each URL can be processed or filtered separately.

- **Nodes Involved:**  
  - Split Out

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Takes the array of URLs and outputs each URL as a separate item.  
    - Configuration: Field to split out set to `urlset.url` (the array of URLs in the sitemap JSON).  
    - Inputs: Receives JSON sitemap from “Convert Sitemap to JSON”.  
    - Outputs: Individual URL objects to “Filter URLs”.  
    - Edge Cases: If the sitemap JSON structure changes or `urlset.url` is missing, this node will fail or output nothing.  
    - Version: 1

---

#### 2.5 URL Filtering

- **Overview:**  
  Filters the extracted URLs based on user-defined conditions, such as URLs ending with `.pdf`. This allows selective processing of specific link types.

- **Nodes Involved:**  
  - Filter URLs

- **Node Details:**

  - **Filter URLs**  
    - Type: Filter  
    - Role: Filters URLs to only those matching the condition(s).  
    - Configuration: Condition set to check if the `loc` field (URL) ends with `.pdf` (case sensitive).  
    - Expression: Left value `={{ $json.loc }}`, operator `endsWith`, right value `.pdf`.  
    - Inputs: Receives individual URL items from “Split Out”.  
    - Outputs: Passes filtered URLs downstream (no further nodes connected in this workflow).  
    - Edge Cases: URLs not ending with `.pdf` are excluded; case sensitivity may exclude some URLs unintentionally; if `loc` field is missing, condition fails.  
    - Version: 2.2  
    - Sticky Note: "**Create your filter here.**"

---

#### 2.6 Notes and Comments

- **Sticky Note (General Workflow Overview)**  
  - Content:  
    ```
    ## Sitemap.xml reader
    This workflow reads an sitemap.xml and filters out the entries you want.

    By default only PDF documents are returned at the end of the workflow.

    **SETUP**
    - Edit the **Set sitemap URL** block and add the url to the sitemap you want to read.

    - Edit the **Filter URLs** to your needs.
    ```
  - Applies to: Entire workflow, positioned near the manual trigger node.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                  | Input Node(s)          | Output Node(s)       | Sticky Note                                  |
|---------------------|-------------------|--------------------------------|-----------------------|----------------------|----------------------------------------------|
| ‘Test workflow’ trigger | Manual Trigger    | Starts the workflow manually    | None                  | Set sitemap URL      |                                              |
| Set sitemap URL     | Set               | Defines sitemap URL variable    | ‘Test workflow’ trigger | Get Sitemap          | **Set your sitemap.xml\nurl here.**          |
| Get Sitemap         | HTTP Request      | Fetches sitemap.xml content     | Set sitemap URL        | Convert Sitemap to JSON |                                              |
| Convert Sitemap to JSON | XML Parser       | Converts XML sitemap to JSON    | Get Sitemap            | Split Out            | ## Fetch and process the sitemap.xml file\nThis part fetches and process the sitemap.xml file from XML data to JSON that we can work with. |
| Split Out           | Split Out         | Splits URLs array into items    | Convert Sitemap to JSON | Filter URLs          |                                              |
| Filter URLs         | Filter            | Filters URLs by condition       | Split Out              | None                 | **Create your filter here.**                  |
| Sticky Note1        | Sticky Note       | Instruction for sitemap URL     | None                  | None                 | **Set your sitemap.xml\nurl here.**          |
| Sticky Note2        | Sticky Note       | Instruction for filter setup    | None                  | None                 | **Create your filter here.**                  |
| Sticky Note3        | Sticky Note       | Explains fetch and process block| None                  | None                 | ## Fetch and process the sitemap.xml file\nThis part fetches and process the sitemap.xml file from XML data to JSON that we can work with. |
| Sticky Note         | Sticky Note       | General workflow overview       | None                  | None                 | ## Sitemap.xml reader\nThis workflow reads an sitemap.xml and filters out the entries you want.\n\nBy default only PDF documents are returned at the end of the workflow.\n\n**SETUP**\n- Edit the **Set sitemap URL** block and add the url to the sitemap you want to read.\n\n- Edit the **Filter URLs** to your needs. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add a Set Node to Define Sitemap URL**  
   - Type: Set  
   - Configuration:  
     - Add a string field named `sitemapUrl`.  
     - Set its value to the desired sitemap URL, e.g., `https://duckduckgo.com/sitemap.xml`.  
   - Connect the Manual Trigger node output to this Set node input.

3. **Add an HTTP Request Node to Fetch Sitemap**  
   - Type: HTTP Request  
   - Configuration:  
     - URL: Use expression `={{ $json.sitemapUrl }}` to dynamically get the URL from the previous node.  
     - Method: GET (default).  
     - No authentication or special headers needed unless your sitemap requires it.  
   - Connect the Set node output to this HTTP Request node input.

4. **Add an XML Node to Convert Sitemap to JSON**  
   - Type: XML  
   - Configuration:  
     - Enable options: Trim, Normalize, Merge Attributes, Ignore Attributes, Normalize Tags.  
     - This ensures the XML is parsed into a clean JSON structure.  
   - Connect the HTTP Request node output to this XML node input.

5. **Add a Split Out Node to Extract URLs**  
   - Type: Split Out  
   - Configuration:  
     - Field to split out: `urlset.url` (this corresponds to the array of URLs in the sitemap JSON).  
   - Connect the XML node output to this Split Out node input.

6. **Add a Filter Node to Filter URLs**  
   - Type: Filter  
   - Configuration:  
     - Condition: Check if the field `loc` ends with `.pdf` (or any other desired extension).  
     - Example: Left value: `={{ $json.loc }}`, Operator: `endsWith`, Right value: `.pdf`.  
     - Case sensitivity: Set according to your needs (default is case sensitive).  
   - Connect the Split Out node output to this Filter node input.

7. **(Optional) Add Nodes for Export or Further Processing**  
   - For example, email the filtered URLs, store them in a database, or pass them to another workflow.  
   - This workflow ends at the Filter node, so add downstream nodes as needed.

8. **Add Sticky Notes for Documentation (Optional)**  
   - Add notes near the “Set sitemap URL” node to remind users to edit the URL.  
   - Add notes near the “Filter URLs” node to guide users on customizing filters.  
   - Add a general note explaining the workflow purpose and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow reads a sitemap.xml and filters out the entries you want. By default, only PDF documents are returned at the end. | General workflow overview (Sticky Note near manual trigger).                                    |
| Edit the **Set sitemap URL** block to specify the sitemap you want to read.                         | Setup instruction (Sticky Note near “Set sitemap URL” node).                                    |
| Edit the **Filter URLs** block to customize which URLs to extract (e.g., PDF, images, etc.).        | Setup instruction (Sticky Note near “Filter URLs” node).                                        |
| Fetch and process the sitemap.xml file from XML to JSON for easier manipulation.                     | Explanation of XML parsing block (Sticky Note near “Convert Sitemap to JSON” node).             |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot it effectively.