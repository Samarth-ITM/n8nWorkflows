Extract Internal Links from a Webpage

https://n8nworkflows.xyz/workflows/extract-internal-links-from-a-webpage-3167


# Extract Internal Links from a Webpage

### 1. Workflow Overview

This workflow automates the extraction of internal links from a specified webpage URL. It is designed primarily for web developers, SEO specialists, and digital marketers who need to analyze website structure and optimize internal linking without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger and setting the base URL to analyze.
- **1.2 Fetching Webpage Content:** HTTP request to retrieve the HTML content of the target webpage.
- **1.3 Parsing and Extracting Links:** HTML parsing to extract all links from the page.
- **1.4 Processing Links:** Splitting extracted links, identifying relative links, appending base URL to relative links, filtering out external links.
- **1.5 Output Preparation:** Merging filtered internal links into a final list.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and sets the target webpage URL for analysis.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Set Base URL

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Configuration: No parameters; triggers workflow execution.  
  - Inputs: None  
  - Outputs: Connects to `Set Base URL`  
  - Edge cases: None typical; user must trigger manually.

- **Set Base URL**  
  - Type: Set  
  - Role: Defines the URL to analyze by setting a variable `url`.  
  - Configuration: User sets the `url` field to the target webpage URL.  
  - Inputs: From manual trigger  
  - Outputs: Connects to `Fetch base URL`  
  - Edge cases: If URL is invalid or empty, subsequent HTTP request will fail.

---

#### 1.2 Fetching Webpage Content

**Overview:**  
Fetches the HTML content of the webpage specified by the base URL.

**Nodes Involved:**  
- Fetch base URL

**Node Details:**

- **Fetch base URL**  
  - Type: HTTP Request  
  - Role: Performs an HTTP GET request to retrieve the webpage HTML.  
  - Configuration:  
    - URL set dynamically from the `url` field in previous node.  
    - Method: GET  
    - Response Format: Text (HTML)  
  - Inputs: From `Set Base URL`  
  - Outputs: Connects to `Extract links`  
  - Edge cases:  
    - Network errors, timeouts, or invalid URLs cause request failure.  
    - HTTP errors (404, 500, etc.) may return error responses or empty content.

---

#### 1.3 Parsing and Extracting Links

**Overview:**  
Parses the fetched HTML content to extract all anchor (`<a>`) tags and their `href` attributes.

**Nodes Involved:**  
- Extract links

**Node Details:**

- **Extract links**  
  - Type: HTML  
  - Role: Parses HTML content and extracts all links.  
  - Configuration:  
    - CSS Selector: `a` (selects all anchor tags)  
    - Attribute to extract: `href`  
  - Inputs: From `Fetch base URL` (HTML content)  
  - Outputs: Connects to `Split Out`  
  - Edge cases:  
    - If HTML is malformed, extraction may fail or miss links.  
    - Links without `href` attributes are ignored.

---

#### 1.4 Processing Links

**Overview:**  
Processes the extracted links by splitting them, identifying relative links, appending base URL to relative links, and filtering out external links to isolate internal links.

**Nodes Involved:**  
- Split Out  
- Find relative links  
- Append base URL  
- Filter external links  
- Merge

**Node Details:**

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the array of extracted links into individual items for processing.  
  - Inputs: From `Extract links`  
  - Outputs: Connects to `Find relative links`  
  - Edge cases: Empty arrays result in no output items.

- **Find relative links**  
  - Type: If  
  - Role: Determines if each link is relative (does not start with `http` or domain).  
  - Configuration:  
    - Condition checks if `href` starts with `/` or does not start with `http`.  
  - Inputs: From `Split Out`  
  - Outputs:  
    - True branch: Connects to `Append base URL` (relative links)  
    - False branch: Connects directly to `Merge` (absolute links)  
  - Edge cases:  
    - Links with unusual formats may be misclassified.

- **Append base URL**  
  - Type: Set  
  - Role: Converts relative links to absolute URLs by prepending the base URL.  
  - Configuration:  
    - Sets new field combining base URL and relative path.  
  - Inputs: From `Find relative links` (true branch)  
  - Outputs: Connects to `Merge`  
  - Edge cases:  
    - Base URL must be properly formatted (with trailing slash if needed).  
    - Relative paths with `../` or complex paths may require normalization (not handled here).

- **Filter external links**  
  - Type: Filter  
  - Role: Filters out links that do not belong to the base domain, ensuring only internal links remain.  
  - Configuration:  
    - Condition checks if link URL contains the base domain.  
  - Inputs: From `Merge`  
  - Outputs: Final output of internal links  
  - Edge cases:  
    - Subdomains or similar domains may be included or excluded depending on condition.  
    - Links with query parameters or anchors are included as is.

- **Merge**  
  - Type: Merge  
  - Role: Combines the two streams of links (absolute and converted relative links) into one list.  
  - Inputs:  
    - From `Append base URL` (relative links converted)  
    - From `Find relative links` (false branch, absolute links)  
  - Outputs: Connects to `Filter external links`  
  - Edge cases: None significant.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                          | Input Node(s)             | Output Node(s)          | Sticky Note                          |
|-------------------------|-------------------|----------------------------------------|---------------------------|-------------------------|------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Starts workflow manually                | -                         | Set Base URL            |                                    |
| Set Base URL            | Set               | Defines target URL                      | When clicking ‘Test workflow’ | Fetch base URL          | Configure the `url` field to target webpage |
| Fetch base URL          | HTTP Request      | Retrieves HTML content of webpage      | Set Base URL              | Extract links           |                                    |
| Extract links           | HTML              | Parses HTML to extract all anchor hrefs | Fetch base URL            | Split Out               |                                    |
| Split Out               | Split Out         | Splits array of links into individual items | Extract links             | Find relative links     |                                    |
| Find relative links     | If                | Checks if link is relative or absolute | Split Out                 | Append base URL (true), Merge (false) |                                    |
| Append base URL         | Set               | Converts relative links to absolute URLs | Find relative links (true) | Merge                   |                                    |
| Merge                   | Merge             | Combines absolute and converted relative links | Append base URL, Find relative links (false) | Filter external links |                                    |
| Filter external links   | Filter            | Filters out external links, keeps internal | Merge                     | -                       |                                    |
| Sticky Note             | Sticky Note       | -                                      | -                         | -                       |                                    |
| Sticky Note1            | Sticky Note       | -                                      | -                         | -                       |                                    |
| Sticky Note2            | Sticky Note       | -                                      | -                         | -                       |                                    |
| Sticky Note3            | Sticky Note       | -                                      | -                         | -                       |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually.

2. **Create a Set node**  
   - Name: `Set Base URL`  
   - Connect input from `When clicking ‘Test workflow’`  
   - Add a field named `url` (string) and set its value to the target webpage URL you want to analyze (e.g., `https://example.com`).

3. **Create an HTTP Request node**  
   - Name: `Fetch base URL`  
   - Connect input from `Set Base URL`  
   - Set HTTP Method to `GET`  
   - Set URL to use the expression referencing the `url` field from previous node: `{{$json["url"]}}`  
   - Set Response Format to `Text` (to get raw HTML).

4. **Create an HTML node**  
   - Name: `Extract links`  
   - Connect input from `Fetch base URL`  
   - Set CSS Selector to `a` (to select all anchor tags)  
   - Set Attribute to Extract to `href` (to get link URLs).

5. **Create a Split Out node**  
   - Name: `Split Out`  
   - Connect input from `Extract links`  
   - Purpose: To split the array of extracted links into individual items.

6. **Create an If node**  
   - Name: `Find relative links`  
   - Connect input from `Split Out`  
   - Set condition to check if the link is relative:  
     - Use expression to check if `href` starts with `/` or does not start with `http` (e.g., `{{$json["href"].startsWith("/") || !$json["href"].startsWith("http")}}`)  
   - True output: relative links  
   - False output: absolute links

7. **Create a Set node**  
   - Name: `Append base URL`  
   - Connect input from `Find relative links` true output  
   - Add a field to construct absolute URL by concatenating base URL and relative path:  
     - For example, set a field `href` with expression: `{{$node["Set Base URL"].json["url"].replace(/\/$/, '') + $json["href"]}}`  
     - This ensures no double slashes.

8. **Create a Merge node**  
   - Name: `Merge`  
   - Connect two inputs:  
     - From `Append base URL` (relative links converted)  
     - From `Find relative links` false output (absolute links)  
   - Set mode to `Merge By Index` or `Append` to combine both streams.

9. **Create a Filter node**  
   - Name: `Filter external links`  
   - Connect input from `Merge`  
   - Set condition to keep only links containing the base domain:  
     - For example, check if `href` contains the domain extracted from `Set Base URL` (e.g., `{{$json["href"].includes($node["Set Base URL"].json["url"].replace(/^https?:\/\//, '').split('/')[0])}}`)  
   - Output: filtered internal links.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Change the target URL in the `Set Base URL` node to analyze different webpages.                     | Workflow customization instruction                                                                 |
| Add nodes to filter, export, or send the extracted links for further processing.                    | Suggested workflow extensions                                                                       |
| This workflow automates internal link extraction to aid SEO and website structure analysis.         | Workflow purpose                                                                                   |
| For more information on n8n HTML node usage, visit: https://docs.n8n.io/nodes/n8n-nodes-base.html/  | Official n8n documentation                                                                          |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling reproduction, modification, and troubleshooting for advanced users and automation agents alike.