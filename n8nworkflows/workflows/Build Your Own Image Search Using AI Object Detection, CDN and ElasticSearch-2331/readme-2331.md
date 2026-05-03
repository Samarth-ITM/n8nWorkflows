Build Your Own Image Search Using AI Object Detection, CDN and ElasticSearch

https://n8nworkflows.xyz/workflows/build-your-own-image-search-using-ai-object-detection--cdn-and-elasticsearch-2331


# Build Your Own Image Search Using AI Object Detection, CDN and ElasticSearch

### 1. Workflow Overview

This n8n workflow automates the process of building an object-based image search index by leveraging AI object detection, image cropping, cloud storage, and Elasticsearch indexing. It targets scenarios where users want to enable granular image search based on the objects contained within images.

**Logical Blocks:**

- **1.1 Input Reception and Variable Setup**  
  The workflow starts by manually triggering the process and setting key variables including source image URL, Cloudflare account details, AI model identifier, and Elasticsearch index name.

- **1.2 Image Retrieval and Object Detection via AI**  
  The source image is fetched, then sent to Cloudflare’s Workers AI API running the Detr-Resnet-50 object detection model. The model returns detected objects with bounding boxes, labels, and confidence scores.

- **1.3 Filtering and Processing Detection Results**  
  Detection results are split into individual object records, filtered to retain only those with confidence scores ≥ 0.9 for quality, then the original image is fetched again for cropping.

- **1.4 Cropping, Uploading, and Indexing**  
  Each filtered object is cropped from the original image based on its bounding box, uploaded to Cloudinary, then indexed in Elasticsearch with metadata, including object label, cropped image URL, and original image URL.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception and Variable Setup

- **Overview:**  
  This block initializes the workflow with required variables and prepares the environment for the image processing pipeline.

- **Nodes Involved:**  
  - When clicking "Test workflow"  
  - Set Variables  
  - Sticky Notes (contextual information)

- **Node Details:**

  1. **When clicking "Test workflow"**  
     - Type: Manual Trigger  
     - Role: Entry point to manually start the workflow.  
     - Config: No parameters; manual trigger node.  
     - Inputs: None  
     - Outputs: Triggers "Set Variables" node.  
     - Failure modes: None significant, manual trigger.

  2. **Set Variables**  
     - Type: Set  
     - Role: Defines key variables for the workflow run.  
     - Config:  
       - CLOUDFLARE_ACCOUNT_ID: Cloudflare account identifier (empty string placeholder).  
       - model: AI model identifier "@cf/facebook/detr-resnet-50".  
       - source_image: URL of the image to process (example given).  
       - elasticsearch_index: Name of the Elasticsearch index to store results.  
     - Inputs: Trigger from manual node.  
     - Outputs: Feeds into "Fetch Source Image" node.  
     - Failure modes: Missing or incorrect variable values (especially Cloudflare Account ID) will cause downstream failures.

  3. **Sticky Notes**  
     - Provide instructions on setting variables and overview of input image setup.  
     - Notes include links to n8n documentation on setting variables.

---

#### Block 1.2: Image Retrieval and Object Detection via AI

- **Overview:**  
  Retrieves the source image and sends it to a Cloudflare AI model that detects objects and returns their labels, bounding boxes, and confidence scores.

- **Nodes Involved:**  
  - Fetch Source Image  
  - Use Detr-Resnet-50 Object Classification  
  - Sticky Notes (explanations)

- **Node Details:**

  1. **Fetch Source Image**  
     - Type: HTTP Request  
     - Role: Downloads the source image binary data using the URL from variables.  
     - Config: URL set dynamically via expression `={{ $json.source_image }}`.  
     - Outputs: Binary data of the image to the AI classification node.  
     - Failure modes: HTTP errors, invalid URL, timeouts.

  2. **Use Detr-Resnet-50 Object Classification**  
     - Type: HTTP Request  
     - Role: Sends the binary image to Cloudflare’s Workers AI API for object detection.  
     - Config:  
       - URL dynamically built using Cloudflare Account ID and model variable.  
       - Method: POST  
       - Content type: Binary data  
       - Authentication: Cloudflare API credentials configured.  
       - Input data field: "data" containing binary image.  
     - Outputs: JSON object containing detected objects with labels, bounding boxes, scores.  
     - Failure modes: Authentication errors, API timeouts, invalid responses, malformed input data.

  3. **Sticky Notes**  
     - Provide background on using Cloudflare Workers AI for object classification.  
     - Link to Cloudflare Workers AI documentation.

---

#### Block 1.3: Filtering and Processing Detection Results

- **Overview:**  
  Splits the AI detection response into individual object records, filters out low-confidence detections, and fetches the source image again for cropping operations.

- **Nodes Involved:**  
  - Split Out Results Only  
  - Filter Score >= 0.9  
  - Fetch Source Image Again  
  - Sticky Notes (contextual info)

- **Node Details:**

  1. **Split Out Results Only**  
     - Type: Split Out  
     - Role: Extracts each detected object from the "result" array into separate workflow items.  
     - Config: Field to split out: "result".  
     - Inputs: AI classification response JSON.  
     - Outputs: Multiple workflow items, each representing one detected object.  
     - Failure modes: Missing or malformed "result" field in input JSON.

  2. **Filter Score >= 0.9**  
     - Type: Filter  
     - Role: Passes only objects with confidence score ≥ 0.9.  
     - Config: Condition: `{{$json.score}} >= 0.9`  
     - Inputs: Individual object items.  
     - Outputs: Filtered detected objects with sufficiently high confidence.  
     - Failure modes: Missing or invalid "score" field.

  3. **Fetch Source Image Again**  
     - Type: HTTP Request  
     - Role: Re-downloads the original image for cropping purposes.  
     - Config: URL from variable `={{ $('Set Variables').item.json.source_image }}`.  
     - Outputs: Binary image data for the next cropping node.  
     - Failure modes: HTTP errors, timeouts.

  4. **Sticky Notes**  
     - Explain filtering rationale and note for cropping preparation.

---

#### Block 1.4: Cropping, Uploading, and Indexing

- **Overview:**  
  Crops each filtered detected object out of the original image, uploads the cropped object image to Cloudinary, then indexes the object data and image URLs in Elasticsearch.

- **Nodes Involved:**  
  - Crop Object From Image  
  - Upload to Cloudinary  
  - Create Docs In Elasticsearch  
  - Sticky Notes (explanations and resource links)

- **Node Details:**

  1. **Crop Object From Image**  
     - Type: Edit Image  
     - Role: Crops the detected object from the source image using bounding box coordinates.  
     - Config:  
       - Operation: Crop  
       - Width: Calculated as `xmax - xmin` from bounding box.  
       - Height: Calculated as `ymax - ymin` from bounding box.  
       - PositionX: xmin  
       - PositionY: ymin  
       - Format: JPEG  
       - Filename: constructed from original file name, object label, and item index (all URL encoded).  
     - Inputs: Binary image from "Fetch Source Image Again" and JSON with bounding box.  
     - Outputs: Cropped image binary data for uploading.  
     - Failure modes: Incorrect bounding box coordinates causing invalid crop, missing binary data.

  2. **Upload to Cloudinary**  
     - Type: HTTP Request  
     - Role: Uploads cropped object image to Cloudinary for external hosting.  
     - Config:  
       - URL: Cloudinary upload endpoint.  
       - Method: POST  
       - Content Type: multipart/form-data  
       - Authentication: HTTP Query Auth with Cloudinary preset.  
       - Body: Binary data from cropped image.  
     - Outputs: JSON response with image hosting URLs.  
     - Failure modes: Authentication errors, payload size limits, upload failures.

  3. **Create Docs In Elasticsearch**  
     - Type: Elasticsearch  
     - Role: Indexes a document for each cropped object image with metadata.  
     - Config:  
       - Index ID: from variable `elasticsearch_index`  
       - Fields:  
         - image_url: Cloudinary secure URL with auto format and quality parameters.  
         - source_image_url: Original source image URL.  
         - label: Object label from cropped image JSON.  
         - metadata: JSON string combining crop metadata and original filename.  
       - Operation: Create (index new document).  
       - Credentials: Elasticsearch API credentials.  
     - Inputs: JSON from upload node and cropped image metadata.  
     - Outputs: Confirmation of indexing.  
     - Failure modes: Authentication, network errors, invalid field data, indexing conflicts.

  4. **Sticky Notes**  
     - Provide explanations on cropping, uploading, and indexing steps.  
     - Include references to n8n docs for Edit Image, Elasticsearch usage, and Cloudflare AI.

---

### 3. Summary Table

| Node Name                        | Node Type         | Functional Role                             | Input Node(s)                   | Output Node(s)                       | Sticky Note                                                                                                           |
|---------------------------------|-------------------|--------------------------------------------|--------------------------------|------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow"    | Manual Trigger    | Start workflow manually                     | None                           | Set Variables                      |                                                                                                                       |
| Set Variables                   | Set               | Define workflow variables                   | When clicking "Test workflow"  | Fetch Source Image                 | 🚨**Required** * Set your variables here first!                                                                       |
| Fetch Source Image              | HTTP Request      | Download source image                        | Set Variables                  | Use Detr-Resnet-50 Object Classification | ## 1. Get Source Image [Read more about setting variables for your workflow](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set) |
| Use Detr-Resnet-50 Object Classification | HTTP Request | Send image to Cloudflare AI for object detection | Fetch Source Image             | Split Out Results Only             | ## 2. Use Detr-Resnet-50 Object Classification [Learn more about Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/)    |
| Split Out Results Only          | Split Out         | Split AI results into individual objects   | Use Detr-Resnet-50 Object Classification | Filter Score >= 0.9              |                                                                                                                       |
| Filter Score >= 0.9             | Filter            | Filter objects by confidence score ≥ 0.9   | Split Out Results Only          | Fetch Source Image Again           |                                                                                                                       |
| Fetch Source Image Again        | HTTP Request      | Re-download source image for cropping       | Filter Score >= 0.9             | Crop Object From Image             |                                                                                                                       |
| Crop Object From Image          | Edit Image        | Crop detected objects out of source image   | Fetch Source Image Again        | Upload to Cloudinary              | ## 3. Crop Objects Out of Source Image [Read more about Editing Images in n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage) |
| Upload to Cloudinary            | HTTP Request      | Upload cropped images to Cloudinary          | Crop Object From Image          | Create Docs In Elasticsearch       |                                                                                                                       |
| Create Docs In Elasticsearch    | Elasticsearch     | Index cropped object images and metadata    | Upload to Cloudinary            | None                             | ## 4. Index Object Images In ElasticSearch [Read more about using ElasticSearch](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.elasticsearch) |
| Sticky Note                    | Sticky Note       | Informational notes                          | None                          | None                             | ## 1. Get Source Image [Read more about setting variables for your workflow](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.set) |
| Sticky Note1                   | Sticky Note       | Informational notes                          | None                          | None                             | ## 2. Use Detr-Resnet-50 Object Classification [Learn more about Cloudflare Workers AI](https://developers.cloudflare.com/workers-ai/) |
| Sticky Note2                   | Sticky Note       | Informational notes                          | None                          | None                             | ## 3. Crop Objects Out of Source Image [Read more about Editing Images in n8n](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage) |
| Sticky Note3                   | Sticky Note       | Informational notes                          | None                          | None                             | ## 4. Index Object Images In ElasticSearch [Read more about using ElasticSearch](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.elasticsearch) |
| Sticky Note4                   | Sticky Note       | Visual example of classification result     | None                          | None                             | Fig 1. Result of Classification ![image of classification](https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto,w_300/v1/n8n-workflows/ywtzjcmqrypihci1npgh) |
| Sticky Note8                   | Sticky Note       | Workflow summary and community help links   | None                          | None                             | ## Try It Out! ... Join the [Discord](https://discord.com/invite/XPKeKXeB7d) or ask in the [Forum](https://community.n8n.io/)! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: When clicking "Test workflow"  
   - Purpose: Manual start.

2. **Create Set Node**  
   - Name: Set Variables  
   - Connect from Manual Trigger.  
   - Set variables:  
     - CLOUDFLARE_ACCOUNT_ID (string): *Your Cloudflare Account ID*  
     - model (string): `@cf/facebook/detr-resnet-50`  
     - source_image (string): URL of the image to process (e.g., `https://images.pexels.com/photos/2293367/pexels-photo-2293367.jpeg?auto=compress&cs=tinysrgb&w=600`)  
     - elasticsearch_index (string): e.g., `n8n-image-search`

3. **Create HTTP Request Node to Fetch Source Image**  
   - Name: Fetch Source Image  
   - Connect from Set Variables.  
   - Method: GET  
   - URL expression: `={{ $json.source_image }}`  
   - Response Format: Binary (to get image data)

4. **Create HTTP Request Node to Use Cloudflare AI for Object Detection**  
   - Name: Use Detr-Resnet-50 Object Classification  
   - Connect from Fetch Source Image.  
   - Method: POST  
   - URL expression:  
     `=https://api.cloudflare.com/client/v4/accounts/{{ $('Set Variables').item.json.CLOUDFLARE_ACCOUNT_ID }}/ai/run/{{ $('Set Variables').item.json.model }}`  
   - Authentication: Cloudflare API credentials (set up OAuth or API token with Workers AI access)  
   - Content Type: Binary data (send image binary data)  
   - Input Data Field Name: `data`  
   - Send Body: true

5. **Create Split Out Node**  
   - Name: Split Out Results Only  
   - Connect from Use Detr-Resnet-50 Object Classification.  
   - Field to split out: `result`

6. **Create Filter Node**  
   - Name: Filter Score >= 0.9  
   - Connect from Split Out Results Only.  
   - Condition: Number operation "greater than or equal"  
     - Left value: Expression `={{ $json.score }}`  
     - Right value: `0.9`

7. **Create HTTP Request Node to Fetch Source Image Again**  
   - Name: Fetch Source Image Again  
   - Connect from Filter Score >= 0.9.  
   - Method: GET  
   - URL expression: `={{ $('Set Variables').item.json.source_image }}`  
   - Response Format: Binary

8. **Create Edit Image Node to Crop Object**  
   - Name: Crop Object From Image  
   - Connect from Fetch Source Image Again.  
   - Operation: Crop  
   - Width: Expression `={{ $json.box.xmax - $json.box.xmin }}`  
   - Height: Expression `={{ $json.box.ymax - $json.box.ymin }}`  
   - PositionX: Expression `={{ $json.box.xmin }}`  
   - PositionY: Expression `={{ $json.box.ymin }}`  
   - Format: JPEG  
   - FileName: Expression `={{ $binary.data.fileName.split('.')[0].urlEncode() + '-' + $json.label.urlEncode() + '-' + $itemIndex }}.jpg`

9. **Create HTTP Request Node to Upload Cropped Image to Cloudinary**  
   - Name: Upload to Cloudinary  
   - Connect from Crop Object From Image.  
   - Method: POST  
   - URL: `https://api.cloudinary.com/v1_1/<your-cloud-name>/image/upload`  
   - Content Type: multipart/form-data  
   - Body Parameters:  
     - Name: `file`  
     - Parameter Type: formBinaryData  
     - Input Data Field Name: `data` (cropped image binary)  
   - Query Parameters:  
     - Name: `upload_preset`  
     - Value: your Cloudinary unsigned upload preset (e.g., `n8n-workflows-preset`)  
   - Authentication: HTTP Query Auth with Cloudinary API credentials.

10. **Create Elasticsearch Node to Index Document**  
    - Name: Create Docs In Elasticsearch  
    - Connect from Upload to Cloudinary.  
    - Operation: Create  
    - Index ID: Expression `={{ $('Set Variables').item.json.elasticsearch_index }}`  
    - Fields:  
      - image_url: Expression `={{ $json.secure_url.replace('upload','upload/f_auto,q_auto') }}`  
      - source_image_url: Expression `={{ $('Set Variables').item.json.source_image }}`  
      - label: Expression `={{ $('Crop Object From Image').item.json.label }}`  
      - metadata: Expression `={{ JSON.stringify(Object.assign($('Crop Object From Image').item.json, { filename: $json.original_filename })) }}`  
    - Set Elasticsearch credentials with your instance details.

---

### 5. General Notes & Resources

| Note Content                                                                                                             | Context or Link                                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| This workflow enables object-based image search through AI detection, cropping, and indexing.                            | Workflow overview                                                                                                                  |
| Use Cloudflare Workers AI to run the Detr-Resnet-50 model for object detection.                                           | https://developers.cloudflare.com/workers-ai/                                                                                    |
| Edit Image node supports cropping based on bounding boxes; essential for isolated object images.                         | https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage                                                      |
| Elasticsearch node indexes object metadata to enable searching by object labels and images.                              | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.elasticsearch                                                    |
| Cloudinary is used as an external CDN for efficient hosting and delivery of cropped object images.                       | https://cloudinary.com/documentation/image_upload_api_reference                                                                    |
| Example classification result image provided for visual reference.                                                       | https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto,w_300/v1/n8n-workflows/ywtzjcmqrypihci1npgh                       |
| For support and community help, join n8n Discord or Forum.                                                                | Discord: https://discord.com/invite/XPKeKXeB7d, Forum: https://community.n8n.io/                                                  |

---

This detailed analysis and reconstruction guide enable advanced users and AI agents to fully understand, reproduce, and adapt the workflow for object-based image search use cases, while anticipating common failure modes and integration requirements.