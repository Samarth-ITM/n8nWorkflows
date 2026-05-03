Extract Text from Instagram Posts (Single & Carousel) using HikerAPI & OCR.Space

https://n8nworkflows.xyz/workflows/extract-text-from-instagram-posts--single---carousel--using-hikerapi---ocr-space-10853


# Extract Text from Instagram Posts (Single & Carousel) using HikerAPI & OCR.Space

### 1. Workflow Overview

This workflow extracts raw text from Instagram posts, whether they are single images or carousel posts (multiple images), using two main external services: HikerAPI to retrieve media details from an Instagram post URL, and OCR.Space to perform optical character recognition (OCR) on images. The workflow is designed to handle different Instagram post types—single images, carousels, and reels—by branching logic accordingly.

Logical blocks of the workflow are:

- **1.1 Input Reception:** Receives the Instagram post URL.
- **1.2 Media Retrieval:** Calls HikerAPI to get media metadata from the Instagram post URL.
- **1.3 Post Type Detection & Routing:** Determines whether the post is a single image, a carousel, or a reel.
- **1.4 OCR Processing:**
  - For single posts: sends the image to OCR.Space.
  - For carousels: loops over each slide and sends each image to OCR.Space.
  - For reels: currently no OCR processing is done.
- **1.5 Text Aggregation:** Aggregates all OCR results into a single combined text output.
- **1.6 Output Preparation:** Formats the final extracted text for further usage or export.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block initializes the process by setting the Instagram post URL that the workflow will analyze.
- **Nodes Involved:**  
  - IGPost URL  
  - When clicking ‘Execute workflow’

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Serves as the workflow starting point for manual execution in n8n.  
    - Configuration: No parameters; simply a trigger node.  
    - Input/Output: No input; output connects to IGPost URL node.  
    - Edge cases: None significant; manual trigger only.

  - **IGPost URL**  
    - Type: Set Node  
    - Role: Holds a static Instagram post URL for testing or demonstration.  
    - Configuration: Assigns a string value to the variable `post_url` with example URL `https://www.instagram.com/p/DQf3KpkE4cW/?img_index=11`.  
    - Input/Output: Input from manual trigger; output to Retrieve Media node.  
    - Edge cases: URL must be valid Instagram post URL; malformed or private posts may cause downstream errors.

---

#### 2.2 Media Retrieval

- **Overview:** Retrieves detailed media information about the Instagram post via HikerAPI using the post URL.
- **Nodes Involved:**  
  - Retrieve Media  
  - Post Type Selector

- **Node Details:**

  - **Retrieve Media**  
    - Type: HTTP Request  
    - Role: Calls HikerAPI endpoint `https://api.hikerapi.com/v1/media/by/url` with query parameter `url` set to the Instagram post URL.  
    - Configuration: Uses HTTP header authentication with a HikerAPI key credential. Sets `accept: application/json` header.  
    - Input/Output: Input from IGPost URL node; output to Post Type Selector node.  
    - Edge cases:  
      - API key invalid or rate-limited.  
      - Instagram post URL invalid or media not found.  
      - Network timeouts or HTTP errors.  
    - Version: HTTP Request node v4.3.

  - **Post Type Selector**  
    - Type: Switch  
    - Role: Determines the type of Instagram post based on `product_type` field in HikerAPI response JSON.  
    - Configuration:  
      - Routes to `single_post` if `product_type` equals `feed`.  
      - Routes to `carousel` if `product_type` equals `carousel_container`.  
      - Routes to `reels` if `product_type` equals `clips`.  
    - Input/Output: Input from Retrieve Media; outputs to OCR_Single, get_all_slide, or No Operation nodes accordingly.  
    - Edge cases: If `product_type` is missing or unexpected, node routes to no-op; no OCR done.

---

#### 2.3 OCR Processing for Single Post

- **Overview:** Processes a single Instagram image by sending its URL to OCR.Space and extracting text.
- **Nodes Involved:**  
  - OCR_Single  
  - getSingleText  
  - Merge All Parsed Text  
  - Result of Raw Text

- **Node Details:**

  - **OCR_Single**  
    - Type: HTTP Request  
    - Role: Sends the first image URL from the media data (`image_versions[0].url`) to OCR.Space API endpoint for parsing the image text.  
    - Configuration: Uses HTTP query authentication with OCR.Space API key credential. Sends URL as query parameter `url`.  
    - Input/Output: Input from Post Type Selector node (single_post output); output to getSingleText node.  
    - Edge cases:  
      - OCR.Space API limits or failures.  
      - Image URL invalid or expired.  
      - Network errors.  
    - Version: HTTP Request node v4.3.

  - **getSingleText**  
    - Type: Code (JavaScript)  
    - Role: Extracts the parsed text from OCR.Space response JSON (`ParsedResults[0].ParsedText`).  
    - Configuration: Returns JSON object with a single field `value` containing extracted text string.  
    - Input/Output: Input from OCR_Single; output to Merge All Parsed Text node.  
    - Edge cases: If no parsed results or empty text, returns empty string.

  - **Merge All Parsed Text**  
    - Type: Aggregate  
    - Role: Aggregates text results—though for single post only one item expected—into one dataset for consistency with carousel flow.  
    - Configuration: Aggregates all item data without filtering.  
    - Input/Output: Input from getSingleText; output to Result of Raw Text node.  
    - Edge cases: Should handle empty inputs gracefully.

  - **Result of Raw Text**  
    - Type: Code (JavaScript)  
    - Role: Concatenates all extracted text from aggregated results into a single text string separated by line breaks.  
    - Configuration:  
      - Flattens array of data items.  
      - Extracts `value` fields.  
      - Filters out empty strings.  
      - Joins with newline characters.  
    - Input/Output: Input from Merge All Parsed Text; output is final workflow output.  
    - Edge cases: No extracted text returns empty string.

---

#### 2.4 OCR Processing for Carousel Post

- **Overview:** Handles Instagram carousel posts by looping through each slide image, sending each to OCR.Space, and collecting all text results.
- **Nodes Involved:**  
  - get_all_slide  
  - Loop Over Items1  
  - OCR_Slide  
  - getOnlyText  
  - Merge All Parsed Text  
  - Result of Raw Text

- **Node Details:**

  - **get_all_slide**  
    - Type: Code (JavaScript)  
    - Role: Extracts the `resources` array representing each image slide from the media data JSON.  
    - Configuration: Returns the array of slide objects for further processing.  
    - Input/Output: Input from Post Type Selector node (carousel output); output to Loop Over Items1 node.  
    - Edge cases: If no slides present, returns empty array.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Iterates over each slide item to process them individually.  
    - Configuration: Default batch size; processes each slide one at a time.  
    - Input/Output: Input from get_all_slide; outputs to OCR_Slide node (main) and Merge All Parsed Text node (secondary).  
    - Edge cases: Large carousel posts may cause longer execution times or rate limits.

  - **OCR_Slide**  
    - Type: HTTP Request  
    - Role: Sends each slide image URL (`image_versions[0].url`) to OCR.Space for text extraction.  
    - Configuration: Same as OCR_Single but in loop context.  
    - Input/Output: Input from Loop Over Items1; output to getOnlyText node.  
    - Edge cases: Same as OCR_Single; failure on any slide should be handled gracefully.

  - **getOnlyText**  
    - Type: Code (JavaScript)  
    - Role: Extracts OCR text from each slide response similarly to getSingleText.  
    - Configuration: Returns JSON with `value` field for parsed text.  
    - Input/Output: Input from OCR_Slide; output to Loop Over Items1 to continue iteration.  
    - Edge cases: Empty or missing OCR results handled by returning empty string.

  - **Merge All Parsed Text**  
    - (Same node as in Single Post flow) Receives all extracted slide texts and aggregates them.

  - **Result of Raw Text**  
    - (Same node as in Single Post flow) Produces final combined text output.

---

#### 2.5 Handling Reels and Other Post Types

- **Overview:** For Instagram reels (`product_type` = `clips`), the workflow currently performs no OCR processing.
- **Nodes Involved:**  
  - No Operation, do nothing

- **Node Details:**

  - **No Operation, do nothing**  
    - Type: NoOp Node  
    - Role: Placeholder to handle reels gracefully by doing nothing.  
    - Input/Output: Input from Post Type Selector node (reels output); no output connections.  
    - Edge cases: No processing, so no errors but no output generated.

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                         | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                         |
|------------------------|--------------------|---------------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Workflow entry trigger                 |                           | IGPost URL               |                                                                                                                                    |
| IGPost URL             | Set                | Sets Instagram post URL                | When clicking ‘Execute workflow’ | Retrieve Media           |                                                                                                                                    |
| Retrieve Media         | HTTP Request       | Calls HikerAPI to get media info       | IGPost URL                | Post Type Selector       | See "1. Get Media of IG Post" note                                                                                                 |
| Post Type Selector     | Switch             | Routes flow based on Instagram post type | Retrieve Media            | OCR_Single, get_all_slide, No Operation, do nothing |                                                                                                                                    |
| OCR_Single             | HTTP Request       | Sends single image to OCR.Space        | Post Type Selector (single_post) | getSingleText            | See "2a. Extract the Text for Image on Single Post with OCR API" note                                                              |
| getSingleText          | Code               | Extracts text from OCR response (single) | OCR_Single                | Merge All Parsed Text    |                                                                                                                                    |
| get_all_slide          | Code               | Extracts carousel slides array          | Post Type Selector (carousel) | Loop Over Items1         | See "2b. Extract the Text within the loop of Carousel Post to scan with OCR API" note                                               |
| Loop Over Items1       | SplitInBatches     | Iterates over each carousel slide       | get_all_slide             | OCR_Slide, Merge All Parsed Text |                                                                                                                                    |
| OCR_Slide              | HTTP Request       | Sends each carousel slide image to OCR | Loop Over Items1          | getOnlyText              |                                                                                                                                    |
| getOnlyText            | Code               | Extracts text from OCR response (carousel) | OCR_Slide                 | Loop Over Items1         |                                                                                                                                    |
| Merge All Parsed Text  | Aggregate          | Combines all extracted texts            | getSingleText, Loop Over Items1 | Result of Raw Text       | See "3. The Result" note                                                                                                           |
| Result of Raw Text     | Code               | Concatenates all parsed text into one string | Merge All Parsed Text     |                          |                                                                                                                                    |
| No Operation, do nothing | NoOp              | Handles reels with no processing        | Post Type Selector (reels) |                          |                                                                                                                                    |
| Sticky Note            | Sticky Note        | Workflow explanation and instructions   |                           |                          | "Get Raw Text of Instagram Post with OCR" overview, setup instructions, API key notes                                              |
| Sticky Note1           | Sticky Note        | Workflow creator profile and contact    |                           |                          | "Workflow Creator Profile – Pake.AI" including website link https://pake.ai                                                        |
| Sticky Note2           | Sticky Note        | Annotation for media retrieval block    |                           |                          | "1. Get Media of IG Post"                                                                                                         |
| Sticky Note3           | Sticky Note        | Annotation for carousel OCR loop block  |                           |                          | "2b. Extract the Text within the loop of Carousel Post to scan with OCR API"                                                      |
| Sticky Note4           | Sticky Note        | Annotation for single image OCR block   |                           |                          | "2a. Extract the Text for Image on Single Post with OCR API"                                                                       |
| Sticky Note5           | Sticky Note        | Annotation for final results block       |                           |                          | "3. The Result\nYou can connect this nodes with your purpose"                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Node Type: Manual Trigger  
   - Position: Start node  
   - No parameters required.

2. **Create Set Node ("IGPost URL"):**  
   - Node Type: Set  
   - Add a string field `post_url` with value set to an example Instagram post URL (e.g., `https://www.instagram.com/p/DQf3KpkE4cW/?img_index=11`).  
   - Connect Manual Trigger output to this node.

3. **Create HTTP Request Node ("Retrieve Media"):**  
   - Node Type: HTTP Request  
   - HTTP Method: GET  
   - URL: `https://api.hikerapi.com/v1/media/by/url`  
   - Add Query Parameter: `url` = `={{ $json.post_url }}` (expression)  
   - Add Header: `accept: application/json`  
   - Authentication: HTTP Header Auth  
   - Credentials: Configure with valid HikerAPI key  
   - Connect "IGPost URL" output to this node.

4. **Create Switch Node ("Post Type Selector"):**  
   - Node Type: Switch  
   - Property to check: Expression `{{$json["product_type"]}}`  
   - Add three rules:  
     - If equals `feed` → output "single_post"  
     - If equals `carousel_container` → output "carousel"  
     - If equals `clips` → output "reels"  
   - Connect "Retrieve Media" output to this node.

5. **For Single Post Path:**

   - **Create HTTP Request Node ("OCR_Single"):**  
     - HTTP Method: GET  
     - URL: `https://api.ocr.space/parse/imageurl`  
     - Query Parameter: `url` = `={{ $json.image_versions[0].url }}` (expression from media data)  
     - Authentication: HTTP Query Auth  
     - Credentials: OCR.Space API key  
     - Connect "Post Type Selector" output "single_post" to this node.

   - **Create Code Node ("getSingleText"):**  
     - Language: JavaScript  
     - Code: Extract `ParsedResults[0].ParsedText` from OCR response and return as `{ value: text }`.  
     - Connect "OCR_Single" output to this node.

   - **Create Aggregate Node ("Merge All Parsed Text"):**  
     - Aggregate all item data without filters.  
     - Connect "getSingleText" output to this node.

6. **For Carousel Post Path:**

   - **Create Code Node ("get_all_slide"):**  
     - Language: JavaScript  
     - Code: Return `$input.first().json.resources` (array of slide images).  
     - Connect "Post Type Selector" output "carousel" to this node.

   - **Create SplitInBatches Node ("Loop Over Items1"):**  
     - Default batch size (1)  
     - Connect "get_all_slide" output to this node.

   - **Create HTTP Request Node ("OCR_Slide"):**  
     - HTTP Method: GET  
     - URL: `https://api.ocr.space/parse/imageurl`  
     - Query Parameter: `url` = `={{ $json.image_versions[0].url }}` (expression per slide)  
     - Authentication: HTTP Query Auth  
     - Credentials: OCR.Space API key  
     - Connect "Loop Over Items1" main output to this node.

   - **Create Code Node ("getOnlyText"):**  
     - Language: JavaScript  
     - Code: Extract `ParsedResults[0].ParsedText` from OCR response and return `{ value: text }`.  
     - Connect "OCR_Slide" output to this node.

   - **Connect "getOnlyText" output back to "Loop Over Items1" input** to process next slides.

   - **Connect "Loop Over Items1" secondary output to "Merge All Parsed Text"** to aggregate after looping.

7. **For Reels Path:**

   - **Create No Operation Node ("No Operation, do nothing"):**  
     - Connect "Post Type Selector" output "reels" to this node.

8. **Create Code Node ("Result of Raw Text"):**  
   - Language: JavaScript  
   - Code:  
     - Flatten all aggregated item data.  
     - Extract `value` fields.  
     - Filter out empty.  
     - Join with `\n` and return as `text` field.  
   - Connect "Merge All Parsed Text" output to this node.

9. **Add Sticky Notes:**  
   - Add notes explaining workflow purpose, setup instructions, and creator profile using Sticky Note nodes as per original content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                  | Context or Link                    |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------|
| This workflow requires API keys from HikerAPI (paid, affordable) and OCR.Space (free). Replace credentials in HTTP Request nodes accordingly.                                                                                | API key setup                    |
| HikerAPI endpoint used: https://api.hikerapi.com/v1/media/by/url. OCR.Space endpoint used: https://api.ocr.space/parse/imageurl.                                                                                              | API Reference                   |
| Workflow creator: Pake.AI, an AI Enabler from Indonesia, offering automation workflows freely to support the community. See https://pake.ai for more information and contact.                                                  | https://pake.ai                  |
| The workflow supports single image posts and carousel posts but skips reels with no OCR. You can extend the reels path as needed.                                                                                             | Functional scope notice          |
| For best results, ensure Instagram post URLs are public and accessible. Private or restricted posts may cause retrieval errors.                                                                                               | Usage advice                    |
| Workflow can be extended by connecting the final output node to other nodes for further processing (e.g., saving text to file, sending emails, etc.).                                                                          | Integration hint                |

---

**Disclaimer:**  
The text provided is exclusively from an automated n8n workflow. It complies with all current content policies and contains no illegal or offensive material. All data processed is legal and publicly accessible.