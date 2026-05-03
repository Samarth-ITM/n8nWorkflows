Download a file and upload it to an FTP Server

https://n8nworkflows.xyz/workflows/download-a-file-and-upload-it-to-an-ftp-server-663


# Download a file and upload it to an FTP Server

### 1. Workflow Overview

This workflow automates downloading a file from a public URL and uploading it to an FTP server, then listing the contents of the target FTP directory to verify the upload. It is designed as a companion example for n8n’s FTP node documentation, demonstrating a basic file transfer use case involving HTTP download and FTP operations.

The workflow is composed of three logical blocks:

- **1.1 Input Trigger:** Manual execution trigger to start the process.
- **1.2 File Download:** HTTP Request node downloads the file from a specified URL.
- **1.3 FTP Operations:** Upload the downloaded file to an FTP server, then list the contents of the upload directory.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  Initiates the workflow manually by user action.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**

  | Node Name           | Details                                                                                       |
  |---------------------|-----------------------------------------------------------------------------------------------|
  | On clicking 'execute' | - Type: Manual Trigger node <br>- Role: Starts workflow execution on manual user trigger <br>- Configuration: Default, no parameters <br>- Inputs: None (trigger) <br>- Outputs: Triggers HTTP Request node <br>- Version: Standard <br>- Edge cases: None; manual node only runs when user clicks execute |

---

#### 1.2 File Download

- **Overview:**  
  Downloads an image file from the n8n.io website as a file stream, preparing it for FTP upload.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  | Node Name     | Details                                                                                           |
  |---------------|-------------------------------------------------------------------------------------------------|
  | HTTP Request  | - Type: HTTP Request node <br>- Role: Downloads the file from a public URL <br>- Configuration: URL set to `https://n8n.io/n8n-logo.png` <br>- Response Format: Set to `file` to download as file binary data <br>- Inputs: Triggered by Manual node <br>- Outputs: Passes downloaded file binary data to FTP upload node <br>- Version: Standard <br>- Edge cases: Possible HTTP errors (404, timeout, network issues), file size limits, invalid URL |

---

#### 1.3 FTP Operations

- **Overview:**  
  Uploads the downloaded file to a specified path on the FTP server, then lists the contents of the upload directory to confirm the upload.

- **Nodes Involved:**  
  - FTP  
  - FTP1

- **Node Details:**

  | Node Name | Details                                                                                                                                                                                                                                                                                  |
  |-----------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
  | FTP       | - Type: FTP node <br>- Role: Uploads the file received from HTTP Request node to FTP server <br>- Configuration: <br>  • Operation: Upload <br>  • Path: `/upload/n8n_logo.png` (target location on FTP) <br>- Credentials: Uses `ftp_creds` (preconfigured FTP credentials) <br>- Inputs: Receives binary file data from HTTP Request node <br>- Outputs: Passes execution to FTP1 node <br>- Version: Standard <br>- Edge cases: FTP authentication failure, permission denied, upload timeout, file path errors |
  | FTP1      | - Type: FTP node <br>- Role: Lists the contents of the `/upload/` directory on FTP server <br>- Configuration: <br>  • Operation: List <br>  • Path: `/upload/` <br>- Credentials: Uses `ftp_creds` <br>- Inputs: Triggered after FTP upload node <br>- Outputs: Final output showing directory listing <br>- Version: Standard <br>- Edge cases: FTP connection issues, permission errors, empty directory |

---

### 3. Summary Table

| Node Name          | Node Type               | Functional Role                  | Input Node(s)       | Output Node(s) | Sticky Note                                                                                  |
|--------------------|-------------------------|---------------------------------|---------------------|----------------|----------------------------------------------------------------------------------------------|
| On clicking 'execute' | Manual Trigger          | Workflow start trigger           | None                | HTTP Request   |                                                                                              |
| HTTP Request       | HTTP Request             | Downloads file from URL          | On clicking 'execute' | FTP            |                                                                                              |
| FTP                | FTP                      | Uploads downloaded file to FTP   | HTTP Request        | FTP1           |                                                                                              |
| FTP1               | FTP                      | Lists contents of FTP directory  | FTP                 | None           |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a new node of type **Manual Trigger**.  
   - Leave default settings. This node will start the workflow on manual execution.

2. **Create HTTP Request Node**  
   - Add a node of type **HTTP Request**.  
   - Set the **URL** parameter to `https://n8n.io/n8n-logo.png`.  
   - Set **Response Format** to `file` to ensure the response is downloaded as binary data.  
   - Connect the Manual Trigger node’s output to this HTTP Request node’s input.

3. **Create FTP Upload Node**  
   - Add a node of type **FTP**.  
   - Set **Operation** to `upload`.  
   - Set **Path** to `/upload/n8n_logo.png` (target upload path on FTP server).  
   - Under **Credentials**, select or create new FTP credentials (`ftp_creds`) with the server address, username, password, and any required settings.  
   - Connect the HTTP Request node’s output to this FTP node’s input.

4. **Create FTP List Node**  
   - Add another node of type **FTP**.  
   - Set **Operation** to `list`.  
   - Set **Path** to `/upload/` to list files in the upload directory.  
   - Use the same FTP credentials (`ftp_creds`).  
   - Connect the FTP upload node’s output to this FTP list node’s input.

5. **Save and Execute**  
   - Save the workflow.  
   - Click **Execute** on the Manual Trigger node to run the entire sequence: download file → upload to FTP → list directory contents.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                    |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------|
| This workflow serves as a companion example for the FTP node documentation in n8n.                 | n8n FTP node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.ftp/ |
| Ensure FTP credentials are properly set with correct permissions to avoid authentication failures. | FTP credentials setup: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base/ftp/#credentials |
| The HTTP Request node downloads a binary file; confirm the URL is accessible and the file size is manageable for your environment. |                                                                                   |