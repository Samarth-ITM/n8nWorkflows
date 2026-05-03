Automatically Upload Scanned Documents to Nextcloud via ScanservJS

https://n8nworkflows.xyz/workflows/automatically-upload-scanned-documents-to-nextcloud-via-scanservjs-3861


# Automatically Upload Scanned Documents to Nextcloud via ScanservJS

### 1. Workflow Overview

This workflow automates the process of uploading scanned documents from a USB scanner interfaced through ScanservJS to a Nextcloud server. It is designed for environments such as home offices, small offices, or maker projects (e.g., Raspberry Pi setups) where scans need to be automatically transferred and archived without manual intervention.

The workflow consists of three primary logical blocks:

- **1.1 Scheduled Trigger:** Periodically initiates the workflow to check for new scanned documents.
- **1.2 Document Retrieval via ScanservJS API:** Requests the list of scanned files and fetches each scanned file's content.
- **1.3 Upload to Nextcloud:** Uploads the retrieved scanned files to a specific folder on the Nextcloud server.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the workflow cyclically at a set interval (default every hour) to check for new scanned documents available via ScanservJS API.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  

  - **Schedule Trigger**  
    - **Type & Role:** A time-based trigger node that initiates workflow execution on a recurring schedule.  
    - **Configuration:**  
      - Set to trigger every hour (interval with field "hours").  
      - No additional input parameters.  
    - **Inputs:** None (trigger node).  
    - **Outputs:** Initiates the HTTP Request node.  
    - **Version Requirements:** Compatible with n8n version supporting scheduleTrigger typeVersion 1.2+.  
    - **Potential Failures:** Misconfiguration of interval could cause too frequent or infrequent scans. System clock issues could affect trigger accuracy.  
    - **Notes:** Serves as the entry point for the workflow.

#### 1.2 Document Retrieval via ScanservJS API

- **Overview:**  
  This block queries the ScanservJS API to retrieve the list of scanned files. For each file listed, it performs a second HTTP request to download the actual file data.

- **Nodes Involved:**  
  - HTTP Request  
  - HTTP Request1

- **Node Details:**  

  - **HTTP Request**  
    - **Type & Role:** HTTP GET request node to fetch metadata about available scanned files.  
    - **Configuration:**  
      - URL: `http://192.168.1.100:8080/api/v1/files` (local ScanservJS API endpoint).  
      - Headers: `accept: application/json` to receive JSON metadata.  
      - No additional authentication configured, assuming local network access.  
    - **Inputs:** Triggered by Schedule Trigger.  
    - **Outputs:** Outputs JSON array of files to HTTP Request1 node.  
    - **Version Requirements:** Uses `typeVersion` 4.2; ensure n8n version supports this.  
    - **Potential Failures:**  
      - Network issues if ScanservJS is unreachable.  
      - API endpoint changes or ScanservJS service down.  
      - Empty or malformed JSON response.  
      - No authentication configured; risk if ScanservJS is exposed externally.  

  - **HTTP Request1**  
    - **Type & Role:** HTTP GET request node to download the binary content of each scanned file.  
    - **Configuration:**  
      - URL uses expression: `http://192.168.1.100:8080/api/v1/files/{{ $json.name }}` where `$json.name` is the filename from prior node.  
      - Headers: `accept: */*` to accept any file type.  
      - Response expected as binary data.  
    - **Inputs:** Receives JSON file metadata from HTTP Request node.  
    - **Outputs:** Outputs binary file data to Nextcloud node.  
    - **Version Requirements:** `typeVersion` 4.2.  
    - **Potential Failures:**  
      - File not found if filename is incorrect or deleted between list and fetch.  
      - Network timeouts or errors.  
      - Incorrect handling of binary data if content-type unknown.

#### 1.3 Upload to Nextcloud

- **Overview:**  
  This block uploads each fetched scanned file to a specified folder in Nextcloud using the Nextcloud node.

- **Nodes Involved:**  
  - Nextcloud

- **Node Details:**  

  - **Nextcloud**  
    - **Type & Role:** Specialized node to upload files directly to Nextcloud via its API.  
    - **Configuration:**  
      - Target path: `/Scans/{{ $json.name }}` — uses expression to dynamically name files based on the filename.  
      - Upload mode: binaryDataUpload enabled to send file content.  
      - Credentials: uses stored Nextcloud API credentials with write permissions.  
    - **Inputs:** Accepts binary data and metadata from HTTP Request1.  
    - **Outputs:** Workflow ends here (no further nodes).  
    - **Version Requirements:** `typeVersion` 1.  
    - **Potential Failures:**  
      - Authentication failure due to invalid or expired credentials.  
      - Permission denied errors if Nextcloud folder access is restricted.  
      - Filename conflicts or invalid characters causing upload failure.  
      - Network errors or API rate limiting.

---

### 3. Summary Table

| Node Name       | Node Type            | Functional Role                    | Input Node(s)      | Output Node(s)       | Sticky Note                                               |
|-----------------|----------------------|----------------------------------|--------------------|----------------------|-----------------------------------------------------------|
| Schedule Trigger| scheduleTrigger      | Periodic workflow initiation     | -                  | HTTP Request         |                                                           |
| HTTP Request    | httpRequest          | Fetch list of scanned files      | Schedule Trigger   | HTTP Request1        |                                                           |
| HTTP Request1   | httpRequest          | Download each scanned file       | HTTP Request      | Nextcloud            |                                                           |
| Nextcloud       | nextCloud            | Upload scanned files to Nextcloud| HTTP Request1     | -                    |                                                           |
| Sticky Note     | stickyNote           | Informational note               | -                  | -                    | ## Copy Scanner Documents to Nextcloud<br>** Needed USB-Scanner and Program ScanServJS with an API |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it appropriately (e.g., "Scans from PDF to Nextcloud").

2. **Add a Schedule Trigger node**  
   - Set the interval to trigger every hour (or desired frequency).  
   - Use the "hours" field and set interval to 1 (or customize).  
   - Connect this node as the workflow’s starting point.

3. **Add an HTTP Request node** (named "HTTP Request")  
   - Set HTTP Method: GET (default).  
   - URL: `http://192.168.1.100:8080/api/v1/files` (adjust IP and port to your ScanservJS instance).  
   - Add header: `accept` with value `application/json`.  
   - No authentication needed assuming local network access.  
   - Connect Schedule Trigger output to this node’s input.

4. **Add another HTTP Request node** (named "HTTP Request1")  
   - Set HTTP Method: GET.  
   - URL: Use an expression: `http://192.168.1.100:8080/api/v1/files/{{ $json.name }}` where `$json.name` is the filename from previous request.  
   - Add header: `accept` with value `*/*` to accept any file type.  
   - Set the response format to binary (so file data can be handled).  
   - Connect "HTTP Request" node's output to this node.

5. **Add a Nextcloud node**  
   - Configure Nextcloud credentials with API access and write permissions.  
   - Set the path to upload files to: `/Scans/{{ $json.name }}` — dynamically naming each file based on the filename field.  
   - Enable "Binary Data Upload".  
   - Connect "HTTP Request1" output to the Nextcloud node input.

6. **Add a Sticky Note (optional)**  
   - Content:  
     ```
     ## Copy Scanner Documents to Nextcloud
     ** Needed USB-Scanner and Program ScanServJS with an API
     ```  
   - Position it near the start nodes as a reminder.

7. **Activate the workflow** and test by ensuring ScanservJS is running with scans available, and Nextcloud credentials are valid.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                   |
|----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Local installation of ScanservJS is required (e.g., on Raspberry Pi) with a configured USB scanner. | Setup prerequisite for the ScanservJS API server |
| Nextcloud account must have write permissions on the target folder `/Scans`.                       | Ensure API credentials used have appropriate access rights |
| Workflow assumes ScanservJS API is accessible at `http://192.168.1.100:8080` (local network IP).   | Adjust IP and port to your environment            |
| Useful for automating paper document digitization workflows in home or small office environments.  | Ideal for Raspberry Pi maker projects and similar setups |
| Reference image screenshots included in original workflow documentation provide UI context.        | Original screenshots named Bildschrimfoto 20250504 |

---

This documentation provides a complete, clear, and actionable description of the workflow enabling users and AI agents to understand, reproduce, and troubleshoot the automatic upload of scanned documents to Nextcloud.