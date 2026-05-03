Compare Your n8n Version with Latest Release using n8n API

https://n8nworkflows.xyz/workflows/compare-your-n8n-version-with-latest-release-using-n8n-api-7797


# Compare Your n8n Version with Latest Release using n8n API

### 1. Workflow Overview

This workflow is designed to compare the version of your currently running n8n instance against the latest official release version of n8n. Its primary use case is for administrators or developers who want to verify if their n8n environment is up-to-date or if an upgrade is recommended.

The workflow is logically divided into the following functional blocks:

- **1.1 Setup and Credential Configuration**: Instructions and preparation for connecting securely to your own n8n instance via API.
- **1.2 Fetch Latest Official n8n Version**: Retrieves the most recent version number from the official n8n release notes webpage.
- **1.3 Fetch Your Instance Version**: Queries your own n8n instance REST API to obtain the current version running.
- **1.4 Version Parsing and Cleaning**: Extracts and cleans the version strings obtained from the release notes and your instance.
- **1.5 Version Comparison**: Compares the cleaned version strings to determine if your instance is up-to-date.

---

### 2. Block-by-Block Analysis

#### 2.1 Setup and Credential Configuration

- **Overview:**  
  This block contains instructions and a disabled node placeholder for configuring the n8n API credentials. It ensures that the workflow can authenticate and communicate with your own n8n instance securely.

- **Nodes Involved:**  
  - Sticky Note (e70dcb2b-184f-4eb1-a9e3-69b0c583d411)  
  - Sticky Note23 (1680,6544)  
  - Sticky Note55 (eec0952b-c17b-4961-85d3-92146b64d488)  
  - Set up your n8n credentials (disabled) (9b890a79-1052-4bcc-a2d3-500e85ddb511)

- **Node Details:**

  - **Sticky Note (e70dcb2b-184f-4eb1-a9e3-69b0c583d411)**  
    - Type: Sticky Note  
    - Role: Provides a high-level explanation of the workflow’s purpose and the need for API credentials.  
    - Configuration: Large note with content explaining the workflow’s goal and prerequisite of setting up n8n API credentials.  
    - Input/Output: None (informational only).

  - **Sticky Note23**  
    - Type: Sticky Note  
    - Role: Detailed step-by-step setup instructions for the API credential creation and usage, and an overview of workflow operation.  
    - Configuration: Includes instructions for copying API Key from n8n admin panel, creating credentials in n8n, and attaching them to the node.  
    - Input/Output: None.

  - **Sticky Note55**  
    - Type: Sticky Note  
    - Role: Reiterates the credential setup steps for emphasis, located close to the disabled n8n node.  
    - Input/Output: None.

  - **Set up your n8n credentials**  
    - Type: n8n Node (n8n-nodes-base.n8n)  
    - Role: Performs an authenticated GET request to your own n8n instance to fetch execution information (used to verify connectivity and credentials).  
    - Configuration:  
      - Resource: execution  
      - Operation: get  
      - Credentials: Uses the "n8n API" type credential configured with your API Key.  
      - Disabled by default in this workflow (likely for demonstration or manual enabling).  
    - Input: None  
    - Output: Passes data downstream if enabled.  
    - Failure Modes: Credential errors (invalid API key), network issues, instance offline.

---

#### 2.2 Fetch Latest Official n8n Version

- **Overview:**  
  Retrieves the latest n8n version by scraping the official release notes page.

- **Nodes Involved:**  
  - Get Most Recent n8n version (3487da8d-0334-4a25-9fb2-c025677557d6)  
  - Extract Version (71face2b-d555-41e9-ad80-0128d6b56278)  
  - Clean Value (2b506d24-badc-4af6-8d40-27a7ec3336e8)

- **Node Details:**

  - **Get Most Recent n8n version**  
    - Type: HTTP Request  
    - Role: Fetch the HTML content of https://docs.n8n.io/release-notes/  
    - Configuration:  
      - HTTP Method: GET (default)  
      - URL: https://docs.n8n.io/release-notes/  
      - No authentication or special headers set.  
    - Output: HTML content of the release notes page.  
    - Failure Modes: Network errors, URL unreachable, changes to page structure.

  - **Extract Version**  
    - Type: HTML Extract  
    - Role: Parses the HTML to find the version string.  
    - Configuration:  
      - Operation: extractHtmlContent  
      - Extraction CSS Selector: `h2:contains("n8n@")` — targets h2 elements containing the text "n8n@" which represents version headings.  
      - Output key: `versions`  
    - Input: HTML content from previous node  
    - Output: Extracted raw version string (e.g., "n8n@0.215.0#").  
    - Failure Modes: Page structure changes; selector returns no matches.

  - **Clean Value**  
    - Type: Code (JavaScript)  
    - Role: Cleans the raw version string to isolate the pure version number.  
    - Configuration:  
      - Code strips prefix "n8n@" and trailing "#" characters from the extracted string.  
      - Returns JSON object with key `version`.  
    - Input: JSON with `versions` field from Extract Version node.  
    - Output: Clean version string (e.g., "0.215.0").  
    - Failure Modes: Unexpected string format, missing input data.

---

#### 2.3 Fetch Your Instance Version

- **Overview:**  
  Queries your own n8n instance REST API to obtain the current version your instance is running.

- **Nodes Involved:**  
  - Get your n8n version (4b4ae554-2376-4b2a-9a99-4815e532666d4)

- **Node Details:**

  - **Get your n8n version**  
    - Type: HTTP Request  
    - Role: Calls the endpoint `/rest/settings` on your n8n instance to fetch instance information, including version.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: `https://yourn8nurl.app.n8n.cloud/rest/settings` (replace with your actual instance URL)  
      - Authentication: Uses predefined n8n API credentials (API Key)  
    - Input: Triggered from Clean Value node (ensures sequential flow)  
    - Output: JSON response containing `data.versionCli` field with the installed version.  
    - Failure Modes: Auth failure, incorrect URL, instance offline, API changes.

---

#### 2.4 Version Comparison

- **Overview:**  
  Compares the version string from your instance against the latest official release version to determine if they match.

- **Nodes Involved:**  
  - Test your version (4c7d9901-72b8-4e22-836e-525c24400729)

- **Node Details:**

  - **Test your version**  
    - Type: IF (Conditional) node  
    - Role: Compares two version strings for equality.  
    - Configuration:  
      - Condition: Checks if `data.versionCli` from "Get your n8n version" equals the `version` from "Clean Value".  
      - Case sensitive and strict type validation enabled.  
    - Input: Output from "Get your n8n version" node  
    - Output: Two branches: true (versions equal), false (versions different)  
    - Failure Modes: Missing fields, malformed data.

---

### 3. Summary Table

| Node Name                   | Node Type                 | Functional Role                              | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                     |
|-----------------------------|---------------------------|----------------------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note                 | Sticky Note               | Workflow overview and purpose                 | None                            | None                             | # 🔑 Test your n8n Version with HTTP Request This workflow compares your current n8n instance version with the latest release. To make it work, you need to set up your n8n API credentials. |
| Sticky Note23               | Sticky Note               | Setup instructions and workflow explanation  | None                            | None                             | ## ⚙️ Setup Instructions 1️⃣ Set Up n8n API Credentials - Detailed steps and contact info.       |
| Sticky Note55               | Sticky Note               | Credential setup emphasis                      | None                            | None                             | ### 1️⃣ Set Up n8n API Credentials - Steps to generate and attach API Key credential.           |
| Set up your n8n credentials | n8n API Request           | Get authenticated info from your instance    | None                            | None                             |                                                                                                |
| Get Most Recent n8n version | HTTP Request              | Fetch official latest n8n release notes page | None                            | Extract Version                  |                                                                                                |
| Extract Version             | HTML Extract              | Parse HTML to extract raw version string      | Get Most Recent n8n version      | Clean Value                     |                                                                                                |
| Clean Value                 | Code                      | Clean and format extracted version string    | Extract Version                 | Get your n8n version             |                                                                                                |
| Get your n8n version        | HTTP Request              | Query your own instance version via API      | Clean Value                    | Test your version               |                                                                                                |
| Test your version           | IF                        | Compare your instance version with latest    | Get your n8n version            | None                           |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note**:  
   - Name: "Sticky Note"  
   - Content: Workflow overview highlighting the purpose and API credential requirement.  
   - Position roughly top-left.

2. **Create Sticky Note for Setup Instructions**:  
   - Name: "Sticky Note23"  
   - Content: Detailed setup instructions for creating and attaching n8n API credentials, workflow logic overview, and contact info.  
   - Position next to first sticky note.

3. **Create Sticky Note for Credential Emphasis**:  
   - Name: "Sticky Note55"  
   - Content: Repeat essential steps for API Key creation and usage.  
   - Position near the n8n API node.

4. **Create n8n API Node ("Set up your n8n credentials")**:  
   - Type: n8n (n8n-nodes-base.n8n)  
   - Resource: execution  
   - Operation: get  
   - Credentials: Create new credential of type "n8n API"  
     - Obtain API Key from your instance under Admin Panel → API.  
     - Paste API Key into new credential and save.  
   - Attach the credential to this node.  
   - Initially disable this node for testing; enable if you want to verify connectivity.

5. **Create HTTP Request Node ("Get Most Recent n8n version")**:  
   - Type: HTTP Request  
   - Method: GET  
   - URL: https://docs.n8n.io/release-notes/  
   - No authentication required.

6. **Create HTML Extract Node ("Extract Version")**:  
   - Operation: extractHtmlContent  
   - Extraction Values:  
     - Key: versions  
     - CSS Selector: `h2:contains("n8n@")`  
   - Connect input from "Get Most Recent n8n version".

7. **Create Code Node ("Clean Value")**:  
   - Language: JavaScript  
   - Code:  
     ```javascript
     // Strip "n8n@" prefix and trailing "#"
     const raw = $json.versions || '';
     const clean = raw.replace(/^n8n@/, '').replace(/#$/, '');
     return [{ version: clean }];
     ```  
   - Connect input from "Extract Version".

8. **Create HTTP Request Node ("Get your n8n version")**:  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Your n8n instance REST endpoint for settings, e.g., `https://<your-n8n-instance>/rest/settings`  
   - Authentication: Use the predefined "n8n API" credential created earlier.  
   - Connect input from "Clean Value".

9. **Create IF Node ("Test your version")**:  
   - Condition Type: String equals  
   - Left Value: `{{$json.data.versionCli}}` (your instance version)  
   - Right Value: `{{$node["Clean Value"].json.version}}` (latest release version)  
   - Case-sensitive and strict validation enabled.  
   - Connect input from "Get your n8n version".

10. **Arrange Nodes and Sticky Notes**:  
    - Position nodes to reflect logical flow:  
      "Get Most Recent n8n version" → "Extract Version" → "Clean Value" → "Get your n8n version" → "Test your version".  
    - Place sticky notes nearby relevant nodes for context and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                        |
|-----------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| To enable the workflow fully, ensure you have valid API credentials with permission to access your n8n instance. | n8n Admin Panel → API                                  |
| The latest version is scraped from https://docs.n8n.io/release-notes/, so changes to that page may break parsing.| Official n8n Documentation Release Notes              |
| Contact for assistance: Robert Breen, robert@ynteractive.com; LinkedIn: https://www.linkedin.com/in/robert-breen-29429625/ | Workflow author contact information                    |
| This workflow requires the n8n version to expose the `/rest/settings` endpoint with the `versionCli` field.      | n8n REST API documentation                             |

---

**Disclaimer:**  
The content provided is extracted exclusively from an automated workflow built with n8n. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.