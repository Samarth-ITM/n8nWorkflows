Generate Images with Replicate and Flux

https://n8nworkflows.xyz/workflows/generate-images-with-replicate-and-flux-7192


# Generate Images with Replicate and Flux

### 1. Workflow Overview

This workflow automates the generation of images using the Replicate API models, uploads the generated images to both a WordPress site and Twitter (X), and aggregates the results. It is designed for users who want to programmatically create AI-generated images based on prompts, manage multiple model options with different configurations and pricing, and seamlessly upload and share images on web and social platforms.

**Logical blocks:**

- **1.1 Input Reception:** Manual or triggered input reception of prompt, slug, and model parameters.
- **1.2 Model Configuration:** Configuration of the Replicate model request based on the selected model.
- **1.3 Image Generation:** Calling Replicate API to create the image according to the model configuration.
- **1.4 Image Retrieval:** Fetching the generated image output from Replicate.
- **1.5 Upload to Destinations:** Uploading the generated image to WordPress and Twitter.
- **1.6 Result Aggregation and Formatting:** Aggregating upload responses and preparing final outputs for downstream use or display.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Accepts initial workflow trigger inputs either manually or from another workflow, including the image prompt, slug, and model.
- **Nodes Involved:** 
  - `When clicking ‘Execute workflow’`
  - `When Executed by Another Workflow`
  - `Code1`
  
- **Node Details:**

  - **When clicking ‘Execute workflow’**
    - Type: Manual Trigger
    - Role: Entry point for manual runs.
    - Config: No parameters.
    - Connections: Outputs to `Code1`.
    - Edge cases: User must input or have prefilled parameters via pinData.
  
  - **When Executed by Another Workflow**
    - Type: Execute Workflow Trigger
    - Role: Entry point when called from another workflow.
    - Config: Expects inputs `prompt`, `slug`, and `model`.
    - Connections: Outputs to `Code1`.
    - Edge cases: Missing inputs cause errors.
  
  - **Code1**
    - Type: Code (JavaScript)
    - Role: Validates the input model parameter and sets model-specific configuration for Replicate.
    - Config:
      - Defines configurations for three models: `black-forest-labs/flux-dev`, `flux-schnell`, and `flux-1.1-pro`.
      - Throws an error if the model is unsupported.
      - Outputs combined input plus model config.
    - Key Expressions:
      - Uses `$input.first().json` to access workflow inputs.
      - Returns `model_config` object tailored to the model.
    - Connections: Outputs to `HTTP Request1`.
    - Edge cases:
      - Unsupported model names cause explicit errors.
      - Input JSON must include `prompt` and `model`.

#### 2.2 Image Generation

- **Overview:** Sends the model-configured request to Replicate API to generate the image.
- **Nodes Involved:** 
  - `HTTP Request1`
  - `HTTP Request2`
  
- **Node Details:**

  - **HTTP Request1**
    - Type: HTTP Request
    - Role: Initiates image generation on Replicate API.
    - Config:
      - POST to `https://api.replicate.com/v1/models/{{ $json.model }}/predictions`.
      - Sends JSON body from `model_config`.
      - Authentication: Uses either `httpBearerAuth` or `httpHeaderAuth` with Replicate API credentials.
      - Headers include `content-type: application/json` and `Prefer: wait` (waits synchronously for completion).
    - Connections: Outputs to `HTTP Request2`.
    - Edge cases:
      - API quota exceeded or auth failures.
      - Network timeouts.
      - Invalid model or malformed JSON.
  
  - **HTTP Request2**
    - Type: HTTP Request
    - Role: Retrieves the generated image URL or output from the Replicate prediction response.
    - Config:
      - GET request to URL indicated by `{{$json.output || $json.output[0]}}`.
      - Authentication: Same as above.
      - Accept header: `application/json`.
    - Connections: Outputs to both upload nodes (`Upload image2`, `Upload Media (X)`).
    - Edge cases:
      - Output URL missing or invalid.
      - API rate limiting or failures.

#### 2.3 Upload to Destinations

- **Overview:** Uploads the generated image to WordPress media library and Twitter as a tweet image.
- **Nodes Involved:** 
  - `Upload image2`
  - `Upload Media (X)`
  - `Merge`
  
- **Node Details:**

  - **Upload image2**
    - Type: HTTP Request
    - Role: Upload image to WordPress via REST API.
    - Config:
      - POST to `https://articles.emp0.com/wp-json/wp/v2/media`.
      - Sends binary image data.
      - Uses WordPress API OAuth2 credential.
      - Sets header `Content-Disposition` with filename from slug parameter: `"attachment; filename=\"img-{{ $('Code1').item.json.slug }}.jpg\""`
      - Content-Type: binaryData.
      - Retries on failure with 5-second wait.
    - Connections: Outputs to `Merge`.
    - Edge cases:
      - Auth failures with WordPress.
      - File upload errors.
      - Filename conflicts or invalid slug.
  
  - **Upload Media (X)**
    - Type: HTTP Request
    - Role: Upload image to Twitter (X) media endpoint.
    - Config:
      - POST to `https://upload.twitter.com/1.1/media/upload.json?media_category=TWEET_IMAGE`
      - Multipart form-data with media binary.
      - Uses Twitter OAuth1 credential.
      - Expects JSON response.
    - Connections: Outputs to `Merge`.
    - Edge cases:
      - Twitter OAuth failures.
      - Media size limits or format issues.
      - Network issues.
  
  - **Merge**
    - Type: Merge
    - Role: Combines results from WordPress and Twitter uploads.
    - Config: Default merge (combine inputs).
    - Connections: Outputs to `Aggregate`.
    - Edge cases:
      - One upload failing could produce incomplete merged output.

#### 2.4 Result Aggregation and Formatting

- **Overview:** Aggregates upload results and formats final output including URLs for public consumption.
- **Nodes Involved:** 
  - `Aggregate`
  - `Code`
  
- **Node Details:**

  - **Aggregate**
    - Type: Aggregate
    - Role: Combines all item data into a single aggregated JSON object.
    - Config: Aggregates all item data from inputs.
    - Connections: Outputs to `Code`.
  
  - **Code**
    - Type: Code (JavaScript)
    - Role: Extracts URLs and relevant data from upload responses.
    - Config:
      - Returns an object containing:
        - `public_image_url`: WordPress image URL (`data[0].guid.raw`)
        - `wordpress`: Full WordPress response object (`data[0]`)
        - `twitter`: Twitter upload response (`data[1]`)
    - Connections: No further outputs.
    - Edge cases:
      - Assumes array order for WordPress and Twitter responses.
      - Missing data fields cause errors.

---

### 3. Summary Table

| Node Name                 | Node Type                  | Functional Role                      | Input Node(s)               | Output Node(s)                  | Sticky Note                                                                                   |
|---------------------------|----------------------------|------------------------------------|-----------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual workflow start               |                             | Code1                          |                                                                                              |
| When Executed by Another Workflow | Execute Workflow Trigger  | Triggered start with inputs         |                             | Code1                          |                                                                                              |
| Code1                     | Code                       | Validate input, set model config    | When clicking ‘Execute workflow’, When Executed by Another Workflow | HTTP Request1                 |                                                                                              |
| HTTP Request1             | HTTP Request               | Send generation request to Replicate| Code1                       | HTTP Request2                  |                                                                                              |
| HTTP Request2             | HTTP Request               | Retrieve generated image URL/output | HTTP Request1               | Upload image2, Upload Media (X)|                                                                                              |
| Upload image2             | HTTP Request               | Upload image to WordPress           | HTTP Request2               | Merge                         |                                                                                              |
| Upload Media (X)          | HTTP Request               | Upload image to Twitter (X)         | HTTP Request2               | Merge                         |                                                                                              |
| Merge                    | Merge                      | Combine upload responses            | Upload image2, Upload Media (X)| Aggregate                    |                                                                                              |
| Aggregate                | Aggregate                  | Aggregate combined upload data      | Merge                       | Code                         |                                                                                              |
| Code                     | Code                       | Format and output URLs and data     | Aggregate                   |                                |                                                                                              |
| Sticky Note              | Sticky Note                | Label “Generate image with replicate”|                             |                                |                                                                                              |
| Sticky Note1             | Sticky Note                | Label “Upload”                      |                             |                                |                                                                                              |
| Sticky Note2             | Sticky Note                | Display generated image example     |                             |                                | ![batman-typing-on-a-laptop](https://articles.emp0.com/wp-content/uploads/2025/08/img-joker-watching-batman.webp) |
| Sticky Note3             | Sticky Note                | Model pricing information           |                             |                                | Models & Pricing / img<br> black-forest-labs/flux-schnell -> $0.003<br> black-forest-labs/flux-dev -> $0.025<br> black-forest-labs/flux-1.1-pro -> $0.04 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Nodes:**

   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`.
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.
     - Configure to accept inputs: `prompt` (string), `slug` (string), `model` (string).

2. **Add a Code Node (`Code1`) to Validate Inputs and Configure Model:**

   - Use JavaScript to:
     - Read inputs `prompt`, `slug`, and `model`.
     - Define model configurations for:
       - `black-forest-labs/flux-dev`
       - `black-forest-labs/flux-schnell`
       - `black-forest-labs/flux-1.1-pro`
     - Throw error if unsupported model.
     - Output combined input plus `model_config`.
   - Connect both trigger nodes to `Code1`.

3. **Add HTTP Request Node (`HTTP Request1`) to Call Replicate API:**

   - Set method to POST.
   - URL: `https://api.replicate.com/v1/models/{{ $json.model }}/predictions`.
   - Body type: JSON.
   - Body: Use `{{ $json.model_config }}` from `Code1`.
   - Headers:
     - `content-type: application/json`
     - `Prefer: wait`
   - Authentication:
     - Add credentials for Replicate API using either HTTP Bearer or Header Auth.
   - Connect `Code1` output to this node.

4. **Add HTTP Request Node (`HTTP Request2`) to Retrieve Image Output:**

   - Set method to GET.
   - URL: `{{$json.output || $json.output[0]}}` (from previous response).
   - Headers:
     - `accept: application/json`
   - Use same authentication as `HTTP Request1`.
   - Connect `HTTP Request1` output to this node.

5. **Add Upload Nodes to Send Image to Destinations:**

   - **Upload to WordPress (`Upload image2`):**
     - Method: POST
     - URL: `https://articles.emp0.com/wp-json/wp/v2/media`
     - Content type: binary data
     - Send headers: true
     - Header `Content-Disposition` set to `attachment; filename="img-{{ $('Code1').item.json.slug }}.jpg"`
     - Use WordPress OAuth2 credential.
     - Send binary data from `HTTP Request2`.
     - Enable retry on failure with 5-second delay.
   
   - **Upload to Twitter (X) (`Upload Media (X)`):**
     - Method: POST
     - URL: `https://upload.twitter.com/1.1/media/upload.json?media_category=TWEET_IMAGE`
     - Content type: multipart-form-data
     - Form parameter `media` set as binary data from `HTTP Request2`.
     - Use Twitter OAuth1 credential.
     - Expect JSON response.

   - Connect `HTTP Request2` output to both upload nodes in parallel.

6. **Add a Merge Node (`Merge`):**

   - Merge inputs from `Upload image2` and `Upload Media (X)`.
   - Use default merge mode (combine).
   - Connect outputs of both upload nodes to `Merge`.

7. **Add an Aggregate Node (`Aggregate`):**

   - Aggregate all item data from `Merge`.
   - Connect `Merge` output to `Aggregate`.

8. **Add Code Node (`Code`) to Format Final Output:**

   - Extract and return:
     - `public_image_url` from WordPress response `data[0].guid.raw`
     - `wordpress` full response from WordPress
     - `twitter` response from Twitter upload
   - Connect `Aggregate` output to this node.

9. **Add Sticky Notes (Optional):**

   - Label workflow sections:
     - “Generate image with replicate”
     - “Upload”
     - Example image display with URL
     - Model pricing info

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow uses Replicate API models from black-forest-labs with different pricing tiers and model capabilities.       | Pricing details shown in Sticky Note3: <br>flux-schnell $0.003, flux-dev $0.025, flux-1.1-pro $0.04       |
| WordPress media upload requires OAuth2 credentials configured for the target WordPress site.                         | WordPress REST API media endpoint: https://articles.emp0.com/wp-json/wp/v2/media                         |
| Twitter (X) media upload uses OAuth1 credentials for authorized API access.                                          | Twitter media upload docs: https://developer.twitter.com/en/docs/twitter-api/v1/media/upload-media        |
| The workflow waits synchronously for the Replicate prediction to complete using the `Prefer: wait` header.           | This ensures the image URL is available immediately after the first request completes.                     |
| Filename for WordPress upload uses slug input to create unique image filenames to avoid overwriting.                  | Header set as `Content-Disposition: attachment; filename="img-{{slug}}.jpg"`                             |
| Example image output shown in Sticky Note2 uses a publicly accessible image for demonstration.                        | ![batman-typing-on-a-laptop](https://articles.emp0.com/wp-content/uploads/2025/08/img-joker-watching-batman.webp) |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is lawful and public.