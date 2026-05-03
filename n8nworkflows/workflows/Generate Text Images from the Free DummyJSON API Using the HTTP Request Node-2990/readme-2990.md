Generate Text Images from the Free DummyJSON API Using the HTTP Request Node

https://n8nworkflows.xyz/workflows/generate-text-images-from-the-free-dummyjson-api-using-the-http-request-node-2990


# Generate Text Images from the Free DummyJSON API Using the HTTP Request Node

### 1. Workflow Overview

This workflow, titled **Generate Image Workflow**, is designed to dynamically create custom text-based images using a free, no-authentication API. It targets marketers, designers, content creators, and developers who want to automate the generation of images with specific text, colors, fonts, and formats without manual graphic design effort or complex integrations.

The workflow is logically divided into three main blocks:

- **1.1 Input Trigger**: Manual initiation of the workflow.
- **1.2 Image Property Definition**: Setting all parameters required to customize the image (text, size, colors, font, format).
- **1.3 API Request and Image Retrieval**: Constructing and sending an HTTP request to the DummyJSON API to generate the image based on the defined properties, and retrieving the resulting image.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger

- **Overview:**  
  This block initiates the workflow manually, allowing users to test or trigger the image generation process on demand.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **Node Name:** When clicking ‘Test workflow’  
  - **Type:** Manual Trigger (n8n-nodes-base.manualTrigger)  
  - **Role:** Starts the workflow manually without any input data.  
  - **Configuration:** No parameters set; default manual trigger.  
  - **Input/Output:** No input; outputs an empty JSON object to the next node.  
  - **Version Requirements:** Compatible with n8n v1 and above.  
  - **Potential Failures:** None expected; manual trigger is stable.  
  - **Sub-workflow:** None.

#### 2.2 Image Property Definition

- **Overview:**  
  This block defines all the customizable properties for the image to be generated, including text content, image size, colors, font, and output format.

- **Nodes Involved:**  
  - Set Image Properties

- **Node Details:**  
  - **Node Name:** Set Image Properties  
  - **Type:** Set (n8n-nodes-base.set)  
  - **Role:** Assigns static values for image customization parameters to be used in the HTTP request.  
  - **Configuration:**  
    - `size`: "600x400" (image dimensions in pixels)  
    - `backgroundColor`: "cc22e3" (hex code for background color)  
    - `textColor`: "ffffff" (hex code for text color)  
    - `text`: "Generated!" (text displayed on the image)  
    - `fontSize`: "100" (font size in pixels)  
    - `fontFamily`: "pacifico" (font family name)  
    - `type`: "png" (image format)  
  - **Expressions/Variables:** Static string assignments; no dynamic expressions used here.  
  - **Input:** Receives trigger from manual trigger node.  
  - **Output:** Passes JSON object with all image properties to the next node.  
  - **Version Requirements:** Compatible with n8n v1 and above.  
  - **Potential Failures:** None expected unless values are invalid or missing; no validation is performed here.  
  - **Sub-workflow:** None.

#### 2.3 API Request and Image Retrieval

- **Overview:**  
  This block sends an HTTP GET request to the DummyJSON API to generate the image based on the parameters set previously. It constructs the URL dynamically using the input parameters and retrieves the generated image.

- **Nodes Involved:**  
  - Fetch Image from API

- **Node Details:**  
  - **Node Name:** Fetch Image from API  
  - **Type:** HTTP Request (n8n-nodes-base.httpRequest)  
  - **Role:** Performs the API call to generate and fetch the image.  
  - **Configuration:**  
    - URL Template: `https://dummyjson.com/image/{{ $json.size }}/{{ $json.backgroundColor }}/{{ $json.textColor }}`  
    - Query Parameters:  
      - `text`: from `$json.text`  
      - `fontSize`: from `$json.fontSize`  
      - `type`: from `$json.type`  
      - `fontFamily`: from `$json.fontFamily`  
    - HTTP Method: GET (default)  
    - Options: Default (no authentication, no special headers)  
  - **Expressions/Variables:** Uses n8n expression syntax to dynamically insert parameters from previous node’s JSON data.  
  - **Input:** Receives image property JSON from Set node.  
  - **Output:** Returns the generated image data (likely as binary or base64 depending on API response).  
  - **Version Requirements:** Uses HTTP Request node version 4.2 features (query parameters support).  
  - **Potential Failures:**  
    - Network errors or API downtime.  
    - Invalid parameter values causing API errors or malformed images.  
    - Timeout if API response is slow.  
    - Incorrect URL or expression errors if parameters are missing or malformed.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                  | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                  |
|--------------------------|--------------------|---------------------------------|-----------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Initiates workflow manually      | None                        | Set Image Properties      |                                                                                              |
| Set Image Properties      | Set                | Defines image customization parameters | When clicking ‘Test workflow’ | Fetch Image from API      | Modify this node to define your preferred image properties: text, size, colors, font, format |
| Fetch Image from API      | HTTP Request       | Calls API to generate and fetch image | Set Image Properties         | None                     | Uses DummyJSON API to generate text images dynamically without authentication                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name it `When clicking ‘Test workflow’`.  
   - No special configuration needed.

3. **Add a Set node:**  
   - Name it `Set Image Properties`.  
   - Connect the Manual Trigger node’s output to this Set node’s input.  
   - Configure the Set node to assign the following fields (all as strings):  
     - `size`: `"600x400"`  
     - `backgroundColor`: `"cc22e3"`  
     - `textColor`: `"ffffff"`  
     - `text`: `"Generated!"`  
     - `fontSize`: `"100"`  
     - `fontFamily`: `"pacifico"`  
     - `type`: `"png"`  
   - These values can be modified later to customize the image.

4. **Add an HTTP Request node:**  
   - Name it `Fetch Image from API`.  
   - Connect the Set node’s output to this HTTP Request node’s input.  
   - Set HTTP Method to `GET`.  
   - Set the URL to:  
     ```
     https://dummyjson.com/image/{{ $json.size }}/{{ $json.backgroundColor }}/{{ $json.textColor }}
     ```  
     Use n8n expression editor to insert the dynamic parts from the incoming JSON.  
   - Under Query Parameters, add the following key-value pairs, all using expressions from the incoming JSON:  
     - `text`: `{{$json.text}}`  
     - `fontSize`: `{{$json.fontSize}}`  
     - `type`: `{{$json.type}}`  
     - `fontFamily`: `{{$json.fontFamily}}`  
   - Leave authentication empty (the API requires none).  
   - Leave other options as default.

5. **Save and activate the workflow.**

6. **Test the workflow:**  
   - Click “Execute Workflow” or “Test Workflow” in n8n.  
   - The HTTP Request node will call the API and return the generated image.  
   - The output can be used downstream or exported as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                   |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow uses the DummyJSON free API for generating text-based images with no authentication required.    | https://dummyjson.com/                                                                           |
| Customize the Set node to change image text, size, colors, font, and format dynamically.                       | Instructions included in the workflow description section.                                       |
| Ideal for automating social media graphics, placeholder images, or content marketing pipelines.                | Workflow description.                                                                            |
| No external design software or complex graphic integrations needed.                                            | Workflow description.                                                                            |

---

This documentation provides a complete, structured understanding of the workflow, enabling users and automation agents to reproduce, modify, and troubleshoot the image generation process effectively.