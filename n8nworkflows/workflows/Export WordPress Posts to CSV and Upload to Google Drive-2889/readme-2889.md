Export WordPress Posts to CSV and Upload to Google Drive

https://n8nworkflows.xyz/workflows/export-wordpress-posts-to-csv-and-upload-to-google-drive-2889


# Export WordPress Posts to CSV and Upload to Google Drive

### 1. Workflow Overview

This workflow automates exporting published WordPress posts into a CSV file and uploading that file to Google Drive. It is designed for content backup, SEO audits, and data migration tasks. The workflow is logically divided into the following blocks:

- **1.1 Manual Trigger**: Initiates the workflow on user command.
- **1.2 WordPress Data Retrieval**: Fetches all published posts from WordPress via its API.
- **1.3 Data Formatting**: Extracts and restructures key post fields for CSV compatibility.
- **1.4 CSV Conversion**: Converts the structured data into a CSV file.
- **1.5 Google Drive Upload**: Uploads the CSV file to a specified Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 2.1 Manual Trigger

- **Overview:**  
  This block starts the workflow manually, allowing the user to control when the export runs.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **Node:** When clicking ‘Test workflow’  
    - Type: Manual Trigger  
    - Configuration: No parameters; triggers workflow on manual activation.  
    - Inputs: None (start node)  
    - Outputs: Connects to “Get Wordpress Posts” node  
    - Edge Cases: None typical; user must manually trigger workflow.  
    - Version: 1

#### 2.2 WordPress Data Retrieval

- **Overview:**  
  Retrieves all published posts from the WordPress site using the WordPress API node, returning full post data.

- **Nodes Involved:**  
  - Get Wordpress Posts

- **Node Details:**  
  - **Node:** Get Wordpress Posts  
    - Type: WordPress API  
    - Configuration:  
      - Operation: getAll (fetch all posts)  
      - Options: status set to “publish” to fetch only published posts  
      - ReturnAll: true (fetch all posts without pagination limits)  
    - Credentials: Uses configured WordPress API credentials  
    - Inputs: From manual trigger  
    - Outputs: Full post data JSON objects to “Adjust Fields” node  
    - Edge Cases:  
      - API authentication failure (invalid credentials)  
      - Network timeouts or API rate limits  
      - Large data sets may cause performance delays  
    - Version: 1

#### 2.3 Data Formatting

- **Overview:**  
  Extracts and restructures the relevant post fields (ID, Title, Link, Content) into a simplified format suitable for CSV conversion.

- **Nodes Involved:**  
  - Adjust Fields  
  - Sticky Note (comment on extensibility)

- **Node Details:**  
  - **Node:** Adjust Fields  
    - Type: Set node  
    - Configuration:  
      - Assigns four fields:  
        - id: post ID (number)  
        - title: rendered post title (string)  
        - link: post URL (string)  
        - content: rendered post content (string)  
      - Uses expressions to extract nested JSON fields (e.g., `$json.title.rendered`)  
    - Inputs: From “Get Wordpress Posts”  
    - Outputs: Structured JSON with selected fields to “Convert to CSV File”  
    - Edge Cases:  
      - Missing or malformed fields in WordPress response could cause expression errors  
      - Large content fields may increase CSV size significantly  
    - Version: 3.4  
  - **Node:** Sticky Note  
    - Content: "### Adjust fields\nYou can add more fields to the CSV file by editing this node"  
    - Purpose: Guidance for users to customize CSV content

#### 2.4 CSV Conversion

- **Overview:**  
  Converts the structured JSON data into a CSV file format for export.

- **Nodes Involved:**  
  - Convert to CSV File

- **Node Details:**  
  - **Node:** Convert to CSV File  
    - Type: Convert To File  
    - Configuration: Default CSV conversion options (no custom delimiter or encoding specified)  
    - Inputs: From “Adjust Fields”  
    - Outputs: CSV file binary data to “Upload to Google Drive”  
    - Edge Cases:  
      - Data with embedded commas or line breaks may require proper escaping (handled by default)  
      - Very large datasets may cause memory or performance issues  
    - Version: 1.1

#### 2.5 Google Drive Upload

- **Overview:**  
  Uploads the generated CSV file to a specified folder in Google Drive using service account authentication.

- **Nodes Involved:**  
  - Upload to Google Drive

- **Node Details:**  
  - **Node:** Upload to Google Drive  
    - Type: Google Drive  
    - Configuration:  
      - File name: “Wordpress-Posts.csv” (fixed)  
      - Drive: “My Drive” (default Google Drive root)  
      - Folder: Root folder ("/ (Root folder)")  
      - Authentication: Service Account (recommended for automated workflows)  
    - Credentials: Configured Google API credentials with service account access  
    - Inputs: CSV file from “Convert to CSV File”  
    - Outputs: None (end node)  
    - Edge Cases:  
      - Authentication failure (invalid or expired service account credentials)  
      - Insufficient permissions to upload to the specified folder  
      - API rate limits or quota exceeded errors  
    - Version: 3

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                  |
|---------------------------|---------------------|-------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Start workflow manually        | None                        | Get Wordpress Posts         | "### Export WordPress Posts to CSV and Upload to Google Drive\n\nSteps:\n- Set your WordPress credentials in the \"Get WordPress Posts\" node\n- Set your Google Drive access in the Drive node\n- Click Test workflow" |
| Get Wordpress Posts       | WordPress API       | Fetch all published posts      | When clicking ‘Test workflow’ | Adjust Fields              |                                                                                              |
| Adjust Fields             | Set                 | Extract and format post fields | Get Wordpress Posts          | Convert to CSV File         | "### Adjust fields\nYou can add more fields to the CSV file by editing this node"            |
| Convert to CSV File       | Convert To File     | Convert JSON data to CSV file  | Adjust Fields                | Upload to Google Drive      |                                                                                              |
| Upload to Google Drive    | Google Drive        | Upload CSV file to Google Drive| Convert to CSV File          | None                       |                                                                                              |
| Sticky Note              | Sticky Note         | Instructional comment          | None                        | None                       | See above sticky notes content                                                               |
| Sticky Note1             | Sticky Note         | Instructional comment          | None                        | None                       | See above sticky notes content                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named “When clicking ‘Test workflow’”.  
   - No parameters needed. This node starts the workflow manually.

2. **Add WordPress Node**  
   - Add a **WordPress** node named “Get Wordpress Posts”.  
   - Set **Operation** to “getAll”.  
   - Under **Options**, set **Status** to “publish” to fetch only published posts.  
   - Enable **Return All** to true to fetch all posts without pagination.  
   - Connect credentials for your WordPress API (OAuth or Basic Auth as configured).  
   - Connect output of the Manual Trigger node to this node’s input.

3. **Add Set Node for Field Adjustment**  
   - Add a **Set** node named “Adjust Fields”.  
   - Configure to assign the following fields using expressions:  
     - `id` (Number): `{{$json["id"]}}`  
     - `title` (String): `{{$json["title"]["rendered"]}}`  
     - `link` (String): `{{$json["link"]}}`  
     - `content` (String): `{{$json["content"]["rendered"]}}`  
   - Connect output of “Get Wordpress Posts” to this node.

4. **Add Convert To File Node**  
   - Add a **Convert To File** node named “Convert to CSV File”.  
   - Set **File Type** to CSV (default).  
   - No additional parameters needed unless custom CSV formatting is desired.  
   - Connect output of “Adjust Fields” to this node.

5. **Add Google Drive Node**  
   - Add a **Google Drive** node named “Upload to Google Drive”.  
   - Set **Operation** to “Upload”.  
   - Set **File Name** to “Wordpress-Posts.csv”.  
   - Set **Drive** to “My Drive” (default).  
   - Set **Folder** to “root” (or specify a folder ID if desired).  
   - Set **Authentication** to “Service Account” (recommended for automated workflows).  
   - Connect your Google API credentials with service account access.  
   - Connect output of “Convert to CSV File” to this node.

6. **Add Sticky Notes (Optional)**  
   - Add sticky notes for user guidance:  
     - One near “Get Wordpress Posts” and “Upload to Google Drive” nodes explaining credential setup and usage.  
     - One near “Adjust Fields” node explaining how to add more fields to the CSV.

7. **Connect Nodes in Sequence**  
   - Manual Trigger → Get Wordpress Posts → Adjust Fields → Convert to CSV File → Upload to Google Drive.

8. **Test Workflow**  
   - Save and execute the workflow manually to verify successful export and upload.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                          |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| The workflow supports adding more WordPress post fields by editing the “Adjust Fields” node.       | See sticky note near “Adjust Fields” node.                                                              |
| Use Google Service Account credentials for seamless automated uploads without user interaction.    | Google Drive node configured with “Service Account” authentication.                                     |
| For large WordPress sites, consider pagination or batch processing to avoid timeouts or API limits.| WordPress API node supports pagination but is set to “Return All” here for simplicity.                   |
| Workflow ideal for SEO audits, content backup, and data migration tasks.                            | Described in the workflow overview and use cases section.                                               |
| Video tutorial and detailed instructions available at: https://n8n.io/workflows/1234 (example link)| Replace with actual workflow documentation or video link if available.                                  |

---

This documentation provides a complete, structured reference for understanding, reproducing, and extending the “Export WordPress Posts to CSV and Upload to Google Drive” workflow in n8n.