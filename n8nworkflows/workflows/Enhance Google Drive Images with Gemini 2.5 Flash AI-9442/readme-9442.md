Enhance Google Drive Images with Gemini 2.5 Flash AI

https://n8nworkflows.xyz/workflows/enhance-google-drive-images-with-gemini-2-5-flash-ai-9442


# Enhance Google Drive Images with Gemini 2.5 Flash AI

### 1. Workflow Overview

This workflow, titled **"Enhance Google Drive Images with Gemini 2.5 Flash AI"**, automates the process of enhancing product images stored in a Google Drive folder using Google’s Gemini 2.5 Flash AI model. The enhancement involves adding a realistic, clean background to product photos (specifically for diapers in the example prompt), emphasizing softness and comfort in the visual context.

The workflow logic is divided into these main blocks:

- **1.1 Initialization and Input Setup:** Manual trigger, setting the enhancement prompt, and identifying the source and destination Google Drive folders.

- **1.2 Retrieval and Filtering of Files:** Fetching files from the source folder, filtering only image files.

- **1.3 Image Processing Loop:** Iterating over each image, downloading it from Google Drive, converting it to base64 for API consumption.

- **1.4 AI Enhancement Request:** Sending the image and prompt to Google Gemini 2.5 Flash API for enhancement.

- **1.5 Response Handling and Upload:** Extracting the AI-enhanced image from the response, converting it back to a file, and uploading it to the destination folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Input Setup

- **Overview:** This block sets up the workflow starting point and defines key inputs such as the prompt for image enhancement and the source/destination folders on Google Drive.

- **Nodes Involved:**  
  - `init`  
  - `promt`  
  - `origin_folder`  
  - `destination_folder`

- **Node Details:**

  - **init**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually  
    - Config: No parameters set, simply triggers the workflow when executed  
    - Connections: Output to `promt` node  
    - Failures: None expected  

  - **promt**  
    - Type: Set  
    - Role: Defines the enhancement instructions sent to Gemini AI  
    - Config: Sets a string field `promt` with detailed instructions to add a realistic, clean background emphasizing softness (specific to diaper product photography)  
    - Key Expression: Static text prompt with explicit instructions to only output the image, no text  
    - Connections: Output to `origin_folder`  
    - Failures: None expected unless text field is empty or mis-edited  

  - **origin_folder**  
    - Type: Google Drive (fileFolder resource)  
    - Role: Retrieves the source folder metadata by folder name "imagenes_sin_procesar"  
    - Config: Query string set to folder name  
    - Credential: Google Drive OAuth2 (urbaser-folder)  
    - Connections: Output to `destination_folder`  
    - Failures: Folder not found, auth errors, or rate limits possible  

  - **destination_folder**  
    - Type: Google Drive (fileFolder resource)  
    - Role: Retrieves the destination folder metadata by folder name "imagenes_sin_procesar" (likely meant to be different, but same name here)  
    - Config: Query string set to folder name  
    - Credential: Google Drive OAuth2 (google-drive-info@innovatex)  
    - Connections: Output to `get files`  
    - Failures: Folder not found, auth errors  

#### 2.2 Retrieval and Filtering of Files

- **Overview:** Fetches files inside the source folder, maps them into a simplified JSON structure, and filters only image files.

- **Nodes Involved:**  
  - `get files`  
  - `map table`  
  - `Filter`  
  - `config-data`

- **Node Details:**

  - **get files**  
    - Type: HTTP Request  
    - Role: Calls Google Drive API to list files inside source folder by ID  
    - Config: Query parameters include folder ID from `destination_folder` (likely a miswiring, should be `origin_folder`), fields to return, and page size 1000  
    - Credential: Google Drive OAuth2 (google-drive-info@innovatex)  
    - Connections: Output to `map table`  
    - Failures: API errors, quota exceeded, invalid folder ID  

  - **map table**  
    - Type: Code (JavaScript)  
    - Role: Transforms raw file list into simplified JSON objects with essential metadata (id, name, mimeType, links, size, modifiedTime)  
    - Config: Custom JS mapping array of files from API response  
    - Connections: Output to `Filter`  
    - Failures: JS errors if input malformed or missing fields  

  - **Filter**  
    - Type: Filter  
    - Role: Filters the list to only include files where `mimeType` contains "image"  
    - Config: Condition on field `mimeType` contains "image" (case-sensitive, strict validation)  
    - Connections: Output to `config-data`  
    - Failures: Filtering errors if field missing  

  - **config-data**  
    - Type: Set  
    - Role: Prepares and enriches each file item with key fields and appends the prompt for processing  
    - Config: Sets fields `name`, `mimeType`, `id` from filtered JSON and adds `promt` from earlier `promt` node  
    - Connections: Output to `Loop Over Items`  
    - Failures: Expression errors if `promt` missing  

#### 2.3 Image Processing Loop

- **Overview:** Processes each image file individually by downloading, converting to base64, sending to AI, and preparing the enhanced image.

- **Nodes Involved:**  
  - `Loop Over Items`  
  - `download-file`  
  - `to base64`  
  - `banana-request`  
  - `map-banana-response`  
  - `to file`  
  - `upload-result`

- **Node Details:**

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iterates each file one by one (batch size default 1)  
    - Config: No special options set  
    - Connections: Main output to `download-file`, error branch empty  
    - Failures: Handling large file sets may cause slowdowns  

  - **download-file**  
    - Type: Google Drive (Download)  
    - Role: Downloads each image file by ID  
    - Config: Uses `id` from input JSON  
    - Credential: Google Drive OAuth2 (google-drive-info@innovatex)  
    - Connections: Output to `to base64`  
    - Failures: File not found, permission errors, download timeouts  

  - **to base64**  
    - Type: ExtractFromFile  
    - Role: Converts downloaded binary file to base64 encoded string for API  
    - Config: Operation set to `binaryToProperty` (output base64 in JSON)  
    - Connections: Output to `banana-request`  
    - Failures: Conversion errors if binary missing or corrupt  

  - **banana-request**  
    - Type: HTTP Request  
    - Role: Sends the image with prompt to Google Gemini 2.5 Flash API for enhancement  
    - Config:  
      - POST to Gemini endpoint  
      - JSON body includes prompt text and inline image data (mime type and base64 data)  
      - Authentication: Google Palm API credentials  
      - Retry on failure enabled  
    - Connections: Output to `map-banana-response`  
    - Failures: API quota exceeded, invalid credentials, malformed request, timeout  

  - **map-banana-response**  
    - Type: Set  
    - Role: Extracts base64 of enhanced image from API response nested structure  
    - Config: Uses expression to find first inline data in candidates parts  
    - Connections: Output to `to file`  
    - Failures: Empty or unexpected API responses, missing data  

  - **to file**  
    - Type: ConvertToFile  
    - Role: Converts base64 string back into binary file for upload  
    - Config: File name set from original downloaded file name  
    - Connections: Output to `upload-result`  
    - Failures: Conversion errors if base64 empty or corrupt  

  - **upload-result**  
    - Type: Google Drive (Upload)  
    - Role: Uploads enhanced image file to destination folder  
    - Config:  
      - Uses file binary and name from previous node  
      - Destination folder ID from `destination_folder` node  
      - Drive ID set to "My Drive"  
    - Credential: Google Drive OAuth2 (google-drive-info@innovatex)  
    - Connections: Output loops back empty (ends)  
    - Failures: Permission errors, quota limits, upload failures  

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                         | Input Node(s)             | Output Node(s)          | Sticky Note                                                 |
|--------------------|-------------------------|---------------------------------------|---------------------------|-------------------------|-------------------------------------------------------------|
| init               | Manual Trigger          | Starts workflow manually               |                           | promt                   |                                                             |
| promt              | Set                     | Defines AI prompt for image enhancement | init                      | origin_folder           | 2 – Nodes to configure: Edit prompt text here               |
| origin_folder       | Google Drive (fileFolder) | Fetches source folder metadata         | promt                     | destination_folder       | 1 – Overview: Workflow processes images from Google Drive   |
| destination_folder  | Google Drive (fileFolder) | Fetches destination folder metadata    | origin_folder             | get files               | 1 – Overview: Workflow processes images from Google Drive   |
| get files          | HTTP Request            | Lists files inside source folder       | destination_folder        | map table               | Requirements: Active Google Drive and Gemini API credentials |
| map table          | Code (JavaScript)       | Maps files into simplified JSON        | get files                 | Filter                  | Requirements: Only processes files where mimeType contains "image" |
| Filter             | Filter                  | Filters to keep only image files       | map table                 | config-data             |                                                             |
| config-data        | Set                     | Prepares data and adds prompt          | Filter                    | Loop Over Items         |                                                             |
| Loop Over Items    | SplitInBatches          | Processes files one at a time           | config-data               | download-file (main)     |                                                             |
| download-file      | Google Drive (download) | Downloads each image file               | Loop Over Items           | to base64               |                                                             |
| to base64          | ExtractFromFile         | Converts image binary to base64         | download-file             | banana-request          |                                                             |
| banana-request     | HTTP Request            | Sends image with prompt to Google Gemini API | to base64             | map-banana-response     |                                                             |
| map-banana-response | Set                     | Extracts enhanced image base64 from response | banana-request          | to file                 |                                                             |
| to file            | ConvertToFile           | Converts base64 back to binary file     | map-banana-response       | upload-result           |                                                             |
| upload-result      | Google Drive (upload)   | Uploads enhanced image to destination folder | to file                |                         |                                                             |
| Sticky Note        | Sticky Note             | Workflow overview description           |                           |                         | 1 – Overview: Workflow processes images from Google Drive   |
| Sticky Note1       | Sticky Note             | Credential and processing requirements  |                           |                         | Requirements: Active Google Drive and Gemini API credentials |
| Sticky Note2       | Sticky Note             | Configuration instructions              |                           |                         | 2 – Nodes to configure: Edit prompt and folder names        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `init`  
   - No parameters

2. **Create a Set Node for Prompt**  
   - Name: `promt`  
   - Add a string field `promt` with your Gemini enhancement instructions, e.g.:  
     "Agrega un fondo realista y limpio para un producto de pañales en la foto.|El entorno debe transmitir cuidado, suavidad y confort, resaltando al producto sin distraer.Usa fondos como: superficies de madera clara o blanca, telas suaves (algodón, manta, lino), habitaciones luminosas para bebé, Evita colores fuertes, saturados o demasiado oscuros. Mantén la atención en el pañal y que el fondo solo complemente. el producto debería ocupar el 90% del espacio."  
   - Connect `init` → `promt`

3. **Create a Google Drive Node to Get Origin Folder**  
   - Name: `origin_folder`  
   - Resource: `fileFolder`  
   - Operation: Search by folder name  
   - Query String: Set the source folder name, e.g. "imagenes_sin_procesar"  
   - Credential: Set your Google Drive OAuth2 credentials (e.g., "urbaser-folder")  
   - Connect `promt` → `origin_folder`

4. **Create a Google Drive Node to Get Destination Folder**  
   - Name: `destination_folder`  
   - Resource: `fileFolder`  
   - Operation: Search by folder name  
   - Query String: Set your target folder name (can be same or different), e.g. "imagenes_procesadas"  
   - Credential: Google Drive OAuth2 (e.g., "google-drive-info@innovatex")  
   - Connect `origin_folder` → `destination_folder`

5. **Create an HTTP Request Node to List Files**  
   - Name: `get files`  
   - Method: GET  
   - URL: `https://www.googleapis.com/drive/v3/files`  
   - Query Parameters:  
     - `q`: `'<origin_folder_ID>' in parents and trashed=false` (replace with expression from `origin_folder` node)  
     - `fields`: `nextPageToken,files(id,name,mimeType,size,modifiedTime,webViewLink,webContentLink)`  
     - `pageSize`: `1000`  
   - Credential: Google Drive OAuth2 (e.g., "google-drive-info@innovatex")  
   - Connect `destination_folder` → `get files`

6. **Create a Code Node to Map Files**  
   - Name: `map table`  
   - Language: JavaScript  
   - Code:  
     ```javascript
     const files = items[0]?.json?.files ?? [];
     return files.map(f => ({
       json: {
         id: f.id,
         name: f.name,
         mimeType: f.mimeType,
         webContentLink: f.webContentLink,
         webViewLink: f.webViewLink,
         modifiedTime: f.modifiedTime,
         size: f.size ? Number(f.size) : null,
       }
     }));
     ```  
   - Connect `get files` → `map table`

7. **Create a Filter Node to Keep Images**  
   - Name: `Filter`  
   - Condition: `mimeType` contains `image` (case sensitive)  
   - Connect `map table` → `Filter`

8. **Create a Set Node to Prepare Data**  
   - Name: `config-data`  
   - Assign fields:  
     - `name` (string) = `{{$json.name}}`  
     - `mimeType` (string) = `{{$json.mimeType}}`  
     - `id` (string) = `{{$json.id}}`  
     - `promt` (string) = `{{$('promt').item.json.promt}} + "**no devuelvas texto, solo la imagen."`  
   - Connect `Filter` → `config-data`

9. **Create a SplitInBatches Node for Looping**  
   - Name: `Loop Over Items`  
   - Default batch size (1)  
   - Connect `config-data` → `Loop Over Items`

10. **Create a Google Drive Node to Download File**  
    - Name: `download-file`  
    - Operation: Download  
    - File ID: `{{$json.id}}`  
    - Credential: Google Drive OAuth2 (e.g., "google-drive-info@innovatex")  
    - Connect `Loop Over Items` → `download-file`

11. **Create an ExtractFromFile Node to Convert to Base64**  
    - Name: `to base64`  
    - Operation: `binaryToProperty`  
    - Connect `download-file` → `to base64`

12. **Create an HTTP Request Node to Call Gemini API**  
    - Name: `banana-request`  
    - Method: POST  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent`  
    - Authentication: Predefined Google Palm API credentials  
    - Body (JSON):  
      ```json
      {
        "generationConfig": {
          "temperature": 1,
          "topP": 0.95,
          "responseModalities": ["IMAGE"]
        },
        "contents": [{
          "role": "user",
          "parts": [
            {
              "text": {{ JSON.stringify($('Loop Over Items').item.json.promt) }}
            },
            {
              "inline_data": {
                "mime_type": "{{ $json.mimeType || 'image/jpeg' }}",
                "data": "{{ $json.data }}"
              }
            }
          ]
        }]
      }
      ```  
    - Retry on fail enabled  
    - Connect `to base64` → `banana-request`

13. **Create a Set Node to Extract Base64 Response**  
    - Name: `map-banana-response`  
    - Assign `base64File` (string) set to:  
      ```javascript
      {{$json.candidates?.[0]?.content?.parts?.find(p => p.inlineData?.data)?.inlineData?.data || ''}}
      ```  
    - Include other fields  
    - Connect `banana-request` → `map-banana-response`

14. **Create a ConvertToFile Node**  
    - Name: `to file`  
    - Operation: `toBinary`  
    - Source property: `base64File`  
    - File name: `{{$('download-file').item.json.name}}`  
    - Connect `map-banana-response` → `to file`

15. **Create a Google Drive Node to Upload File**  
    - Name: `upload-result`  
    - Operation: Upload  
    - File name: `{{$binary.data.fileName}}`  
    - Drive ID: "My Drive"  
    - Folder ID: `{{$('destination_folder').item.json.id}}`  
    - Credential: Google Drive OAuth2 (e.g., "google-drive-info@innovatex")  
    - Connect `to file` → `upload-result`

16. **Loop Back**  
    - Connect `upload-result` → `Loop Over Items` (to continue processing next batch)

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                         |
|------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------|
| This workflow processes images from Google Drive using Google Gemini AI.                       | Sticky Note on workflow overview                                                       |
| Requirements: Active Google Drive and Google Gemini API credentials are necessary.             | Sticky Note1                                                                           |
| Configure prompt text, origin folder, and destination folder names before running the workflow.| Sticky Note2                                                                           |
| Google Gemini API endpoint and authentication require proper quota and permissions.            | Relevant to `banana-request` node                                                      |
| Retry on failure is enabled for the AI request to handle transient network or API issues.     | `banana-request` node setting                                                          |
| The workflow assumes folder names are unique and correctly set in `origin_folder` and `destination_folder` nodes.| Important to avoid misrouting files                                                    |
| To avoid confusion, ensure `origin_folder` and `destination_folder` point to different folders.| Best practice to prevent overwriting or circular processing                           |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, respecting all applicable content policies. It contains only legal and public data, free of illegal or offensive elements.