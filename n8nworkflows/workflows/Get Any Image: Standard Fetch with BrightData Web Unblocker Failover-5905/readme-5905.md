Get Any Image: Standard Fetch with BrightData Web Unblocker Failover

https://n8nworkflows.xyz/workflows/get-any-image--standard-fetch-with-brightdata-web-unblocker-failover-5905


# Get Any Image: Standard Fetch with BrightData Web Unblocker Failover

### 1. Workflow Overview

This workflow is designed to fetch any image URL with a failover mechanism involving BrightData’s Web Unblocker API. It targets scenarios where direct image retrieval might fail due to network restrictions, blocking, or other errors. The workflow groups into three main logical blocks:

- **1.1 Trigger and Input Setup:** Manual execution trigger and image URL assignment.
- **1.2 Primary Image Fetch:** Attempt to retrieve the image directly via HTTP request.
- **1.3 Failover Unlock and Fetch:** If the primary fetch fails, the workflow calls the BrightData Web Unblocker API to retrieve the image.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Setup

- **Overview:**  
  This block initiates the workflow manually and sets the target image URL for retrieval.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - image

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually  
    - Configuration: No parameters; triggers on user command  
    - Inputs: None  
    - Outputs: Connected to the “image” node  
    - Potential Issues: None typical, manual trigger depends on user action  

  - **image**  
    - Type: Set  
    - Role: Assigns the image URL to a workflow variable for use downstream  
    - Configuration: Sets a single string field named `image` with a static URL value `https://example.com/image.jpg`  
    - Inputs: Receives trigger from manual node  
    - Outputs: Connects to “Classic Image Getter” node  
    - Edge Cases: Ensure the URL is valid and reachable; static URL may need dynamic substitution for real use  

#### 1.2 Primary Image Fetch

- **Overview:**  
  Attempts to download the image directly from the assigned URL with a short timeout to avoid hanging.

- **Nodes Involved:**  
  - Classic Image Getter

- **Node Details:**  

  - **Classic Image Getter**  
    - Type: HTTP Request  
    - Role: Directly fetches the image from the URL stored in `$json.image`  
    - Configuration:  
      - URL: Dynamic via expression `={{ $json.image }}`  
      - Timeout: 1000 ms (1 second)  
      - Retry enabled on failure  
      - On error: Continue workflow with error output (does not stop workflow)  
    - Inputs: From “image” node  
    - Outputs:  
      - Main output (success) is empty (no next node connected)  
      - Second output (error) connected to “Unlock Image” node for failover  
    - Edge Cases:  
      - Timeout will cause failover  
      - Network errors, 404 or other HTTP errors handled by failover  
      - Large images might exceed timeout or memory limits  

#### 1.3 Failover Unlock and Fetch

- **Overview:**  
  If the direct fetch fails, this block calls BrightData’s Web Unblocker API to retrieve the image via a proxy/unblocker service.

- **Nodes Involved:**  
  - Unlock Image  
  - Sticky Note (informational)

- **Node Details:**  

  - **Unlock Image**  
    - Type: HTTP Request  
    - Role: Fetches the image through BrightData’s Web Unblocker API as a fallback  
    - Configuration:  
      - URL: `https://api.brightdata.com/request` (fixed)  
      - Method: POST  
      - Body parameters:  
        - `zone`: fixed value “web_unlocker”  
        - `url`: dynamic, same as original image URL `={{ $json.image }}`  
        - `format`: “raw” (to get raw image)  
      - Headers: Authorization with Bearer token from credential `{{BRIGHTDATA_TOKEN}}`  
      - Response configured to get full response as a file  
      - Allow unauthorized certificates enabled  
      - Retry enabled on fail  
      - On error: continue workflow (non-blocking)  
    - Inputs: From error output of “Classic Image Getter”  
    - Outputs: None connected (terminal node)  
    - Edge Cases:  
      - Invalid or expired BrightData token causes authorization errors  
      - API downtime or network issues  
      - Unexpected response formats  
      - Large image files could cause timeout or memory issues  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Provides user info about the BrightData Unlock API and signup link  
    - Content: Markdown with heading and link  
    - Position: Near “Unlock Image” node for visibility  
    - Inputs/Outputs: None  
    - Notes: Useful for users to know how to get a free trial token  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                 | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                   |
|----------------------------|--------------------|--------------------------------|----------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Start workflow manually          | None                       | image                    |                                                                                              |
| image                      | Set                | Assign image URL for processing | When clicking ‘Execute workflow’ | Classic Image Getter      |                                                                                              |
| Classic Image Getter        | HTTP Request       | Fetch image directly             | image                      | Unlock Image (on error)   |                                                                                              |
| Unlock Image               | HTTP Request       | Failover fetch via BrightData API | Classic Image Getter (error) | None                     | ## Unlock image API<br><br>[Inscription - Free Trial](https://get.brightdata.com/image-unlocker) |
| Sticky Note                | Sticky Note        | Informational note about API     | None                       | None                     | ## Unlock image API<br><br>[Inscription - Free Trial](https://get.brightdata.com/image-unlocker) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a “Manual Trigger” node. No configuration needed. This node will start the workflow execution manually.

2. **Create Set Node (image)**  
   - Add a “Set” node named `image`.  
   - Add one field assignment:  
     - Name: `image`  
     - Type: String  
     - Value: `https://example.com/image.jpg` (replace with desired image URL)  
   - Connect Manual Trigger node’s output to this node’s input.

3. **Create HTTP Request Node (Classic Image Getter)**  
   - Add an “HTTP Request” node named `Classic Image Getter`.  
   - Configure:  
     - URL: Use expression `={{ $json.image }}` to dynamically use the URL from previous node.  
     - Method: GET (default)  
     - Options: Set timeout to 1000 ms (1 second) to avoid long delays.  
     - Retry On Fail: Enable (to retry request on failure).  
     - Error Handling: Set “On Error” to “Continue” with error output.  
   - Connect the `image` node’s output to `Classic Image Getter` node’s main input.

4. **Create HTTP Request Node (Unlock Image)**  
   - Add a second “HTTP Request” node named `Unlock Image`.  
   - Configure:  
     - URL: `https://api.brightdata.com/request`  
     - Method: POST  
     - Body Parameters (as JSON parameters):  
       - `zone`: “web_unlocker”  
       - `url`: expression `={{ $json.image }}` (same image URL)  
       - `format`: “raw”  
     - Header Parameters:  
       - Key: `Authorization`  
       - Value: `Bearer {{BRIGHTDATA_TOKEN}}` (configure an environment variable or credential named `BRIGHTDATA_TOKEN` with your API token)  
     - Options:  
       - Enable “Allow Unauthorized Certificates”  
       - Response: Set to “Full Response” with “Response Format” as “File” to retrieve raw image data  
     - Retry On Fail: Enable  
     - Error Handling: Set “On Error” to “Continue”  
   - Connect the **error output** (second output) of `Classic Image Getter` node to the main input of `Unlock Image` node.

5. **Add Sticky Note**  
   - Add a “Sticky Note” node near the `Unlock Image` node for documentation.  
   - Paste the following content:  
     ```
     ## Unlock image API

     [Inscription - Free Trial](https://get.brightdata.com/image-unlocker)
     ```

6. **Credential Setup**  
   - Create or import BrightData API credentials with the token named `BRIGHTDATA_TOKEN`.  
   - Ensure the token is valid and has access to the “web_unlocker” zone.

7. **Final Checks**  
   - Confirm all connections are correct and that the workflow triggers from manual trigger → set URL → HTTP Request → fallback HTTP Request on error.  
   - Save and test with a known image URL that may or may not be blocked to verify failover.

---

### 5. General Notes & Resources

| Note Content                                                                       | Context or Link                                                       |
|------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| BrightData Web Unblocker allows bypassing network restrictions for image fetching. | https://get.brightdata.com/image-unlocker                            |
| Short timeout on direct fetch aims to reduce waiting time before failover triggers | Timeout configured to 1000 ms in “Classic Image Getter” HTTP node.  |
| Retry enabled to improve robustness on transient network errors                    | Both HTTP Request nodes have retry enabled.                          |

---

_Disclaimer: The text provided derives solely from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public._