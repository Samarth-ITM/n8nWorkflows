Posting from Wordpress to Medium

https://n8nworkflows.xyz/workflows/posting-from-wordpress-to-medium-2062


# Posting from Wordpress to Medium

### 1. Workflow Overview

This n8n workflow automates the process of reposting WordPress blog posts to Medium. It is designed to extract posts from a specified WordPress blog URL, parse and format their content, and then publish them on a Medium account with appropriate metadata.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Triggering the workflow manually.
- **1.2 WordPress Blog Index Fetching:** Requesting the main blog page to retrieve links to individual posts.
- **1.3 Post Links Extraction and Splitting:** Parsing the blog index page to extract blog post URLs and splitting them for individual processing.
- **1.4 Individual Blog Post Content Retrieval:** Fetching each blog post's full content using the extracted URLs.
- **1.5 Content Extraction and Formatting:** Parsing individual blog posts to extract title, introduction, and main content in HTML format.
- **1.6 Publishing to Medium:** Sending the formatted content to Medium with metadata like tags and publication status.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow manually by the user.
- **Nodes Involved:**  
  - *When clicking "Execute Workflow"*

- **Node Details:**
  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Configuration: Default manual trigger, no parameters set  
    - Inputs: None (manual activation)  
    - Outputs: Triggers the next node  
    - Potential Failures: None (manual trigger)  
    - Notes: Acts as the entry point for the workflow.

---

#### 1.2 WordPress Blog Index Fetching

- **Overview:** Fetch the main blog page from the WordPress site to obtain the HTML content containing blog post links.
- **Nodes Involved:**  
  - *HTTP Request*

- **Node Details:**
  - **HTTP Request**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: `https://mailsafi.com/blog/` (configurable to other blog URLs)  
      - Method: GET (default)  
      - Options: Default (no special headers or authentication)  
    - Inputs: Trigger from manual trigger node  
    - Outputs: Raw HTML content of the blog index page  
    - Potential Failures:  
      - Network issues (timeouts, DNS failures)  
      - HTTP errors (404, 500, etc.)  
    - Version: 4.1  
    - Notes: Critical to ensure the URL is correct and reachable.

---

#### 1.3 Post Links Extraction and Splitting

- **Overview:** Extract blog post titles and URLs from the fetched HTML, then split the list for individual processing.
- **Nodes Involved:**  
  - *HTML1*  
  - *Item Lists*  
  - *Item Lists1*  
  - *Loop Over Items*

- **Node Details:**

  - **HTML1**  
    - Type: HTML Extract  
    - Configuration:  
      - Operation: Extract HTML content  
      - Extraction rules:  
        - `post`: Selects `.entry-title > a` elements (blog post titles), returns an array  
        - `Link`: Selects `.lae-read-more > a` elements and extracts their `href` attribute, returns an array  
    - Inputs: Output from HTTP Request node (blog index HTML)  
    - Outputs: JSON array with post titles and links  
    - Potential Failures:  
      - CSS selector changes on the WordPress site breaking extraction  
      - Empty results if site structure changes  
    - Version: 1

  - **Item Lists**  
    - Type: Item Lists  
    - Configuration:  
      - Field to split out: `"post , Link"` (splits these arrays into individual items)  
    - Inputs: Output from HTML1  
    - Outputs: List of individual posts with title and link  
    - Potential Failures: Improper splitting if input data malformed  
    - Version: 3.1

  - **Item Lists1**  
    - Type: Item Lists  
    - Configuration:  
      - Operation: Limit  
      - Max Items: 5 (limits processing to first 5 posts for performance or testing)  
    - Inputs: Output from Item Lists  
    - Outputs: Limited list of posts  
    - Potential Failures: None significant, but limits workflow scope  
    - Version: 3.1

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Configuration: Default (no batch size set explicitly)  
    - Inputs: Output from Item Lists1  
    - Outputs: Splits list into batches (default batch size is 1) for sequential processing  
    - Potential Failures: None significant  
    - Version: 3

---

#### 1.4 Individual Blog Post Content Retrieval

- **Overview:** For each individual blog post URL, fetch the full HTML content.
- **Nodes Involved:**  
  - *HTTP Request1*

- **Node Details:**
  - **HTTP Request1**  
    - Type: HTTP Request  
    - Configuration:  
      - URL: Dynamic — expression `{{$json.Link}}` (uses current item's link)  
      - Method: GET (default)  
      - Options: Default  
    - Inputs: Output from the "Loop Over Items" node (second output)  
    - Outputs: HTML content of the individual blog post  
    - Potential Failures:  
      - Broken or inaccessible URLs  
      - HTTP errors  
      - Rate limiting or blocking by source site  
    - Version: 4.1  
    - Notes: Must handle various site response delays or errors gracefully.

---

#### 1.5 Content Extraction and Formatting

- **Overview:** Parse the fetched blog post HTML and extract key content parts: the post title, an introduction paragraph, and the main content in HTML format.
- **Nodes Involved:**  
  - *HTML*

- **Node Details:**
  - **HTML**  
    - Type: HTML Extract  
    - Configuration:  
      - Operation: Extract HTML content  
      - Extraction rules:  
        - `Title`: Select `h2.single-post-title` (post title)  
        - `Introduction`: Select first paragraph inside `.kiwi-highlighter-content-area > p:nth-child(1)`  
        - `Header`: Select entire `.kiwi-highlighter-content-area` div, returning raw HTML  
    - Inputs: Output from HTTP Request1 (individual blog post HTML)  
    - Outputs: JSON object with extracted title, intro, and content  
    - Potential Failures:  
      - CSS selectors become invalid if blog structure changes  
      - Missing elements leading to empty fields  
    - Version: 1

---

#### 1.6 Publishing to Medium

- **Overview:** Publish the extracted and formatted blog content to Medium, setting title, HTML content, tags, and publication status.
- **Nodes Involved:**  
  - *Medium*

- **Node Details:**
  - **Medium**  
    - Type: Medium Node (API integration)  
    - Configuration:  
      - Title: Dynamic from extracted `{{$json.Title}}`  
      - Content: Dynamic from extracted `{{$json.Header}}` (HTML content)  
      - Content Format: HTML  
      - Additional Fields:  
        - Tags: `"Email Hosting, Email, Email Marketing"` (static tags)  
        - Publish Status: `public`  
        - Notify Followers: `false`  
    - Inputs: Output from HTML node (content extraction)  
    - Outputs: Response from Medium API with published post info  
    - Potential Failures:  
      - Authentication errors (invalid or expired Medium credentials)  
      - API rate limits or quota exceeded  
      - Content rejected by Medium (too short, invalid HTML)  
    - Version: 1  
    - Credential Requirements: Medium OAuth2 credentials configured in n8n

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                     | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                                         |
|----------------------------|-----------------------|-----------------------------------|-------------------------------|-------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger         | Start workflow manually            | None                          | HTTP Request                  |                                                                                                                     |
| HTTP Request               | HTTP Request          | Fetch WordPress blog index page   | When clicking "Execute Workflow" | HTML1                         |                                                                                                                     |
| HTML1                      | HTML Extract          | Extract blog post titles & URLs   | HTTP Request                  | Item Lists                    |                                                                                                                     |
| Item Lists                 | Item Lists            | Split arrays of posts & links     | HTML1                         | Item Lists1                   |                                                                                                                     |
| Item Lists1                | Item Lists            | Limit number of posts processed   | Item Lists                    | Loop Over Items               |                                                                                                                     |
| Loop Over Items            | Split In Batches      | Process posts one at a time       | Item Lists1                  | HTTP Request1 (via 2nd output)|                                                                                                                     |
| HTTP Request1              | HTTP Request          | Fetch individual blog post content| Loop Over Items (2nd output)  | HTML                         |                                                                                                                     |
| HTML                       | HTML Extract          | Extract title, intro, content     | HTTP Request1                 | Medium                       |                                                                                                                     |
| Medium                     | Medium                | Publish formatted post to Medium  | HTML                          | Loop Over Items (1st output)  |                                                                                                                     |
| Sticky Note                | Sticky Note           | Usage instructions                | None                         | None                         | ## Usage \n**How to use me** This workflow gets all the posts from your WordPress site and sorts them into a clear format before publishing them to medium. Step 1. Set up the HTTP node and set the URL of the source destination. This will be the URL of the blog you want to use. We shall be using https://mailsafi.com/blog for this. Step 2. Extract the URLs of all the blogs on the page This gets all the blog titles and their URLs. Its an easy way to sort ou which blogs to share and which not to share. Step 3. Split the entries for easy sorting or a cleaner view. Step 4. Set a new https node with all the blog URLs that we got from the previous steps. Step 5. Extract the contents of the blog Step 6. Add the medium node and then set the contents that you want to be shared out. Execute your work flow and you are good to go |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: *Manual Trigger*  
   - No special configuration  
   - Name: `When clicking "Execute Workflow"`  

2. **Create HTTP Request Node to Fetch Blog Index**  
   - Type: *HTTP Request*  
   - Name: `HTTP Request`  
   - URL: `https://mailsafi.com/blog/` (replace with your WordPress blog URL)  
   - Method: GET (default)  
   - Connect output of manual trigger to this node.

3. **Create HTML Extract Node to Parse Blog Posts Links**  
   - Type: *HTML*  
   - Name: `HTML1`  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `post`, CSS Selector: `.entry-title > a`, Return Array: true  
     - Key: `Link`, CSS Selector: `.lae-read-more > a`, Attribute: `href`, Return Array: true  
   - Connect output of HTTP Request to this node.

4. **Create Item Lists Node to Split Posts and Links**  
   - Type: *Item Lists*  
   - Name: `Item Lists`  
   - Field to split out: `post , Link` (note the comma and space)  
   - Connect output of HTML1 to this node.

5. **Create Item Lists Node to Limit Number of Posts**  
   - Type: *Item Lists*  
   - Name: `Item Lists1`  
   - Operation: Limit  
   - Max Items: 5 (or set as required)  
   - Connect output of Item Lists to this node.

6. **Create Split In Batches Node to Loop Over Posts**  
   - Type: *Split In Batches*  
   - Name: `Loop Over Items`  
   - Default batch size (1) is fine for sequential processing  
   - Connect output of Item Lists1 to this node.

7. **Create HTTP Request Node to Fetch Individual Blog Post Content**  
   - Type: *HTTP Request*  
   - Name: `HTTP Request1`  
   - URL: Expression: `{{$json.Link}}` (dynamic per item)  
   - Method: GET (default)  
   - Connect second output of Loop Over Items (batches) to this node.

8. **Create HTML Extract Node to Parse Individual Blog Post Content**  
   - Type: *HTML*  
   - Name: `HTML`  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - Key: `Title`, CSS Selector: `h2.single-post-title`  
     - Key: `Introduction`, CSS Selector: `.kiwi-highlighter-content-area > p:nth-child(1)`  
     - Key: `Header`, CSS Selector: `div.kiwi-highlighter-content-area`, Return Value: `html`  
   - Connect output of HTTP Request1 to this node.

9. **Create Medium Node to Publish Posts**  
   - Type: *Medium*  
   - Name: `Medium`  
   - Title: Expression: `{{$json.Title}}`  
   - Content: Expression: `{{$json.Header}}`  
   - Content Format: `html`  
   - Additional Fields:  
     - Tags: `Email Hosting, Email, Email Marketing` (static tags)  
     - Publish Status: `public`  
     - Notify Followers: `false`  
   - Connect output of HTML node to this node.

10. **Connect Medium Node Output Back to Loop Over Items**  
    - Connect output of Medium node to first input of Loop Over Items node (to continue batch processing).

11. **Credentials Setup**  
    - Ensure Medium OAuth2 credentials are configured in n8n for the Medium node to authenticate.

12. **Final Check**  
    - Confirm all nodes are connected as per above steps.  
    - Test workflow manually by clicking `Execute Workflow`.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                 |
|----------------------------------------------------------------------------------------------------------------------|------------------------------------------------|
| This workflow is designed for the blog at https://mailsafi.com/blog but can be adapted by changing the initial URL. | Usage instructions embedded in Sticky Note.    |
| Medium node requires proper OAuth2 credentials setup in n8n to publish content.                                      | n8n Medium node documentation for credential setup. |
| Limiting posts to 5 in `Item Lists1` node is for performance/testing; adjust as needed for production.                | Can be removed or increased to process more posts. |
| CSS selectors used in HTML extraction nodes depend on the target website’s structure and may require updates if site changes. | Monitor site updates to maintain workflow reliability. |

---

This documentation fully captures the structure, logic, and configuration of the "Posting from Wordpress to Medium" workflow, enabling reliable reproduction, modification, and error anticipation.