Automatic Blog Post Generation from WooCommerce Products with GPT-4.1-mini

https://n8nworkflows.xyz/workflows/automatic-blog-post-generation-from-woocommerce-products-with-gpt-4-1-mini-5445


# Automatic Blog Post Generation from WooCommerce Products with GPT-4.1-mini

### 1. Workflow Overview

This workflow automates the generation and publication of blog posts based on WooCommerce products using GPT-4.1-mini from OpenAI. It is designed for WooCommerce store owners or marketers who want to create engaging, quirky, and shareable blog content about their products automatically, without manual writing.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the workflow every 6 hours.
- **1.2 WooCommerce Product Retrieval:** Fetches up to 100 products from the WooCommerce REST API using Basic Authentication.
- **1.3 Product Selection:** Randomly sorts the fetched products and selects one product for blog post generation.
- **1.4 AI Blog Post Generation:** Sends the selected product permalink as prompt input to OpenAI’s GPT-4.1-mini model to generate a fun, curiosity-driven blog post.
- **1.5 Post Formatting:** Parses and formats the AI-generated content to extract title, content, excerpt, and slug suitable for WordPress.
- **1.6 WordPress Publishing:** Publishes the formatted blog post to a WordPress site using Basic Authentication.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  This block periodically triggers the workflow every 6 hours at 17 minutes past the hour.

- **Nodes Involved:**  
  - Trigger on Schedule

- **Node Details:**  
  - **Trigger on Schedule**  
    - Type: Schedule Trigger  
    - Configuration: Runs every 6 hours, triggering at minute 17.  
    - Input: None (trigger node)  
    - Output: Starts the workflow by sending a trigger event.  
    - Notes: User can customize the schedule timing.  
    - Potential Failures: None typical, but workflow won't run if the n8n instance is offline or scheduler disabled.

#### 2.2 WooCommerce Product Retrieval

- **Overview:**  
  Retrieves up to 100 products from the WooCommerce store REST API using Basic Authentication credentials.

- **Nodes Involved:**  
  - Pull Woocommerce Product  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Pull Woocommerce Product**  
    - Type: HTTP Request  
    - Configuration:  
      - URL set to WooCommerce REST API endpoint for products with per_page=100.  
      - Uses Basic Auth credentials stored in n8n credentials manager.  
      - Pulls products in paginated batches if needed (but current config fetches 100 at once).  
    - Input: Trigger from schedule node  
    - Output: JSON array of product objects  
    - Notes: Requires valid WooCommerce API key with read permissions; domain must be updated for the actual store URL.  
    - Edge Cases: Authentication failure, API rate limits, network timeouts, empty product lists.  
  - **Sticky Note**  
    - Content: "Pull 100 Woocommerce Products. Set shop's authentication credentials here."

#### 2.3 Product Selection

- **Overview:**  
  Randomizes the list of retrieved products and selects one product to generate a blog post about.

- **Nodes Involved:**  
  - Sort Products Randomly  
  - Return Only One

- **Node Details:**  
  - **Sort Products Randomly**  
    - Type: Sort  
    - Configuration: Sort type set to "random" to shuffle product order.  
    - Input: Output from WooCommerce product retrieval node  
    - Output: Randomly sorted list of products  
    - Edge Cases: Empty input list results in no output; no error but downstream nodes will have no data.  
  - **Return Only One**  
    - Type: Code (JavaScript)  
    - Configuration: Returns only the first item from the input array (the first product after random sort).  
    - Input: Randomly sorted products array  
    - Output: Single product object  
    - Potential Failures: If input is empty, function returns undefined or error downstream.

#### 2.4 AI Blog Post Generation

- **Overview:**  
  Sends the selected product permalink to OpenAI GPT-4.1-mini with a prompt to create a fun and engaging blog post.

- **Nodes Involved:**  
  - Create Blog Post About Product Selected Via GPT-4.1-MINI  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Create Blog Post About Product Selected Via GPT-4.1-MINI**  
    - Type: OpenAI (Langchain node)  
    - Configuration:  
      - Model: gpt-4.1-mini  
      - Prompt: Requests a fun, curiosity-driven post about the product permalink, targeting quirky, unique gift seekers.  
      - Tone: Casual, catchy, shareable, avoiding corporate style.  
      - Includes product link in the post.  
    - Input: Single product JSON with permalink  
    - Output: AI-generated blog post text in message content  
    - Credentials: OpenAI API key  
    - Edge Cases: API limits, authentication errors, prompt failures, empty or irrelevant output.  
  - **Sticky Note1**  
    - Content: "Create blog post about selected product via OpenAi. Set OpenAi credentials here."

#### 2.5 Post Formatting

- **Overview:**  
  Parses the AI-generated blog post content to extract a suitable blog post title, slug, excerpt, and content for WordPress publishing.

- **Nodes Involved:**  
  - Format Post For Publishing  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Format Post For Publishing**  
    - Type: Code (JavaScript)  
    - Configuration Logic:  
      - Extracts the AI text from first input item’s `message.content`.  
      - Attempts to separate title and content by detecting double newlines.  
      - If no explicit title found, uses first line as title if under 120 chars; else defaults to "Untitled Blog Post".  
      - Generates slug by lowercasing title and replacing non-alphanumeric characters with hyphens.  
      - Creates excerpt from first ~30 words of content.  
      - Outputs JSON with `title`, `slug`, `excerpt`, and `content`.  
    - Input: AI-generated blog post content  
    - Output: Structured post object, ready for publishing  
    - Potential Failures: Missing or malformed AI response, empty content, errors in regex or string operations.  
  - **Sticky Note2**  
    - Content: "Set your Wordpress authentication credentials here."

#### 2.6 WordPress Publishing

- **Overview:**  
  Publishes the formatted blog post to the WordPress site via REST API using Basic Authentication.

- **Nodes Involved:**  
  - Publish to Wordpress  
  - Sticky Note (instructional)

- **Node Details:**  
  - **Publish to Wordpress**  
    - Type: HTTP Request  
    - Configuration:  
      - Method: POST  
      - URL: WordPress REST API endpoint for posts (`/wp-json/wp/v2/posts`)  
      - Body includes title, content, excerpt, slug, status as "publish", and category ID 1.  
      - Uses Basic Auth credentials stored in n8n credentials manager.  
    - Input: Formatted blog post JSON from formatting node  
    - Output: HTTP response from WordPress confirming post creation  
    - Edge Cases: Authentication failure, permission denied, invalid post data, network issues.  
  - **Sticky Note2**  
    - Content: "Set your Wordpress authentication credentials here."

---

### 3. Summary Table

| Node Name                                | Node Type                       | Functional Role                         | Input Node(s)             | Output Node(s)                                | Sticky Note                                               |
|-----------------------------------------|--------------------------------|---------------------------------------|---------------------------|-----------------------------------------------|-----------------------------------------------------------|
| Trigger on Schedule                      | Schedule Trigger                | Initiates workflow on scheduled timer| None                      | Pull Woocommerce Product                       | Change day and time schedule for the triggering of this node. |
| Pull Woocommerce Product                 | HTTP Request                   | Retrieves WooCommerce products         | Trigger on Schedule       | Sort Products Randomly                         | Pull 100 Woocommerce Products. Set shop's authentication credentials here. |
| Sort Products Randomly                   | Sort                          | Randomizes product list                 | Pull Woocommerce Product  | Return Only One                               |                                                           |
| Return Only One                         | Code                          | Selects single product                  | Sort Products Randomly    | Create Blog Post About Product Selected Via GPT-4.1-MINI |                                                           |
| Create Blog Post About Product Selected Via GPT-4.1-MINI | OpenAI (Langchain)            | Generates blog post using GPT-4.1-mini| Return Only One           | Format Post For Publishing                     | Create blog post about selected product via OpenAi. Set OpenAi credentials here. |
| Format Post For Publishing               | Code                          | Extracts title, slug, excerpt, content | Create Blog Post About Product Selected Via GPT-4.1-MINI | Publish to Wordpress                           | Set your Wordpress authentication credentials here.       |
| Publish to Wordpress                     | HTTP Request                   | Publishes blog post to WordPress       | Format Post For Publishing | None                                          | Set your Wordpress authentication credentials here.       |
| Sticky Note                             | Sticky Note                   | Instructional comment                   | None                      | None                                          | Pull 100 Woocommerce Products. Set shop's authentication credentials here. |
| Sticky Note1                            | Sticky Note                   | Instructional comment                   | None                      | None                                          | Create blog post about selected product via OpenAi. Set OpenAi credentials here. |
| Sticky Note2                            | Sticky Note                   | Instructional comment                   | None                      | None                                          | Set your Wordpress authentication credentials here.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger node**  
   - Type: Schedule Trigger  
   - Configure to run every 6 hours at minute 17.

2. **Add an HTTP Request node for WooCommerce Product Retrieval**  
   - Name: Pull Woocommerce Product  
   - Method: GET  
   - URL: `https://yourwebsite.com/wp-json/wc/v3/products?per_page=100` (replace with actual domain)  
   - Authentication: Basic Auth  
   - Credentials: Select or create WooCommerce API credentials with Read permissions  
   - Connect output of Scheduled Trigger node to this node.

3. **Add a Sort node to randomize products**  
   - Name: Sort Products Randomly  
   - Sort type: Random  
   - Connect output of WooCommerce Product Retrieval node to this node.

4. **Add a Code node to select only one product**  
   - Name: Return Only One  
   - Language: JavaScript  
   - Code:  
     ```javascript
     return items[0];
     ```  
   - Connect output of Sort node to this node.

5. **Add OpenAI node to generate blog post**  
   - Name: Create Blog Post About Product Selected Via GPT-4.1-MINI  
   - Provider: OpenAI (Langchain)  
   - Model: gpt-4.1-mini  
   - Credentials: Configure OpenAI API credentials  
   - Message prompt:  
     ```
     Write a fun, curiosity-driven blog post for the following product {{ $json.permalink }}. The post should appeal to people looking for quirky, unique, or thoughtful gifts—like birthdays, holidays, viral TikTok items, minimalist gifts, or AI-themed stuff. Make it casual, catchy, and shareable. Avoid sounding corporate. Only write one. Include the link to the product in the post.
     ```  
   - Connect output of Return Only One node to this node.

6. **Add a Code node to format AI output**  
   - Name: Format Post For Publishing  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const raw = $input.first().json.message.content || "";

     if (!raw) {
       throw new Error("No content found in 'content' field.");
     }

     let title = "";
     let content = "";

     if (raw.includes("\n\n")) {
       const [maybeTitle, ...rest] = raw.split("\n\n");
       if (maybeTitle.length < 120) {
         title = maybeTitle.trim();
         content = rest.join("\n\n").trim();
       } else {
         content = raw.trim();
       }
     } else {
       content = raw.trim();
     }

     if (!title) {
       const firstLine = content.split("\n")[0];
       if (firstLine.length < 120) {
         title = firstLine.trim();
         content = content.replace(firstLine, "").trim();
       } else {
         title = "Untitled Blog Post";
       }
     }

     const slug = title
       .toLowerCase()
       .replace(/[^a-z0-9]+/g, "-")
       .replace(/(^-|-$)/g, "");

     const excerpt = content.split(/\s+/).slice(0, 30).join(" ") + "...";

     return [
       {
         json: {
           title,
           slug,
           excerpt,
           content,
         },
       },
     ];
     ```  
   - Connect output of OpenAI node to this node.

7. **Add an HTTP Request node to publish to WordPress**  
   - Name: Publish to Wordpress  
   - Method: POST  
   - URL: `https://yourwebsite.com/wp-json/wp/v2/posts` (replace with your WordPress domain)  
   - Authentication: Basic Auth  
   - Credentials: Configure WordPress API credentials with permission to publish posts  
   - Body parameters (JSON):  
     - title: `={{$json["title"]}}`  
     - content: `={{$json.content}}`  
     - excerpt: `={{$json.excerpt}}`  
     - slug: `={{$json["slug"]}}`  
     - status: `publish`  
     - categories[0]: `1` (or adjust category as needed)  
   - Connect output of Format Post For Publishing node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                    |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| Pull 100 WooCommerce Products. Set shop's authentication credentials here.                     | Instructional sticky note near WooCommerce HTTP Request node.    |
| Create blog post about selected product via OpenAI. Set OpenAI credentials here.               | Instructional sticky note near OpenAI node.                      |
| Set your WordPress authentication credentials here.                                          | Instructional sticky note near WordPress HTTP Request node.      |
| Adjust domain URLs in WooCommerce and WordPress nodes to match your live site URLs.           | Critical for successful API communication.                       |
| OpenAI prompt is designed to create casual, quirky blog posts aimed at gift seekers and viral trends. | Can be customized to fit brand tone or target audience.          |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.