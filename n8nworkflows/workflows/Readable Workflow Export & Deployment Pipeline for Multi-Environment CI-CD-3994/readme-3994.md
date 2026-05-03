Readable Workflow Export & Deployment Pipeline for Multi-Environment CI/CD

https://n8nworkflows.xyz/workflows/readable-workflow-export---deployment-pipeline-for-multi-environment-ci-cd-3994


# Readable Workflow Export & Deployment Pipeline for Multi-Environment CI/CD

### 1. Workflow Overview

This workflow automates the export, naming, categorization, and deployment preparation of n8n workflows for multi-environment CI/CD pipelines. Its primary purpose is to export all current workflows, create human-readable JSON files, and organize them into folders based on deployment tags for development and production environments. This enables automated deployment pipelines to pick up the workflows for container image builds and imports on startup.

The workflow is logically structured into the following blocks:

- **1.1 Initialization and Export:** Creates necessary folders and exports all n8n workflows using the CLI.
- **1.2 Workflow Processing:** Reads exported workflow files, parses JSON, and removes redundant root nodes.
- **1.3 Naming and Storing:** Converts workflows to JSON files with readable names and stores them.
- **1.4 Deployment Tag Checks:** Determines if workflows should be auto-deployed to development or production based on tags, then stores accordingly.
- **1.5 Documentation and Supplemental Files:** Contains sticky notes with documentation, Dockerfile, and Docker entrypoint scripts to support deployment and CI/CD.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Export

**Overview:**  
This block initializes the export environment by creating folders, clearing previous exports, and running the n8n CLI export command to export all workflows into a dedicated folder.

**Nodes Involved:**  
- Start export workflows  
- Create folders and run n8n cli

**Node Details:**

- **Start export workflows**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow export process manually.  
  - *Config:* No parameters, triggers downstream export process.  
  - *Connections:* Outputs to "Create folders and run n8n cli".  
  - *Edge cases:* User must manually trigger; no automated scheduling.  

- **Create folders and run n8n cli**  
  - *Type:* Execute Command  
  - *Role:* Prepares folder structure and runs the CLI export command.  
  - *Config:* Executes shell commands that:  
    - Create directories `export`, `export/n8n-workflows`, `export/named-workflows`, and `import-dev/workflows`.  
    - Removes existing files in export folders to ensure a clean export.  
    - Runs `n8n export:workflow --all --output=export/n8n-workflows --pretty --separate` to export all workflows individually in pretty JSON format.  
  - *Connections:* Outputs to "load exported workflows".  
  - *Edge cases:* Requires CLI installed and accessible; permission errors possible on mkdir or rm commands; CLI export failures must be handled externally.

---

#### 2.2 Workflow Processing

**Overview:**  
Reads all exported workflow JSON files from the export folder and parses them as JSON to prepare for further processing.

**Nodes Involved:**  
- load exported workflows  
- parse workflow  
- Remove root node

**Node Details:**

- **load exported workflows**  
  - *Type:* Read/Write File  
  - *Role:* Reads all JSON files in the `export/n8n-workflows` folder.  
  - *Config:* File selector pattern: `export/n8n-workflows/*.*`. Reads all files regardless of extension.  
  - *Connections:* Outputs to "parse workflow".  
  - *Edge cases:* Files missing or unreadable will cause failures; no filtering on file types.  

- **parse workflow**  
  - *Type:* Extract From File  
  - *Role:* Parses file content from JSON string to JSON object.  
  - *Config:* Operation set to "fromJson", stores result in `parsedData` key.  
  - *Connections:* Outputs to "Remove root node".  
  - *Edge cases:* Malformed JSON will cause parsing errors.  

- **Remove root node**  
  - *Type:* Set  
  - *Role:* Cleans data by replacing the root JSON with the parsed workflow data under `parsedData`.  
  - *Config:* Uses expression `={{ $json.parsedData }}` to set the whole output JSON to the parsed data.  
  - *Connections:* Outputs to three nodes in parallel:  
    - "Create JSON file with readable name" (base export)  
    - "TAG? Auto deploy to dev"  
    - "TAG? Auto deploy to PROD"  
  - *Edge cases:* If `parsedData` is missing, outputs blank or invalid data.

---

#### 2.3 Naming and Storing

**Overview:**  
This block converts raw workflow JSON into readable filenames and stores them in the base export folder.

**Nodes Involved:**  
- Create JSON file with readable name  
- Store named workflow

**Node Details:**

- **Create JSON file with readable name**  
  - *Type:* Convert To File  
  - *Role:* Converts JSON workflow objects into JSON files with readable names.  
  - *Config:*  
    - Mode: "each" (processes each item individually)  
    - File name format: `./export/named-workflows/{{ $json.name }} ({{ $json.id }}).json`  
    - Pretty formatting enabled.  
  - *Connections:* Outputs to "Store named workflow".  
  - *Edge cases:* Workflow names with invalid filesystem characters may cause errors.  

- **Store named workflow**  
  - *Type:* Read/Write File  
  - *Role:* Writes the generated JSON files to the filesystem.  
  - *Config:*  
    - File name sourced from binary data metadata (`{{$binary.data.directory}}/{{$binary.data.fileName}}`).  
    - Operation: write.  
  - *Connections:* Terminal node in this branch.  
  - *Edge cases:* Filesystem write permission errors; filename conflicts.

---

#### 2.4 Deployment Tag Checks and Environment Specific Storage

**Overview:**  
Evaluates workflow tags to determine if workflows should be automatically deployed to development or production environments and stores those workflows accordingly.

**Nodes Involved:**  
- TAG? Auto deploy to dev  
- Create JSON file with readable name (dev)  
- Store named workflow (dev)  
- TAG? Auto deploy to PROD  
- Create JSON file with readable name (prod)  
- Store named workflow (prod)

**Node Details:**

- **TAG? Auto deploy to dev**  
  - *Type:* If  
  - *Role:* Checks if the workflow has a tag named "Auto deploy to dev".  
  - *Config:*  
    - Condition checks if the array of tag names (`{{ $json.tags.map(obj => obj.name) }}`) contains "Auto deploy to dev".  
    - Case sensitive.  
  - *Connections:* If true, outputs to "Create JSON file with readable name (dev)".  
  - *Edge cases:* Workflows without tags or malformed tags will not pass condition.

- **Create JSON file with readable name (dev)**  
  - *Type:* Convert To File  
  - *Role:* Converts qualified workflows to JSON files for development environment.  
  - *Config:*  
    - Mode: "each"  
    - File name: `./import-dev/workflows/{{ $json.name }} ({{ $json.id }}).json`  
    - Pretty formatting enabled.  
  - *Connections:* Outputs to "Store named workflow (dev)".  
  - *Edge cases:* Same as base naming node.  

- **Store named workflow (dev)**  
  - *Type:* Read/Write File  
  - *Role:* Saves development deployment JSON files.  
  - *Config:* Writes file as per binary metadata.  
  - *Connections:* Terminal node.  
  - *Edge cases:* Filesystem permission conflicts.

- **TAG? Auto deploy to PROD**  
  - *Type:* If  
  - *Role:* Checks if the workflow has the "Auto deploy to PROD" tag.  
  - *Config:* Similar condition as dev tag check but for "Auto deploy to PROD".  
  - *Connections:* If true, outputs to "Create JSON file with readable name (prod)".  
  - *Edge cases:* Same as dev tag check.

- **Create JSON file with readable name (prod)**  
  - *Type:* Convert To File  
  - *Role:* Converts qualified workflows to JSON files for production environment.  
  - *Config:*  
    - Mode: "each"  
    - File name: `./import-prod/workflows/{{ $json.name }} ({{ $json.id }}).json`  
    - Pretty formatting enabled.  
  - *Connections:* Outputs to "Store named workflow (prod)".  
  - *Edge cases:* Same as above.

- **Store named workflow (prod)**  
  - *Type:* Read/Write File  
  - *Role:* Saves production deployment JSON files.  
  - *Config:* Writes file as per binary metadata.  
  - *Connections:* Terminal node.  
  - *Edge cases:* Filesystem permission conflicts.

---

#### 2.5 Documentation and Supplemental Files

**Overview:**  
This block contains sticky notes documenting the workflow logic, folder structure, Dockerfile, and Docker entrypoint scripts used to automate importing and starting workflows in containerized environments.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5  
- Sticky Note6  
- Sticky Note7

**Node Details:**

- **Sticky Note** (positioned near export nodes)  
  - *Content:* Highlights exporting data to the exports folder with readable names.

- **Sticky Note1**  
  - *Content:* Notes that exported workflows are saved per workflow in the exports folder.

- **Sticky Note2 and Sticky Note3**  
  - *Content:* Instructions to add files to the `import-dev` folder if they should auto deploy to dev.

- **Sticky Note4** (large detailed note)  
  - *Content:* Comprehensive explanation of export behavior:  
    - All workflows stored first by ID, then renamed to readable format.  
    - Workflows tagged with "Auto deploy to dev" or "Auto deploy to PROD" are saved in respective folders.  
    - On commit, build pipeline creates images that auto-deploy these workflows.  
    - On startup, workflows are imported and disabled; an AutoStarter workflow starts workflows with the "AutoStarted" tag.

- **Sticky Note5**  
  - *Content:* Dockerfile example for creating a custom n8n image that supports auto importing credentials and workflows, with environment variables and user permissions handled.

- **Sticky Note6**  
  - *Content:* Importing Docker entrypoint script example, which imports credentials and workflows from mounted folders and executes a specified workflow on startup.

- **Sticky Note7**  
  - *Content:* Notes about example files used in the delivery pipeline.

---

### 3. Summary Table

| Node Name                       | Node Type           | Functional Role                                            | Input Node(s)                 | Output Node(s)                                     | Sticky Note                                                                                       |
|--------------------------------|---------------------|------------------------------------------------------------|------------------------------|---------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Start export workflows          | Manual Trigger      | Entry point to manually trigger export process             |                              | Create folders and run n8n cli                    |                                                                                                 |
| Create folders and run n8n cli  | Execute Command     | Prepares folders and runs n8n CLI export                    | Start export workflows        | load exported workflows                           |                                                                                                 |
| load exported workflows         | Read/Write File     | Reads exported workflow JSON files                          | Create folders and run n8n cli | parse workflow                                   |                                                                                                 |
| parse workflow                 | Extract From File   | Parses workflow JSON content                                | load exported workflows       | Remove root node                                  |                                                                                                 |
| Remove root node                | Set                 | Sets JSON data to parsed workflow object                    | parse workflow                | Create JSON file with readable name, TAG? Auto deploy to dev, TAG? Auto deploy to PROD |                                                                                                 |
| Create JSON file with readable name | Convert To File   | Converts workflow JSON to readable file names               | Remove root node              | Store named workflow                              | Sticky Note: "## export\nadd the data in exports folder with readable names"                     |
| Store named workflow            | Read/Write File     | Saves named workflow JSON files                              | Create JSON file with readable name |                                              |                                                                                                 |
| TAG? Auto deploy to dev         | If                  | Checks for "Auto deploy to dev" tag                         | Remove root node              | Create JSON file with readable name (dev)         | Sticky Note2, Sticky Note3: "## export\nadd files to the import-dev folder if they should auto deploy to dev" |
| Create JSON file with readable name (dev) | Convert To File   | Converts dev-tagged workflows to JSON files                 | TAG? Auto deploy to dev       | Store named workflow (dev)                        |                                                                                                 |
| Store named workflow (dev)      | Read/Write File     | Saves dev environment workflows                              | Create JSON file with readable name (dev) |                                            |                                                                                                 |
| TAG? Auto deploy to PROD        | If                  | Checks for "Auto deploy to PROD" tag                        | Remove root node              | Create JSON file with readable name (prod)        |                                                                                                 |
| Create JSON file with readable name (prod) | Convert To File  | Converts prod-tagged workflows to JSON files                | TAG? Auto deploy to PROD      | Store named workflow (prod)                       |                                                                                                 |
| Store named workflow (prod)     | Read/Write File     | Saves prod environment workflows                             | Create JSON file with readable name (prod) |                                           |                                                                                                 |
| Sticky Note                    | Sticky Note         | Documentation about export folder usage                      |                              |                                                   | "## export\nadd the data in exports folder with readable names"                                 |
| Sticky Note1                   | Sticky Note         | Documentation about exporting workflows per workflow        |                              |                                                   | "## n8n export workflows\nadd the data in exports folder per workflow"                            |
| Sticky Note2                   | Sticky Note         | Info on dev auto deployment file placement                   |                              |                                                   | "## export\nadd files to the import-dev folder if they should auto deploy to dev"                |
| Sticky Note3                   | Sticky Note         | Same as Sticky Note2                                         |                              |                                                   | "## export\nadd files to the import-dev folder if they should auto deploy to dev"                |
| Sticky Note4                   | Sticky Note         | Detailed explanation of export and deployment process       |                              |                                                   | Large note explaining workflow export, tagging, import process, and startup behavior            |
| Sticky Note5                   | Sticky Note         | Dockerfile example for auto-importing workflows             |                              |                                                   | Dockerfile content showing environment variables and entrypoint setup                           |
| Sticky Note6                   | Sticky Note         | Docker entrypoint script for import and auto-execution      |                              |                                                   | Shell script to import credentials and workflows, execute startup workflow, start n8n           |
| Sticky Note7                   | Sticky Note         | Notes about example files in delivery pipeline               |                              |                                                   | "# Example files for use in a delivery pipeline"                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `Start export workflows`  
   - Type: Manual Trigger  
   - Purpose: Start the export process manually.

2. **Create Execute Command Node**  
   - Name: `Create folders and run n8n cli`  
   - Type: Execute Command  
   - Command:  
     ```
     mkdir export; 
     mkdir export/n8n-workflows; 
     mkdir export/named-workflows;
     rm -Rf export/n8n-workflows/*;
     rm -Rf export/named-workflows/*;

     mkdir import-dev/workflows;

     n8n export:workflow --all --output=export/n8n-workflows --pretty --separate
     ```  
   - Connect `Start export workflows` → `Create folders and run n8n cli`.

3. **Create Read/Write File Node**  
   - Name: `load exported workflows`  
   - Type: Read/Write File  
   - File Selector: `export/n8n-workflows/*.*`  
   - Connect `Create folders and run n8n cli` → `load exported workflows`.

4. **Create Extract From File Node**  
   - Name: `parse workflow`  
   - Type: Extract From File  
   - Operation: fromJson  
   - Destination Key: `parsedData`  
   - Connect `load exported workflows` → `parse workflow`.

5. **Create Set Node**  
   - Name: `Remove root node`  
   - Type: Set  
   - Mode: Raw  
   - JSON Output: `={{ $json.parsedData }}`  
   - Connect `parse workflow` → `Remove root node`.

6. **Create Convert To File Node - Base Export**  
   - Name: `Create JSON file with readable name`  
   - Type: Convert To File  
   - Mode: each  
   - Format: true (pretty)  
   - File Name: `=./export/named-workflows/{{ $json.name }} ({{ $json.id }}).json`  
   - Connect `Remove root node` → `Create JSON file with readable name`.

7. **Create Read/Write File Node - Base Export Store**  
   - Name: `Store named workflow`  
   - Type: Read/Write File  
   - File Name: `={{ $binary.data.directory }}/{{ $binary.data.fileName }}`  
   - Operation: write  
   - Connect `Create JSON file with readable name` → `Store named workflow`.

8. **Create If Node - Auto deploy to dev**  
   - Name: `TAG? Auto deploy to dev`  
   - Type: If  
   - Condition:  
     - Left Value: `={{ $json.tags.map(obj => obj.name) }}` (array of tag names)  
     - Operator: array contains  
     - Right Value: "Auto deploy to dev"  
   - Connect `Remove root node` → `TAG? Auto deploy to dev`.

9. **Create Convert To File Node - Dev**  
   - Name: `Create JSON file with readable name (dev)`  
   - Type: Convert To File  
   - Mode: each  
   - Format: true  
   - File Name: `=./import-dev/workflows/{{ $json.name }} ({{ $json.id }}).json`  
   - Connect `TAG? Auto deploy to dev` (true) → `Create JSON file with readable name (dev)`.

10. **Create Read/Write File Node - Dev Store**  
    - Name: `Store named workflow (dev)`  
    - Type: Read/Write File  
    - File Name: `={{ $binary.data.directory }}/{{ $binary.data.fileName }}`  
    - Operation: write  
    - Connect `Create JSON file with readable name (dev)` → `Store named workflow (dev)`.

11. **Create If Node - Auto deploy to PROD**  
    - Name: `TAG? Auto deploy to PROD`  
    - Type: If  
    - Condition:  
      - Left Value: `={{ $json.tags.map(obj => obj.name) }}`  
      - Operator: array contains  
      - Right Value: "Auto deploy to PROD"  
    - Connect `Remove root node` → `TAG? Auto deploy to PROD`.

12. **Create Convert To File Node - Prod**  
    - Name: `Create JSON file with readable name (prod)`  
    - Type: Convert To File  
    - Mode: each  
    - Format: true  
    - File Name: `=./import-prod/workflows/{{ $json.name }} ({{ $json.id }}).json`  
    - Connect `TAG? Auto deploy to PROD` (true) → `Create JSON file with readable name (prod)`.

13. **Create Read/Write File Node - Prod Store**  
    - Name: `Store named workflow (prod)`  
    - Type: Read/Write File  
    - File Name: `={{ $binary.data.directory }}/{{ $binary.data.fileName }}`  
    - Operation: write  
    - Connect `Create JSON file with readable name (prod)` → `Store named workflow (prod)`.

14. **Add Sticky Notes for Documentation**  
    - Add all sticky notes at respective positions with content from the original workflow to provide context and documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| The workflow uses n8n CLI commands (`n8n export:workflow`, `n8n import:workflow`) which require the n8n CLI installed and available in the environment running the Execute Command node.                                                                                                                                                                                                                                                                                                                                                                                             | n8n CLI Documentation: https://docs.n8n.io/reference/cli/                                                                           |
| The Dockerfile and entrypoint script examples in sticky notes illustrate how to build a custom n8n Docker image that automatically imports workflows and credentials stored in mounted folders before starting n8n. The entrypoint script also supports auto-executing a workflow by ID after startup delays, useful for AutoStarter workflows.                                                                                                                                                                                                                                                      | GitHub n8n Docker repo: https://github.com/n8n-io/n8n/tree/master/docker/images/n8n                                                |
| Filenames for exported workflows are constructed as `<Workflow Name> (<Workflow ID>).json` for clarity and traceability. Workflow names must be sanitized if they contain filesystem-invalid characters to avoid errors.                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                    |
| The deployment tagging relies on workflows being tagged in n8n with exact tag names "Auto deploy to dev" or "Auto deploy to PROD". Tags are case sensitive and must be maintained accordingly to trigger auto deployment.                                                                                                                                                                                                                                                                                                                                                                   |                                                                                                                                    |
| On startup, the import folders (`import-dev`, `import-prod`) are used to automatically import workflows. Workflows are imported as disabled by default and require manual or automated enabling via secondary workflows (e.g., AutoStarter).                                                                                                                                                                                                                                                                                                                                        |                                                                                                                                    |
| The overall system supports multi-environment CI/CD pipelines by allowing exported workflows to be picked up by container build processes to create deployable n8n images per environment.                                                                                                                                                                                                                                                                                                                                                                                               |                                                                                                                                    |

---

_Disclaimer: The text provided is exclusively extracted from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and publicly accessible._