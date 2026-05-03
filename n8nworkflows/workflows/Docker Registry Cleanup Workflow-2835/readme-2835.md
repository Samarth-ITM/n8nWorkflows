Docker Registry Cleanup Workflow

https://n8nworkflows.xyz/workflows/docker-registry-cleanup-workflow-2835


# Docker Registry Cleanup Workflow

### 1. Workflow Overview

This workflow automates the cleanup of old Docker image tags in a Docker Registry v2, helping maintain registry hygiene and save storage space. It is designed for Docker Registry administrators who want to automatically delete outdated image tags while preserving recent and important tags, and to perform garbage collection on the registry server.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Image Listing:** Periodically triggers the workflow and fetches the list of all Docker images in the registry.
- **1.2 Tag Retrieval & Processing:** For each image, retrieves all tags, splits them, filters valid tags, fetches detailed manifest information, and enriches tag data.
- **1.3 Tag Grouping & Sorting:** Groups tags by image and sorts them by creation date to prepare for retention policy application.
- **1.4 Tag Retention & Deletion Identification:** Applies retention policy to identify which tags to keep and which to delete.
- **1.5 Tag Deletion & Notification:** Deletes identified old tags and sends success or failure email notifications.
- **1.6 Garbage Collection Execution:** Runs a garbage collection command on the Docker registry server via SSH to clean up unreferenced data.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Image Listing

**Overview:**  
This block initiates the workflow on a schedule and retrieves the list of all images from the Docker registry.

**Nodes Involved:**  
- Scheduled Trigger  
- List Images  
- Extract Image Names

**Node Details:**

- **Scheduled Trigger**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow daily at 1 AM (configurable)  
  - Configuration: Interval set to trigger at hour 1 daily  
  - Inputs: None  
  - Outputs: Triggers "List Images" node  
  - Edge Cases: Missed triggers if n8n is down; time zone considerations  

- **List Images**  
  - Type: HTTP Request  
  - Role: Fetches the list of all repositories/images from the Docker registry API endpoint `/v2/_catalog`  
  - Configuration: Uses HTTP Basic Auth credentials; URL requires replacing `<<your-registry-url>>` with actual registry URL  
  - Inputs: Trigger from Scheduled Trigger  
  - Outputs: JSON list of image names to "Extract Image Names"  
  - Edge Cases: Authentication failure, network errors, empty catalog response  

- **Extract Image Names**  
  - Type: Code (JavaScript)  
  - Role: Transforms the raw list of repositories into individual items with an `image` property for downstream processing  
  - Configuration: Maps `repositories` array to separate items  
  - Inputs: JSON from "List Images"  
  - Outputs: One item per image name to "Retrieve Image Tags"  
  - Edge Cases: Empty or malformed response from registry  

---

#### 1.2 Tag Retrieval & Processing

**Overview:**  
Retrieves tags for each image, splits tags into individual items, filters out invalid tags, fetches manifest digests, and enriches tag data with creation dates and digests.

**Nodes Involved:**  
- Retrieve Image Tags  
- Split Tags  
- Filter Valid Tags  
- Fetch Manifest Digest  
- Fetch Manifest Digest for Blob  
- Update Fields

**Node Details:**

- **Retrieve Image Tags**  
  - Type: HTTP Request  
  - Role: Fetches all tags for a given image from `/v2/{image}/tags/list`  
  - Configuration: HTTP Basic Auth; URL dynamically constructed per image  
  - Inputs: Image names from "Extract Image Names"  
  - Outputs: JSON with tags array to "Split Tags"  
  - Edge Cases: Empty tags array, authentication errors  

- **Split Tags**  
  - Type: Split Out  
  - Role: Splits the tags array into individual items, each representing a single tag with the image name  
  - Configuration: Splits `tags` field, includes `name` field  
  - Inputs: Tags list from "Retrieve Image Tags"  
  - Outputs: One item per tag to "Filter Valid Tags"  
  - Edge Cases: Empty tags array  

- **Filter Valid Tags**  
  - Type: Filter  
  - Role: Filters out any items where tag or image name is empty or missing  
  - Configuration: Conditions ensure `tag` and `name` are not empty strings  
  - Inputs: Individual tags from "Split Tags"  
  - Outputs: Valid tags to "Fetch Manifest Digest"  
  - Edge Cases: Tags with empty names, malformed data  

- **Fetch Manifest Digest**  
  - Type: HTTP Request  
  - Role: Fetches the manifest digest for each tag from `/v2/{image}/manifests/{tag}` with Accept headers for Docker and OCI manifest types  
  - Configuration: HTTP Basic Auth; full HTTP response requested to access headers  
  - Inputs: Valid tags from "Filter Valid Tags"  
  - Outputs: Manifest response (including headers) to "Fetch Manifest Digest for Blob"  
  - Edge Cases: Manifest not found, authentication errors, timeout  

- **Fetch Manifest Digest for Blob**  
  - Type: HTTP Request  
  - Role: Fetches blob data for the config digest extracted from previous manifest response, to obtain creation date metadata  
  - Configuration: HTTP Basic Auth; URL constructed using config digest from previous node  
  - Inputs: Manifest data from "Fetch Manifest Digest"  
  - Outputs: Blob JSON containing creation date to "Update Fields"  
  - Edge Cases: Blob not found, malformed digest, authentication errors  

- **Update Fields**  
  - Type: Set  
  - Role: Combines and sets key fields: image name, tag, creation date (from blob), and manifest digest (from headers) for downstream processing  
  - Configuration: Uses expressions to pull data from previous nodes  
  - Inputs: Blob data from "Fetch Manifest Digest for Blob" and context from "Filter Valid Tags" and "Fetch Manifest Digest"  
  - Outputs: Enriched tag data to "Sort by Creation Date"  
  - Edge Cases: Missing or invalid creation date, missing digest header  

---

#### 1.3 Tag Grouping & Sorting

**Overview:**  
Groups enriched tags by image and sorts each group’s tags by creation date descending to prepare for retention policy application.

**Nodes Involved:**  
- Sort by Creation Date  
- Group Tags by Image

**Node Details:**

- **Sort by Creation Date**  
  - Type: Sort  
  - Role: Sorts all tag items by their `created` field in descending order (newest first)  
  - Configuration: Sort field set to `created`, order descending  
  - Inputs: Enriched tag data from "Update Fields"  
  - Outputs: Sorted tags to "Group Tags by Image"  
  - Edge Cases: Missing or invalid creation dates may cause sorting issues  

- **Group Tags by Image**  
  - Type: Code (JavaScript)  
  - Role: Groups sorted tags by their image name, producing one item per image with an array of tags (each tag includes tag name, creation date, digest)  
  - Configuration: Aggregates items into an object keyed by image name, then maps to output array  
  - Inputs: Sorted tags from "Sort by Creation Date"  
  - Outputs: Grouped tags per image to "Identify Tags to Remove"  
  - Edge Cases: Images with no tags, inconsistent data  

---

#### 1.4 Tag Retention & Deletion Identification

**Overview:**  
Applies retention policy to keep the latest 10 tags plus the `latest` tag per image, and identifies tags to delete.

**Nodes Involved:**  
- Identify Tags to Remove

**Node Details:**

- **Identify Tags to Remove**  
  - Type: Code (JavaScript)  
  - Role: For each image, preserves the `latest` tag and the 10 most recent tags by creation date, marks older tags for deletion  
  - Configuration:  
    - Checks if `latest` tag exists and preserves it  
    - Sorts tags excluding `latest` by creation date descending  
    - Keeps first 10 tags, deletes the rest  
  - Inputs: Grouped tags per image from "Group Tags by Image"  
  - Outputs: List of tags to delete (with image and tag name) to "Remove Old Tags" and "Send Notification Email"  
  - Edge Cases: Images with fewer than 10 tags, missing `latest` tag, tags without creation dates  

---

#### 1.5 Tag Deletion & Notification

**Overview:**  
Deletes identified old tags from the registry and sends email notifications about success or failure.

**Nodes Involved:**  
- Remove Old Tags  
- Send Notification Email  
- Send Failure Notification Email

**Node Details:**

- **Remove Old Tags**  
  - Type: HTTP Request  
  - Role: Sends DELETE requests to the registry API to remove manifests by digest for each tag marked for deletion  
  - Configuration:  
    - URL constructed as `/v2/{image}/manifests/{digest}`  
    - HTTP Basic Auth  
    - Accept header set to Docker manifest v2 media type  
    - On error, continues workflow to allow failure notification  
  - Inputs: Tags to delete from "Identify Tags to Remove"  
  - Outputs: On success, triggers "Execute Garbage Collection" and "Send Notification Email"; on failure triggers "Send Failure Notification Email"  
  - Edge Cases: Authorization errors, manifest not found, network timeouts, partial failures  

- **Send Notification Email**  
  - Type: Email Send  
  - Role: Sends success notification emails for each deleted tag  
  - Configuration:  
    - Subject: "Docker Registry Cleaner Notification"  
    - To/From emails configured (replace with real addresses)  
    - Email body includes image and tag info  
  - Inputs: Successful deletions from "Remove Old Tags"  
  - Outputs: None  
  - Edge Cases: SMTP configuration errors, email delivery failures  

- **Send Failure Notification Email**  
  - Type: Email Send  
  - Role: Sends failure notification emails when tag deletion fails  
  - Configuration:  
    - Subject: "[FAIL] Docker Registry Cleaner Notification"  
    - To/From emails configured  
    - Email body includes image and tag info with failure notice  
  - Inputs: Failed deletions from "Remove Old Tags"  
  - Outputs: None  
  - Edge Cases: SMTP errors, email delivery failures  

---

#### 1.6 Garbage Collection Execution

**Overview:**  
Executes a garbage collection command on the Docker registry server via SSH to remove untagged blobs and free storage.

**Nodes Involved:**  
- Execute Garbage Collection

**Node Details:**

- **Execute Garbage Collection**  
  - Type: SSH  
  - Role: Runs the Docker registry garbage collection command on the server to clean untagged blobs  
  - Configuration:  
    - Working directory: `/opt/services/`  
    - Command: `docker compose exec -it -u root registry bin/registry garbage-collect --delete-untagged /etc/docker/registry/config.yml`  
    - Authentication: SSH Private Key credential  
  - Inputs: Triggered after successful tag deletions  
  - Outputs: None  
  - Edge Cases: SSH connection failures, command errors, registry in read-only mode, maintenance mode requirements  

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                         | Input Node(s)             | Output Node(s)                          | Sticky Note                                                                                      |
|-----------------------------|---------------------|---------------------------------------|---------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| Scheduled Trigger            | Schedule Trigger    | Initiates workflow on schedule        | None                      | List Images                            | Change schedule frequency here                                                                 |
| List Images                 | HTTP Request        | Fetches list of images from registry  | Scheduled Trigger         | Extract Image Names                    | Replace `<<your-registry-url>>` with your registry URL                                        |
| Extract Image Names         | Code                | Extracts image names from catalog     | List Images               | Retrieve Image Tags                    |                                                                                                |
| Retrieve Image Tags         | HTTP Request        | Retrieves tags for each image         | Extract Image Names       | Split Tags                            |                                                                                                |
| Split Tags                 | Split Out           | Splits tags array into individual tags| Retrieve Image Tags       | Filter Valid Tags                     |                                                                                                |
| Filter Valid Tags          | Filter              | Filters out invalid or empty tags     | Split Tags                | Fetch Manifest Digest                  |                                                                                                |
| Fetch Manifest Digest      | HTTP Request        | Fetches manifest digest for tag       | Filter Valid Tags         | Fetch Manifest Digest for Blob         |                                                                                                |
| Fetch Manifest Digest for Blob | HTTP Request    | Fetches blob data for creation date   | Fetch Manifest Digest     | Update Fields                        |                                                                                                |
| Update Fields             | Set                 | Enriches tag data with creation date and digest | Fetch Manifest Digest for Blob | Sort by Creation Date             |                                                                                                |
| Sort by Creation Date     | Sort                | Sorts tags by creation date descending| Update Fields             | Group Tags by Image                   |                                                                                                |
| Group Tags by Image       | Code                | Groups tags by image                  | Sort by Creation Date     | Identify Tags to Remove               |                                                                                                |
| Identify Tags to Remove   | Code                | Applies retention policy, identifies tags to delete | Group Tags by Image       | Remove Old Tags, Send Notification Email | Change retention slice(0,10) value here to adjust number of tags kept                         |
| Remove Old Tags           | HTTP Request        | Deletes old tags from registry        | Identify Tags to Remove   | Execute Garbage Collection, Send Failure Notification Email | Check DELETE operations in test environment before running                                   |
| Send Notification Email   | Email Send          | Sends success notification emails     | Remove Old Tags (success) | None                               | Customize email templates as needed                                                           |
| Send Failure Notification Email | Email Send     | Sends failure notification emails     | Remove Old Tags (failure) | None                               | Customize email templates as needed                                                           |
| Execute Garbage Collection | SSH                 | Runs garbage collection command on server | Remove Old Tags (success) | None                               | Requires SSH Private Key credential; registry may need maintenance mode for GC               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Scheduled Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 1 AM (adjust as needed)  

2. **Create HTTP Request Node "List Images"**  
   - URL: `https://<<your-registry-url>>/v2/_catalog` (replace placeholder)  
   - Authentication: HTTP Basic Auth with Docker Registry credentials  
   - Connect Scheduled Trigger → List Images  

3. **Create Code Node "Extract Image Names"**  
   - JavaScript:  
     ```js
     const images = items[0].json.repositories;
     return images.map(image => ({ json: { image } }));
     ```  
   - Connect List Images → Extract Image Names  

4. **Create HTTP Request Node "Retrieve Image Tags"**  
   - URL: `https://<<your-registry-url>>/v2/{{$json.image}}/tags/list`  
   - Authentication: HTTP Basic Auth  
   - Connect Extract Image Names → Retrieve Image Tags  

5. **Create Split Out Node "Split Tags"**  
   - Field to split out: `tags`  
   - Include field: `name` (image name)  
   - Destination field name: `tag`  
   - Connect Retrieve Image Tags → Split Tags  

6. **Create Filter Node "Filter Valid Tags"**  
   - Conditions:  
     - `$json.tag` is not empty  
     - `$json.name` is not empty  
   - Connect Split Tags → Filter Valid Tags  

7. **Create HTTP Request Node "Fetch Manifest Digest"**  
   - URL: `https://<<your-registry-url>>/v2/{{$json.name}}/manifests/{{$json.tag}}`  
   - Authentication: HTTP Basic Auth  
   - Set header `Accept` to:  
     `application/vnd.docker.distribution.manifest.v2+json, application/vnd.oci.image.manifest.v1+json, application/vnd.oci.image.index.v1+json, application/vnd.docker.distribution.manifest.list.v2+json`  
   - Enable full response to access headers  
   - Connect Filter Valid Tags → Fetch Manifest Digest  

8. **Create HTTP Request Node "Fetch Manifest Digest for Blob"**  
   - URL: `https://<<your-registry-url>>/v2/{{$('Filter Valid Tags').item.json.name}}/blobs/{{$json.body.config.digest}}`  
   - Authentication: HTTP Basic Auth  
   - Set header `Accept` to: `application/vnd.docker.distribution.manifest.v2+json`  
   - Connect Fetch Manifest Digest → Fetch Manifest Digest for Blob  

9. **Create Set Node "Update Fields"**  
   - Set fields:  
     - `name` = `{{$('Filter Valid Tags').item.json.name}}`  
     - `tag` = `{{$('Filter Valid Tags').item.json.tag}}`  
     - `created` = `{{$json.created}}` (from blob)  
     - `digest` = `{{$('Fetch Manifest Digest').item.json.headers['docker-content-digest']}}`  
   - Connect Fetch Manifest Digest for Blob → Update Fields  

10. **Create Sort Node "Sort by Creation Date"**  
    - Sort by field: `created` descending  
    - Connect Update Fields → Sort by Creation Date  

11. **Create Code Node "Group Tags by Image"**  
    - JavaScript:  
      ```js
      const groupedData = items.reduce((acc, item) => {
        const name = item.json.name;
        if (!acc[name]) acc[name] = [];
        acc[name].push({
          tag: item.json.tag,
          created: item.json.created,
          digest: item.json.digest,
        });
        return acc;
      }, {});
      return Object.keys(groupedData).map(name => ({ json: { name, tags: groupedData[name] } }));
      ```  
    - Connect Sort by Creation Date → Group Tags by Image  

12. **Create Code Node "Identify Tags to Remove"**  
    - JavaScript:  
      ```js
      const result = [];
      for (const item of items) {
        const tags = item.json.tags;
        if (tags) {
          const latestTag = tags.find(t => t.tag === 'latest') ? 'latest' : null;
          const sortedTags = tags.filter(t => t.tag !== 'latest')
                                 .sort((a, b) => new Date(b.created) - new Date(a.created));
          const keepTags = sortedTags.slice(0, 10);
          if (latestTag) keepTags.push('latest');
          const deleteTags = sortedTags.slice(10);
          result.push(...deleteTags.map(t => ({ json: { image: item.json.name, tag: t.tag, digest: t.digest } })));
        }
      }
      return result;
      ```  
    - Connect Group Tags by Image → Identify Tags to Remove  

13. **Create HTTP Request Node "Remove Old Tags"**  
    - URL: `https://<<your-registry-url>>/v2/{{$json.image}}/manifests/{{$json.digest}}`  
    - Method: DELETE  
    - Authentication: HTTP Basic Auth  
    - Header `Accept`: `application/vnd.docker.distribution.manifest.v2+json`  
    - On error: Continue workflow (to allow failure notification)  
    - Connect Identify Tags to Remove → Remove Old Tags  

14. **Create Email Send Node "Send Notification Email"**  
    - To: your notification recipient  
    - From: your sender email  
    - Subject: "Docker Registry Cleaner Notification"  
    - Body (text):  
      ```
      Image : {{$json.image}}
      Tag : {{$json.tag}}

      Removed
      ```  
    - Connect Remove Old Tags (success output) → Send Notification Email  

15. **Create Email Send Node "Send Failure Notification Email"**  
    - To: your notification recipient  
    - From: your sender email  
    - Subject: "[FAIL] Docker Registry Cleaner Notification"  
    - Body (text):  
      ```
      Image : {{$json.image}}
      Tag : {{$json.tag}}

      Failed
      ```  
    - Connect Remove Old Tags (error output) → Send Failure Notification Email  

16. **Create SSH Node "Execute Garbage Collection"**  
    - Working Directory: `/opt/services/`  
    - Command:  
      ```
      docker compose exec -it -u root registry bin/registry garbage-collect --delete-untagged /etc/docker/registry/config.yml
      ```  
    - Authentication: SSH Private Key credential with access to Docker registry server  
    - Connect Remove Old Tags (success output) → Execute Garbage Collection  

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Replace all instances of `<<your-registry-url>>` with your actual Docker registry URL                    | Critical for HTTP requests to function correctly                                                      |
| Retention policy can be adjusted by changing the slice value in "Identify Tags to Remove" node          | Default keeps last 10 tags plus `latest` tag                                                          |
| Ensure Docker Registry v2 API is accessible and supports Basic Authentication                            | Prerequisite for API calls                                                                             |
| SMTP credentials must be configured in n8n for email notifications                                      | Required for "Send Notification Email" and "Send Failure Notification Email" nodes                     |
| SSH Private Key credential must have access to Docker registry server for garbage collection command    | Required for "Execute Garbage Collection" node                                                        |
| Registry may need to be in maintenance mode or read-write mode for garbage collection to succeed       | Check Docker Registry documentation for maintenance mode procedures                                   |
| Test DELETE operations in a non-production environment before running in production                      | To avoid accidental data loss                                                                          |
| Customize email templates in notification nodes as needed                                               | To fit organizational communication standards                                                         |
| Video explanation and detailed blog post available at: https://n8n.io/blog/docker-registry-cleanup      | Additional resource for understanding and extending this workflow                                     |

---

This documentation provides a detailed and structured reference for the Docker Registry Cleanup Workflow, enabling users and automation agents to understand, reproduce, and modify the workflow confidently.