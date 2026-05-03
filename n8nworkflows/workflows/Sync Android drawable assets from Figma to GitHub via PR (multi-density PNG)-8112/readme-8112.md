Sync Android drawable assets from Figma to GitHub via PR (multi-density PNG)

https://n8nworkflows.xyz/workflows/sync-android-drawable-assets-from-figma-to-github-via-pr--multi-density-png--8112


# Sync Android drawable assets from Figma to GitHub via PR (multi-density PNG)

### 1. Workflow Overview

This workflow automates the synchronization of Android drawable assets from a Figma design project into a GitHub repository through a Pull Request (PR). It is designed to export multi-density PNG images (mdpi, hdpi, xhdpi, xxhdpi) for Android drawable folders, ensuring that assets are available in all necessary resolutions.

**Target Use Cases:**  
- Android developers or design teams who maintain UI assets in Figma and want to automate the export and integration of updated drawable assets into their codebase.  
- Continuous integration scenarios where design updates trigger automated asset synchronization, reducing manual export and commit errors.

**Logical Blocks:**

- **1.1 Input Reception and Trigger**  
  Starts the workflow manually and fetches initial data from the Figma API.

- **1.2 Filtering and Metadata Preparation**  
  Extracts relevant nodes (icons/buttons), defines Android drawable density folders, and merges metadata for processing.

- **1.3 Image Export URL Retrieval and Validation**  
  Obtains export URLs from the Figma API for the specified nodes and filters out any invalid or empty URLs.

- **1.4 Image Download and Processing**  
  Downloads the PNG files from Figma export URLs, merges with metadata, and renames files to comply with Android resource naming conventions.

- **1.5 Pull Request Preparation and Submission**  
  Creates a GitHub Pull Request containing the new drawable assets, ensuring no duplicate PRs are created.


---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception and Trigger

**Overview:**  
This block initiates the workflow manually and fetches the Figma file data required for asset extraction.

**Nodes Involved:**  
- Execute workflow (Manual Trigger)  
- Get Figma Export URL

**Node Details:**

- **Execute workflow**  
  - Type: Manual Trigger  
  - Role: Entry point for manual workflow execution.  
  - Configuration: No parameters; triggers the workflow on demand.  
  - Connections: Output → Get Figma Export URL  
  - Failure modes: None specific; manual start.

- **Get Figma Export URL**  
  - Type: HTTP Request  
  - Role: Fetches Figma file data with specific node IDs to access design components.  
  - Configuration:  
    - URL: `https://api.figma.com/v1/files/FIGMA_PROJECT_ID?ids=2-20347` (replace `FIGMA_PROJECT_ID`)  
    - Headers: `X-Figma-Token` with valid Figma API token  
  - Inputs: Trigger node output  
  - Outputs: JSON data representing Figma file nodes  
  - Failure modes: HTTP errors (auth failure, rate limiting), invalid project ID, network timeouts.

---

#### Block 1.2: Filtering and Metadata Preparation

**Overview:**  
Filters the Figma design nodes to identify only icons and buttons, defines standard Android drawable folders and densities, then merges this metadata for further processing.

**Nodes Involved:**  
- Find Icons & Buttons  
- Predefine drawable folders  
- Merge

**Node Details:**

- **Find Icons & Buttons**  
  - Type: Code (JavaScript)  
  - Role: Recursively traverses Figma document nodes to find nodes whose names contain “icon” or “button” (case-insensitive), and extracts relevant node metadata including export settings.  
  - Configuration: Custom JS traversing document structure, filtering by name regex `/icon|button/i`.  
  - Inputs: Output of Get Figma Export URL (JSON Figma document)  
  - Outputs: Array of filtered node metadata objects  
  - Failure modes: Unexpected JSON structure, missing children arrays, empty results when no matching nodes found.

- **Predefine drawable folders**  
  - Type: Code (JavaScript)  
  - Role: Defines static Android drawable folders with associated scale factors as JSON objects:  
    - drawable-mdpi (scale 1)  
    - drawable-hdpi (scale 1.5)  
    - drawable-xhdpi (scale 2)  
    - drawable-xxhdpi (scale 3)  
  - Inputs: Output of Find Icons & Buttons (parallel; nodes merged later)  
  - Outputs: List of drawable folder metadata  
  - Failure modes: None significant; static data.

- **Merge**  
  - Type: Merge (combine mode)  
  - Role: Combines results from Find Icons & Buttons and Predefine drawable folders into a Cartesian product for multi-density processing.  
  - Configuration: Combine all items from both inputs into combined pairs.  
  - Inputs: From Find Icons & Buttons and Predefine drawable folders  
  - Outputs: Combined metadata pairs (node + folder)  
  - Failure modes: Large datasets causing performance issues, empty inputs causing empty outputs.

---

#### Block 1.3: Image Export URL Retrieval and Validation

**Overview:**  
Requests actual export URLs for each node and density from Figma, then filters out any null or missing URLs to avoid processing errors.

**Nodes Involved:**  
- get figma Url  
- Filter nullable url of nodes

**Node Details:**

- **get figma Url**  
  - Type: HTTP Request  
  - Role: Calls Figma API to get export URLs for each node ID and scale defined in merged metadata.  
  - Configuration:  
    - URL template: `https://api.figma.com/v1/images/FIGMA_PROJECT_ID?ids={{$json["id"]}}&format=png&scale={{$json["scale"]}}`  
    - Header: `X-Figma-Token` with Figma API token  
  - Inputs: Output from Merge (node+folder pairs)  
  - Outputs: JSON with export URLs keyed by node IDs  
  - Failure modes: API rate limits, invalid tokens, malformed IDs, network issues.

- **Filter nullable url of nodes**  
  - Type: Code (JavaScript)  
  - Role: Parses the JSON response to extract valid URLs, filters out any entries with null or empty URLs.  
  - Configuration: Iterates over images map, returns only entries with non-null URLs.  
  - Inputs: Output from get figma Url  
  - Outputs: List of nodeId-URL pairs for valid images.  
  - Failure modes: Empty or malformed response, no valid URLs found.

---

#### Block 1.4: Image Download and Processing

**Overview:**  
Downloads the image files from Figma export URLs, merges downloaded binary data with prior metadata, then renames files to comply with Android resource naming conventions.

**Nodes Involved:**  
- Download Each Image from the Figma Export URL  
- Merge1  
- Edit File names

**Node Details:**

- **Download Each Image from the Figma Export URL**  
  - Type: HTTP Request  
  - Role: Downloads the binary PNG file from each valid URL.  
  - Configuration:  
    - URL: dynamically set from the filtered URL field  
    - Response Format: file (binary)  
  - Inputs: Output from Filter nullable url of nodes  
  - Outputs: Binary image data with associated metadata  
  - Failure modes: Network errors, URL expiration, HTTP 404 if asset missing.

- **Merge1**  
  - Type: Merge (combineByPosition)  
  - Role: Combines the downloaded images with their corresponding metadata based on position in data arrays (index-based matching).  
  - Configuration: Combine by position to align metadata and binary data.  
  - Inputs: From Download Each Image and from previous Merge output chain  
  - Outputs: Combined record with metadata + binary image  
  - Failure modes: Mismatched array lengths causing data misalignment.

- **Edit File names**  
  - Type: Code (JavaScript)  
  - Role: Renames each file path according to Android naming standards:  
    - Lowercase  
    - Replace non-alphanumeric with underscore  
    - Places files under correct drawable folder (mdpi, hdpi, etc.)  
    - Sets commit message, branch, repository owner and name for GitHub commit  
  - Inputs: Output from Merge1  
  - Outputs: Metadata enriched with file paths and commit info  
  - Failure modes: Missing or malformed names or folders, static strings for repo info must be replaced.

---

#### Block 1.5: Pull Request Preparation and Submission

**Overview:**  
Checks if this is the first item (index 0) to avoid duplicate PR creation, then creates a GitHub Pull Request with the new drawable assets committed into the repo.

**Nodes Involved:**  
- If  
- Prepare Pull Request

**Node Details:**

- **If**  
  - Type: If  
  - Role: Conditional check to only proceed for the first item (index 0) in the batch to prevent multiple PRs creation.  
  - Configuration: Checks if item index equals 0.  
  - Inputs: Output from Edit File names  
  - Outputs: If true → Prepare Pull Request  
  - Failure modes: Incorrect branching logic could cause duplicate PRs or no PR creation.

- **Prepare Pull Request**  
  - Type: HTTP Request  
  - Role: Creates a Pull Request in GitHub using the GitHub API.  
  - Configuration:  
    - URL: `https://api.github.com/repos/REPO_OWNER/REPO_NAME/pulls` (replace placeholders)  
    - Method: POST  
    - Headers: Authorization Bearer token, Accept JSON, Content-Type JSON  
    - Body: JSON with PR title, branch head (`add-assets-from-figma`), base branch (`main`), and description  
  - Inputs: Output from If (true branch)  
  - Outputs: GitHub PR creation response  
  - Failure modes: Auth failure (token invalid or missing), branch conflicts, repo not found, rate limits.

---

### 3. Summary Table

| Node Name                                 | Node Type           | Functional Role                                     | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                               |
|-------------------------------------------|---------------------|----------------------------------------------------|---------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------------|
| Execute workflow                          | Manual Trigger      | Starts the workflow manually                        | -                               | Get Figma Export URL           |                                                                                                          |
| Get Figma Export URL                      | HTTP Request        | Fetches Figma file node data                        | Execute workflow                | Find Icons & Buttons, Predefine drawable folders |                                                                                                          |
| Find Icons & Buttons                      | Code                | Filters Figma nodes to icons & buttons              | Get Figma Export URL            | Merge                         |                                                                                                          |
| Predefine drawable folders                | Code                | Defines Android drawable folders and scales         | Get Figma Export URL            | Merge                         |                                                                                                          |
| Merge                                    | Merge (combine all) | Combines filtered nodes with drawable folders       | Find Icons & Buttons, Predefine drawable folders | get figma Url                 |                                                                                                          |
| get figma Url                            | HTTP Request        | Fetches export URLs from Figma for each node/density | Merge                         | Filter nullable url of nodes   |                                                                                                          |
| Filter nullable url of nodes              | Code                | Filters out null or empty export URLs                | get figma Url                  | Download Each Image from the Figma Export URL |                                                                                                          |
| Download Each Image from the Figma Export URL | HTTP Request    | Downloads PNG files from export URLs                 | Filter nullable url of nodes   | Merge1                        |                                                                                                          |
| Merge1                                   | Merge (combineByPosition) | Merges downloaded images with metadata              | Download Each Image, Merge      | Edit File names               |                                                                                                          |
| Edit File names                          | Code                | Renames files and sets commit/branch/repo metadata | Merge1                        | If                           |                                                                                                          |
| If                                       | If                  | Ensures PR creation only once per batch              | Edit File names                | Prepare Pull Request           |                                                                                                          |
| Prepare Pull Request                      | HTTP Request        | Creates GitHub Pull Request with new assets          | If                           | -                             |                                                                                                          |
| Sticky Note                              | Sticky Note         | Workflow title banner                                | -                             | -                             | Sync Android drawable assets from Figma to GitHub via PR (multi‑density PNG)                              |
| Sticky Note1                             | Sticky Note         | Detailed node descriptions and workflow overview    | -                             | -                             | See detailed node descriptions in section 1 overview for function roles                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger Node** (`Execute workflow`)  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually.

2. **Create HTTP Request Node** (`Get Figma Export URL`)  
   - URL: `https://api.figma.com/v1/files/FIGMA_PROJECT_ID?ids=2-20347` (replace `FIGMA_PROJECT_ID`)  
   - Method: GET  
   - Headers: Add `X-Figma-Token` with your Figma API token.  
   - Connect: Output from Manual Trigger.

3. **Create Code Node** (`Find Icons & Buttons`)  
   - Paste the provided JS code that recursively finds nodes with “icon” or “button” in their names inside a specific frame named “forgot password”.  
   - Connect input from `Get Figma Export URL`.

4. **Create Code Node** (`Predefine drawable folders`)  
   - JS code returns static array for drawable folders with scales: mdpi(1), hdpi(1.5), xhdpi(2), xxhdpi(3).  
   - Connect input from `Get Figma Export URL`.

5. **Create Merge Node** (`Merge`)  
   - Mode: Combine (combine all)  
   - Connect inputs:  
     - From `Find Icons & Buttons` (index 0)  
     - From `Predefine drawable folders` (index 1)

6. **Create HTTP Request Node** (`get figma Url`)  
   - URL expression:  
     `https://api.figma.com/v1/images/FIGMA_PROJECT_ID?ids={{$json["id"]}}&format=png&scale={{$json["scale"]}}`  
     (replace `FIGMA_PROJECT_ID`)  
   - Headers: `X-Figma-Token` with Figma token  
   - Connect input from `Merge`.

7. **Create Code Node** (`Filter nullable url of nodes`)  
   - JS code to filter out empty or null URLs from the response images map.  
   - Connect input from `get figma Url`.

8. **Create HTTP Request Node** (`Download Each Image from the Figma Export URL`)  
   - URL: Use expression `{{$json["url"]}}`  
   - Response format: File (binary)  
   - Connect input from `Filter nullable url of nodes`.

9. **Create Merge Node** (`Merge1`)  
   - Mode: Combine by Position  
   - Connect inputs:  
     - From `Download Each Image from the Figma Export URL` (index 0)  
     - From `Merge` (index 1) (previous metadata)

10. **Create Code Node** (`Edit File names`)  
    - JS code to set file paths: e.g. `app/src/main/res/${folder}/${name}.png`  
    - Normalize names to lowercase and underscores, set commit message, branch `add-assets-from-figma`, repository owner/name placeholders.  
    - Connect input from `Merge1`.

11. **Create If Node** (`If`)  
    - Condition: Check if `{{$itemIndex}}` equals 0 (only first item proceeds)  
    - Connect input from `Edit File names`.

12. **Create HTTP Request Node** (`Prepare Pull Request`)  
    - URL: `https://api.github.com/repos/REPO_OWNER/REPO_NAME/pulls` (replace placeholders)  
    - Method: POST  
    - Headers:  
      - Authorization: Bearer `GITHUB_TOKEN` (GitHub personal access token)  
      - Accept: application/vnd.github+json  
      - Content-Type: application/json  
    - Body JSON:  
      ```json
      {
        "title": "Add drawable assets from Figma",
        "head": "add-assets-from-figma",
        "base": "main",
        "body": "This PR contains Android drawable assets exported from Figma in all resolutions."
      }
      ```  
    - Connect input from `If` (true branch).

**Note on Credentials:**  
- Replace `FIGMA_TOKEN` with your actual Figma API token in HTTP request headers.  
- Replace `GITHUB_TOKEN` with a GitHub personal access token with repo write permissions.  
- Replace `FIGMA_PROJECT_ID`, `REPO_OWNER`, and `REPO_NAME` with your actual project and repository identifiers.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                   |
|-----------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow syncs Android drawable assets exported from Figma into a GitHub repo via a Pull Request.    | Primary workflow purpose                          |
| Node descriptions provide detailed roles for each step of the process, supporting maintenance.       | Sticky Note1 content                              |
| Replace placeholders (`FIGMA_PROJECT_ID`, `REPO_OWNER`, `REPO_NAME`, tokens) with actual values.     | Required for proper API integration               |
| To avoid duplicate PRs, the workflow restricts PR creation to the first item processed in batch.     | Prevents GitHub clutter                           |
| For advanced customization, adjust regex in "Find Icons & Buttons" node to match specific asset names. | Custom filtering logic                            |

---

*Disclaimer: This document is based exclusively on an automated n8n workflow export. It complies fully with content policies and contains no illegal or sensitive data.*