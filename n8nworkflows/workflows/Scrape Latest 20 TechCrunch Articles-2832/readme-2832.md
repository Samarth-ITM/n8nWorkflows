Scrape Latest 20 TechCrunch Articles

https://n8nworkflows.xyz/workflows/scrape-latest-20-techcrunch-articles-2832


# Scrape Latest 20 TechCrunch Articles

### 1. Workflow Overview

This workflow automates the scraping of the latest 20 articles from TechCrunch’s "Recent" page. It is designed for developers, content creators, and data analysts who require up-to-date news articles with metadata for aggregation, analysis, or integration purposes.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Fetch Latest Articles Page:** HTTP request to retrieve the TechCrunch "Recent" page HTML.
- **1.3 Extract Articles List:** Parse the HTML to isolate the container holding article listings and then extract individual article entries.
- **1.4 Process Each Article Summary:** Split the list of articles and parse each article’s summary metadata (title, URL, image, publication date).
- **1.5 Fetch and Parse Article Details:** For each article URL, send an HTTP request to retrieve the full article page and parse detailed content and metadata.
- **1.6 Save Extracted Data:** Consolidate and save the extracted article information for downstream use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block initiates the workflow manually.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’
- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connected to the HTTP Request node fetching the latest TechCrunch page.  
    - Edge Cases: None; manual trigger is straightforward.

#### 1.2 Fetch Latest Articles Page

- **Overview:** Retrieves the HTML content of TechCrunch’s "Recent" page to access the latest articles.
- **Nodes Involved:**  
  - Request Techcrunsh Latest Page
- **Node Details:**

  - **Request Techcrunsh Latest Page**  
    - Type: HTTP Request  
    - Role: Fetches the HTML page from https://techcrunch.com/latest/0  
    - Configuration:  
      - Method: GET (default)  
      - URL: `https://techcrunch.com/latest/0` (static URL targeting the first page of recent articles)  
      - No authentication or headers configured by default; user may add if needed.  
    - Inputs: From Manual Trigger  
    - Outputs: HTML content passed to the next node for parsing.  
    - Edge Cases:  
      - Network errors or timeouts.  
      - Website structure changes causing unexpected HTML.  
      - Potential rate limiting or blocking by TechCrunch.  
    - Version: HTTP Request node version 4.2

#### 1.3 Extract Articles List

- **Overview:** Parses the fetched HTML to isolate the container holding the list of articles, then extracts individual article entries as HTML snippets.
- **Nodes Involved:**  
  - Parse a posts box  
  - Parse all posts
- **Node Details:**

  - **Parse a posts box**  
    - Type: HTML Extract  
    - Role: Extracts the main container `<ul>` element with class `wp-block-post-template` that holds the articles.  
    - Configuration:  
      - Operation: Extract HTML content  
      - CSS Selector: `ul.wp-block-post-template`  
      - Return Value: HTML content of the container, stored under key `box`  
    - Inputs: HTML from HTTP Request node  
    - Outputs: Passes extracted container HTML to next node  
    - Edge Cases:  
      - Selector may fail if TechCrunch changes page structure.  
      - Empty or missing container leads to no articles extracted.

  - **Parse all posts**  
    - Type: HTML Extract  
    - Role: Extracts each article `<li>` element inside the container as an array of HTML snippets.  
    - Configuration:  
      - Operation: Extract HTML content  
      - Data Property Name: `box` (input property)  
      - CSS Selector: `li.wp-block-post`  
      - Return Array: true  
      - Return Value: HTML snippets of each article under key `posts`  
      - Trim Values: enabled to clean whitespace  
    - Inputs: Container HTML from previous node  
    - Outputs: Array of article HTML snippets passed downstream  
    - Edge Cases:  
      - Empty array if no articles found.  
      - Selector failure if page structure changes.

#### 1.4 Process Each Article Summary

- **Overview:** Splits the array of article HTML snippets into individual items and parses each to extract summary metadata.
- **Nodes Involved:**  
  - split out the posts  
  - Parse each post in detail
- **Node Details:**

  - **split out the posts**  
    - Type: Split Out  
    - Role: Splits the array of posts into individual workflow items for parallel processing.  
    - Configuration:  
      - Field to Split Out: `posts`  
    - Inputs: Array of posts from previous node  
    - Outputs: One item per article HTML snippet  
    - Edge Cases:  
      - Empty input array results in no output items.

  - **Parse each post in detail**  
    - Type: HTML Extract  
    - Role: Extracts key metadata from each article snippet: image URL, title, article URL, and publication date.  
    - Configuration:  
      - Operation: Extract HTML content  
      - Data Property Name: `posts` (each item’s HTML snippet)  
      - Extraction Values:  
        - `image`: `img` tag’s `src` attribute  
        - `title`: text inside `h3.loop-card__title`  
        - `url`: `data-destinationlink` attribute of `h3 > a`  
        - `created_at`: `datetime` attribute of `time` tag  
      - Trim Values: enabled  
    - Inputs: Single article snippet from Split Out node  
    - Outputs: Parsed metadata JSON per article  
    - Edge Cases:  
      - Missing attributes or tags may result in null or empty fields.  
      - URL extraction depends on correct attribute presence.

#### 1.5 Fetch and Parse Article Details

- **Overview:** For each article URL, fetches the full article page and extracts detailed content and metadata.
- **Nodes Involved:**  
  - Request a post detail page  
  - Parse a post's content and metadata
- **Node Details:**

  - **Request a post detail page**  
    - Type: HTTP Request  
    - Role: Fetches the full article HTML page using the URL extracted previously.  
    - Configuration:  
      - URL: Dynamic expression `={{ $json.url }}` to use each article’s URL  
      - Method: GET (default)  
      - No authentication or headers by default  
    - Inputs: Parsed article metadata from previous node  
    - Outputs: Full article HTML content  
    - Edge Cases:  
      - Invalid or broken URLs cause request failures.  
      - Network errors or timeouts.  
      - Rate limiting or blocking possible.  
    - Version: HTTP Request node version 4.2

  - **Parse a post's content and metadata**  
    - Type: HTML Extract  
    - Role: Extracts main article content and metadata from the full article page.  
    - Configuration:  
      - Operation: Extract HTML content  
      - Extraction Values:  
        - `content`: HTML inside `div.entry-content` (main article body)  
        - `title`: text inside `h1.wp-block-post-title`  
        - `thumbnail`: `src` attribute of `img.attachment-post-thumbnail`  
        - `created_at`: `datetime` attribute of `time` tag  
      - Trim Values: enabled  
      - Clean Up Text: enabled to remove extraneous whitespace and HTML artifacts  
      - Execute Once: false (runs per item)  
    - Inputs: Full article HTML from HTTP Request node  
    - Outputs: Detailed article content and metadata JSON  
    - Edge Cases:  
      - Missing selectors or attributes may yield incomplete data.  
      - Changes in article page structure can break extraction.

#### 1.6 Save Extracted Data

- **Overview:** Consolidates and saves the extracted article data into a structured format for further use.
- **Nodes Involved:**  
  - Save the values
- **Node Details:**

  - **Save the values**  
    - Type: Set  
    - Role: Assigns final output fields combining summary and detailed data for each article.  
    - Configuration:  
      - Assignments:  
        - `url`: from "Parse each post in detail" node’s `url` field  
        - `created_at`: from "Parse each post in detail" node’s `created_at`  
        - `image`: from "Parse each post in detail" node’s `image`  
        - `content`: from current node’s `content` field (detailed content)  
        - `title`: from current node’s `title` field (detailed title)  
    - Inputs: Detailed article content and metadata from previous node  
    - Outputs: Final structured article data per item  
    - Edge Cases:  
      - Missing fields in source nodes propagate as empty values.

---

### 3. Summary Table

| Node Name                     | Node Type          | Functional Role                          | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------------|--------------------|----------------------------------------|-------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’  | Manual Trigger     | Starts the workflow manually            | None                          | Request Techcrunsh Latest Page  |                                                                                              |
| Request Techcrunsh Latest Page | HTTP Request       | Fetches TechCrunch "Recent" page HTML  | When clicking ‘Test workflow’ | Parse a posts box               | Be sure to update HTTP Request nodes with any necessary headers or authentication.           |
| Parse a posts box              | HTML Extract       | Extracts container holding article list| Request Techcrunsh Latest Page| Parse all posts                |                                                                                              |
| Parse all posts               | HTML Extract       | Extracts individual article snippets    | Parse a posts box             | split out the posts             |                                                                                              |
| split out the posts           | Split Out          | Splits array of articles into items     | Parse all posts               | Parse each post in detail       |                                                                                              |
| Parse each post in detail     | HTML Extract       | Extracts summary metadata per article   | split out the posts           | Request a post detail page      |                                                                                              |
| Request a post detail page    | HTTP Request       | Fetches full article page HTML           | Parse each post in detail     | Parse a post's content and metadata |                                                                                              |
| Parse a post's content and metadata | HTML Extract | Extracts detailed content and metadata  | Request a post detail page    | Save the values                |                                                                                              |
| Save the values              | Set                | Consolidates and saves final article data| Parse a post's content and metadata | None                         |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To manually start the workflow.

2. **Add an HTTP Request node**  
   - Name: `Request Techcrunsh Latest Page`  
   - URL: `https://techcrunch.com/latest/0`  
   - Method: GET (default)  
   - Connect output of Manual Trigger to this node.

3. **Add an HTML Extract node**  
   - Name: `Parse a posts box`  
   - Operation: Extract HTML content  
   - CSS Selector: `ul.wp-block-post-template`  
   - Return Value: HTML  
   - Output Key: `box`  
   - Connect output of HTTP Request node to this node.

4. **Add another HTML Extract node**  
   - Name: `Parse all posts`  
   - Operation: Extract HTML content  
   - Data Property Name: `box`  
   - CSS Selector: `li.wp-block-post`  
   - Return Array: true  
   - Return Value: HTML  
   - Trim Values: enabled  
   - Connect output of `Parse a posts box` node to this node.

5. **Add a Split Out node**  
   - Name: `split out the posts`  
   - Field to Split Out: `posts`  
   - Connect output of `Parse all posts` node to this node.

6. **Add an HTML Extract node**  
   - Name: `Parse each post in detail`  
   - Operation: Extract HTML content  
   - Data Property Name: `posts`  
   - Extraction Values:  
     - `image`: CSS Selector `img`, Return Attribute `src`  
     - `title`: CSS Selector `h3.loop-card__title`  
     - `url`: CSS Selector `h3 > a`, Return Attribute `data-destinationlink`  
     - `created_at`: CSS Selector `time`, Return Attribute `datetime`  
   - Trim Values: enabled  
   - Connect output of `split out the posts` node to this node.

7. **Add an HTTP Request node**  
   - Name: `Request a post detail page`  
   - URL: Expression `={{ $json.url }}` (dynamic URL from previous node)  
   - Method: GET (default)  
   - Connect output of `Parse each post in detail` node to this node.

8. **Add an HTML Extract node**  
   - Name: `Parse a post's content and metadata`  
   - Operation: Extract HTML content  
   - Extraction Values:  
     - `content`: CSS Selector `div.entry-content`  
     - `title`: CSS Selector `h1.wp-block-post-title`  
     - `thumbnail`: CSS Selector `img.attachment-post-thumbnail`, Return Attribute `src`  
     - `created_at`: CSS Selector `time`, Return Attribute `datetime`  
   - Trim Values: enabled  
   - Clean Up Text: enabled  
   - Connect output of `Request a post detail page` node to this node.

9. **Add a Set node**  
   - Name: `Save the values`  
   - Assignments:  
     - `url`: Expression `={{ $('Parse each post in detail').item.json.url }}`  
     - `created_at`: Expression `={{ $('Parse each post in detail').item.json.created_at }}`  
     - `image`: Expression `={{ $('Parse each post in detail').item.json.image }}`  
     - `content`: Expression `={{ $json.content }}`  
     - `title`: Expression `={{ $json.title }}`  
   - Connect output of `Parse a post's content and metadata` node to this node.

10. **Verify all connections follow the sequence:**  
    Manual Trigger → Request Techcrunsh Latest Page → Parse a posts box → Parse all posts → split out the posts → Parse each post in detail → Request a post detail page → Parse a post's content and metadata → Save the values

11. **Credentials:**  
    - No credentials are required by default since the workflow scrapes publicly accessible web pages.  
    - If TechCrunch requires headers or authentication in the future, configure HTTP Request nodes accordingly.

12. **Defaults and Constraints:**  
    - The workflow fetches only the first page (`/latest/0`) which contains the latest 20 articles.  
    - To change the number of articles or pages, modify the URL in the first HTTP Request node.  
    - CSS selectors are tightly coupled to TechCrunch’s current page structure; monitor for changes.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Be sure to update HTTP Request nodes with any necessary headers or authentication to work with TechCrunch’s website. | Workflow Setup Instructions (from description)                                                  |
| This workflow is ideal for developers, content creators, and data analysts needing automated article scraping. | Workflow Purpose (from description)                                                             |
| Customize by modifying HTTP Request URLs or adding processing steps to filter or analyze articles. | Workflow Customization Suggestions (from description)                                           |
| TechCrunch page structure changes may require updating CSS selectors in HTML Extract nodes.        | Maintenance Consideration                                                                       |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Scrape Latest 20 TechCrunch Articles" workflow in n8n.