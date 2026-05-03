Upload Carousel of Images to Tiktok and Instagram  with upload-post.com

https://n8nworkflows.xyz/workflows/upload-carousel-of-images-to-tiktok-and-instagram--with-upload-post-com-3524


# Upload Carousel of Images to Tiktok and Instagram  with upload-post.com

### 1. Workflow Overview

This workflow automates the uploading of image carousels to Instagram and slideshows to TikTok using the third-party service upload-post.com. It is designed for content creators, digital marketers, and social media managers who want to streamline multi-image posting across these platforms with minimal manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and fetch images from external URLs.
- **1.2 Image Preparation:** Rename and organize binary image data for multi-image upload compatibility.
- **1.3 Image Merging:** Combine multiple renamed image binaries into a single item for multipart form submission.
- **1.4 Platform Upload:** Send the prepared images as multipart form-data HTTP POST requests to upload-post.com API endpoints for Instagram and TikTok.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow manually and retrieves the images to be uploaded from specified URLs.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Get Image 1 (HTTP Request)  
- Get Image 2 (HTTP Request)

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Triggers two parallel HTTP Request nodes to fetch images.  
  - Edge Cases: None typical; user must manually trigger.  

- **Get Image 1**  
  - Type: HTTP Request  
  - Role: Downloads the first image from a public URL.  
  - Configuration:  
    - Method: GET (default)  
    - URL: https://upload.wikimedia.org/wikipedia/commons/7/70/Example.png  
    - Options: Default (no authentication or special headers)  
  - Inputs: Trigger from manual node  
  - Outputs: Binary data of the image under default binary key `data`  
  - Edge Cases: Network errors, URL unavailability, or invalid image format.  

- **Get Image 2**  
  - Type: HTTP Request  
  - Role: Downloads the second image from the same public URL as Image 1 (for demonstration).  
  - Configuration: Same as Get Image 1.  
  - Inputs: Trigger from manual node  
  - Outputs: Binary data of the image under default binary key `data`  
  - Edge Cases: Same as Get Image 1.

---

#### 2.2 Image Preparation

**Overview:**  
This block renames the binary data keys of each image to unique identifiers (`photo1`, `photo2`) to prepare for multipart form submission.

**Nodes Involved:**  
- Change name to photo1 (Code)  
- Change name to photo2 (Code)

**Node Details:**

- **Change name to photo1**  
  - Type: Code (JavaScript)  
  - Role: Renames the binary data key from `data` to `photo1` for the first image.  
  - Configuration:  
    - Code iterates over items, extracts binary data under `data`, and reassigns it to `photo1`.  
  - Inputs: Binary data from Get Image 1  
  - Outputs: Items with binary key renamed to `photo1`  
  - Edge Cases: Missing binary data or unexpected binary key names could cause failures.  

- **Change name to photo2**  
  - Type: Code (JavaScript)  
  - Role: Renames the binary data key from `data` to `photo2` for the second image.  
  - Configuration: Similar to photo1 node, but renames to `photo2`.  
  - Inputs: Binary data from Get Image 2  
  - Outputs: Items with binary key renamed to `photo2`  
  - Edge Cases: Same as photo1 node.

---

#### 2.3 Image Merging

**Overview:**  
This block merges the separately renamed image binaries into a single item, enabling the HTTP request node to send multiple images in one multipart form-data request.

**Nodes Involved:**  
- Merge (Merge node)  
- Send as 1 merged file (Code)

**Node Details:**

- **Merge**  
  - Type: Merge  
  - Role: Combines two input streams (photo1 and photo2 renamed items) into one output stream.  
  - Configuration: Default merge mode (likely "Merge by position" or "Wait for all inputs")  
  - Inputs: Outputs from Change name to photo1 and Change name to photo2  
  - Outputs: Two items forwarded to the next node  
  - Edge Cases: If one input is missing or delayed, merge may stall or error.  

- **Send as 1 merged file**  
  - Type: Code (JavaScript)  
  - Role: Consolidates multiple items (each with one photo binary) into a single item containing all photo binaries.  
  - Configuration:  
    - Iterates over all incoming items, copies their binary data into one combined item.  
    - Returns a single-item array with all photos as separate binary keys.  
  - Inputs: Two items from Merge node  
  - Outputs: Single item with binary keys `photo1`, `photo2`  
  - Edge Cases: Missing binary data in any item could cause incomplete merges.

---

#### 2.4 Platform Upload

**Overview:**  
This block sends the merged images as multipart form-data POST requests to upload-post.com API endpoints for Instagram and TikTok, using the provided API key and user details.

**Nodes Involved:**  
- POST TO INSTAGRAM1 (HTTP Request)  
- POST TO TIKTOK (HTTP Request)  
- Sticky Note7 (Sticky Note)

**Node Details:**

- **POST TO INSTAGRAM1**  
  - Type: HTTP Request  
  - Role: Uploads images as an Instagram carousel via upload-post.com API.  
  - Configuration:  
    - URL: https://api.upload-post.com/api/upload_photos  
    - Method: POST  
    - Content-Type: multipart/form-data  
    - Headers: Authorization with API key (`Apikey api`)  
    - Body Parameters:  
      - title: "title-ig" (static string, should be parameterized)  
      - user: "user_name" (static string, should be parameterized)  
      - platform[]: "instagram"  
      - photos[]: binary data fields `photo1` and `photo2`  
  - Inputs: Single merged item with photo binaries  
  - Outputs: API response with upload status  
  - Edge Cases:  
    - Authentication errors if API key is invalid or missing  
    - API rate limits or downtime  
    - Incorrect binary data format or missing photos  
    - Static title and user values may need dynamic configuration  

- **POST TO TIKTOK**  
  - Type: HTTP Request  
  - Role: Uploads images as a TikTok slideshow via upload-post.com API.  
  - Configuration: Same as Instagram node except:  
    - platform[]: "tiktok"  
  - Inputs: Single merged item with photo binaries  
  - Outputs: API response with upload status  
  - Edge Cases: Same as Instagram node  

- **Sticky Note7**  
  - Type: Sticky Note  
  - Role: Visual label for the Instagram POST block  
  - Content: "# POST : to Instagram"  
  - Position: Covers the Instagram POST node area  
  - No functional impact  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)                  | Output Node(s)                 | Sticky Note                  |
|-------------------------|---------------------|----------------------------------------|-------------------------------|-------------------------------|------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Starts the workflow manually            | None                          | Get Image 1, Get Image 2       |                              |
| Get Image 1             | HTTP Request        | Downloads first image from URL          | When clicking ‘Test workflow’  | Change name to photo1          |                              |
| Get Image 2             | HTTP Request        | Downloads second image from URL         | When clicking ‘Test workflow’  | Change name to photo2          |                              |
| Change name to photo1   | Code                | Renames binary key to photo1             | Get Image 1                   | Merge                         |                              |
| Change name to photo2   | Code                | Renames binary key to photo2             | Get Image 2                   | Merge                         |                              |
| Merge                  | Merge               | Combines two streams into one            | Change name to photo1, photo2 | Send as 1 merged file          |                              |
| Send as 1 merged file   | Code                | Merges multiple items into one with all photos | Merge                        | POST TO INSTAGRAM1, POST TO TIKTOK |                              |
| POST TO INSTAGRAM1      | HTTP Request        | Uploads images as Instagram carousel     | Send as 1 merged file          | None                          | # POST : to Instagram         |
| POST TO TIKTOK          | HTTP Request        | Uploads images as TikTok slideshow       | Send as 1 merged file          | None                          |                              |
| Sticky Note7            | Sticky Note         | Visual label for Instagram POST block    | None                          | None                          | # POST : to Instagram         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No special parameters.

2. **Create HTTP Request Node for First Image**  
   - Name: `Get Image 1`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://upload.wikimedia.org/wikipedia/commons/7/70/Example.png`  
     - Method: GET (default)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node for Second Image**  
   - Name: `Get Image 2`  
   - Type: HTTP Request  
   - Parameters: Same as `Get Image 1`  
   - Connect output of Manual Trigger to this node.

4. **Create Code Node to Rename Binary to photo1**  
   - Name: `Change name to photo1`  
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     return items.map((item, index) => {
       const buffer = item.binary.data;
       return {
         json: item.json,
         binary: {
           photo1: buffer
         }
       };
     });
     ```
   - Connect output of `Get Image 1` to this node.

5. **Create Code Node to Rename Binary to photo2**  
   - Name: `Change name to photo2`  
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     return items.map((item, index) => {
       const buffer = item.binary.data;
       return {
         json: item.json,
         binary: {
           photo2: buffer
         }
       };
     });
     ```
   - Connect output of `Get Image 2` to this node.

6. **Create Merge Node**  
   - Name: `Merge`  
   - Type: Merge  
   - Default settings (merge by position or wait for all inputs)  
   - Connect outputs of `Change name to photo1` and `Change name to photo2` to this node.

7. **Create Code Node to Merge Items into One**  
   - Name: `Send as 1 merged file`  
   - Type: Code (JavaScript)  
   - Code:
     ```javascript
     const mergedItem = { json: {}, binary: {} };
     for (const item of items) {
       for (const [key, bin] of Object.entries(item.binary || {})) {
         mergedItem.binary[key] = bin;
       }
     }
     return [mergedItem];
     ```
   - Connect output of `Merge` node to this node.

8. **Create HTTP Request Node for Instagram Upload**  
   - Name: `POST TO INSTAGRAM1`  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.upload-post.com/api/upload_photos`  
     - Method: POST  
     - Content-Type: multipart/form-data  
     - Headers:  
       - `Authorization`: `Apikey YOUR_API_KEY` (replace with your actual API key)  
     - Body Parameters:  
       - `title`: `"title-ig"` (replace or parameterize as needed)  
       - `user`: `"user_name"` (replace or parameterize as needed)  
       - `platform[]`: `"instagram"`  
       - `photos[]`: Binary data fields `photo1` and `photo2` (select as formBinaryData)  
   - Connect output of `Send as 1 merged file` to this node.

9. **Create HTTP Request Node for TikTok Upload**  
   - Name: `POST TO TIKTOK`  
   - Type: HTTP Request  
   - Parameters: Same as Instagram node except:  
     - `platform[]`: `"tiktok"`  
   - Connect output of `Send as 1 merged file` to this node.

10. **(Optional) Add Sticky Note**  
    - Name: `Sticky Note7`  
    - Content: `# POST : to Instagram`  
    - Position it near the Instagram POST node for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                         |
|---------------------------------------------------------------------------------------------------------|---------------------------------------|
| This workflow requires an API token from upload-post.com for authentication with their API endpoints.   | https://upload-post.com                |
| Images must meet platform-specific requirements: Instagram supports up to 10 images per carousel with aspect ratios 1:1 to 4:5; TikTok prefers 9:16 aspect ratio for slideshows. | Workflow description                   |
| The workflow currently uses static values for `title` and `user` in HTTP Request nodes; these should be parameterized for production use. | Workflow analysis                     |
| upload-post.com provides cross-platform posting, image optimization, and queue management for scheduled posts. | Workflow description                   |
| For testing, example images are fetched from Wikimedia Commons; replace URLs with your own image sources. | Workflow input block                   |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling reproduction, modification, and troubleshooting for advanced users and automation agents alike.