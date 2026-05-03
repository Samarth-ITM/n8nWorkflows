Create Deepfake Videos by Swapping Faces with Fal.ai Wan 2.2 and AWS S3

https://n8nworkflows.xyz/workflows/create-deepfake-videos-by-swapping-faces-with-fal-ai-wan-2-2-and-aws-s3-9541


# Create Deepfake Videos by Swapping Faces with Fal.ai Wan 2.2 and AWS S3

### 1. Workflow Overview

This workflow automates the creation of deepfake-style videos by swapping faces in a source video with a provided image avatar using the Fal.ai Wan 2.2 model, leveraging AWS S3 for file storage. It is designed for content creators, marketers, and developers aiming to generate animated videos with face replacements without manual video editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives user-uploaded video and image files via a form.
- **1.2 File Upload to AWS S3**: Uploads the received video and image files to an AWS S3 bucket to make them publicly accessible.
- **1.3 URL Preparation**: Constructs public URLs for the uploaded video and image to feed into the Fal.ai API.
- **1.4 Face Swap Request to Fal.ai**: Sends a POST request to Fal.ai API to start the face replacement animation process using the URLs.
- **1.5 Status Polling Loop**: Waits and periodically checks the status of the animation job until it completes.
- **1.6 Final Video Retrieval**: Once completed, fetches the final animated video from Fal.ai.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Initiates the workflow by accepting user uploads of the source video and face image via a form trigger node.

- **Nodes Involved:**  
  - On form for S3  
  - Sticky Note (form explanation)

- **Node Details:**  

  - **On form for S3**  
    - Type: Form Trigger  
    - Role: Captures user uploads of video and image files.  
    - Configuration: Form titled "Input" with two required file fields labeled "Video" and "Image".  
    - Inputs: External user submission.  
    - Outputs: Binary data of uploaded files, JSON metadata with filenames.  
    - Edge Cases: User may upload unsupported file types or no files; form validation handles required fields.  
    - No specific version requirements.  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides user instructions for the form inputs: the video for face replacement and the source image for swapping.  
    - Content: "Form Entry: 1. Upload Video - face from this video will be swapped with the 2. Image - This is the source that will be used to swap the character in the video."

#### 2.2 File Upload to AWS S3

- **Overview:**  
  Uploads the video and image files received from the form trigger to a public AWS S3 bucket for external access by the Fal.ai API.

- **Nodes Involved:**  
  - Upload Video1  
  - Upload Image1  
  - Sticky Note (public storage explanation)

- **Node Details:**  

  - **Upload Video1**  
    - Type: AWS S3  
    - Role: Uploads the video file to the "n8n-marketplace" S3 bucket.  
    - Key Config: Filename derived from the form upload metadata, binary property set to "Video".  
    - Credentials: AWS account with S3 permissions named "sandy4v AWS account".  
    - Inputs: Binary video file from form trigger.  
    - Outputs: Metadata including uploaded file info.  
    - Edge Cases: Upload failures due to credential/permission issues or network errors.  
    - Version: 2  

  - **Upload Image1**  
    - Type: AWS S3  
    - Role: Uploads the image file to the same S3 bucket.  
    - Key Config: Filename from form metadata, binary property "Image".  
    - Credentials: Same as Upload Video1.  
    - Inputs: Binary image file from form trigger.  
    - Outputs: Metadata including uploaded file info.  
    - Edge Cases: Same as Upload Video1.  
    - Version: 2  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Role: Explains that uploaded video and images are stored in a publicly accessible URL in AWS S3.

#### 2.3 URL Preparation

- **Overview:**  
  Converts the uploaded files’ S3 metadata into publicly accessible URLs required by the Fal.ai API.

- **Nodes Involved:**  
  - Edit Fields2 (Video URL)  
  - Edit Fields3 (Image URL)  

- **Node Details:**  

  - **Edit Fields2**  
    - Type: Set Node  
    - Role: Creates a new string field "Video_URL" by concatenating the S3 bucket base URL with the uploaded video filename.  
    - Expression: `https://n8n-marketplace.s3.us-east-1.amazonaws.com/{{ $('On form for S3').item.json.Video[0].filename }}`  
    - Inputs: Output from Upload Video1.  
    - Outputs: JSON with "Video_URL".  
    - Edge Cases: Filename missing or incorrect path leads to invalid URL.  
    - Version: 3.4  

  - **Edit Fields3**  
    - Type: Set Node  
    - Role: Similar to Edit Fields2, but for the "Image_URL" field.  
    - Expression: `https://n8n-marketplace.s3.us-east-1.amazonaws.com/{{ $('On form for S3').item.json.Image[0].filename }}`  
    - Inputs: Output from Upload Image1.  
    - Outputs: JSON with "Image_URL".  
    - Edge Cases: Same as Edit Fields2.  
    - Version: 3.4  

#### 2.4 Face Swap Request to Fal.ai

- **Overview:**  
  Sends a POST request to the Fal.ai API with the video and image URLs to start the face replacement animation process.

- **Nodes Involved:**  
  - Merge  
  - Fal.ai Animate Request1  
  - Sticky Note2 (Fal.ai model explanation)

- **Node Details:**  

  - **Merge**  
    - Type: Merge  
    - Role: Combines the outputs of Edit Fields2 and Edit Fields3 into a single JSON object with both URLs.  
    - Mode: Combine by position (merges items in parallel arrays by index).  
    - Inputs: Outputs of Edit Fields2 and Edit Fields3.  
    - Outputs: Combined object with "Video_URL" and "Image_URL".  
    - Version: 3.2  

  - **Fal.ai Animate Request1**  
    - Type: HTTP Request  
    - Role: Sends POST request to Fal.ai endpoint `https://queue.fal.run/fal-ai/wan/v2.2-14b/animate/replace`.  
    - Authentication: HTTP Header Auth with Fal.ai API key credential.  
    - Body Parameters: JSON with "video_url" and "image_url" from merged data.  
    - Inputs: Combined URLs from Merge node.  
    - Outputs: JSON containing job request ID and initial status.  
    - Edge Cases: HTTP errors, authentication failures, or invalid URLs may cause request failures.  
    - Version: 4.1  

  - **Sticky Note2**  
    - Type: Sticky Note  
    - Role: Explains that the Wan 2.2 model is called through the Fal.ai API and that Replicate.ai could be used alternatively.

#### 2.5 Status Polling Loop

- **Overview:**  
  Periodically checks the status of the face replacement job until the process is complete.

- **Nodes Involved:**  
  - Wait  
  - Get Video Status  
  - If  

- **Node Details:**  

  - **Wait**  
    - Type: Wait  
    - Role: Pauses the workflow for 1 minute before polling again.  
    - Parameter: Wait time = 1 minute.  
    - Inputs: Output from Fal.ai Animate Request1 or from If node if job incomplete.  
    - Outputs: Triggers Get Video Status node after delay.  
    - Edge Cases: If the wait node is interrupted or workflow is deactivated, polling stops.  
    - Version: 1.1  

  - **Get Video Status**  
    - Type: HTTP Request  
    - Role: Sends GET request to `https://queue.fal.run/fal-ai/wan/requests/{{ $json.request_id }}/status` to check job status.  
    - Authentication: Same Fal.ai HTTP header credential.  
    - Input: Job request ID from previous node.  
    - Output: JSON with current status field.  
    - Edge Cases: Network or auth errors; invalid request ID.  
    - Version: 4.1  

  - **If**  
    - Type: If  
    - Role: Conditional check whether the job status is "COMPLETED".  
    - Condition: `$json.status == "COMPLETED"`  
    - Inputs: Output from Get Video Status.  
    - Outputs:  
      - True branch: proceeds to Get Final Video node.  
      - False branch: loops back to Wait node for another delay cycle.  
    - Version: 1  

#### 2.6 Final Video Retrieval

- **Overview:**  
  Once the Fal.ai job is completed, retrieves the final animated video URL and data.

- **Nodes Involved:**  
  - Get Final Video  

- **Node Details:**  

  - **Get Final Video**  
    - Type: HTTP Request  
    - Role: Fetches the final result from `https://queue.fal.run/fal-ai/wan/requests/{{ $json.request_id }}`.  
    - Authentication: Fal.ai HTTP header credential.  
    - Input: Job request ID.  
    - Output: JSON containing the completed animated video details and URLs.  
    - Edge Cases: Request failure, incomplete data, or expired request ID.  
    - Version: 4.1  

---

### 3. Summary Table

| Node Name             | Node Type          | Functional Role                      | Input Node(s)             | Output Node(s)                | Sticky Note                                                       |
|-----------------------|--------------------|------------------------------------|---------------------------|------------------------------|------------------------------------------------------------------|
| On form for S3        | Form Trigger       | Receive user uploads (video, image) | -                         | Upload Video1, Upload Image1  | Form Entry instructions for uploads                              |
| Upload Video1          | AWS S3             | Upload video to S3 bucket           | On form for S3            | Edit Fields2                  | Public storage info                                               |
| Upload Image1          | AWS S3             | Upload image to S3 bucket           | On form for S3            | Edit Fields3                  | Public storage info                                               |
| Edit Fields2           | Set                | Generate public Video URL           | Upload Video1             | Merge                        |                                                                  |
| Edit Fields3           | Set                | Generate public Image URL           | Upload Image1             | Merge                        |                                                                  |
| Merge                  | Merge              | Combine video and image URLs        | Edit Fields2, Edit Fields3 | Fal.ai Animate Request1       |                                                                  |
| Fal.ai Animate Request1 | HTTP Request       | Initiate face swap animation job   | Merge                     | Wait                        | Wan 2.2 model via Fal.ai; Replicate.ai alternative               |
| Wait                   | Wait               | Delay between status checks         | Fal.ai Animate Request1, If (false branch) | Get Video Status          |                                                                  |
| Get Video Status       | HTTP Request       | Poll Fal.ai job status              | Wait                      | If                          |                                                                  |
| If                     | If                 | Check if job status is COMPLETED    | Get Video Status           | Get Final Video (true), Wait (false) |                                                          |
| Get Final Video        | HTTP Request       | Retrieve final animated video       | If (true)                 | -                            |                                                                  |
| Sticky Note            | Sticky Note        | Form input instructions             | -                         | -                            | "Form Entry: Upload Video and Image"                             |
| Sticky Note1           | Sticky Note        | Public storage explanation          | -                         | -                            | "Video and Images stored on a publicly accessible URL"           |
| Sticky Note2           | Sticky Note        | Fal.ai model explanation            | -                         | -                            | "Wan 2.2 model is called via HTTP node via the Fal.ai"           |
| Sticky Note3           | Sticky Note        | Enterprise AWS S3 setup info        | -                         | -                            | "Enterprise Setup: AWS S3 Bucket storage used here"              |
| Sticky Note5           | Sticky Note        | Full workflow description and setup| -                         | -                            | Detailed project description, usage, and setup instructions      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Title: "Input"  
   - Fields:  
     - File upload, label "Video", required  
     - File upload, label "Image", required  
   - This node starts the workflow on form submission with video and image files.

2. **Create AWS S3 Upload Nodes**  
   - **Upload Video1**:  
     - Type: AWS S3  
     - Operation: Upload  
     - Bucket Name: "n8n-marketplace"  
     - File Name: Expression `{{$json.Video[0].filename}}`  
     - Binary Property Name: "Video"  
     - Credentials: AWS account with S3 permissions  
     - Connect input from Form Trigger node.  
   - **Upload Image1**:  
     - Same as Upload Video1, but:  
     - File Name: `{{$json.Image[0].filename}}`  
     - Binary Property Name: "Image"  
     - Connect input from Form Trigger node.

3. **Create Set Nodes to Form Public URLs**  
   - **Edit Fields2** (Video URL):  
     - Type: Set  
     - Create string field "Video_URL" with expression:  
       `https://n8n-marketplace.s3.us-east-1.amazonaws.com/{{ $('On form for S3').item.json.Video[0].filename }}`  
     - Connect input from Upload Video1.  
   - **Edit Fields3** (Image URL):  
     - Type: Set  
     - Create string field "Image_URL" with expression:  
       `https://n8n-marketplace.s3.us-east-1.amazonaws.com/{{ $('On form for S3').item.json.Image[0].filename }}`  
     - Connect input from Upload Image1.

4. **Create Merge Node**  
   - Type: Merge  
   - Mode: Combine  
   - Combine By: Position  
   - Inputs: Connect Edit Fields2 and Edit Fields3 nodes.  
   - Outputs combined JSON containing both URLs.

5. **Create HTTP Request Node for Face Swap**  
   - Name: Fal.ai Animate Request1  
   - Method: POST  
   - URL: `https://queue.fal.run/fal-ai/wan/v2.2-14b/animate/replace`  
   - Authentication: HTTP Header Auth with Fal.ai API key credentials  
   - Body Parameters (JSON):  
     ```json
     {
       "video_url": "={{ $json.Video_URL }}",
       "image_url": "={{ $json.Image_URL }}"
     }
     ```  
   - Connect input from Merge node.

6. **Create Wait Node**  
   - Type: Wait  
   - Duration: 1 minute  
   - Connect input from Fal.ai Animate Request1 (main flow).

7. **Create HTTP Request Node for Status Polling**  
   - Name: Get Video Status  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/wan/requests/{{ $json.request_id }}/status`  
   - Authentication: Same Fal.ai HTTP Header Auth credentials  
   - Connect input from Wait node.

8. **Create If Node to Check Completion**  
   - Condition: String equals  
   - Expression: `$json.status == "COMPLETED"`  
   - Connect input from Get Video Status node.  
   - True output: Connect to Get Final Video node.  
   - False output: Loop back to Wait node.

9. **Create HTTP Request Node to Retrieve Final Video**  
   - Name: Get Final Video  
   - Method: GET  
   - URL: `https://queue.fal.run/fal-ai/wan/requests/{{ $json.request_id }}`  
   - Authentication: Same Fal.ai HTTP Header Auth credentials  
   - Connect input from If node’s true branch.

10. **Activate the Workflow**  
    - Ensure all credentials are correctly set up:  
      - AWS S3 credentials with permission to upload to "n8n-marketplace" bucket.  
      - Fal.ai API key stored as HTTP Header Auth credential.  
    - Activate the workflow to start accepting form submissions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                       |
|--------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Enterprise AWS S3 bucket is used for storing uploaded video and image files publicly accessible to Fal.ai API.                             | Sticky Note3                                                                                         |
| The workflow is designed for content creators, marketers, and developers needing automated video face replacement without complex editing. | Sticky Note5 - Detailed description and setup instructions                                          |
| Fal.ai Wan 2.2 model is used for face swapping; Replicate.ai can be an alternative choice for similar functionality.                       | Sticky Note2                                                                                         |
| Join the Skool community for n8n + AI automation tutorials and workflows: https://www.skool.com/n8n-ai-automation-champions               | Sticky Note5 (project credits and support links)                                                    |
| Videos and images are stored on publicly accessible URLs in AWS S3 bucket "n8n-marketplace" (region: us-east-1).                            | Sticky Note1                                                                                        |

---

**Disclaimer:** The provided text originates solely from an n8n automated workflow. It fully complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.