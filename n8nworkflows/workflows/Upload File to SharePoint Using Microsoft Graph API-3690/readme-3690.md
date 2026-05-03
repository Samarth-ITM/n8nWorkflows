Upload File to SharePoint Using Microsoft Graph API

https://n8nworkflows.xyz/workflows/upload-file-to-sharepoint-using-microsoft-graph-api-3690


# Upload File to SharePoint Using Microsoft Graph API

### 1. Workflow Overview

This workflow automates the process of uploading a photo to a SharePoint folder by leveraging the Microsoft Graph API. It is designed for users who want to streamline image uploads to SharePoint, such as developers and IT administrators managing digital assets.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Configuration Setup**: Initiates the workflow manually and sets sensitive authentication data.
- **1.2 Authentication**: Obtains an OAuth2 access token from Microsoft Graph API using client credentials.
- **1.3 Photo Retrieval**: Downloads a sample photo from a public URL for upload testing.
- **1.4 Destination Setup**: Defines the SharePoint folder path and file name for the upload.
- **1.5 Upload Execution**: Uploads the photo to the specified SharePoint folder using the Microsoft Graph API.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Configuration Setup

- **Overview:**  
  This block starts the workflow manually and sets the required sensitive configuration parameters (tenant ID, client ID, client secret) needed for authentication.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Set config (sensitive data)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user action (button click).  
    - Configuration: No parameters; default manual trigger.  
    - Inputs: None  
    - Outputs: Connected to "Set config (sensitive data)"  
    - Edge Cases: None typical; user must manually trigger.  

  - **Set config (sensitive data)**  
    - Type: Set  
    - Role: Stores sensitive OAuth2 credentials as workflow variables.  
    - Configuration: Sets three string variables:  
      - TENANT_ID (placeholder UUID)  
      - CLIENT_ID (placeholder UUID)  
      - CLIENT_SECRET (example secret string)  
    - Inputs: From manual trigger  
    - Outputs: Connected to "Authentication" node  
    - Edge Cases: Storing secrets in a Set node is insecure for production; recommended to use n8n credentials or vaults.  

---

#### 2.2 Authentication

- **Overview:**  
  This block obtains an OAuth2 access token from Microsoft Graph API using the client credentials grant flow.

- **Nodes Involved:**  
  - Authentication

- **Node Details:**

  - **Authentication**  
    - Type: HTTP Request  
    - Role: Sends a POST request to Microsoft identity platform token endpoint to get an access token.  
    - Configuration:  
      - URL: `https://login.microsoftonline.com/{{ TENANT_ID }}/oauth2/token` (tenant ID injected dynamically)  
      - Method: POST  
      - Content-Type: `application/x-www-form-urlencoded`  
      - Body parameters:  
        - grant_type = client_credentials  
        - client_id = CLIENT_ID from Set node  
        - client_secret = CLIENT_SECRET from Set node  
        - resource = https://graph.microsoft.com  
    - Inputs: From "Set config (sensitive data)"  
    - Outputs: Access token JSON, passed to "Get photo (for testing purposes)"  
    - Edge Cases:  
      - Authentication failure due to invalid credentials or expired secrets  
      - Network timeouts or endpoint unavailability  
      - Rate limiting by Microsoft Graph API  

---

#### 2.3 Photo Retrieval

- **Overview:**  
  Downloads a sample photo from a public URL to use as the file content for upload testing.

- **Nodes Involved:**  
  - Get photo (for testing purposes)

- **Node Details:**

  - **Get photo (for testing purposes)**  
    - Type: HTTP Request  
    - Role: Fetches a binary image file from a fixed URL.  
    - Configuration:  
      - URL: `https://fastly.picsum.photos/id/459/200/300.jpg?hmac=4Cn5sZqOdpuzOwSTs65XA75xvN-quC4t9rfYYyoTCEI`  
      - Method: GET (default)  
      - No special headers or authentication  
    - Inputs: From "Authentication" node  
    - Outputs: Binary image data passed to "Set destination"  
    - Edge Cases:  
      - URL unreachable or image removed  
      - Network errors or slow response  
      - Content type mismatch (non-image)  

---

#### 2.4 Destination Setup

- **Overview:**  
  Sets the target SharePoint folder path and file name for the upload, augmenting the binary data with these parameters.

- **Nodes Involved:**  
  - Set destination

- **Node Details:**

  - **Set destination**  
    - Type: Set  
    - Role: Adds two string variables to the workflow data: TARGET_FOLDER and FILE_NAME  
    - Configuration:  
      - TARGET_FOLDER = `/uploads/pictures from n8n`  
      - FILE_NAME = `example.jpg`  
      - Includes other fields (binary data from previous node)  
    - Inputs: From "Get photo (for testing purposes)"  
    - Outputs: Connected to "Upload photo"  
    - Edge Cases:  
      - Incorrect folder path or file name format could cause upload failure  
      - Special characters or spaces in folder/file name may require encoding  

---

#### 2.5 Upload Execution

- **Overview:**  
  Uploads the photo binary data to the specified SharePoint folder using Microsoft Graph API with the access token.

- **Nodes Involved:**  
  - Upload photo

- **Node Details:**

  - **Upload photo**  
    - Type: HTTP Request  
    - Role: Sends a PUT request to Microsoft Graph API to upload the file content.  
    - Configuration:  
      - URL: `https://graph.microsoft.com/v1.0/sites/root/drive/root:{{TARGET_FOLDER}}/{{FILE_NAME}}:/content` (folder and file name dynamically injected)  
      - Method: PUT  
      - Content-Type: `application/octet-stream` (binary data)  
      - Headers:  
        - Authorization: `Bearer {{access_token}}` (from Authentication node)  
      - Body: Binary data field named "data" (from photo retrieval)  
    - Inputs: From "Set destination"  
    - Outputs: HTTP response from Microsoft Graph API (upload result)  
    - Edge Cases:  
      - Authorization failure due to expired or invalid token  
      - Incorrect folder path causing 404 or permission denied  
      - File size limits or network interruptions  
      - API rate limiting or throttling  

---

### 3. Summary Table

| Node Name                  | Node Type          | Functional Role                          | Input Node(s)               | Output Node(s)              | Sticky Note                                                                                              |
|----------------------------|--------------------|----------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Starts the workflow manually            | None                        | Set config (sensitive data) | Workflow Overview                                                                                       |
| Set config (sensitive data) | Set                | Stores sensitive OAuth2 credentials     | When clicking ‘Test workflow’ | Authentication              | Authentication Details                                                                                  |
| Authentication             | HTTP Request       | Obtains OAuth2 access token             | Set config (sensitive data) | Get photo (for testing purposes) | Authentication Details                                                                                  |
| Get photo (for testing purposes) | HTTP Request       | Downloads sample photo for upload       | Authentication              | Set destination             | Workflow Overview                                                                                       |
| Set destination            | Set                | Sets SharePoint folder path and file name | Get photo (for testing purposes) | Upload photo                | Set Destination Details                                                                                 |
| Upload photo               | HTTP Request       | Uploads photo to SharePoint via Graph API | Set destination             | None                        | Workflow Overview                                                                                       |
| Sticky Note                | Sticky Note        | Prerequisites and permissions           | None                        | None                        | ## Prerequisites 1. [Create an application user](https://learn.microsoft.com/en-us/power-platform/admin/manage-application-users) 2. Ensure the following permissions are set: - Sites.ReadWrite.All - for SharePoint site access - Files.ReadWrite.All - for file upload operations |
| Sticky Note1               | Sticky Note        | Authentication security notes            | None                        | None                        | ## Authentication For a succesful authentication it is required to provide: - TENANT_ID - CLIENT_ID - CLIENT_SECRET --- ## Attention! For demonstration purposes and template restrictions we store these values in a 'Set' node but in production environment please ensure safety of such data via utilizing credentials, secure vault or any other safe way of storing such information. |
| Sticky Note2               | Sticky Note        | Destination folder and file name details | None                        | None                        | ## Set destination In this step we will set the destination. The destination is made of two parameters: - TARGET_FOLDER - FILE_NAME --- ### Example Let's say this is our desired file location: `https://contoso.sharepoint.com/uploads/pictures from n8n/example.jpg` Thus we will set the following: - TARGET_FOLDER = `/uploads/pictures from n8n` - FILE_NAME = `example.jpg` |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No additional configuration needed.

2. **Create a Set Node for Sensitive Data**  
   - Name: `Set config (sensitive data)`  
   - Type: Set  
   - Add three string fields:  
     - TENANT_ID: Your Azure tenant ID (e.g., `00000000-0000-0000-0000-000000000000`)  
     - CLIENT_ID: Your Azure app client ID (e.g., `00000000-0000-0000-0000-000000000000`)  
     - CLIENT_SECRET: Your Azure app client secret (e.g., `uU~8Q~THEQLIE2TX7UsecretT2g_JCADyxBxN0bx3`)  
   - Connect output of Manual Trigger to this node.

3. **Create HTTP Request Node for Authentication**  
   - Name: `Authentication`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://login.microsoftonline.com/{{ $json.TENANT_ID }}/oauth2/token`  
   - Content-Type: `application/x-www-form-urlencoded`  
   - Body Parameters (form-urlencoded):  
     - grant_type = `client_credentials`  
     - client_id = `{{ $json.CLIENT_ID }}`  
     - client_secret = `{{ $json.CLIENT_SECRET }}`  
     - resource = `https://graph.microsoft.com`  
   - Connect output of Set config node to this node.

4. **Create HTTP Request Node to Get Photo**  
   - Name: `Get photo (for testing purposes)`  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `https://fastly.picsum.photos/id/459/200/300.jpg?hmac=4Cn5sZqOdpuzOwSTs65XA75xvN-quC4t9rfYYyoTCEI`  
   - Connect output of Authentication node to this node.

5. **Create Set Node to Define Destination**  
   - Name: `Set destination`  
   - Type: Set  
   - Add two string fields:  
     - TARGET_FOLDER = `/uploads/pictures from n8n`  
     - FILE_NAME = `example.jpg`  
   - Enable "Include Other Fields" to keep binary data from previous node.  
   - Connect output of Get photo node to this node.

6. **Create HTTP Request Node to Upload Photo**  
   - Name: `Upload photo`  
   - Type: HTTP Request  
   - Method: PUT  
   - URL: `https://graph.microsoft.com/v1.0/sites/root/drive/root:{{ $json.TARGET_FOLDER }}/{{ $json.FILE_NAME }}:/content`  
   - Content-Type: `application/octet-stream`  
   - Headers:  
     - Authorization: `Bearer {{ $json.access_token }}` (access token from Authentication node)  
     - Content-Type: `application/octet-stream`  
   - Body: Send binary data field named `data` (the photo)  
   - Connect output of Set destination node to this node.

7. **Credentials Setup**  
   - Create Microsoft OAuth2 credentials or use existing credentials for Microsoft Graph API if you want to avoid storing secrets in Set node.  
   - Alternatively, keep secrets in the Set node for testing only.

8. **Test the Workflow**  
   - Click the manual trigger node's "Execute Workflow" button to run the process.  
   - Verify the photo is uploaded to the specified SharePoint folder.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                             | Context or Link                                                                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Prerequisites include creating an application user and assigning `Sites.ReadWrite.All` and `Files.ReadWrite.All` permissions in Azure AD for the app.                                                                                   | [Create an application user](https://learn.microsoft.com/en-us/power-platform/admin/manage-application-users) |
| Authentication requires providing `TENANT_ID`, `CLIENT_ID`, and `CLIENT_SECRET`. For production, store these securely using n8n credentials or vaults instead of a Set node.                                                             | Authentication Details sticky note                                                                             |
| Destination folder and file name must be set carefully to match your SharePoint folder structure. Spaces and special characters may require URL encoding.                                                                               | Set Destination Details sticky note                                                                            |
| Microsoft Graph API documentation for uploading files: https://learn.microsoft.com/en-us/graph/api/driveitem-put-content?view=graph-rest-1.0&tabs=http                                                                                  | Official Microsoft Graph API docs                                                                               |
| For testing, the photo is fetched from a public image URL (picsum.photos). Replace this URL to upload different images.                                                                                                                | Get photo (for testing purposes) node                                                                           |

---

This document fully describes the workflow logic, node configurations, and setup instructions to enable reproduction, modification, and error anticipation for automating photo uploads to SharePoint via Microsoft Graph API.