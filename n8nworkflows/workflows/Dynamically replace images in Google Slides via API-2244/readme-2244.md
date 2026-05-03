Dynamically replace images in Google Slides via API

https://n8nworkflows.xyz/workflows/dynamically-replace-images-in-google-slides-via-api-2244


# Dynamically replace images in Google Slides via API

---

### 1. Workflow Overview

This workflow provides a dynamic API endpoint to replace images in a Google Slides presentation by using Google Slides API. It targets scenarios where presentations need to be updated programmatically, such as changing client logos, backgrounds, or any images tagged with unique identifiers in slides.

**Logical Blocks:**

- **1.1 Input Reception & Validation:**  
  Receives a POST request containing necessary parameters (`presentation_id`, `image_key`, `image_url`) and validates their presence.

- **1.2 Slide Content Retrieval:**  
  Queries the Google Slides presentation to retrieve all slide elements.

- **1.3 Image Identification:**  
  Filters slide elements to find images tagged with the specified unique identifier (`image_key`).

- **1.4 Image Replacement:**  
  Sends a batch update request to Google Slides API to replace the identified images with a new image URL and update their alt text.

- **1.5 Response Handling:**  
  Returns success or error messages back to the API caller based on the outcome.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block listens for incoming POST requests on a webhook, extracts the required parameters, and verifies that all mandatory fields are present.

- **Nodes Involved:**  
  - Webhook  
  - Check if all params are provided  
  - Error Missing Fields

- **Node Details:**

  - **Webhook**  
    - *Type & Role:* HTTP webhook node; entry point for the API endpoint `/replace-image-in-slide`.  
    - *Configuration:* Accepts POST requests. Response mode is set to `responseNode` to delegate response to downstream nodes.  
    - *Expressions:* Uses `body.presentation_id`, `body.image_key`, and `body.image_url` from HTTP request body.  
    - *Connections:* Outputs to `Check if all params are provided`.  
    - *Potential Failures:* Invalid HTTP method, malformed requests, missing body.  
    - *Version:* 2 (supports webhookId).

  - **Check if all params are provided**  
    - *Type & Role:* Conditional node (IF) to ensure all three required fields exist.  
    - *Configuration:* Checks existence of `presentation_id`, `image_key`, and `image_url` in the webhook request body using strict type validation.  
    - *Connections:* If all exist, proceeds to `Retrieve All Slide Elements`; else, routes to `Error Missing Fields`.  
    - *Edge Cases:* Partial or empty fields, type mismatches, missing keys.

  - **Error Missing Fields**  
    - *Type & Role:* Responds to webhook with HTTP 500 error and JSON message indicating missing fields.  
    - *Configuration:* Response code 500, body: `{"error": "Missing fields."}`.  
    - *Connections:* Terminal node for invalid requests.  
    - *Edge Cases:* None beyond receiving improper input.

---

#### 2.2 Slide Content Retrieval

- **Overview:**  
  Fetches the entire slide deck data from Google Slides API to analyze all page elements.

- **Nodes Involved:**  
  - Retrieve All Slide Elements

- **Node Details:**

  - **Retrieve All Slide Elements**  
    - *Type & Role:* HTTP Request node to Google Slides API.  
    - *Configuration:* Performs GET request on `https://slides.googleapis.com/v1/presentations/{{presentation_id}}`.  
    - *Authentication:* Google Slides OAuth2 credential required.  
    - *Expressions:* Uses `presentation_id` from webhook body.  
    - *Connections:* Outputs to `Retrieve matching Images ObjectIds`.  
    - *Potential Failures:* Authentication errors, invalid presentation ID, network timeout, API rate limits, large presentations causing timeouts.

---

#### 2.3 Image Identification

- **Overview:**  
  Processes the slide data to identify images whose alt text description matches the given `image_key`.

- **Nodes Involved:**  
  - Retrieve matching Images ObjectIds

- **Node Details:**

  - **Retrieve matching Images ObjectIds**  
    - *Type & Role:* Code node running JavaScript to filter slide elements.  
    - *Configuration:*  
      - Extracts `image_key` from webhook body.  
      - Flattens all slides and their page elements.  
      - Filters page elements where an image exists and alt text description equals `image_key`.  
      - Returns an array of objects with `objectId` properties for matched images.  
    - *Expressions/Code:*  
      ```javascript
      const key = $('Webhook').item.json.body.image_key;

      const pageElements = $input
        .all()
        .flatMap(item => item.json.slides)
        .flatMap(slide => slide.pageElements.filter(el => el.image && el.description === key));

      const objectIds = pageElements.map(el => ({ objectId: el.objectId }));

      return objectIds;
      ```  
    - *Connections:* Outputs to `Replace Images`.  
    - *Edge Cases:* No matching images (empty array), malformed slide data, missing alt text fields.

---

#### 2.4 Image Replacement

- **Overview:**  
  For each identified image, sends a batch update request to Google Slides API to replace the image with the new URL and update its alt text.

- **Nodes Involved:**  
  - Replace Images

- **Node Details:**

  - **Replace Images**  
    - *Type & Role:* HTTP Request node executing batchUpdate on Google Slides API.  
    - *Configuration:*  
      - POST to `https://slides.googleapis.com/v1/presentations/{{presentation_id}}:batchUpdate`.  
      - JSON body contains two requests per image:  
        1. `replaceImage` with `imageObjectId`, new `url`, and `imageReplaceMethod` set to `"CENTER_CROP"`.  
        2. `updatePageElementAltText` updates the alt text description to the `image_key`.  
      - Uses expressions to dynamically inject `presentation_id`, `image_url`, `image_key`, and `objectId` from previous nodes.  
    - *Authentication:* Google Slides OAuth2 credential.  
    - *Connections:* Outputs to `Respond to Webhook`.  
    - *Edge Cases:*  
      - API errors (invalid objectId, invalid URL, permission errors).  
      - Multiple images replaced in one request.  
      - Network failures or quota limits.  
      - If no images found, this node might not be called or called with empty data (should be handled upstream).  
    - *Version:* 4.2.

---

#### 2.5 Response Handling

- **Overview:**  
  Sends a JSON success message back to the API client indicating that the image replacement was completed.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - *Type & Role:* HTTP response node for webhook to send back success confirmation.  
    - *Configuration:* Responds with JSON: `{"message": "Image replaced."}`.  
    - *Connections:* Terminal node following `Replace Images`.  
    - *Edge Cases:* None; assumes prior nodes succeeded.

---

#### 2.6 Sticky Note (Documentation Node)

- **Overview:**  
  Contains detailed documentation for users, including setup instructions and example usage.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**

  - **Sticky Note**  
    - *Type & Role:* Visual aid containing workflow description, usage steps, and example Curl request.  
    - *Content:*  
      - How to add unique key identifiers in Google Slides images via alt text.  
      - Required POST request parameters and example.  
      - Workflow purpose and flexibility.  
      - YouTube overview link: [https://www.youtube.com/watch?v=3kM7lKorWkQ](https://www.youtube.com/watch?v=3kM7lKorWkQ)  
    - *Connections:* None; standalone visual node.

---

### 3. Summary Table

| Node Name                     | Node Type                  | Functional Role                      | Input Node(s)            | Output Node(s)                | Sticky Note                                                    |
|-------------------------------|----------------------------|------------------------------------|--------------------------|------------------------------|----------------------------------------------------------------|
| Webhook                       | Webhook                    | API entry point                    |                          | Check if all params are provided |                                                                |
| Check if all params are provided | IF (Conditional)          | Validate presence of required fields | Webhook                  | Retrieve All Slide Elements, Error Missing Fields |                                                                |
| Error Missing Fields          | Respond to Webhook         | Return error if required fields missing | Check if all params are provided |                              |                                                                |
| Retrieve All Slide Elements   | HTTP Request               | Get full slide deck data from API  | Check if all params are provided | Retrieve matching Images ObjectIds |                                                                |
| Retrieve matching Images ObjectIds | Code (JavaScript)          | Filter images matching key identifier | Retrieve All Slide Elements | Replace Images                |                                                                |
| Replace Images               | HTTP Request               | Replace and update images in Google Slides | Retrieve matching Images ObjectIds | Respond to Webhook            |                                                                |
| Respond to Webhook            | Respond to Webhook         | Return success response             | Replace Images            |                              |                                                                |
| Sticky Note                  | Sticky Note                | Documentation and user instructions |                          |                              | See full workflow description and YouTube overview link (https://www.youtube.com/watch?v=3kM7lKorWkQ) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `replace-image-in-slide`  
   - Response Mode: `responseNode` (response delegated to downstream)  
   - No authentication required on webhook.  

2. **Add IF Node to Validate Inputs:**  
   - Type: IF  
   - Conditions: Check existence (strict) of all three fields in the webhook body:  
     - `presentation_id`  
     - `image_key`  
     - `image_url`  
   - On TRUE (all present): proceed to next step.  
   - On FALSE: connect to error response node.

3. **Add Error Response Node:**  
   - Type: Respond to Webhook  
   - Response Code: 500  
   - Response Body (JSON): `{"error": "Missing fields."}`  
   - Connect this node from IF false output.

4. **Add HTTP Request Node to Retrieve Slides:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://slides.googleapis.com/v1/presentations/{{ $json.body.presentation_id }}`  
   - Authentication: Use Google Slides OAuth2 credentials.  
   - Connect from IF true output.

5. **Add Code Node to Filter Images:**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const key = $('Webhook').item.json.body.image_key;

     const pageElements = $input
       .all()
       .flatMap(item => item.json.slides)
       .flatMap(slide => slide.pageElements.filter(el => el.image && el.description === key));

     const objectIds = pageElements.map(el => ({ objectId: el.objectId }));

     return objectIds;
     ```  
   - Connect output from HTTP Request node.

6. **Add HTTP Request Node to Replace Images:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://slides.googleapis.com/v1/presentations/{{ $json.body.presentation_id }}:batchUpdate`  
   - Authentication: Google Slides OAuth2 credential.  
   - Body Type: JSON  
   - JSON Body (use expression and loop over matched images):  
     ```json
     {
       "requests": [
         {
           "replaceImage": {
             "imageObjectId": "{{ $json.objectId }}",
             "url": "{{ $json.body.image_url }}",
             "imageReplaceMethod": "CENTER_CROP"
           }
         },
         {
           "updatePageElementAltText": {
             "objectId": "{{ $json.objectId }}",
             "description": "{{ $json.body.image_key }}"
           }
         }
       ]
     }
     ```  
   - Connect output from Code node.

7. **Add Respond to Webhook Node for Success:**  
   - Type: Respond to Webhook  
   - Response Body (JSON): `{"message": "Image replaced."}`  
   - Connect from the "Replace Images" node.

8. **Create Google Slides OAuth2 Credential:**  
   - Configure OAuth2 credentials with Google API Console.  
   - Enable Google Slides API.  
   - Set appropriate scopes (e.g., `https://www.googleapis.com/auth/presentations`).  
   - Add credential to HTTP request nodes requiring it.

9. **Optional: Add Sticky Note Node:**  
   - Add a sticky note on the canvas with documentation:  
     - Workflow purpose  
     - Setup steps for alt text key identifiers  
     - Example Curl request  
     - Link to YouTube overview: https://www.youtube.com/watch?v=3kM7lKorWkQ

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| This workflow requires that images in Google Slides be tagged with unique keys in their alt text descriptions (Format Options > Alt Text). | Setup instruction for image identification                      |
| Example Curl to invoke the workflow:                                                             | `curl --location 'https://workflow.url' --form 'presentation_id="google-presentation-id"' --form 'image_key="background"' --form 'image_url="https://picsum.photos/536/354"'` |
| Official Google Slides API documentation: https://developers.google.com/slides/api/reference/rest | Reference for API calls used                                      |
| YouTube overview video explaining this workflow: https://www.youtube.com/watch?v=3kM7lKorWkQ     | Video demonstration and explanation                              |
| OAuth2 credentials must have Google Slides API enabled and proper scopes for presentation editing. | Credential requirement                                          |

---

This structured documentation enables deep understanding, reproduction, and modification of the workflow while anticipating common errors and integration challenges.