Find the Best Favicon from Multiple Sources with GPT-4 Vision Analysis

https://n8nworkflows.xyz/workflows/find-the-best-favicon-from-multiple-sources-with-gpt-4-vision-analysis-5599


# Find the Best Favicon from Multiple Sources with GPT-4 Vision Analysis

---

### 1. Workflow Overview

This workflow is designed to identify the best favicon image for a given website by fetching potential icons from multiple sources, analyzing their quality using GPT-4 Vision capabilities, and then selecting and returning the highest quality favicon URL. Its main use case is in directories or platforms listing AI tools or websites, where presenting a clean, high-quality brand icon is essential.

The workflow is logically divided into three main blocks:

- **1.1 Fetch Favicon Images**: Retrieve favicon images from three different providers using the input URL and domain.
- **1.2 Analyze Each Image**: Use OpenAI's GPT-4 Vision API to assign a quality score to each fetched favicon image.
- **1.3 Build Result**: Aggregate the analysis results, identify the best scoring icon, and output the final favicon URL.

---

### 2. Block-by-Block Analysis

#### 2.1 Block 1: Fetch Favicon Images

- **Overview:**  
Fetches favicon images from three distinct external sources based on the provided website URL and domain. It prepares the input data and sends HTTP requests to retrieve favicon images in PNG or JPEG formats, handling errors gracefully.

- **Nodes Involved:**  
  - `workflow_trigger`  
  - `set_common_fields`  
  - `google`  
  - `logo_dev`  
  - `clearbit`  
  - `merge`  
  - `filter_errors`  

- **Node Details:**

  - **workflow_trigger**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point that accepts inputs `url` and `domain` for which to find favicons.  
    - *Configuration:* Defines `url` and `domain` as inputs expected from caller workflows or manual execution.  
    - *Input/Output:* No inputs; outputs the received parameters for downstream nodes.  
    - *Failure Cases:* Missing or malformed inputs could lead to downstream failures.  
    - *Sticky Note:* Explains this block fetches favicon images from multiple providers.

  - **set_common_fields**  
    - *Type:* Set Node  
    - *Role:* Normalizes and propagates `url` and `domain` fields for consistent use in downstream HTTP requests.  
    - *Configuration:* Assigns `url` and `domain` from input JSON to node output.  
    - *Input:* Receives from `workflow_trigger`  
    - *Output:* Feeds to favicon fetch nodes.  
    - *Edge Cases:* None critical; ensures consistent data structure.  

  - **google**  
    - *Type:* HTTP Request  
    - *Role:* Fetches favicon from Google's favicon service using the input URL.  
    - *Configuration:*  
      - URL constructed dynamically: `https://t3.gstatic.com/faviconV2?...&url={{url}}&size=256`  
      - Requests full response as a file, allows unauthorized certificates.  
      - `onError` set to continue output (does not fail workflow if request fails).  
    - *Input:* From `set_common_fields`  
    - *Output:* To `merge`  
    - *Potential Failures:* Network issues, invalid URL, or no favicon found.  
    - *Edge Cases:* May return default or placeholder icons if the site has none.

  - **logo_dev**  
    - *Type:* HTTP Request  
    - *Role:* Fetches favicon from Logo.dev API using the domain.  
    - *Configuration:*  
      - URL: `https://img.logo.dev/{{domain}}?format=png&size=512`  
      - Authentication: Uses HTTP Query Auth credential named "Logo.dev API".  
      - Full response as file, continues on error.  
    - *Input:* From `set_common_fields`  
    - *Output:* To `merge`  
    - *Failures:* API limit exceeded, invalid domain, auth issues.  

  - **clearbit**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves favicon from Clearbit’s logo API using the domain.  
    - *Configuration:*  
      - URL: `https://logo.clearbit.com/{{domain}}?size=256&format=png`  
      - Full response as file, continues on error.  
    - *Input:* From `set_common_fields`  
    - *Output:* To `merge`  
    - *Failures:* No logo found, network errors.  

  - **merge**  
    - *Type:* Merge  
    - *Role:* Combines outputs of the three favicon fetch requests into one stream with three inputs.  
    - *Configuration:* Number of inputs set to 3.  
    - *Input:* From `google`, `logo_dev`, and `clearbit` nodes  
    - *Output:* To `filter_errors`  
    - *Edge Cases:* Handles cases where one or two requests fail but others succeed.  

  - **filter_errors**  
    - *Type:* Filter  
    - *Role:* Filters out any items (icon responses) that contain an error property.  
    - *Configuration:* Checks if `error` field does not exist in the JSON.  
    - *Input:* From `merge`  
    - *Output:* To `filter_mime_type`  
    - *Edge Cases:* Ensures only successful HTTP responses continue.

---

#### 2.2 Block 2: Analyze Each Image

- **Overview:**  
Filters the images by MIME type, sends each valid image to OpenAI’s GPT-4 Vision model for quality analysis, and aggregates results for final extraction.

- **Nodes Involved:**  
  - `filter_mime_type`  
  - `analyze_each_icon`  
  - `aggregate_results`  

- **Node Details:**

  - **filter_mime_type**  
    - *Type:* Filter  
    - *Role:* Allows only images with MIME types `image/png` or `image/jpeg` to proceed.  
    - *Configuration:* OR condition on content-type header equals "image/png" or "image/jpeg" (case-insensitive).  
    - *Input:* From `filter_errors`  
    - *Output:* To `analyze_each_icon`  
    - *Edge Cases:* Could exclude valid images if content-type header is missing or malformed.

  - **analyze_each_icon**  
    - *Type:* OpenAI (Langchain OpenAI Node) with Image Analysis  
    - *Role:* Uses GPT-4 Vision to score each image on a 0.0 to 1.0 scale for favicon suitability.  
    - *Configuration:*  
      - Model: chatgpt-4o-latest (GPT-4 Vision)  
      - Input: Base64 encoded image file from previous step  
      - Prompt instructs scoring based on clarity, resolution, background, artifact presence, and brand relevance without design bias.  
    - *Input:* From `filter_mime_type`  
    - *Output:* To `aggregate_results`  
    - *Failures:* API auth errors, rate limits, image decoding errors.  
    - *Sticky Note:* Describes the image quality scoring process.

  - **aggregate_results**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all analysis outputs’ `content` fields into a single array for extraction.  
    - *Configuration:* Aggregates field "content" renamed as "results".  
    - *Input:* From `analyze_each_icon`  
    - *Output:* To `extract_best_icon`  
    - *Edge Cases:* Handles empty or partial results gracefully.

---

#### 2.3 Block 3: Build Result

- **Overview:**  
Extracts the best favicon image index from aggregated scores, then constructs and returns the final best favicon URL using stored inputs.

- **Nodes Involved:**  
  - `extract_best_icon`  
  - `return_final_image_url`  

- **Node Details:**

  - **extract_best_icon**  
    - *Type:* Langchain Information Extractor  
    - *Role:* Parses the aggregated text results to find the index of the highest quality favicon image.  
    - *Configuration:*  
      - System prompt instructs extraction of only relevant info, specifically the best image index (number).  
      - Input text constructed by mapping through results with image numbers and scores.  
    - *Input:* From `aggregate_results`  
    - *Output:* To `return_final_image_url`  
    - *Failures:* Parsing errors, ambiguous results if scores are equal or missing.  
    - *Sticky Note:* Describes this step as picking the best scoring image.

  - **return_final_image_url**  
    - *Type:* Code (JavaScript)  
    - *Role:* Based on the extracted best image index, returns the corresponding favicon URL from the known sources.  
    - *Configuration:*  
      - Reads best image index from input JSON.  
      - Uses stored `tool_url` and `domain` for URL construction.  
      - Switch statement selects correct URL template based on index:  
        - 0 = Google  
        - 1 = Logo.dev  
        - 2 = Clearbit  
      - Throws error on invalid index.  
    - *Input:* From `extract_best_icon`  
    - *Output:* Final output with `icon_url` field.  
    - *Failures:* Unknown image index from extraction, missing inputs.  
    - *Sticky Note:* Describes final selection and URL output.

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                         | Input Node(s)                | Output Node(s)          | Sticky Note                                                                                             |
|---------------------|--------------------------------|---------------------------------------|-----------------------------|-------------------------|-------------------------------------------------------------------------------------------------------|
| workflow_trigger     | Execute Workflow Trigger        | Entry point, receives `url` & `domain`| None                        | set_common_fields        | ## 1. Fetch Favicon Images Attempts to fetch favicon images from multiple providers by using the website's `url` and `domain` inputs. |
| set_common_fields    | Set                            | Normalizes and forwards input fields  | workflow_trigger             | google, logo_dev, clearbit |                                                                                                       |
| google              | HTTP Request                   | Fetch favicon from Google service      | set_common_fields            | merge                   |                                                                                                       |
| logo_dev            | HTTP Request                   | Fetch favicon from Logo.dev API        | set_common_fields            | merge                   |                                                                                                       |
| clearbit            | HTTP Request                   | Fetch favicon from Clearbit API        | set_common_fields            | merge                   |                                                                                                       |
| merge               | Merge                          | Combines all favicon fetch results     | google, logo_dev, clearbit   | filter_errors            |                                                                                                       |
| filter_errors        | Filter                         | Filters out errored HTTP responses     | merge                       | filter_mime_type         |                                                                                                       |
| filter_mime_type     | Filter                         | Filters images by MIME type (PNG/JPEG) | filter_errors               | analyze_each_icon        |                                                                                                       |
| analyze_each_icon    | OpenAI (Langchain OpenAI Image)| Scores each image's favicon quality    | filter_mime_type            | aggregate_results        | ## 2. Analyze Each Image Use OpenAI's vision API to analyze each image and assign a "Quality Score" for each favicon previously fetched. |
| aggregate_results    | Aggregate                      | Aggregates analysis results             | analyze_each_icon           | extract_best_icon        |                                                                                                       |
| extract_best_icon    | Langchain Information Extractor| Extracts best image index from scores  | aggregate_results           | return_final_image_url   | ## 3. Build Result The final step here picks out the best score from the previous analysis and returns the url for the highest quality favicon image. |
| return_final_image_url| Code                           | Builds and returns final favicon URL   | extract_best_icon           | None                    |                                                                                                       |
| Sticky Note         | Sticky Note                    | Informational notes                     | None                        | None                    | See block-specific sticky notes above.                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create an Execute Workflow Trigger node** named `workflow_trigger`:  
   - Add two string inputs: `url` and `domain`.  
   - Position at the start of the workflow.

2. **Add a Set node** named `set_common_fields`:  
   - Connect from `workflow_trigger`.  
   - Assign two fields: `url` (copy from input JSON `url`) and `domain` (copy from input JSON `domain`).  

3. **Add three HTTP Request nodes** to fetch favicons:  
   - `google`:  
     - Connect from `set_common_fields`.  
     - URL: `https://t3.gstatic.com/faviconV2?client=SOCIAL&type=FAVICON&fallback_opts=TYPE,SIZE,URL&url={{ $json.url }}&size=256`  
     - Response format: File (full response).  
     - On error: Continue workflow.  
     - Allow unauthorized certs: True.  
   - `logo_dev`:  
     - Connect from `set_common_fields`.  
     - URL: `https://img.logo.dev/{{ $json.domain }}?format=png&size=512`  
     - Authentication: Setup HTTP Query Auth credential with Logo.dev API key.  
     - Response format: File (full response).  
     - On error: Continue workflow.  
     - Allow unauthorized certs: True.  
   - `clearbit`:  
     - Connect from `set_common_fields`.  
     - URL: `https://logo.clearbit.com/{{ $json.domain }}?size=256&format=png`  
     - Response format: File (full response).  
     - On error: Continue workflow.  
     - Allow unauthorized certs: True.  

4. **Add a Merge node** named `merge`:  
   - Set number of inputs to 3.  
   - Connect each HTTP request node's main output into one input of `merge`.

5. **Add a Filter node** named `filter_errors`:  
   - Connect from `merge`.  
   - Condition: Only allow items where the `error` property does NOT exist in the JSON.  

6. **Add another Filter node** named `filter_mime_type`:  
   - Connect from `filter_errors`.  
   - Condition: Allow if HTTP header `content-type` equals "image/png" OR "image/jpeg" (case-insensitive).  

7. **Add an OpenAI (Langchain OpenAI) node** named `analyze_each_icon`:  
   - Connect from `filter_mime_type`.  
   - Set operation to `analyze` with resource type `image`.  
   - Model: Use GPT-4 Vision model `chatgpt-4o-latest`.  
   - Input type: Base64 encoded image file from HTTP response.  
   - Prompt: Provide instructions to score image quality from 0.0 to 1.0 based on clarity, resolution, background, artifact presence, and brand relevance.  
   - Credentials: Setup OpenAI API credentials.  

8. **Add an Aggregate node** named `aggregate_results`:  
   - Connect from `analyze_each_icon`.  
   - Aggregate the `content` field from each analysis into a new field named `results`.  

9. **Add a Langchain Information Extractor node** named `extract_best_icon`:  
   - Connect from `aggregate_results`.  
   - Input text: Construct a string enumerating each image’s result with index.  
   - System prompt: Extract only the best image index (number) with the highest score (0.0 to 1.0).  
   - Define attribute to extract: `best_image_index` (number, required).  

10. **Add a Code node** named `return_final_image_url`:  
    - Connect from `extract_best_icon`.  
    - JavaScript code:  
      - Read `best_image_index` from input data.  
      - Read `tool_url` and `domain` from `set_common_fields` node’s output.  
      - Use a switch-case to construct the favicon URL for the best image index:  
        - 0 → Google favicon URL using `tool_url`  
        - 1 → Logo.dev favicon URL using `domain`  
        - 2 → Clearbit favicon URL using `domain`  
      - Return the constructed URL as `icon_url`.  
    - Configure to run once per item.  

11. **Add Sticky Notes** at appropriate positions for documentation:  
    - Near the favicon fetch nodes: Describe fetching favicons from multiple providers.  
    - Near the analysis nodes: Describe the image quality scoring process.  
    - Near the final extraction and code nodes: Describe picking and returning the best favicon URL.  

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                          |
|-----------------------------------------------------------------------------------------------------|----------------------------------------|
| Uses GPT-4 Vision (chatgpt-4o-latest) for image quality scoring based on favicon suitability criteria. | OpenAI GPT-4 Vision API                 |
| Logo.dev API requires HTTP Query Auth credentials with a valid API token.                            | https://logo.dev                        |
| Clearbit logo API is a free service but may have rate limits and no guaranteed availability.         | https://clearbit.com/logo-api          |
| Google favicon service provides favicons for any URL but may return default or placeholder icons.    | https://developers.google.com/speed/docs/insights/v5/about#favicons |
| Workflow designed to gracefully continue on HTTP request errors to allow partial favicon availability.| Robust error handling                   |
| The final favicon URL is constructed only from known sources indexed by the analysis result.         | Code node switch-case logic             |

---

**Disclaimer:**  
The provided description and analysis derive exclusively from an n8n workflow automation. The workflow respects all current content policies and does not contain illegal or protected elements. All data processed is legal and public.

---