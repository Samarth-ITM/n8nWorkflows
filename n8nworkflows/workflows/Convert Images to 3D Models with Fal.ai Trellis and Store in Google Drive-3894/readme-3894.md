Convert Images to 3D Models with Fal.ai Trellis and Store in Google Drive

https://n8nworkflows.xyz/workflows/convert-images-to-3d-models-with-fal-ai-trellis-and-store-in-google-drive-3894


# Convert Images to 3D Models with Fal.ai Trellis and Store in Google Drive

### 1. Workflow Overview

This workflow automates the conversion of 2D images into 3D models using Fal.ai Trellis API and manages the complete lifecycle from image input to 3D model storage and result update. It is designed for users who want to generate 3D assets without manual modeling, leveraging AI-powered generation and cloud integration.

**Target Use Cases:**  
- Artists, designers, and developers automating 3D model creation from 2D images.  
- Batch processing of images listed in Google Sheets for scalable 3D asset generation.  
- Integration with cloud storage (Google Drive) for easy access and sharing of generated models.  

**Logical Blocks:**  
- **1.1 Trigger & Input Acquisition:** Initiates workflow manually or on schedule; retrieves new image data from Google Sheets.  
- **1.2 3D Model Generation Request:** Sends image URL to Fal.ai Trellis API to start 3D model generation.  
- **1.3 Status Polling:** Periodically checks the generation job status until completion.  
- **1.4 Result Retrieval & Storage:** Fetches the generated 3D model file, uploads it to Google Drive, and updates the Google Sheet with the download URL.  

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Acquisition

**Overview:**  
This block initiates the workflow either manually or on a schedule and retrieves the next image to process from a Google Sheet, filtering for entries without a 3D result.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger  
- Get new image (Google Sheets)  
- Set data (Set node)

**Node Details:**  

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command for testing or manual runs.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Connected to "Get new image" node.  
  - Edge Cases: No input data if triggered without data in Google Sheet.  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automates workflow execution at defined intervals (default every minute but recommended 5 minutes).  
  - Configuration: Interval set to minutes (can be customized).  
  - Inputs: None  
  - Outputs: Not connected in the provided workflow JSON but intended for automation.  
  - Edge Cases: Overlapping executions if interval too short; rate limits on API calls.  

- **Get new image**  
  - Type: Google Sheets  
  - Role: Retrieves rows from Google Sheet where the "3D RESULT" column is empty (filter applied).  
  - Configuration: Reads from a specific Google Sheet document and sheet (gid=0). Filters rows where "3D RESULT" is empty to find new images to process.  
  - Credentials: Google Sheets OAuth2 credentials required.  
  - Inputs: Trigger nodes  
  - Outputs: Connected to "Set data" node.  
  - Edge Cases: Empty sheet or no new rows; API quota limits; authentication failures.  

- **Set data**  
  - Type: Set  
  - Role: Extracts and assigns the "IMAGE" field from the Google Sheet row to a variable `image` for downstream use.  
  - Configuration: Assigns `image` = `{{$json['IMAGE']}}` (the image URL or path).  
  - Inputs: From "Get new image"  
  - Outputs: Connected to "Create 3D Image" node.  
  - Edge Cases: Missing or malformed image URLs; expression evaluation errors.  

---

#### 2.2 3D Model Generation Request

**Overview:**  
This block sends the image URL and generation parameters to the Fal.ai Trellis API to initiate the 3D model creation process.

**Nodes Involved:**  
- Create 3D Image (HTTP Request)  
- Wait 60 sec. (Wait node)

**Node Details:**  

- **Create 3D Image**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Fal.ai Trellis API with image URL and generation parameters to start 3D model creation.  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/trellis`  
    - Method: POST  
    - Body (JSON): Includes `image_url` (from `image` variable), guidance strengths, sampling steps, mesh simplification, texture size.  
    - Headers: Content-Type: application/json; Authorization header with API key (configured via HTTP Header Auth credential).  
  - Credentials: Fal.run API key via HTTP Header Auth.  
  - Inputs: From "Set data" node.  
  - Outputs: Connected to "Wait 60 sec." node.  
  - Edge Cases: API authentication failure; invalid image URL; API rate limits; network timeouts; malformed JSON body.  

- **Wait 60 sec.**  
  - Type: Wait  
  - Role: Pauses workflow for 60 seconds before polling status to allow processing time.  
  - Configuration: Wait time set to 60 seconds.  
  - Inputs: From "Create 3D Image"  
  - Outputs: Connected to "Get status" node.  
  - Edge Cases: Excessive wait time delaying workflow; insufficient wait causing premature status checks.  

---

#### 2.3 Status Polling

**Overview:**  
This block polls the Fal.ai Trellis API to check the status of the 3D model generation job until it is marked as "COMPLETED".

**Nodes Involved:**  
- Get status (HTTP Request)  
- Completed? (If node)  
- Wait 60 sec. (Wait node)

**Node Details:**  

- **Get status**  
  - Type: HTTP Request  
  - Role: Sends a GET request to Fal.ai API endpoint to retrieve the current status of the generation job using the `request_id` from "Create 3D Image".  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/trellis/requests/{{ $('Create 3D Image').item.json.request_id }}/status`  
    - Method: GET  
    - Authentication: HTTP Header Auth with Fal.run API key.  
  - Inputs: From "Wait 60 sec." node.  
  - Outputs: Connected to "Completed?" node.  
  - Edge Cases: API errors; invalid request_id; network issues; authentication errors.  

- **Completed?**  
  - Type: If  
  - Role: Checks if the status field in the response equals "COMPLETED".  
  - Configuration: Condition: `{{$json.status}} === "COMPLETED"`  
  - Inputs: From "Get status"  
  - Outputs:  
    - True branch: proceeds to "Get Url 3D image" node.  
    - False branch: loops back to "Wait 60 sec." node to retry after delay.  
  - Edge Cases: Status field missing or unexpected; infinite loops if job never completes.  

- **Wait 60 sec.** (same as above)  
  - Used here to delay repeated status polling.  

---

#### 2.4 Result Retrieval & Storage

**Overview:**  
Once the 3D model generation is completed, this block retrieves the model URL, downloads the 3D model file, uploads it to Google Drive, and updates the Google Sheet with the new model URL.

**Nodes Involved:**  
- Get Url 3D image (HTTP Request)  
- Get File 3D image (HTTP Request)  
- Upload 3D Image (Google Drive)  
- Update result (Google Sheets)

**Node Details:**  

- **Get Url 3D image**  
  - Type: HTTP Request  
  - Role: Retrieves detailed information about the completed 3D model request, including the download URL.  
  - Configuration:  
    - URL: `https://queue.fal.run/fal-ai/trellis/requests/{{ $json.request_id }}`  
    - Method: GET  
    - Authentication: HTTP Header Auth with Fal.run API key.  
  - Inputs: From "Completed?" node (true branch)  
  - Outputs: Connected to "Get File 3D image" node.  
  - Edge Cases: API errors; missing URL in response; authentication failure.  

- **Get File 3D image**  
  - Type: HTTP Request  
  - Role: Downloads the 3D model file from the URL obtained in the previous node.  
  - Configuration:  
    - URL: `{{$json.model_mesh.url}}` (dynamic URL from previous node)  
    - Method: GET  
    - No authentication needed (public URL assumed).  
  - Inputs: From "Get Url 3D image"  
  - Outputs: Connected to "Upload 3D Image" node.  
  - Edge Cases: Download failures; invalid URL; network timeouts.  

- **Upload 3D Image**  
  - Type: Google Drive  
  - Role: Uploads the downloaded 3D model file to a specified Google Drive folder.  
  - Configuration:  
    - File name: Timestamp + original file name from model metadata.  
    - Drive: "My Drive"  
    - Folder ID: Specific folder for Fal.run uploads.  
  - Credentials: Google Drive OAuth2 credentials.  
  - Inputs: From "Get File 3D image"  
  - Outputs: Connected to "Update result" node.  
  - Edge Cases: Upload failures; permission issues; quota limits.  

- **Update result**  
  - Type: Google Sheets  
  - Role: Updates the Google Sheet row corresponding to the processed image with the URL of the uploaded 3D model.  
  - Configuration:  
    - Matches row by `row_number` from "Get new image".  
    - Updates "IMAGE RESULT" column with the Google Drive URL from "Get Url 3D image".  
  - Credentials: Google Sheets OAuth2 credentials.  
  - Inputs: From "Upload 3D Image"  
  - Outputs: None (end of workflow)  
  - Edge Cases: Sheet update failures; row mismatch; API quota limits.  

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                                                      |
|-------------------------|---------------------|----------------------------------------|-----------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual start of workflow                | None                        | Get new image               |                                                                                                |
| Schedule Trigger        | Schedule Trigger    | Automated periodic start                | None                        | (Not connected in JSON)      | Recommended interval: 5 minutes                                                                |
| Get new image           | Google Sheets       | Retrieve new images without 3D result  | When clicking ‘Test workflow’ | Set data                    | See Step 1 - Google Sheet instructions with example link                                      |
| Set data                | Set                 | Assign image URL variable                | Get new image               | Create 3D Image             |                                                                                                |
| Create 3D Image         | HTTP Request       | Send image to Fal.ai Trellis API        | Set data                    | Wait 60 sec.                | API key required; set Authorization header as "Key YOURAPIKEY" (see Step 2)                    |
| Wait 60 sec.            | Wait                | Delay before status check                | Create 3D Image / Completed? | Get status / Get status     |                                                                                                |
| Get status              | HTTP Request       | Poll generation job status               | Wait 60 sec.                | Completed?                  |                                                                                                |
| Completed?              | If                  | Check if job status is "COMPLETED"      | Get status                  | Get Url 3D image / Wait 60 sec. |                                                                                                |
| Get Url 3D image        | HTTP Request       | Retrieve 3D model download URL           | Completed?                  | Get File 3D image           |                                                                                                |
| Get File 3D image       | HTTP Request       | Download 3D model file                    | Get Url 3D image            | Upload 3D Image             |                                                                                                |
| Upload 3D Image         | Google Drive        | Upload 3D model to Google Drive           | Get File 3D image           | Update result               |                                                                                                |
| Update result           | Google Sheets       | Update Google Sheet with 3D model URL    | Upload 3D Image             | None                        |                                                                                                |
| Sticky Note3            | Sticky Note         | Workflow overview and image illustration | None                        | None                        | Contains workflow description and image link                                                  |
| Sticky Note4            | Sticky Note         | Google Sheet setup instructions          | None                        | None                        | Link to Google Sheet template                                                                  |
| Sticky Note5            | Sticky Note         | Main flow execution instructions         | None                        | None                        | Advises manual or scheduled start                                                             |
| Sticky Note6            | Sticky Note         | API key setup instructions                | None                        | None                        | Link to Fal.ai signup and API key setup                                                       |
| Sticky Note7            | Sticky Note         | API key reminder                          | None                        | None                        | Reminder to set API key in "Create 3D Image" node                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’" for manual execution.  
   - Optionally, add a **Schedule Trigger** node named "Schedule Trigger" configured to run every 5 minutes for automation.  

2. **Add Google Sheets Node to Retrieve Images:**  
   - Add a **Google Sheets** node named "Get new image".  
   - Configure it to read from your Google Sheet document (use the provided template or your own).  
   - Set a filter to only retrieve rows where the "3D RESULT" column is empty.  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Connect "When clicking ‘Test workflow’" (and/or "Schedule Trigger") to this node.  

3. **Add Set Node to Extract Image URL:**  
   - Add a **Set** node named "Set data".  
   - Assign a new field `image` with the value from the Google Sheet column containing the image URL (e.g., `{{$json['IMAGE MODEL']}}` or `{{$json['IMAGE']}}` depending on your sheet).  
   - Connect "Get new image" to this node.  

4. **Add HTTP Request Node to Create 3D Image:**  
   - Add an **HTTP Request** node named "Create 3D Image".  
   - Set method to POST.  
   - Set URL to `https://queue.fal.run/fal-ai/trellis`.  
   - Set the body type to JSON and include parameters:  
     ```json
     {
       "image_url": "={{ $json.image }}",
       "ss_guidance_strength": 7.5,
       "ss_sampling_steps": 12,
       "slat_guidance_strength": 3,
       "slat_sampling_steps": 12,
       "mesh_simplify": 0.95,
       "texture_size": 1024
     }
     ```  
   - Add header `Content-Type: application/json`.  
   - Configure HTTP Header Auth credentials with your Fal.ai API key:  
     - Header Name: `Authorization`  
     - Header Value: `Key YOURAPIKEY`  
   - Connect "Set data" to this node.  

5. **Add Wait Node:**  
   - Add a **Wait** node named "Wait 60 sec." configured to wait 60 seconds.  
   - Connect "Create 3D Image" to this node.  

6. **Add HTTP Request Node to Get Status:**  
   - Add an **HTTP Request** node named "Get status".  
   - Set method to GET.  
   - URL: `https://queue.fal.run/fal-ai/trellis/requests/{{ $('Create 3D Image').item.json.request_id }}/status`  
   - Use the same HTTP Header Auth credentials as before.  
   - Connect "Wait 60 sec." to this node.  

7. **Add If Node to Check Completion:**  
   - Add an **If** node named "Completed?".  
   - Condition: Check if `{{$json.status}}` equals `"COMPLETED"`.  
   - Connect "Get status" to this node.  
   - True output connects to next step (Get Url 3D image).  
   - False output loops back to "Wait 60 sec." to retry.  

8. **Add HTTP Request Node to Get 3D Model URL:**  
   - Add an **HTTP Request** node named "Get Url 3D image".  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/trellis/requests/{{ $json.request_id }}`  
   - Use Fal.ai API key credentials.  
   - Connect "Completed?" (true output) to this node.  

9. **Add HTTP Request Node to Download 3D Model File:**  
   - Add an **HTTP Request** node named "Get File 3D image".  
   - Method: GET  
   - URL: `={{ $json.model_mesh.url }}` (dynamic from previous node)  
   - No authentication needed.  
   - Connect "Get Url 3D image" to this node.  

10. **Add Google Drive Node to Upload Model:**  
    - Add a **Google Drive** node named "Upload 3D Image".  
    - Configure to upload file content from "Get File 3D image".  
    - Set file name dynamically, e.g., `{{$now.format('yyyyLLddHHmmss')}}-{{$json.model_mesh.file_name}}`.  
    - Select your Google Drive and target folder.  
    - Authenticate with Google Drive OAuth2 credentials.  
    - Connect "Get File 3D image" to this node.  

11. **Add Google Sheets Node to Update Result:**  
    - Add a **Google Sheets** node named "Update result".  
    - Configure to update the row matching the original row number from "Get new image".  
    - Update the "IMAGE RESULT" column with the Google Drive file URL from "Get Url 3D image" node.  
    - Authenticate with Google Sheets OAuth2 credentials.  
    - Connect "Upload 3D Image" to this node.  

12. **Test and Deploy:**  
    - Run manually using "When clicking ‘Test workflow’" or enable "Schedule Trigger" for automation.  
    - Monitor logs and handle errors as needed.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                    | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow allows batch processing of images listed in Google Sheets, automating 3D model generation and storage.                                                | Workflow description                                                                                |
| Google Sheet template provided for input data structure: includes columns for IMAGE MODEL and 3D RESULT.                                                      | https://docs.google.com/spreadsheets/d/1C0Et6X3Zwr_6CxeNjhLpDwjAfIGeUvLGFawckKb0utY/edit?usp=sharing |
| API key must be obtained from Fal.ai and configured in HTTP Header Auth with header name "Authorization" and value "Key YOURAPIKEY".                           | https://fal.ai/                                                                                     |
| Recommended to run the workflow every 5 minutes for efficient processing and API rate limit management.                                                        | Sticky Note5                                                                                       |
| For support or customization, contact info@n3w.it or LinkedIn profile of author.                                                                                | mailto:info@n3w.it, https://www.linkedin.com/in/davideboizza/                                      |
| The workflow uses Google Sheets and Google Drive OAuth2 credentials that must be set up and authorized in n8n before execution.                               | Credential setup                                                                                   |
| The workflow includes retry logic with a 60-second wait between status checks to handle asynchronous processing by the Fal.ai API.                           | Status polling block                                                                               |
| Potential failure points include API authentication errors, network timeouts, empty or malformed image URLs, and Google API quota limits.                     | General error considerations                                                                       |
| The 3D model is stored in `.glb` format, suitable for web and application use.                                                                                  | Workflow overview                                                                                  |

---

This document provides a detailed and structured reference for understanding, reproducing, and maintaining the "Convert Images to 3D Models with Fal.ai Trellis and Store in Google Drive" workflow in n8n.