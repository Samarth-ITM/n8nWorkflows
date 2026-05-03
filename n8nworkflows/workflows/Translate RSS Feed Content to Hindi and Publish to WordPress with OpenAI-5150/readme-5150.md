Translate RSS Feed Content to Hindi and Publish to WordPress with OpenAI

https://n8nworkflows.xyz/workflows/translate-rss-feed-content-to-hindi-and-publish-to-wordpress-with-openai-5150


# Translate RSS Feed Content to Hindi and Publish to WordPress with OpenAI

---

### 1. Workflow Overview

This workflow automates the process of translating RSS feed content from English to Hindi using OpenAI, then publishing the translated content as draft posts on a WordPress site. It also handles featured image extraction from the original content, uploads the image to WordPress, and sets it as the featured image of the post.

Logical blocks:

- **1.1 RSS Feed Ingestion & Content Parsing:** Triggered every 10 minutes, it fetches new posts from a specified RSS feed and extracts the featured image URL embedded within the content.
- **1.2 Content Translation (English to Hindi):** Uses OpenAI’s Assistant resource to translate the post content and title from English to Hindi.
- **1.3 Image Handling & Upload:** Downloads the extracted featured image and uploads it to WordPress media library.
- **1.4 WordPress Post Creation & Image Assignment:** Creates a draft post on WordPress with the translated content and title, then assigns the uploaded image as the featured image.

---

### 2. Block-by-Block Analysis

#### 2.1 RSS Feed Ingestion & Content Parsing

**Overview:**  
This block triggers based on a recurring RSS feed poll, retrieves new posts, and extracts the featured image URL embedded within the content field of the RSS item.

**Nodes Involved:**  
- RSS Feed Trigger  
- Code1  
- Set Image URL

**Node Details:**

- **RSS Feed Trigger**  
  - Type: RSS Feed Read Trigger  
  - Role: Periodically polls the RSS feed URL for new posts every 10 minutes.  
  - Configuration: Feed URL set to `https://www.abc.com/uncategorized-hi/hindi-blog/feed/`. Poll frequency is every 10 minutes.  
  - Input: None (trigger node)  
  - Output: Emits newly found RSS feed items for processing.  
  - Edge Cases: Feed downtime, empty feed, malformed feed entries, or missing expected content fields.

- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Parses the RSS feed item’s `content:encoded` field to extract the URL inside `<featured-image>...</featured-image>` tags.  
  - Configuration: Uses regex to match the featured image URL inside the content and appends it as `featuredImage` property in the item’s JSON.  
  - Key Expression:  
    ```js
    const match = content.match(/<featured-image>(.*?)<\/featured-image>/);
    item.json.featuredImage = match ? match[1] : null;
    ```  
  - Input: Output from RSS Feed Trigger.  
  - Output: Items enriched with `featuredImage` field (string URL or null).  
  - Edge Cases: Missing or malformed `<featured-image>` tag, no match returns null.

- **Set Image URL**  
  - Type: Set  
  - Role: Assigns the extracted `featuredImage` URL to a new field `image-url` for downstream use.  
  - Configuration: Sets `image-url` = value of `featuredImage` from `Code1`.  
  - Input: Output from Code1.  
  - Output: Item with `image-url` field set.  
  - Edge Cases: Null or invalid URL propagation if no featured image found.

---

#### 2.2 Content Translation (English to Hindi)

**Overview:**  
This block translates both the post content and the post title from English to Hindi by utilizing the OpenAI Assistant resource configured for translation.

**Nodes Involved:**  
- Content English to Hindi  
- Title convert

**Node Details:**

- **Content English to Hindi**  
  - Type: Langchain OpenAI Node  
  - Role: Sends the original post content (`content:encoded`) to OpenAI’s assistant for translation to Hindi.  
  - Configuration:  
    - Text input: `{{$json['content:encoded']}}`  
    - Assistant ID: `asst_ydjGgj8Sax8qJgRgLh6XfBHB` (configured for English to Hindi translation)  
    - No additional options specified.  
  - Credentials: OpenAI API credential named "OpenAi account 3"  
  - Input: Enriched RSS feed item with content field.  
  - Output: Translated content as `output` field in JSON.  
  - Edge Cases: API rate limits, malformed input text, network errors, or empty content.

- **Title convert**  
  - Type: Langchain OpenAI Node  
  - Role: Translates the post title from English to Hindi using the same Assistant resource.  
  - Configuration:  
    - Text input: `{{$('RSS Feed Trigger').item.json.title}}` (original title from RSS feed)  
    - Assistant ID: same as content translation node.  
  - Credentials: Same OpenAI API credential as above.  
  - Input: Original RSS feed title.  
  - Output: Translated title as `output` field.  
  - Edge Cases: Same as content translation node, plus potential differences in text encoding.

---

#### 2.3 Image Handling & Upload

**Overview:**  
Downloads the featured image from the extracted URL and uploads it to the WordPress media library via REST API.

**Nodes Involved:**  
- GET Image  
- Upload Image to Wordpress

**Node Details:**

- **GET Image**  
  - Type: HTTP Request  
  - Role: Downloads the featured image binary data from the URL stored in `image-url`.  
  - Configuration:  
    - URL: `={{ $json["image-url"] }}` (dynamic URL from previous Set node)  
    - Method: GET (default)  
    - No authentication.  
  - Input: Item with `image-url`.  
  - Output: Binary data of the image.  
  - Edge Cases: Invalid URL, 404 not found, slow response, large images, network timeouts.

- **Upload Image to Wordpress**  
  - Type: HTTP Request  
  - Role: Uploads the image binary data to WordPress media endpoint.  
  - Configuration:  
    - URL: `https://www.abc.com/hindi/wp-json/wp/v2/media`  
    - Method: POST  
    - Content Type: binaryData (to send image data)  
    - Header: `Content-Disposition: attachment; filename="cover-image-{{ $('Wordpress').item.json.id }}.jpeg"` (sets filename dynamically based on post ID)  
    - Authentication: Predefined credential type using WordPress API OAuth2 or JWT token (credential "Abc Hindi Website")  
    - Input field: Uses binary data named `data` from GET Image.  
  - Credentials: WordPress API credential named "Abc Hindi Website"  
  - Input: Binary image data from GET Image node.  
  - Output: JSON response with uploaded media details (including media ID).  
  - Edge Cases: Auth failures, file size limits, invalid token, API rate limits, malformed headers.

---

#### 2.4 WordPress Post Creation & Image Assignment

**Overview:**  
Creates a draft WordPress post with translated title and content, then assigns the uploaded image as the featured image using REST API.

**Nodes Involved:**  
- Wordpress  
- Set Image URL (HTTP Request)  
- Set Image on Wordpress Post

**Node Details:**

- **Wordpress**  
  - Type: WordPress node (native n8n integration)  
  - Role: Creates a new draft post on WordPress with translated content and title.  
  - Configuration:  
    - Title: `={{ $json.output }}` (translated title from Title convert node)  
    - Content: `={{ $('Content English to Hindi').item.json.output }}` (translated content)  
    - Status: draft  
    - Author ID: 16 (hardcoded)  
    - Categories: [130] (hardcoded category IDs)  
    - Post template: empty  
  - Credentials: WordPress API credential "ABC Hindi Website"  
  - Input: Translated title and content.  
  - Output: JSON of the created post including post ID.  
  - Edge Cases: Auth failures, invalid credentials, category or author ID mismatch, API limits.

- **Set Image URL** (HTTP Request)  
  - Note: There is a naming overlap. This node is the HTTP Request node named "Set Image URL" responsible for fetching the image binary, already described in 2.3.

- **Set Image on Wordpress Post**  
  - Type: HTTP Request  
  - Role: Updates the created WordPress post by assigning the uploaded media ID as the featured image.  
  - Configuration:  
    - URL: `https://www.abc.com/hindi/wp-json/wp/v2/posts/{{ $('Wordpress').item.json.id }}` (dynamic post ID)  
    - Method: POST  
    - Query Parameter: `featured_media={{ $json.id }}` (media ID from upload image node)  
    - Authentication: WordPress API credential "abc Hindi Website"  
  - Input: Post ID from WordPress node and media ID from Upload Image node.  
  - Output: JSON response confirming featured image assignment.  
  - Edge Cases: API errors, mismatched IDs, auth failures, request timeouts.

---

### 3. Summary Table

| Node Name              | Node Type                      | Functional Role                          | Input Node(s)          | Output Node(s)               | Sticky Note                             |
|------------------------|--------------------------------|----------------------------------------|-----------------------|-----------------------------|---------------------------------------|
| RSS Feed Trigger       | RSS Feed Read Trigger           | Poll RSS feed for new posts            | -                     | Code1                       |                                       |
| Code1                  | Code                           | Extract featured image URL from content| RSS Feed Trigger       | Content English to Hindi     |                                       |
| Content English to Hindi| Langchain OpenAI Node          | Translate post content to Hindi        | Code1                  | Title convert                |                                       |
| Title convert          | Langchain OpenAI Node          | Translate post title to Hindi          | Content English to Hindi| Wordpress                   |                                       |
| Wordpress              | WordPress Node                 | Create draft post with translated data| Title convert           | Set Image URL (HTTP Request) |                                       |
| Set Image URL          | Set                            | Assign extracted featured image URL   | Code1                   | GET Image                   |                                       |
| GET Image              | HTTP Request                   | Download image binary from URL         | Set Image URL           | Upload Image to Wordpress    |                                       |
| Upload Image to Wordpress| HTTP Request                 | Upload image binary to WordPress media | GET Image               | Set Image on Wordpress Post  |                                       |
| Set Image on Wordpress Post| HTTP Request               | Set featured image of the post         | Wordpress, Upload Image to Wordpress | -                   |                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create RSS Feed Trigger Node**  
   - Type: RSS Feed Read Trigger  
   - Configure feed URL: `https://www.abc.com/uncategorized-hi/hindi-blog/feed/`  
   - Set polling interval: Every 10 minutes  
   - No credentials needed

2. **Create Code Node (Code1)**  
   - Type: Code  
   - Paste the following JavaScript to extract featured image URL from `content:encoded`:  
     ```js
     const items = $input.all();
     const updatedItems = items.map((item) => {
       const content = item?.json["content:encoded"];
       const match = content.match(/<featured-image>(.*?)<\/featured-image>/);
       item.json.featuredImage = match ? match[1] : null;
       return item;
     });
     return updatedItems;
     ```  
   - Connect RSS Feed Trigger output to Code1 input

3. **Create Set Node (Set Image URL)**  
   - Type: Set  
   - Add a new field `image-url` (string)  
   - Set value to `={{ $('Code1').item.json.featuredImage }}`  
   - Connect Code1 output to this node

4. **Create Langchain OpenAI Node (Content English to Hindi)**  
   - Type: Langchain OpenAI  
   - Text parameter: `={{ $json['content:encoded'] }}`  
   - Prompt: Use assistant configured for translation (assistant ID: `asst_ydjGgj8Sax8qJgRgLh6XfBHB`)  
   - Credentials: Configure OpenAI API credentials (e.g., "OpenAi account 3")  
   - Connect Code1 output to this node

5. **Create Langchain OpenAI Node (Title convert)**  
   - Type: Langchain OpenAI  
   - Text parameter: `={{ $('RSS Feed Trigger').item.json.title }}`  
   - Same assistant ID and credentials as previous node  
   - Connect Content English to Hindi output to Title convert input

6. **Create WordPress Node (Wordpress)**  
   - Type: WordPress  
   - Title: `={{ $json.output }}` (translated title from Title convert)  
   - Content: `={{ $('Content English to Hindi').item.json.output }}` (translated content)  
   - Status: draft  
   - Author ID: 16  
   - Categories: [130]  
   - Credentials: WordPress API credential (e.g., "ABC Hindi Website")  
   - Connect Title convert output to this node

7. **Create HTTP Request Node (GET Image)**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json["image-url"] }}`  
   - Connect Set Image URL output to this node

8. **Create HTTP Request Node (Upload Image to Wordpress)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://www.abc.com/hindi/wp-json/wp/v2/media`  
   - Content-Type: binaryData  
   - Send Headers: true  
   - Headers:  
     - `Content-Disposition`: `attachment; filename="cover-image-{{ $('Wordpress').item.json.id }}.jpeg"`  
   - Authentication: Use predefined WordPress API credential (e.g., "Abc Hindi Website")  
   - Input: Binary data named `data` (from GET Image)  
   - Connect GET Image output to this node

9. **Create HTTP Request Node (Set Image on Wordpress Post)**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://www.abc.com/hindi/wp-json/wp/v2/posts/{{ $('Wordpress').item.json.id }}`  
   - Query Parameter: `featured_media={{ $json.id }}` (media ID from Upload Image to Wordpress)  
   - Authentication: WordPress API credential (same as above)  
   - Connect Wordpress output and Upload Image to Wordpress output to this node

10. **Connect the image and post nodes**:  
    - Connect Set Image URL → GET Image → Upload Image to Wordpress → Set Image on Wordpress Post  
    - Connect Title convert → Wordpress → Set Image on Wordpress Post

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                            |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------|
| The workflow relies on OpenAI Assistant configured for English to Hindi translation, identified by assistant ID `asst_ydjGgj8Sax8qJgRgLh6XfBHB`. | OpenAI Langchain integration               |
| WordPress REST API endpoints used: `/wp-json/wp/v2/media` for media uploads and `/wp-json/wp/v2/posts/{id}` for post update. | WordPress API documentation                |
| Featured image extraction depends on the presence of `<featured-image>` tags in RSS feed content, which is custom markup. | Input RSS feed format requirement          |
| The workflow creates posts as drafts, allowing review before publishing.                                  | Workflow design choice                      |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---