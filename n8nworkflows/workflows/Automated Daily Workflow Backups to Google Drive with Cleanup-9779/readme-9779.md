Automated Daily Workflow Backups to Google Drive with Cleanup

https://n8nworkflows.xyz/workflows/automated-daily-workflow-backups-to-google-drive-with-cleanup-9779


# Automated Daily Workflow Backups to Google Drive with Cleanup

### 1. Workflow Overview

This workflow automates daily backups of all n8n workflows to Google Drive, organizing them in dated folders and cleaning up older backup folders to save space. It is designed for use cases where automated, reliable, and clean archival of workflow configurations is required, ensuring both backup preservation and storage management.

The workflow consists of two parallel but functionally identical logical blocks (one in Spanish, one in English), each including:

- **1.1 Automatic Trigger:** Runs daily at 4:00 AM to kick off the backup process.
- **1.2 Backup Folder Creation:** Creates a new Google Drive folder named with the current day, time, and date to store backups.
- **1.3 Fetch Workflows:** Retrieves all workflows from the n8n instance via API.
- **1.4 Batch Processing:** Iterates over each workflow in batches.
- **1.5 Backup Conversion and Save:** Converts each workflow to a formatted JSON file and uploads it to the new backup folder.
- **1.6 Old Backup Cleanup:** Lists existing backup folders, filters out the current one, and deletes the older folders permanently.

The Spanish and English blocks are duplicates, presumably for localization or presentation purposes.

---

### 2. Block-by-Block Analysis

---

#### Block 1: Spanish Version Backup Process

---

##### Overview:

This block performs the automated daily backup of n8n workflows to Google Drive, triggered automatically at 4:00 AM. It creates a dated folder, backs up each workflow into JSON files, and cleans old backup folders.

##### Nodes Involved:

- Schedule Trigger1
- create new folder1
- n8n1
- Loop Over Items1
- Get folders1
- Filter1
- delete folder1
- Convert to File1
- Google Drive1
- Sticky Note3, Sticky Note4, Sticky Note5, Sticky Note7, Sticky Note8, Sticky Note9 (documentation nodes)

##### Node Details:

1. **Schedule Trigger1**
   - Type: Schedule Trigger
   - Role: Triggers the workflow daily at 4:00 AM.
   - Configuration: Interval set to trigger at hour 4.
   - Inputs: None (trigger node).
   - Outputs: Connects to "create new folder1".
   - Edge cases: Misconfiguration of trigger time; daylight saving time changes.
   - Sticky Note7 documents timing details.

2. **create new folder1**
   - Type: Google Drive
   - Role: Creates a new folder in Google Drive for storing backups.
   - Configuration: Folder name dynamically generated as "Workflow Backups [DayOfWeek Time dd-MM-yyyy]".
   - Drive ID: "My Drive".
   - Parent Folder ID: Fixed ID ("1dCNZ_oYzukeFsaKWhukX1l_Hsi57ns9m") representing the main "N8N" folder.
   - Inputs: From Schedule Trigger1.
   - Outputs: To "n8n1".
   - Credentials: Google Drive OAuth2.
   - Edge cases: Google Drive API rate limits, permission issues.
   - Sticky Note8 explains folder naming.

3. **n8n1**
   - Type: n8n API node
   - Role: Fetches all workflows from the connected n8n instance.
   - Configuration: No filters, requesting all workflows.
   - Credentials: n8n API credentials with proper base URL (e.g., https://your-instance/api/v1).
   - Inputs: From "create new folder1".
   - Outputs: To "Loop Over Items1".
   - Edge cases: API authentication failures, API downtime.
   - Sticky Note9 details credential setup.

4. **Loop Over Items1**
   - Type: SplitInBatches
   - Role: Iterates over each workflow item, processing them sequentially.
   - Configuration: Default batch size (process all or configured batches).
   - Inputs: From "n8n1".
   - Outputs: Two outputs:
     - First output to "Get folders1" (old backup cleanup path).
     - Second output to "Convert to File1" (backup save path).
   - Edge cases: Large number of workflows could cause long processing times.

5. **Get folders1**
   - Type: Google Drive
   - Role: Retrieves all folders inside the main backup folder ("N8N").
   - Configuration: Filters set to search only folders inside the given parent folder ID.
   - Inputs: From "Loop Over Items1" (first output).
   - Outputs: To "Filter1".
   - Credentials: Google Drive OAuth2.
   - Edge cases: API rate limits, incorrect folder ID.

6. **Filter1**
   - Type: Filter
   - Role: Filters out the current newly created backup folder from the list.
   - Configuration: Compares each folder's ID to the current backup folder ID; excludes the current folder.
   - Inputs: From "Get folders1".
   - Outputs: To "delete folder1".
   - Edge cases: Empty folder list, expression evaluation errors.

7. **delete folder1**
   - Type: Google Drive
   - Role: Deletes old backup folders permanently.
   - Configuration: Permanently deletes folders by ID.
   - Inputs: From "Filter1".
   - Outputs: None (end of cleanup path).
   - Credentials: Google Drive OAuth2 (different credential "HTA" noted).
   - OnError: Continue regular output (failures do not stop workflow).
   - Edge cases: Permissions to delete, network errors.

8. **Convert to File1**
   - Type: Convert To File
   - Role: Converts each workflow JSON to a formatted JSON file.
   - Configuration: Output format JSON with pretty formatting; file named as "[workflow-name].json".
   - Inputs: From "Loop Over Items1" (second output).
   - Outputs: To "Google Drive1".
   - Edge cases: Large workflow JSON size, invalid JSON.

9. **Google Drive1**
   - Type: Google Drive
   - Role: Uploads the JSON backup file to the newly created folder.
   - Configuration: File name set dynamically; folder ID from "create new folder1".
   - Inputs: From "Convert to File1".
   - Outputs: Back to "Loop Over Items1" to continue batch processing.
   - Credentials: Google Drive OAuth2.
   - Edge cases: File overwrite conflicts, upload failures.

10. **Sticky Notes**
    - Provide documentation on batch processing (Sticky Note3), conversion and saving (Sticky Note4), cleanup (Sticky Note5), trigger timing (Sticky Note7), folder creation (Sticky Note8), and API connection (Sticky Note9).

---

#### Block 2: English Version Backup Process

---

##### Overview:

This block is a functional duplicate of the Spanish block, performing the same operations but annotated in English for clarity or localization.

##### Nodes Involved:

- Schedule Trigger2
- create new folder2
- n8n2
- Loop Over Items2
- Get folders2
- Filter2
- delete folder2
- Convert to File2
- Google Drive2
- Sticky Note10, Sticky Note11, Sticky Note12, Sticky Note13, Sticky Note14, Sticky Note15

##### Node Details:

1. **Schedule Trigger2**
   - Same as Schedule Trigger1; triggers at 4:00 AM daily.
   - Sticky Note10 explains timing.

2. **create new folder2**
   - Creates backup folder in Google Drive with same naming convention.
   - Sticky Note11 explains folder creation.

3. **n8n2**
   - Fetches all workflows from n8n API.
   - Sticky Note12 gives credential setup details.

4. **Loop Over Items2**
   - Processes workflows in batches, splitting into cleanup and backup paths.
   - Sticky Note13 documents batch processing.

5. **Get folders2**
   - Lists all backup folders.
   - Sticky Note15 describes old backup cleanup.

6. **Filter2**
   - Filters out current folder.
   - Same logic as Filter1.

7. **delete folder2**
   - Deletes old folders permanently.
   - OnError continue.
   - Uses Google Drive credential "HTA".

8. **Convert to File2**
   - Converts workflow to formatted JSON file.
   - Sticky Note14 explains conversion.

9. **Google Drive2**
   - Uploads JSON file to backup folder.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                               | Input Node(s)       | Output Node(s)        | Sticky Note                                                                                                  |
|---------------------|---------------------|-----------------------------------------------|---------------------|-----------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger1    | Schedule Trigger    | Triggers workflow daily at 4:00 AM            | None                | create new folder1    | ## ⏰ Trigger Automático: Se activa todos los días a las 4:00 AM (Sticky Note7)                              |
| create new folder1   | Google Drive Folder | Creates dated backup folder in Google Drive   | Schedule Trigger1   | n8n1                  | ## 📁 Creación de Carpeta de Backup: Nombre con formato día hora fecha (Sticky Note8)                        |
| n8n1                 | n8n API             | Fetches all workflows from n8n instance       | create new folder1  | Loop Over Items1      | ## 🔌 Conexión con n8n API: URL /api/v1, credenciales (Sticky Note9)                                        |
| Loop Over Items1     | Split In Batches    | Iterates over workflows, splits to two paths  | n8n1                | Get folders1, Convert to File1 | ## 🔄 Procesamiento por Lotes: Dos rutas, limpieza y guardado (Sticky Note3)                                |
| Get folders1         | Google Drive Search | Lists all existing backup folders              | Loop Over Items1    | Filter1               | ## 🧹 Limpieza de Backups Antiguos: Obtiene carpetas existentes (Sticky Note5)                               |
| Filter1              | Filter              | Excludes current backup folder                 | Get folders1        | delete folder1        |                                                                                                              |
| delete folder1       | Google Drive Delete | Deletes old backup folders permanently         | Filter1             | None                  | ⚠️ Nota: Backups antiguos se eliminan automáticamente (Sticky Note5)                                        |
| Convert to File1     | Convert To File     | Converts workflow JSON to formatted file      | Loop Over Items1    | Google Drive1         | ## 📄 Conversión y Guardado: Archivo JSON legible (Sticky Note4)                                            |
| Google Drive1        | Google Drive File   | Uploads JSON backup file to new folder         | Convert to File1    | Loop Over Items1      | ## 📄 Conversión y Guardado: Guarda archivo JSON (Sticky Note4)                                             |
| Schedule Trigger2    | Schedule Trigger    | Triggers English workflow daily at 4:00 AM    | None                | create new folder2    | ## ⏰ Automatic Trigger: Runs every day at 4:00 AM (Sticky Note10)                                           |
| create new folder2   | Google Drive Folder | Creates dated backup folder                     | Schedule Trigger2   | n8n2                  | ## 📁 Backup Folder Creation: Folder naming explained (Sticky Note11)                                       |
| n8n2                 | n8n API             | Fetches all workflows                           | create new folder2  | Loop Over Items2      | ## 🔌 n8n API Connection: Credential details (Sticky Note12)                                                |
| Loop Over Items2     | Split In Batches    | Iterates workflows, splits to cleanup & backup| n8n2                | Get folders2, Convert to File2 | ## 🔄 Batch Processing: Two paths for cleanup and backup (Sticky Note13)                                    |
| Get folders2         | Google Drive Search | Lists backup folders                            | Loop Over Items2    | Filter2               | ## 🧹 Old Backup Cleanup: Retrieves existing folders (Sticky Note15)                                        |
| Filter2              | Filter              | Excludes current backup folder                  | Get folders2        | delete folder2        |                                                                                                              |
| delete folder2       | Google Drive Delete | Deletes old backup folders permanently          | Filter2             | None                  | ⚠️ Old backups are automatically deleted (Sticky Note15)                                                   |
| Convert to File2     | Convert To File     | Converts workflow to JSON file                   | Loop Over Items2    | Google Drive2         | ## 📄 Conversion and Saving: Converts to JSON file (Sticky Note14)                                          |
| Google Drive2        | Google Drive File   | Uploads JSON backup file                         | Convert to File2    | Loop Over Items2      | ## 📄 Conversion and Saving: Saves JSON file (Sticky Note14)                                                |
| Sticky Note3         | Sticky Note         | Explains batch processing logic                  |                     |                       | ## 🔄 Procesamiento por Lotes: Loop and two paths                                                            |
| Sticky Note4         | Sticky Note         | Explains conversion and saving                   |                     |                       | ## 📄 Conversión y Guardado: Convert and save workflow JSON                                                  |
| Sticky Note5         | Sticky Note         | Explains old backup cleanup                       |                     |                       | ## 🧹 Limpieza de Backups Antiguos: Get, filter, delete old folders                                         |
| Sticky Note7         | Sticky Note         | Explains trigger timing                           |                     |                       | ## ⏰ Trigger Automático: Daily at 4:00 AM                                                                  |
| Sticky Note8         | Sticky Note         | Explains folder creation                          |                     |                       | ## 📁 Creación de Carpeta de Backup: Folder naming format                                                   |
| Sticky Note9         | Sticky Note         | Explains n8n API credential setup                 |                     |                       | ## 🔌 Conexión con n8n API: API URL and credential notes                                                    |
| Sticky Note10        | Sticky Note         | Explains English trigger timing                   |                     |                       | ## ⏰ Automatic Trigger: Runs daily at 4:00 AM                                                              |
| Sticky Note11        | Sticky Note         | Explains English backup folder creation           |                     |                       | ## 📁 Backup Folder Creation: Folder naming explained                                                      |
| Sticky Note12        | Sticky Note         | Explains English n8n API connection                |                     |                       | ## 🔌 n8n API Connection: Credential details                                                                |
| Sticky Note13        | Sticky Note         | Explains English batch processing                  |                     |                       | ## 🔄 Batch Processing: Loop over workflows with two paths                                                  |
| Sticky Note14        | Sticky Note         | Explains English conversion and saving             |                     |                       | ## 📄 Conversion and Saving: Convert to JSON and save                                                       |
| Sticky Note15        | Sticky Note         | Explains English old backup cleanup                 |                     |                       | ## 🧹 Old Backup Cleanup: Get, filter, delete old folders                                                   |
| Sticky Note          | Sticky Note         | Marks Spanish language block                        |                     |                       | # Español                                                                                                   |
| Sticky Note1         | Sticky Note         | Marks English language block                        |                     |                       | # English                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

Follow these steps to recreate the workflow in n8n:

**Preparation:**

- Ensure Google Drive OAuth2 credentials are configured with access to the target Drive and folder permissions.
- Set up n8n API credentials with correct base URL (e.g., https://your-instance.n8n.io/api/v1) and API key/token.
- Have the parent Google Drive folder ID ready where backups will be stored (e.g., "1dCNZ_oYzukeFsaKWhukX1l_Hsi57ns9m").

---

#### Step 1: Schedule Trigger Node (Schedule Trigger1)

- Add node "Schedule Trigger".
- Set interval to trigger daily at hour 4 (4:00 AM).
- Name it "Schedule Trigger1".

---

#### Step 2: Create Backup Folder (create new folder1)

- Add "Google Drive" node.
- Set resource to "folder" and operation to "create".
- Name the folder dynamically:  
  `Workflow Backups {{ $now.format('cccc t dd-MM-yyyy') }}`  
  (e.g., "Workflow Backups Monday 4:00 PM 24-04-2024")
- Set parent folderId to your main backup folder ID ("N8N" folder).
- Assign Google Drive OAuth2 credentials.
- Connect output of Schedule Trigger1 to this node.

---

#### Step 3: Fetch Workflows (n8n1)

- Add "n8n" node.
- Configure with your n8n API credentials.
- No filters to fetch all workflows.
- Connect output of "create new folder1" to this node.

---

#### Step 4: Batch Processing Loop (Loop Over Items1)

- Add "SplitInBatches" node.
- Default batch size (or configure as needed).
- Connect output of "n8n1" to this node.

---

#### Step 5a: Old Backup Cleanup Branch

- From first output of "Loop Over Items1", add "Google Drive" node ("Get folders1").
- Configure to list all folders inside your main backup folder (same as parent folder ID).
- Connect output of "Loop Over Items1" (first output) to "Get folders1".

- Add "Filter" node ("Filter1").
- Configure condition to exclude the current backup folder by comparing folder ID with the ID from "create new folder1".
- Connect output of "Get folders1" to "Filter1".

- Add "Google Drive" node ("delete folder1").
- Set resource to "folder", operation to "deleteFolder".
- Use the filtered folder IDs from "Filter1" to delete.
- Set option to delete permanently.
- Use Google Drive credentials that have delete permissions.
- Connect output of "Filter1" to "delete folder1".
- Set onError to "continue" so workflow won't stop if deletion fails.

---

#### Step 5b: Backup Conversion and Save Branch

- From second output of "Loop Over Items1", add "Convert To File" node ("Convert to File1").
- Set operation to "toJson".
- Enable formatting option for readability.
- Set file name to: `{{$json.name + ".json"}}`.
- Connect output of "Loop Over Items1" (second output) to this node.

- Add "Google Drive" node ("Google Drive1").
- Set resource to "file".
- Set file name as dynamic from the previous node.
- Set folder ID to the newly created folder ID from "create new folder1".
- Use same Google Drive OAuth2 credentials.
- Connect output of "Convert to File1" to this node.

- Connect output of "Google Drive1" back to "Loop Over Items1" to continue batch processing.

---

#### Step 6: (Optional) Duplicate the above steps for the English block

- Repeat steps 1–5 with new nodes named Schedule Trigger2, create new folder2, n8n2, Loop Over Items2, Get folders2, Filter2, delete folder2, Convert to File2, Google Drive2.
- Use same configuration but English sticky notes.
- Connect nodes accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| For Google Drive folder ID, use the folder where you want backups stored (e.g., "N8N" folder).    | Google Drive UI or API to find folder ID                                                        |
| n8n API credentials require base URL in the form `https://your-instance.n8n.io/api/v1`.          | Important for successful API connection (see Sticky Note9 and Sticky Note12)                     |
| The workflow triggers automatically every day at 4:00 AM; you can adjust the time in the trigger.| Modify Schedule Trigger node parameter                                                          |
| Old backups are deleted permanently to save space; be cautious with deletion permissions.        | Google Drive permissions and API quotas may affect success                                      |
| Batch processing via SplitInBatches allows handling large numbers of workflows without overload. | n8n documentation on SplitInBatches node                                                        |
| The workflow includes comprehensive sticky notes explaining each step in both Spanish and English.| Helpful for maintenance and localization purposes                                               |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created with n8n, adhering strictly to content policies. It contains no illegal, offensive, or protected elements. All data handled is legal and public.