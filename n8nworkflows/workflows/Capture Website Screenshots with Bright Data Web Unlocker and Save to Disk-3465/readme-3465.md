Capture Website Screenshots with Bright Data Web Unlocker and Save to Disk

https://n8nworkflows.xyz/workflows/capture-website-screenshots-with-bright-data-web-unlocker-and-save-to-disk-3465


# Capture Website Screenshots with Bright Data Web Unlocker and Save to Disk

### 1. Workflow Overview

This workflow automates capturing website screenshots using the Bright Data Web Unlocker API, designed to bypass anti-bot mechanisms like Cloudflare protections. It is tailored for professionals needing reliable visual snapshots of web pages, including compliance teams, marketers, developers, and growth hackers.

The workflow is logically divided into three main blocks:

- **1.1 Input Initialization**: Defines the target URL, output filename, and Bright Data zone to configure the screenshot request parameters.
- **1.2 Screenshot Capture via Bright Data API**: Sends a POST request to the Bright Data Web Unlocker API to obtain a screenshot of the specified URL.
- **1.3 Local File Storage**: Saves the received screenshot file locally to a predefined disk path.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block sets the essential parameters required for the screenshot process: the target website URL, the output image filename, and the Bright Data zone identifier used for proxying and unlocking web content.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set URL, Filename and Bright Data Zone

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type & Role: Manual Trigger; initiates the workflow manually.  
    - Configuration: Default, no parameters.  
    - Inputs: None.  
    - Outputs: Connected to "Set URL, Filename and Bright Data Zone".  
    - Edge Cases: None significant; manual trigger ensures controlled execution.

  - **Set URL, Filename and Bright Data Zone**  
    - Type & Role: Set node; assigns static variables for URL, filename, and Bright Data zone.  
    - Configuration:  
      - `url` set to `"https://dev.to/"` (modifiable to target other websites).  
      - `filename` set to `"devto.png"` (output file name, can be dynamic).  
      - `zone` set to `"web_unlocker1"` (Bright Data zone name for proxy configuration).  
    - Key Expressions: None beyond static assignment, but values support expressions for dynamic assignment.  
    - Inputs: From Manual Trigger node.  
    - Outputs: To "Capture a screenshot" node.  
    - Edge Cases:  
      - Incorrect URL format or unreachable URL leads to API errors downstream.  
      - Invalid or misspelled Bright Data zone causes authentication or proxy failures.

---

#### 2.2 Screenshot Capture via Bright Data API

- **Overview:**  
  Sends an authenticated HTTP POST request to Bright Data’s Web Unlocker API to request a screenshot of the specified URL using the configured proxy zone. It obtains the screenshot as a raw file response.

- **Nodes Involved:**  
  - Capture a screenshot

- **Node Details:**

  - **Capture a screenshot**  
    - Type & Role: HTTP Request node; performs the API call to Bright Data Web Unlocker.  
    - Configuration:  
      - URL: `https://api.brightdata.com/request`  
      - Method: POST  
      - Body Parameters (sent as form data):  
        - `zone`: uses expression `{{$json.zone}}` from prior node  
        - `url`: uses expression `{{$json.url}}`  
        - `format`: `"raw"` (requests raw file response)  
        - `data_format`: `"screenshot"` (specifies screenshot type)  
      - Authentication: Generic HTTP Header Authentication using Bright Data token credential ("Header Auth account").  
      - Response Handling: Configured to receive the response as a file, storing it under the property named by `{{$json.filename}}`.  
      - Additional: Allows unauthorized SSL certificates (to avoid TLS errors).  
    - Inputs: From "Set URL, Filename and Bright Data Zone".  
    - Outputs: To "Write a file to disk".  
    - Edge Cases & Failure Types:  
      - Authentication failures if token is invalid or expired.  
      - Network issues or API downtime.  
      - API limitations such as rate limits or quota exceeded.  
      - Invalid zone or URL leads to API errors or empty responses.  
      - File response parsing failures if API changes response format.

---

#### 2.3 Local File Storage

- **Overview:**  
  Saves the screenshot image file received from the API call to a defined local disk path for later retrieval or processing.

- **Nodes Involved:**  
  - Write a file to disk

- **Node Details:**

  - **Write a file to disk**  
    - Type & Role: Read/Write File node; writes binary data to a local file path.  
    - Configuration:  
      - Operation: `write`  
      - Filename: expression `="c:\\" + $json.filename` (writes to root of C: drive concatenated with filename, e.g., `c:\devto.png`). This path should be customized as needed.  
      - Data Property Name: uses dynamic property named after the filename to access binary file data from previous node.  
    - Inputs: From "Capture a screenshot" node.  
    - Outputs: None (end node).  
    - Edge Cases & Failure Types:  
      - File system permissions may prevent writing to specified path.  
      - Invalid or inaccessible directory paths cause errors.  
      - Filename collisions overwrite existing files unless handled externally.  
      - Windows-specific absolute path; adjust for other OSes.  

---

### 3. Summary Table

| Node Name                        | Node Type             | Functional Role                              | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                                                                    |
|---------------------------------|-----------------------|----------------------------------------------|---------------------------------|---------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’    | Manual Trigger        | Starts workflow manually                      | -                               | Set URL, Filename and Bright Data Zone |                                                                                                                                               |
| Set URL, Filename and Bright Data Zone | Set                  | Defines URL, filename, and Bright Data zone | When clicking ‘Test workflow’    | Capture a screenshot             | The "**Set URL, Filename and Bright Data Zone**" node must be updated with the appropriate URL, filename and **Bright Data Proxies & Infrastructure** zone. |
| Capture a screenshot            | HTTP Request          | Calls Bright Data API to capture screenshot  | Set URL, Filename and Bright Data Zone | Write a file to disk            |                                                                                                                                               |
| Write a file to disk            | Read/Write File       | Saves screenshot image locally                | Capture a screenshot             | -                               | The "**Write a file to disk**" node has the location to download the website screenshot. Please make sure to set the path accordingly.           |
| Sticky Note                    | Sticky Note           | Informational notes                           | -                               | -                               | See above content                                                                                                                              |
| Sticky Note1                   | Sticky Note           | Section label                                | -                               | -                               | Website Screenshot                                                                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add node: **Manual Trigger**.  
   - No special configuration needed. This will start the workflow.

2. **Create Set Node to Define Parameters**  
   - Add node: **Set**.  
   - Name it: "Set URL, Filename and Bright Data Zone".  
   - Add three string fields:  
     - `url`: e.g., `"https://dev.to/"` (replace with your target site).  
     - `filename`: e.g., `"devto.png"` (output screenshot file name).  
     - `zone`: e.g., `"web_unlocker1"` (your Bright Data Web Unlocker zone name).  
   - Connect **Manual Trigger** output to this Set node input.

3. **Create HTTP Request Node for Screenshot Capture**  
   - Add node: **HTTP Request**.  
   - Name it: "Capture a screenshot".  
   - Set HTTP Method to POST.  
   - Set URL to `https://api.brightdata.com/request`.  
   - Under Authentication, select "Generic Credential" and configure a new credential:  
     - Type: HTTP Header Authentication.  
     - Header Name: `Authorization`.  
     - Header Value: `Bearer YOUR_BRIGHT_DATA_TOKEN` (replace with your token).  
   - Enable "Send Body" and "Send Headers".  
   - Set Body Parameters as form parameters:  
     - `zone`: expression `{{$json.zone}}` (from Set node).  
     - `url`: expression `{{$json.url}}`.  
     - `format`: `"raw"`.  
     - `data_format`: `"screenshot"`.  
   - Under Options -> Response, set:  
     - Response Format: `file`.  
     - Output Property Name: expression `{{$json.filename}}` (use the filename as the property name to store the file).  
   - Enable "Allow Unauthorized Certificates" to avoid SSL issues.  
   - Connect the output of the Set node to this HTTP Request node.

4. **Create Read/Write File Node to Save Screenshot**  
   - Add node: **Read/Write File**.  
   - Name it: "Write a file to disk".  
   - Set Operation to `write`.  
   - Set File Name to an expression: `="c:\\" + $json.filename` (adjust path as needed, e.g., use `/tmp/` on Linux/macOS).  
   - Set Data Property Name to the dynamic property named by the filename: `{{$json.filename}}`.  
   - Connect the output of the HTTP Request node to this node.

5. **Connect Nodes to Form Complete Workflow**  
   - Manual Trigger → Set URL, Filename and Bright Data Zone → Capture a screenshot → Write a file to disk.

6. **Credentials Setup**  
   - In n8n, create a new credential of type "HTTP Header Auth".  
   - Enter Header Name: `Authorization`.  
   - Enter Header Value: `Bearer YOUR_BRIGHT_DATA_TOKEN`.  
   - Save and assign this credential to the HTTP Request node.

7. **Test and Validate**  
   - Run the workflow manually using the Manual Trigger node.  
   - Verify the screenshot is saved to the specified local path.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                    | Context or Link                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------|
| Bright Data setup requires creating a Web Unlocker zone under Proxies & Scraping in your Bright Data dashboard.                                                                | https://brightdata.com/                 |
| The Bearer token must be kept secure; do not expose your Bright Data token publicly.                                                                                            | Bright Data API Authentication          |
| Customize filename with dynamic expressions (e.g., date, URL slug) to prevent overwrites and organize screenshots.                                                             | n8n Expressions docs                    |
| Screenshot storage path must have write permissions; adjust for your OS (Windows use `C:\`, Linux/macOS use `/tmp/` or other directories).                                     | OS File Permissions                     |
| To extend, integrate with cloud storage (AWS S3, Google Drive) or notification services (email, Slack) for automated alerts and archival.                                     | n8n Integration Nodes                   |
| Workflow can be scheduled or triggered via other events for automated monitoring or visual regression testing.                                                                  | n8n Scheduling and Triggers             |

---

This document fully describes the workflow structure, logic, and configurations needed for implementation and customization. It enables users and automated systems to recreate, modify, and troubleshoot the workflow efficiently.