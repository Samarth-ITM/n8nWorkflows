YouTube Analytics Data Reporting API Integration for AI Agents

https://n8nworkflows.xyz/workflows/youtube-analytics-data-reporting-api-integration-for-ai-agents-5637


# YouTube Analytics Data Reporting API Integration for AI Agents

### 1. Workflow Overview

This workflow, titled **"YouTube Analytics Data Reporting API Integration for AI Agents,"** is designed to serve as a backend API integration layer for managing YouTube Analytics reporting jobs and reports. The workflow enables AI agents or external systems to interact programmatically with the YouTube Reporting API via a webhook trigger, facilitating automated data reporting tasks.

The logic is grouped into the following blocks:

- **1.1 Input Reception and Trigger**  
  Handles incoming API requests via an MCP (Multi-Channel Platform) trigger node designed for AI agent interaction.

- **1.2 Job Management Operations**  
  Provides HTTP API calls to list, create, delete, and retrieve YouTube reporting jobs.

- **1.3 Report Handling Operations**  
  Includes listing job reports, retrieving report metadata, and downloading report media files.

- **1.4 Report Type Discovery**  
  Enables listing available report types supported by the YouTube Reporting API.

The workflow is structured as a centralized API server for YouTube Reporting API operations, exposing multiple endpoints and HTTP requests connected to a single triggering node.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:**  
  This block receives incoming requests from AI agents or external clients via an MCP webhook trigger. It acts as the main entry point, routing requests to appropriate YouTube API operations.

- **Nodes Involved:**  
  - YouTube Reporting MCP Server  
  - Setup Instructions (sticky note)  
  - Workflow Overview (sticky note)  
  - Sticky Note (near trigger)  

- **Node Details:**  
  - **YouTube Reporting MCP Server**  
    - *Type:* MCP Trigger node (`@n8n/n8n-nodes-langchain.mcpTrigger`)  
    - *Role:* Receives webhook calls representing AI agent commands or API requests.  
    - *Configuration:* Uses a unique webhook ID to listen for incoming HTTP requests. No additional parameters configured explicitly.  
    - *Input/Output:* No input; outputs to all HTTP Request Tool nodes handling specific API calls.  
    - *Edge Cases:* Webhook connectivity issues, invalid request payloads, missing or malformed parameters from clients.  
    - *Integration:* Acts as the hub connecting AI-driven requests to YouTube Reporting API calls.

- **Sticky Notes:**  
  - "Setup Instructions" and "Workflow Overview" notes likely contain documentation or guidance for users but have no content in the JSON provided.  
  - The "Sticky Note" near the trigger node appears empty but is positioned to annotate the trigger or its usage.

---

#### 2.2 Job Management Operations

- **Overview:**  
  This block manages YouTube Reporting API jobs by listing existing jobs, creating new jobs, deleting jobs, and retrieving job details.

- **Nodes Involved:**  
  - List Jobs  
  - Create Job 1  
  - Delete Job 1  
  - Retrieve Job  

- **Node Details:**  
  Each node is an HTTP Request Tool configured to interact with specific YouTube Reporting API endpoints:

  - **List Jobs**  
    - *Type:* HTTP Request Tool  
    - *Role:* Retrieves a list of reporting jobs for a YouTube channel.  
    - *Configuration:* Uses YouTube Reporting API endpoint for listing jobs (e.g., `GET https://youtubereporting.googleapis.com/v1/jobs`).  
    - *Parameters:* Likely includes authentication via OAuth2 credentials (not explicitly shown in JSON) and query parameters such as `channelId`.  
    - *Input:* Trigger output  
    - *Output:* List of jobs data for downstream processing.

  - **Create Job 1**  
    - *Type:* HTTP Request Tool  
    - *Role:* Creates a new reporting job on YouTube Reporting API.  
    - *Configuration:* `POST` request with required payload specifying report type and other configurations.  
    - *Input:* Trigger output  
    - *Output:* Response object containing new job metadata.

  - **Delete Job 1**  
    - *Type:* HTTP Request Tool  
    - *Role:* Deletes a specified reporting job by job ID.  
    - *Configuration:* `DELETE` request targeting a job resource URL.  
    - *Input:* Trigger output  
    - *Output:* Confirmation of deletion or error message.

  - **Retrieve Job**  
    - *Type:* HTTP Request Tool  
    - *Role:* Retrieves detailed information for a specific reporting job.  
    - *Configuration:* `GET` request with job ID as a path parameter.  
    - *Input:* Trigger output  
    - *Output:* Job metadata.

- **Edge Cases:**  
  - OAuth token expiration or invalid credentials causing 401 errors.  
  - API quota limits or rate limiting (403 errors).  
  - Nonexistent job IDs causing 404 errors.  
  - Network timeouts or malformed requests.

- **Sticky Notes:**  
  - "Sticky Note2" is positioned near this block but contains no content.

---

#### 2.3 Report Handling Operations

- **Overview:**  
  This block handles retrieving reports generated by the jobs: listing available reports, fetching metadata, and downloading media content.

- **Nodes Involved:**  
  - List Job Reports  
  - Retrieve Report Metadata  
  - Download Media  

- **Node Details:**  

  - **List Job Reports**  
    - *Type:* HTTP Request Tool  
    - *Role:* Lists reports generated by a specific job.  
    - *Configuration:* `GET` request to API endpoint listing reports associated with a job ID.  
    - *Input:* Trigger output  
    - *Output:* Array of report files metadata.

  - **Retrieve Report Metadata**  
    - *Type:* HTTP Request Tool  
    - *Role:* Retrieves metadata for a specific report file.  
    - *Configuration:* `GET` request with report ID/path.  
    - *Input:* Trigger output  
    - *Output:* Metadata details such as creation time, size, download URL.

  - **Download Media**  
    - *Type:* HTTP Request Tool  
    - *Role:* Downloads the actual media content (report data file).  
    - *Configuration:* `GET` request to the media download URL.  
    - *Input:* Trigger output  
    - *Output:* Binary data of the report file.

- **Edge Cases:**  
  - Missing or expired report files causing 404 errors.  
  - Download interruptions or partial data.  
  - Large file sizes requiring attention to memory limits.  
  - Authentication failures.

- **Sticky Notes:**  
  - "Sticky Note3" is near this block but empty.

---

#### 2.4 Report Type Discovery

- **Overview:**  
  Enables clients to query the YouTube Reporting API for available report types that can be requested or created.

- **Nodes Involved:**  
  - List Report Types  

- **Node Details:**  

  - **List Report Types**  
    - *Type:* HTTP Request Tool  
    - *Role:* Fetches a list of supported report types from the YouTube Reporting API.  
    - *Configuration:* `GET` request to the report types endpoint.  
    - *Input:* Trigger output  
    - *Output:* Array of valid report types with descriptions.

- **Edge Cases:**  
  - API changes or deprecations affecting available report types.  
  - Authentication and authorization errors.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                             | Input Node(s)             | Output Node(s)            | Sticky Note                         |
|-------------------------|----------------------------------|---------------------------------------------|---------------------------|---------------------------|-----------------------------------|
| Setup Instructions      | Sticky Note                      | Documentation / Guidance                     | —                         | —                         | (Empty)                           |
| Workflow Overview       | Sticky Note                      | Documentation / Guidance                     | —                         | —                         | (Empty)                           |
| YouTube Reporting MCP Server | MCP Trigger (`mcpTrigger`)    | Entry point webhook for AI agent requests   | —                         | List Jobs, Create Job 1, Delete Job 1, Retrieve Job, List Job Reports, Retrieve Report Metadata, Download Media, List Report Types | (Empty)                           |
| Sticky Note             | Sticky Note                      | Annotation near trigger                       | —                         | —                         | (Empty)                           |
| List Jobs               | HTTP Request Tool                | Lists YouTube reporting jobs                  | YouTube Reporting MCP Server | Create Job 1, Delete Job 1, Retrieve Job | (Empty)                           |
| Create Job 1            | HTTP Request Tool                | Creates a new YouTube reporting job           | YouTube Reporting MCP Server | Delete Job 1, Retrieve Job | (Empty)                           |
| Delete Job 1            | HTTP Request Tool                | Deletes a specified reporting job             | YouTube Reporting MCP Server | Retrieve Job               | (Empty)                           |
| Retrieve Job            | HTTP Request Tool                | Retrieves metadata for a specific job         | YouTube Reporting MCP Server | List Job Reports           | (Empty)                           |
| List Job Reports        | HTTP Request Tool                | Lists reports generated by a job              | YouTube Reporting MCP Server | Retrieve Report Metadata   | (Empty)                           |
| Retrieve Report Metadata | HTTP Request Tool               | Retrieves metadata for a specific report      | YouTube Reporting MCP Server | Download Media             | (Empty)                           |
| Download Media          | HTTP Request Tool                | Downloads report file media content            | YouTube Reporting MCP Server | —                         | (Empty)                           |
| Sticky Note2            | Sticky Note                      | Annotation near Job Management operations     | —                         | —                         | (Empty)                           |
| Sticky Note3            | Sticky Note                      | Annotation near Report Handling operations    | —                         | —                         | (Empty)                           |
| List Report Types       | HTTP Request Tool                | Lists available YouTube report types          | YouTube Reporting MCP Server | —                         | (Empty)                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n named** "YouTube Reporting API MCP Server."

2. **Add an MCP Trigger node:**  
   - Node name: `YouTube Reporting MCP Server`  
   - Select the MCP Trigger node type (`@n8n/n8n-nodes-langchain.mcpTrigger`).  
   - Configure a webhook ID (auto-generated or custom) for receiving incoming requests.  
   - No additional parameters needed. This node will be the main entry point.

3. **Add HTTP Request Tool nodes for YouTube Reporting API operations:**  
   For all HTTP Request nodes, configure OAuth2 credentials for Google APIs with sufficient scopes to access YouTube Reporting API.

   - **List Jobs:**  
     - Method: GET  
     - URL: `https://youtubereporting.googleapis.com/v1/jobs`  
     - Authentication: OAuth2 (Google)  
     - Query Parameters: As needed, e.g., `channelId`  
     - Connect input from `YouTube Reporting MCP Server`.

   - **Create Job 1:**  
     - Method: POST  
     - URL: `https://youtubereporting.googleapis.com/v1/jobs`  
     - Body Parameters: JSON specifying report type, e.g., `{ "reportType": "channel_basic_a2" }`  
     - Authentication: OAuth2  
     - Connect input from `YouTube Reporting MCP Server`.

   - **Delete Job 1:**  
     - Method: DELETE  
     - URL: `https://youtubereporting.googleapis.com/v1/jobs/{{ $json["jobId"] }}` (use expression to inject job ID)  
     - Authentication: OAuth2  
     - Connect input from `YouTube Reporting MCP Server`.

   - **Retrieve Job:**  
     - Method: GET  
     - URL: `https://youtubereporting.googleapis.com/v1/jobs/{{ $json["jobId"] }}`  
     - Authentication: OAuth2  
     - Connect input from `YouTube Reporting MCP Server`.

4. **Add HTTP Request Tool nodes for report operations:**

   - **List Job Reports:**  
     - Method: GET  
     - URL: `https://youtubereporting.googleapis.com/v1/jobs/{{ $json["jobId"] }}/reports`  
     - Authentication: OAuth2  
     - Connect input from `YouTube Reporting MCP Server`.

   - **Retrieve Report Metadata:**  
     - Method: GET  
     - URL: `https://youtubereporting.googleapis.com/v1/media/{{ $json["reportId"] }}`  
     - Authentication: OAuth2  
     - Connect input from `YouTube Reporting MCP Server`.

   - **Download Media:**  
     - Method: GET  
     - URL: Use the download URL from report metadata or `{{ $json["downloadUrl"] }}`  
     - Authentication: OAuth2 or appropriate method to access media  
     - Set response format to binary (download file)  
     - Connect input from `YouTube Reporting MCP Server`.

5. **Add HTTP Request Tool node for listing report types:**

   - **List Report Types:**  
     - Method: GET  
     - URL: `https://youtubereporting.googleapis.com/v1/reportTypes`  
     - Authentication: OAuth2  
     - Connect input from `YouTube Reporting MCP Server`.

6. **Add Sticky Note nodes for documentation or guidance as needed:**

   - Create notes labeled:  
     - `Setup Instructions`  
     - `Workflow Overview`  
     - `Sticky Note` near the trigger node  
     - `Sticky Note2` near job management nodes  
     - `Sticky Note3` near report handling nodes

7. **Connect all HTTP Request nodes as outputs from the MCP Trigger node.**

8. **Configure Credentials:**

   - Set up Google OAuth2 credentials in n8n with scopes:  
     - `https://www.googleapis.com/auth/yt-analytics.readonly`  
     - `https://www.googleapis.com/auth/yt-analytics-monetary.readonly`  
     - `https://www.googleapis.com/auth/youtubereporting`  

9. **Set workflow timezone:** America/New_York (adjust as needed).

10. **Test each API call independently through the trigger to verify proper authentication and data retrieval.**

---

### 5. General Notes & Resources

| Note Content                                                      | Context or Link                                                |
|------------------------------------------------------------------|---------------------------------------------------------------|
| YouTube Reporting API documentation: https://developers.google.com/youtube/reporting/v1/reports | Official API reference for report types and job management    |
| OAuth2 Setup for Google APIs: https://developers.google.com/identity/protocols/oauth2 | Guidance on configuring OAuth2 credentials for Google services |
| MCP Trigger Node documentation: https://docs.n8n.io/nodes/n8n-nodes-langchain/mcp-trigger/ | Details about using MCP Trigger in n8n workflows              |
| Workflow timezone is set to America/New_York                     | Important for scheduling and timestamp consistency             |

---

**Disclaimer:** The provided text is exclusively extracted from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.