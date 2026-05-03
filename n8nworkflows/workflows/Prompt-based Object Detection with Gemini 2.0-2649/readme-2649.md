Prompt-based Object Detection with Gemini 2.0

https://n8nworkflows.xyz/workflows/prompt-based-object-detection-with-gemini-2-0-2649


# Prompt-based Object Detection with Gemini 2.0

### 1. Workflow Overview

This workflow demonstrates how to leverage Google Gemini 2.0's experimental prompt-based bounding box object detection capabilities within an n8n automation. The core use case is to identify specific objects or subjects in an image based on a natural language prompt (e.g., "all bunnies"), retrieve their bounding box coordinates, and visually mark these detected objects on the original image.

The logical flow is divided into these main blocks:

1.1 **Input Reception & Image Preparation**  
- Download the test image from a URL  
- Extract image metadata (width and height) needed for coordinate scaling  

1.2 **AI Processing with Gemini 2.0**  
- Submit the image and prompt to Gemini 2.0 API for bounding box detection  
- Receive normalized bounding box coordinates in JSON format  

1.3 **Coordinate Scaling and Processing**  
- Parse and extract the bounding box data  
- Rescale normalized coordinates (0 to 1000 scale) to actual pixel values based on image dimensions  

1.4 **Visualization of Results**  
- Draw bounding boxes onto the original image using the scaled coordinates  
- Output the annotated image for validation or further use  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Image Preparation

- **Overview:**  
  This block handles fetching the target image and extracting its size details, which are essential for accurately mapping the bounding box coordinates returned by the AI.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Get Test Image (HTTP Request)  
  - Get Image Info (Edit Image)  

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual testing of the workflow  
    - Config: No parameters; triggers workflow on demand  
    - Input: None  
    - Output: Triggers "Get Test Image"  
    - Edge Cases: None typical; manual trigger ensures controlled start  

  - **Get Test Image**  
    - Type: HTTP Request  
    - Role: Downloads the image from a fixed URL  
    - Config: URL set to a petting zoo image with rabbits; no authentication or special headers  
    - Input: Trigger from manual node  
    - Output: Binary image data passed to "Get Image Info"  
    - Edge Cases:  
      - Network errors or image URL unavailability  
      - Unsupported image format causing failure downstream  
      - Large image size could affect performance  

  - **Get Image Info**  
    - Type: Edit Image (Information extraction)  
    - Role: Extracts metadata (width and height) from the binary image  
    - Config: Operation set to "information" to retrieve image size metadata  
    - Input: Binary image from HTTP Request  
    - Output: JSON containing image width and height, forwarded to Gemini API node  
    - Edge Cases:  
      - Corrupted or unsupported image format causing failure  
      - Missing metadata field errors  

---

#### 1.2 AI Processing with Gemini 2.0

- **Overview:**  
  This block sends the image and a text prompt to Google Gemini 2.0 API to detect bounding boxes of objects matching the prompt ("all bunnies" in this demo). It receives normalized coordinates as JSON.

- **Nodes Involved:**  
  - Gemini 2.0 Object Detection  
  - Get Variables (Set node, prepares extracted data)  

- **Node Details:**  

  - **Gemini 2.0 Object Detection**  
    - Type: HTTP Request  
    - Role: Sends POST request to Google Gemini 2.0 model API for bounding box detection  
    - Config:  
      - URL: Gemini 2.0 API endpoint for generating content with bounding box detection  
      - Method: POST  
      - Body: JSON containing prompt text and inline image data (base64-encoded)  
      - Authentication: Uses predefined Google Palm API credentials  
      - Response schema expects an array of objects, each with `box_2d` (array of coordinates) and `label` (string)  
    - Input: Binary image from "Get Image Info" node  
    - Output: JSON with detected bounding box data passed to "Get Variables" node  
    - Edge Cases:  
      - Authentication failures due to invalid or expired credentials  
      - API rate limits or timeouts  
      - Malformed or unexpected response payloads  
      - Gemini 2.0 model being experimental may produce incomplete or inaccurate detections  

  - **Get Variables**  
    - Type: Set  
    - Role: Extracts and parses the bounding box data and image dimensions for further processing  
    - Config:  
      - Uses expressions to parse the first candidate’s content text as JSON, extracting bounding box `coords`  
      - Retrieves image `width` and `height` from prior node's JSON metadata  
    - Input: JSON response from Gemini node and image info data  
    - Output: Structured JSON with bounding box array and image dimensions forwarded to scaling code node  
    - Edge Cases:  
      - Failure if parsing JSON fails (e.g., API response format changes)  
      - Missing or null data in expected fields  

---

#### 1.3 Coordinate Scaling and Processing

- **Overview:**  
  This block rescales the normalized bounding box coordinates (0-1000 range) to pixel coordinates matching the original image dimensions to allow precise drawing.

- **Nodes Involved:**  
  - Scale Normalised Coords (Code node)  

- **Node Details:**  

  - **Scale Normalised Coords**  
    - Type: Code (JavaScript)  
    - Role: Converts normalized bounding boxes into pixel coordinates  
    - Config:  
      - Extracts `coords`, `width`, and `height` from input JSON  
      - Applies scaling formulas:  
        - X coordinates scaled by `(val * width) / 1000`  
        - Y coordinates scaled by `(val * height) / 1000`  
      - Filters out invalid boxes (must have 4 coordinates)  
      - Outputs normalized boxes with properties `xmin`, `xmax`, `ymin`, `ymax`  
      - Passes along the original binary image for annotation  
    - Input: Parsed coordinates and image dimension data from "Get Variables"  
    - Output: JSON with scaled bounding boxes and original binary image forwarded to drawing node  
    - Edge Cases:  
      - Coordinates missing or malformed (e.g., wrong array length)  
      - Division by zero if width or height missing or zero  
      - Index errors if bounding box arrays are incorrectly structured  

---

#### 1.4 Visualization of Results

- **Overview:**  
  This block draws the scaled bounding boxes onto the original image using the "Edit Image" node to visually confirm the detected objects.

- **Nodes Involved:**  
  - Draw Bounding Boxes (Edit Image)  

- **Node Details:**  

  - **Draw Bounding Boxes**  
    - Type: Edit Image (Draw operations)  
    - Role: Annotates the original image by drawing rectangles for each detected bounding box  
    - Config:  
      - Multi-step drawing operation configured with a fixed set of up to 6 bounding boxes  
      - Each box uses the scaled xmin, ymin, xmax, ymax values for start and end positions  
      - Color set to a semi-transparent pink (#ff00f277) for visibility  
      - Corner radius fixed to 0 (sharp corners)  
      - Coordinates accessed via JSON expressions referencing the scaled coords array  
    - Input: JSON with scaled bounding boxes and binary image from the code node  
    - Output: Modified image with drawn bounding boxes (binary data)  
    - Edge Cases:  
      - If fewer than 6 bounding boxes are returned, references to non-existent array indices will cause errors  
      - Coordinates out of image bounds may cause drawing issues  
      - Image format incompatibility or large image size impacting performance  

---

### 3. Summary Table

| Node Name                 | Node Type          | Functional Role                                | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                                    |
|---------------------------|--------------------|------------------------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Entry point to start the workflow manually      | -                         | Get Test Image            | Try it out! This n8n template demonstrates how to use Gemini 2.0's new Bounding Box detection capabilities your workflows. The key difference being this enables prompt-based object detection for images... [Discord](https://discord.com/invite/XPKeKXeB7d) [Forum](https://community.n8n.io/) |
| Get Test Image            | HTTP Request       | Downloads test image from a fixed URL           | When clicking ‘Test workflow’ | Get Image Info            | ## 1. Download Test Image [HTTP node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) [Image format docs](https://ai.google.dev/gemini-api/docs/vision?lang=rest#technical-details-image)                     |
| Get Image Info            | Edit Image         | Extracts image width and height                  | Get Test Image             | Gemini 2.0 Object Detection |                                                                                                                                                |
| Gemini 2.0 Object Detection | HTTP Request       | Calls Gemini API to detect bounding boxes       | Get Image Info             | Get Variables             | ## 2. Use Prompt-Based Object Detection [HTTP node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest) [Previous object detection example](https://n8n.io/workflows/2331-build-your-own-image-search-using-ai-object-detection-cdn-and-elasticsearch/) |
| Get Variables             | Set                | Parses Gemini response and extracts variables   | Gemini 2.0 Object Detection | Scale Normalised Coords   |                                                                                                                                                |
| Scale Normalised Coords   | Code               | Scales normalized bounding box coordinates      | Get Variables              | Draw Bounding Boxes       | ## 3. Scale Coords to Fit Original Image [Code node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/) [Bounding box explanation](https://ai.google.dev/gemini-api/docs/models/gemini-v2?_gl=1*187cb6v*_up*MQ..*_ga*MTU1ODkzMDc0Mi4xNzM0NDM0NDg2*_ga_P1DBVKWT6V*MTczNDQzNDQ4Ni4xLjAuMTczNDQzNDQ4Ni4wLjAuMjEzNzc5MjU0Ng..#bounding-box) |
| Draw Bounding Boxes       | Edit Image         | Draws bounding boxes on the original image      | Scale Normalised Coords    | -                        | ## 4. Draw! [Edit Image node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage/)                                                                                             |
| Sticky Note               | Sticky Note        | Image example of output bounding box detection  | -                         | -                        | Fig 1. Output of Object Detection ![](https://res.cloudinary.com/daglih2g8/image/upload/f_auto,q_auto/v1/n8n-workflows/download_1_qmqyyo#full-width)                                                      |
| Sticky Note1              | Sticky Note        | Description of downloading the test image       | -                         | -                        | ## 1. Download Test Image [HTTP node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)                                                                                   |
| Sticky Note2              | Sticky Note        | Explanation of prompt-based object detection     | -                         | -                        | ## 2. Use Prompt-Based Object Detection [HTTP node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest)                                                                     |
| Sticky Note3              | Sticky Note        | Explains coordinate scaling logic                | -                         | -                        | ## 3. Scale Coords to Fit Original Image [Code node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/)                                                                          |
| Sticky Note4              | Sticky Note        | Notes on why not using Basic LLM node            | -                         | -                        | ### Q. Why not use the Basic LLM node? At time of writing, Langchain version does not recognise Gemini 2.0 to be a multimodal model.                                                                          |
| Sticky Note5              | Sticky Note        | Notes on drawing bounding boxes                   | -                         | -                        | ## 4. Draw! [Edit Image node docs](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.editimage/)                                                                                             |
| Sticky Note6              | Sticky Note        | Overall workflow description and help links      | -                         | -                        | Try it out! This n8n template demonstrates how to use Gemini 2.0's new Bounding Box detection capabilities your workflows. [Discord](https://discord.com/invite/XPKeKXeB7d) [Forum](https://community.n8n.io/)                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Purpose: To start the workflow manually  

2. **Create HTTP Request Node to Download Image**  
   - Name: `Get Test Image`  
   - URL: `https://www.stonhambarns.co.uk/wp-content/uploads/jennys-ark-petting-zoo-for-website-6.jpg`  
   - Method: GET  
   - Connect output of manual trigger to this node  

3. **Create Edit Image Node to Get Image Info**  
   - Name: `Get Image Info`  
   - Operation: `information` (to extract image metadata)  
   - Input: Connect from `Get Test Image` node's output  
   - This node outputs JSON with image dimensions  

4. **Create HTTP Request Node for Gemini 2.0 API Call**  
   - Name: `Gemini 2.0 Object Detection`  
   - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash-exp:generateContent`  
   - Method: POST  
   - Authentication: Set to Google Palm API credential (OAuth2 or API key as configured)  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "contents": [{
         "parts": [
           {"text": "I want to see all bounding boxes of rabbits in this image."},
           {
             "inline_data": {
               "mime_type": "image/jpeg",
               "data": "={{ $input.item.binary.data.data }}"
             }
           }
         ]
       }],
       "generationConfig": {
         "response_mime_type": "application/json",
         "response_schema": {
           "type": "ARRAY",
           "items": {
             "type": "OBJECT",
             "properties": {
               "box_2d": {"type": "ARRAY", "items": {"type": "NUMBER"}},
               "label": {"type": "STRING"}
             }
           }
         }
       }
     }
     ```  
   - Connect output of `Get Image Info` node as input (image binary passed along)  

5. **Create Set Node to Extract Variables**  
   - Name: `Get Variables`  
   - Purpose: Parse Gemini response and extract coords, width, height  
   - Assignments:  
     - `coords` (array): `={{ $json.candidates[0].content.parts[0].text.parseJson() }}`  
     - `width` (string): `={{ $('Get Image Info').item.json.size.width }}`  
     - `height` (string): `={{ $('Get Image Info').item.json.size.height }}`  
   - Connect from `Gemini 2.0 Object Detection` output  

6. **Create Code Node to Scale Coordinates**  
   - Name: `Scale Normalised Coords`  
   - Language: JavaScript  
   - Code:  
     ```js
     const { coords, width, height } = $input.first().json;
     const scale = 1000;
     const scaleCoordX = (val) => (val * width) / scale;
     const scaleCoordY = (val) => (val * height) / scale;
     
     const normalisedOutput = coords
       .filter(coord => coord.box_2d.length === 4)
       .map(coord => {
         return {
           xmin: coord.box_2d[1] ? scaleCoordX(coord.box_2d[1]) : coord.box_2d[1],
           xmax: coord.box_2d[3] ? scaleCoordX(coord.box_2d[3]) : coord.box_2d[3],
           ymin: coord.box_2d[0] ? scaleCoordY(coord.box_2d[0]) : coord.box_2d[0],
           ymax: coord.box_2d[2] ? scaleCoordY(coord.box_2d[2]) : coord.box_2d[2],
         };
       });
     
     return {
       json: { coords: normalisedOutput },
       binary: $('Get Test Image').first().binary
     };
     ```  
   - Connect from `Get Variables` output  

7. **Create Edit Image Node to Draw Bounding Boxes**  
   - Name: `Draw Bounding Boxes`  
   - Operation: `multiStep`  
   - Configure drawing operations for each bounding box (up to 6), with:  
     - StartPositionX: `={{ $json.coords[index].xmin }}`  
     - StartPositionY: `={{ $json.coords[index].ymin }}`  
     - EndPositionX: `={{ $json.coords[index].xmax }}`  
     - EndPositionY: `={{ $json.coords[index].ymax }}`  
     - Color: `#ff00f277` (semi-transparent pink)  
     - CornerRadius: `0`  
   - Connect from `Scale Normalised Coords` output  

8. **Connect all nodes in sequence:**  
   `When clicking ‘Test workflow’` → `Get Test Image` → `Get Image Info` → `Gemini 2.0 Object Detection` → `Get Variables` → `Scale Normalised Coords` → `Draw Bounding Boxes`  

9. **Configure Credentials:**  
   - Set up Google Palm API credentials (OAuth2 or API key) under n8n Credentials for the Gemini API node  

10. **Optional:** Add Sticky Notes for documentation and reference inside the editor as per original workflow  

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow is an experimental demonstration of Google Gemini 2.0's bounding box detection capabilities.               | Workflow description                                                                                             |
| Gemini 2.0 bounding box coordinates are normalized to a 0–1000 scale and must be rescaled to image pixel dimensions.    | [Gemini 2.0 bounding box docs](https://ai.google.dev/gemini-api/docs/models/gemini-v2?_gl=1*187cb6v*_up*MQ..#bounding-box) |
| The workflow uses an experimental API and may produce incomplete or inaccurate detections; not recommended for production. | Template warning                                                                                                 |
| Join the n8n community Discord or Forum for support and discussion.                                                     | [Discord](https://discord.com/invite/XPKeKXeB7d), [Forum](https://community.n8n.io/)                              |
| Previous related n8n example using ResNet for object detection available for reference.                                  | [Image Search Workflow](https://n8n.io/workflows/2331-build-your-own-image-search-using-ai-object-detection-cdn-and-elasticsearch/) |

---

This documentation provides a comprehensive, structured understanding of the "Prompt-based Object Detection with Gemini 2.0" workflow, facilitating effective use, modification, and troubleshooting.