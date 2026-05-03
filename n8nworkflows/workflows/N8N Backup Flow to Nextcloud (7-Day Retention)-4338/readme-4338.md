N8N Backup Flow to Nextcloud (7-Day Retention)

https://n8nworkflows.xyz/workflows/n8n-backup-flow-to-nextcloud--7-day-retention--4338


# N8N Backup Flow to Nextcloud (7-Day Retention)

---

### 1. Workflow Overview

This workflow automates the backup of all n8n workflows by exporting them as JSON files and storing them in a Nextcloud directory with a 7-day retention policy. It is designed for users who want to ensure regular, time-stamped backups of their workflows in a cloud storage environment, enabling easy restoration and version control.

The workflow logically divides into the following blocks:

- **1.1 Triggering Mechanisms:** Supports manual execution and scheduled automatic backups.
- **1.2 Backup Path Setup:** Defines and manages the backup directory path in Nextcloud, including directory creation.
- **1.3 Workflow Export and Processing:** Fetches all n8n workflows, converts each to a JSON file, and uploads to Nextcloud.
- **1.4 Backup Retention Management:** Lists existing backups in Nextcloud, sorts them, and deletes backups older than 7 days to maintain retention limits.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Triggering Mechanisms

- **Overview:**  
  This block initiates the backup process either manually or on a scheduled interval.

- **Nodes Involved:**  
  - On clicking 'execute' (Manual Trigger)  
  - Schedule Trigger (Scheduled Trigger)  
  - Backup Path (Set node, shared with Block 2.2)

- **Node Details:**

  - **On clicking 'execute'**  
    - Type: Manual Trigger  
    - Role: Allows manual initiation of the backup workflow by user action in the n8n editor UI.  
    - Configuration: Default, no parameters.  
    - Connections: Outputs to Backup Path node.  
    - Edge Cases: None significant, but manual trigger depends on user interaction.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow at defined intervals (default is every minute as per empty interval array).  
    - Configuration: Interval set to default (empty object, meaning every minute or as configured by user).  
    - Connections: Outputs to Backup Path node.  
    - Edge Cases: If interval is misconfigured, the workflow might trigger too frequently or not at all.  
    - Version: 1.2 ensures support for interval scheduling.

---

#### 2.2 Backup Path Setup

- **Overview:**  
  Sets the destination backup directory path with an ISO timestamp and ensures the directory exists in Nextcloud.

- **Nodes Involved:**  
  - Backup Path (Set)  
  - Nextcloud Directory (Nextcloud node)

- **Node Details:**

  - **Backup Path**  
    - Type: Set  
    - Role: Creates a string variable `backup` representing the backup directory path formatted as `/N8N-Backup/YYYY-MM-DD_HHMMSS`.  
    - Configuration: Uses expression `=/N8N-Backup/{{ $now.format('yyyy-MM-dd_HHmmss') }}` to generate a timestamped folder path.  
    - Connections: Outputs to Nextcloud Directory node.  
    - Execute Once: True (runs once per trigger).  
    - Edge Cases: Timezone or formatting errors could lead to incorrect folder names. `now` should be correctly defined by n8n runtime.  
    - Notes: Marked as "Backupdirectory" for clarity.

  - **Nextcloud Directory**  
    - Type: Nextcloud (Folder resource)  
    - Role: Creates the backup directory in Nextcloud if it does not exist.  
    - Configuration: Path is set dynamically via expression from `backup` variable, resource type is folder.  
    - Credentials: Uses configured Nextcloud API credentials.  
    - Connections: Outputs to n8n node (Block 2.3).  
    - Edge Cases: Folder creation might fail due to permissions, connectivity, or if folder already exists (usually safe).  
    - Notes: "Create Directory" annotation.

---

#### 2.3 Workflow Export and Processing

- **Overview:**  
  Retrieves all existing n8n workflows, processes them one by one, converting each to a JSON file and uploading to the created Nextcloud backup directory.

- **Nodes Involved:**  
  - n8n (n8n API node)  
  - Loop Over Items (SplitInBatches)  
  - Convert to File (ConvertToFile)  
  - Nextcloud Upload (Nextcloud node)

- **Node Details:**

  - **n8n**  
    - Type: n8n API node  
    - Role: Retrieves all workflows from the n8n instance via API.  
    - Configuration: No filters applied, retrieves all workflows. Retry enabled on failure.  
    - Credentials: Uses n8n API credentials.  
    - Connections: Outputs to Loop Over Items node.  
    - Edge Cases: API authentication failures, network timeouts, or empty workflow list.  
    - Always Output Data: True ensuring output data is always forwarded.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Iteratively processes workflows one by one to handle file conversion and upload.  
    - Configuration: Default batch size (likely 1).  
    - Connections: Has two output paths:
      - Main output to Convert to File node for export.
      - Secondary output to Nextcloud List Dir node for retention management (see Block 2.4).  
    - Edge Cases: Large workflow counts could slow process; batch size can be adjusted for performance.

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts each workflow JSON data into a JSON file binary format suitable for upload.  
    - Configuration: Format set to JSON, filename dynamically assigned as the workflow name plus `.json` extension.  
    - Connections: Outputs binary data to Nextcloud Upload node.  
    - Edge Cases: File name conflicts if workflows have same names, JSON conversion errors if data malformed.

  - **Nextcloud Upload**  
    - Type: Nextcloud (File resource)  
    - Role: Uploads the generated JSON file to the Nextcloud backup directory.  
    - Configuration: Path dynamically built combining backup path and converted file name; binary data upload enabled.  
    - Credentials: Uses Nextcloud API credentials.  
    - Connections: Loops back to Loop Over Items node for next item processing.  
    - Edge Cases: Upload failures due to permissions, storage quota, or connectivity issues.

---

#### 2.4 Backup Retention Management

- **Overview:**  
  Maintains backup retention by listing existing backups in Nextcloud, sorting them, and deleting backups that exceed the 7 most recent entries.

- **Nodes Involved:**  
  - Nextcloud List Dir (Nextcloud node)  
  - Backups (Sort node)  
  - Limits Backups (Code node)  
  - Nextcloud - Delete old backups (Nextcloud node)

- **Node Details:**

  - **Nextcloud List Dir**  
    - Type: Nextcloud (Folder resource)  
    - Role: Lists all folders/files in the `/N8N-Backup` directory on Nextcloud.  
    - Configuration: Path set statically to `/N8N-Backup`, operation set to list folder contents.  
    - Credentials: Nextcloud API credentials.  
    - Connections: Outputs to Backups node.  
    - Execute Once: True (runs once per trigger).  
    - Edge Cases: Permissions or connectivity issues could block directory listing.

  - **Backups**  
    - Type: Sort  
    - Role: Sorts listed backups in descending order based on the `path` field to prioritize newest backups first.  
    - Configuration: Sort by field `path` descending.  
    - Connections: Outputs to Limits Backups node.  
    - Edge Cases: Sorting may fail if path values are missing or malformed.

  - **Limits Backups**  
    - Type: Code (JavaScript)  
    - Role: Implements retention logic by slicing the list to retain only backups beyond the 7 most recent (i.e., the older backups to be deleted).  
    - Configuration: JavaScript code `return items.slice(7);`  
    - Connections: Outputs to Nextcloud - Delete old backups node.  
    - Edge Cases: If fewer than 7 backups exist, result is an empty list, so no deletion occurs.  
    - Notes: Annotated as "Retention 7".

  - **Nextcloud - Delete old backups**  
    - Type: Nextcloud (Folder resource)  
    - Role: Deletes folders/files identified by the retention code node as older backups from Nextcloud.  
    - Configuration: Path dynamically set from JSON `path` property of each item.  
    - Credentials: Nextcloud API credentials.  
    - Connections: Terminal node (no output).  
    - Edge Cases: Deletion failures due to permissions, missing files, or connectivity issues.  
    - Notes: Marked "Delete old Backups".

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                        | Input Node(s)                    | Output Node(s)                         | Sticky Note                                       |
|---------------------------|------------------------|-------------------------------------|---------------------------------|--------------------------------------|--------------------------------------------------|
| On clicking 'execute'      | Manual Trigger         | Manual workflow start trigger       | —                               | Backup Path                          |                                                  |
| Schedule Trigger           | Schedule Trigger       | Scheduled workflow start trigger    | —                               | Backup Path                          |                                                  |
| Backup Path               | Set                    | Defines backup directory path       | On clicking 'execute', Schedule Trigger | Nextcloud Directory               | Backupdirectory                                   |
| Nextcloud Directory        | Nextcloud              | Creates backup directory in Nextcloud| Backup Path                    | n8n                                 | Create Directory                                  |
| n8n                       | n8n API                | Fetches all workflows               | Nextcloud Directory             | Loop Over Items                     |                                                  |
| Loop Over Items            | SplitInBatches         | Processes workflows one by one      | n8n                            | Convert to File, Nextcloud List Dir |                                                  |
| Convert to File            | ConvertToFile          | Converts workflow JSON to file      | Loop Over Items                 | Nextcloud Upload                    |                                                  |
| Nextcloud Upload           | Nextcloud              | Uploads workflow backup file        | Convert to File                 | Loop Over Items                    |                                                  |
| Nextcloud List Dir         | Nextcloud              | Lists existing backups folder       | Loop Over Items (secondary output) | Backups                        |                                                  |
| Backups                   | Sort                   | Sorts backup entries by path desc   | Nextcloud List Dir              | Limits Backups                    | Backup List                                       |
| Limits Backups             | Code                   | Selects backups older than 7 days   | Backups                        | Nextcloud - Delete old backups      | Retention 7                                      |
| Nextcloud - Delete old backups | Nextcloud          | Deletes old backups from Nextcloud  | Limits Backups                 | —                                  | Delete old Backups                                |
| Sticky Note               | Sticky Note            | Documentation note                  | —                               | —                                  | ## N8N Backup Flow\n### - Create /N8N-Backup Directory manual in Nextcloud\n### - Edit Backup Path |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Create a **Manual Trigger** node named `On clicking 'execute'` with default settings.
   - Create a **Schedule Trigger** node named `Schedule Trigger`. Set the trigger interval as desired (default is every minute).

2. **Create Backup Path Node:**

   - Add a **Set** node named `Backup Path`.
   - Configure it to assign a string variable `backup` with the value `/N8N-Backup/{{ $now.format('yyyy-MM-dd_HHmmss') }}`.
   - Set "Execute Once" to true.
   - Connect both trigger nodes (`On clicking 'execute'` and `Schedule Trigger`) outputs to this node.

3. **Create Nextcloud Directory Node:**

   - Add a **Nextcloud** node named `Nextcloud Directory`.
   - Configure it to create a folder resource.
   - Set the path parameter to `={{ $json.backup }}` to use the backup path from the previous node.
   - Select or create credentials for your Nextcloud account.
   - Connect output of `Backup Path` to this node.

4. **Create n8n API Node:**

   - Add an **n8n** node named `n8n`.
   - Configure to retrieve workflows with no filters.
   - Use your n8n API credentials.
   - Enable retry on failure.
   - Connect output of `Nextcloud Directory` to this node.

5. **Create Loop Over Items Node:**

   - Add a **SplitInBatches** node named `Loop Over Items`.
   - Use default batch size (1).
   - Connect output of `n8n` to this node.

6. **Create Convert to File Node:**

   - Add a **ConvertToFile** node named `Convert to File`.
   - Set operation to convert JSON.
   - Set filename to `={{ $json.name + ".json" }}`.
   - Connect main output of `Loop Over Items` to this node.

7. **Create Nextcloud Upload Node:**

   - Add a **Nextcloud** node named `Nextcloud Upload`.
   - Configure to upload a file (binary data upload enabled).
   - Set path to `={{ $('Backup Path').item.json.backup }}/{{ $binary.data.fileName }}`.
   - Use Nextcloud credentials.
   - Connect output of `Convert to File` to this node.
   - Connect output of `Nextcloud Upload` back to `Loop Over Items` to continue processing remaining items.

8. **Create Nextcloud List Dir Node:**

   - Add a **Nextcloud** node named `Nextcloud List Dir`.
   - Configure to list folder contents of path `/N8N-Backup`.
   - Use Nextcloud credentials.
   - Connect secondary output of `Loop Over Items` to this node.

9. **Create Backups Sort Node:**

   - Add a **Sort** node named `Backups`.
   - Configure to sort by field `path` descending.
   - Connect output of `Nextcloud List Dir` to this node.

10. **Create Limits Backups Code Node:**

    - Add a **Code** node named `Limits Backups`.
    - Use JavaScript code:
      ```js
      // Keep only backups from index 7 onward (older backups)
      return items.slice(7);
      ```
    - Connect output of `Backups` to this node.

11. **Create Nextcloud Delete Node:**

    - Add a **Nextcloud** node named `Nextcloud - Delete old backups`.
    - Configure to delete folder resource.
    - Set path to `={{ $json.path }}`.
    - Use Nextcloud credentials.
    - Connect output of `Limits Backups` to this node.

12. **Add Sticky Note:**

    - Add a **Sticky Note** node with content:
      ```
      ## N8N Backup Flow
      ### - Create /N8N-Backup Directory manual in Nextcloud
      ### - Edit Backup Path
      ```
    - Position near the start of the workflow for documentation.

13. **Verify Connections:**

    - Confirm the following connections:
      - `On clicking 'execute'` → `Backup Path`
      - `Schedule Trigger` → `Backup Path`
      - `Backup Path` → `Nextcloud Directory`
      - `Nextcloud Directory` → `n8n`
      - `n8n` → `Loop Over Items`
      - `Loop Over Items` → `Convert to File` → `Nextcloud Upload` → back to `Loop Over Items`
      - `Loop Over Items` (secondary output) → `Nextcloud List Dir` → `Backups` → `Limits Backups` → `Nextcloud - Delete old backups`

14. **Credential Setup:**

    - Ensure you have valid credentials for:
      - n8n API access (API key or OAuth as configured)
      - Nextcloud API (username/password or OAuth)
    - Assign these credentials to the respective nodes.

15. **Test Workflow:**

    - Run manually to verify backup creation.
    - Check Nextcloud for backup folder and files.
    - Monitor retention by verifying older backups are deleted after more than 7 exist.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Create `/N8N-Backup` directory manually in Nextcloud before running workflow or ensure permissions allow creation | Sticky note in workflow emphasizes manual directory creation or permissions setup for backup directory |
| The retention policy is set to keep only the 7 most recent backups, older ones are automatically deleted      | Code node "Limits Backups" enforces this logic                                                        |
| Retry logic enabled on n8n API node to improve robustness against transient failures                           | Prevents workflow interruption due to temporary API issues                                            |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---