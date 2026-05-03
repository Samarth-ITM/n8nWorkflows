Generate Audio from Text Scripts using Self-Hosted Bark Model and Google Drive

https://n8nworkflows.xyz/workflows/generate-audio-from-text-scripts-using-self-hosted-bark-model-and-google-drive-4241


# Generate Audio from Text Scripts using Self-Hosted Bark Model and Google Drive

### 1. Workflow Overview

This workflow automates the generation of audio files from text scripts stored in Google Drive, leveraging a self-hosted Bark voice synthesis Python script. It is designed for scenarios where batches of text scripts need to be converted into narrated audio files and stored back into Google Drive for further use or distribution.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception**: Entry points to trigger the workflow, either externally with parameters or manually for testing.
- **1.2 Input Aggregation**: Collect and aggregate input folder IDs for scripts and target audio repository.
- **1.3 Script Retrieval**: Listing and batch processing of text script files from the specified Google Drive folder.
- **1.4 Script Download and Preparation**: Download each script file content and write it locally to a temporary file.
- **1.5 Audio Generation**: Invoke the self-hosted Python Bark model to synthesize audio from the script.
- **1.6 Audio Upload**: Read the generated audio file and upload it back to the specified Google Drive folder.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block provides two entry points to start the workflow: an external trigger with input parameters and a manual trigger for testing purposes.
- **Nodes Involved:** 
  - Start: External Trigger
  - Start: Manual Test
  - Test Values
- **Node Details:**

1. **Start: External Trigger**
   - Type: Execute Workflow Trigger
   - Role: Receives external triggers with inputs `id_repo_audio` and `id_repo_script`.
   - Configuration: Defines expected input variables to receive folder IDs for audio output and script input.
   - Connections: Outputs to Aggregate Inputs.
   - Edge Cases: Missing or invalid input variables can cause downstream failures.

2. **Start: Manual Test**
   - Type: Manual Trigger
   - Role: Allows manual initiation of the workflow for testing.
   - Configuration: No parameters; triggers workflow on user action.
   - Connections: Outputs to Test Values.

3. **Test Values**
   - Type: Set
   - Role: Provides hardcoded test values for the folder IDs for manual testing.
   - Configuration: Sets `id_repo_audio` and `id_repo_script` to specific Google Drive folder IDs.
   - Connections: Outputs to Aggregate Inputs.
   - Edge Cases: Hardcoded IDs must be valid and accessible.

#### 1.2 Input Aggregation

- **Overview:** Aggregates the received input folder IDs from either trigger into a single data structure to streamline downstream usage.
- **Nodes Involved:** 
  - Aggregate Inputs
- **Node Details:**

1. **Aggregate Inputs**
   - Type: Aggregate
   - Role: Collects `id_repo_audio` and `id_repo_script` into arrays (even if single values) to normalize data format.
   - Configuration: Aggregates fields `id_repo_audio` and `id_repo_script`.
   - Connections: Outputs to Get Scripts.
   - Edge Cases: Empty or missing inputs cause failure in subsequent data retrieval.

#### 1.3 Script Retrieval

- **Overview:** Retrieves the list of script files stored in the specified Google Drive folder and prepares them for batch processing.
- **Nodes Involved:** 
  - Get Scripts
  - Loop Scripts
- **Node Details:**

1. **Get Scripts**
   - Type: Google Drive
   - Role: Lists files in the folder identified by the aggregated `id_repo_script`.
   - Configuration: Uses folder ID dynamically from `Aggregate Inputs` node JSON output to list all files within.
   - Credentials: Google Drive OAuth2.
   - Connections: Outputs to Loop Scripts.
   - Edge Cases: Invalid folder ID, permission errors, empty folder yield no scripts.

2. **Loop Scripts**
   - Type: SplitInBatches
   - Role: Iterates over each script file one by one for sequential processing.
   - Configuration: Default batch size (1), no additional parameters.
   - Connections: 
     - Main output (empty) for loop control.
     - Second output to Download Script node.
   - Edge Cases: Large number of files will result in long processing times.

#### 1.4 Script Download and Preparation

- **Overview:** Downloads each script file from Google Drive and saves its content locally to a fixed temporary file.
- **Nodes Involved:** 
  - Download Script
  - Save Script
- **Node Details:**

1. **Download Script**
   - Type: Google Drive
   - Role: Downloads the current script file content by file ID.
   - Configuration: Uses dynamic file ID from the current loop item.
   - Credentials: Google Drive OAuth2.
   - Connections: Outputs to Save Script.
   - Edge Cases: File missing or permission denied errors.

2. **Save Script**
   - Type: ReadWriteFile
   - Role: Writes the downloaded script content to `/tmp/script.txt` to prepare for audio generation.
   - Configuration: Write operation, data taken from `data` property of previous node output.
   - Connections: Outputs to Generate WAV.
   - Edge Cases: File system write permissions, disk space issues.

#### 1.5 Audio Generation

- **Overview:** Executes a command-line Python script that uses the Bark model to generate a WAV audio file from the text script.
- **Nodes Involved:** 
  - Generate WAV
- **Node Details:**

1. **Generate WAV**
   - Type: Execute Command
   - Role: Runs the self-hosted Python voice synthesis script `generate_voice.py`.
   - Configuration: Command template dynamically constructs output WAV filename based on script name, replacing `.txt` extension.
   - Command example: `/opt/venv/bin/python /scripts/generate_voice.py /tmp/script.txt /tmp/output_<script_name>.wav`
   - Connections: Outputs to Read Audio.
   - Edge Cases: Python environment or dependencies missing, script errors, timeout, invalid input script content.

#### 1.6 Audio Upload

- **Overview:** Reads the generated WAV file from the local filesystem and uploads it to the specified Google Drive folder.
- **Nodes Involved:** 
  - Read Audio
  - Upload Audio
- **Node Details:**

1. **Read Audio**
   - Type: ReadWriteFile
   - Role: Reads the generated WAV audio file from `/tmp/output_<script_name>.wav`.
   - Configuration: File path dynamically resolves from current loop script name.
   - Connections: Outputs to Upload Audio.
   - Edge Cases: File not found if generation failed.

2. **Upload Audio**
   - Type: Google Drive
   - Role: Uploads the audio file to the Google Drive folder identified by `id_repo_audio`.
   - Configuration: Uses folder ID from aggregated inputs, filename from file metadata.
   - Credentials: Google Drive OAuth2.
   - Connections: Outputs back to Loop Scripts to continue processing next script.
   - Edge Cases: Insufficient permissions, quota limits, file overwrite conflicts.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                     | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                          |
|---------------------|----------------------------|-----------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Start: External Trigger | Execute Workflow Trigger | External workflow start with inputs | —                      | Aggregate Inputs        |                                                                                                    |
| Start: Manual Test   | Manual Trigger             | Manual workflow start for testing | —                      | Test Values             |                                                                                                    |
| Test Values         | Set                        | Provides test folder IDs           | Start: Manual Test      | Aggregate Inputs        |                                                                                                    |
| Aggregate Inputs    | Aggregate                  | Aggregates input folder IDs        | Start: External Trigger, Test Values | Get Scripts       |                                                                                                    |
| Get Scripts         | Google Drive               | Lists script files in folder       | Aggregate Inputs        | Loop Scripts            |                                                                                                    |
| Loop Scripts        | SplitInBatches             | Processes scripts one by one       | Get Scripts, Upload Audio | Download Script      |                                                                                                    |
| Download Script     | Google Drive               | Downloads script content            | Loop Scripts            | Save Script             |                                                                                                    |
| Save Script         | ReadWriteFile              | Saves script content locally       | Download Script         | Generate WAV            |                                                                                                    |
| Generate WAV        | Execute Command            | Runs Python Bark audio synthesis   | Save Script             | Read Audio              |                                                                                                    |
| Read Audio          | ReadWriteFile              | Reads generated WAV audio          | Generate WAV            | Upload Audio            |                                                                                                    |
| Upload Audio        | Google Drive               | Uploads audio to Google Drive      | Read Audio              | Loop Scripts            |                                                                                                    |
| 📝 Workflow Instructions | Sticky Note            | Describes workflow purpose, inputs | —                      | —                      | “## Audio Generation Workflow\n\n1. This workflow automates the conversion of text scripts into audio files using a Python voice synthesis script.\n2. It retrieves scripts from Google Drive, generates audio with Bark-style narration, and reuploads the result.\n3. Script filenames must be .txt. Only clean plain text is supported.\n\n### Inputs:\n- id_repo_script: Google Drive folder with scripts\n- id_repo_audio: Google Drive folder to upload audio files\n\nMake sure the Python script is deployed at /scripts/generate_voice.py and has access to Bark or other required models.” |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Start Nodes**
   - Add an **Execute Workflow Trigger** node named `Start: External Trigger`.
     - Configure inputs: `id_repo_audio`, `id_repo_script` (string).
   - Add a **Manual Trigger** node named `Start: Manual Test`.

2. **Add Test Values Node**
   - Add a **Set** node named `Test Values`.
   - Set fields:
     - `id_repo_audio`: string, e.g. `"1wrnIskVWQJRAvKs04bhXavgDv2FokabG"`
     - `id_repo_script`: string, e.g. `"12tIJJuO_DfwJbtOuO72mpk2_3s9O4xZc"`
   - Connect `Start: Manual Test` to `Test Values`.

3. **Add Aggregate Inputs Node**
   - Add an **Aggregate** node named `Aggregate Inputs`.
   - Configure to aggregate fields:
     - `id_repo_audio`
     - `id_repo_script`
   - Connect outputs of both `Start: External Trigger` and `Test Values` to `Aggregate Inputs`.

4. **Add Get Scripts Node**
   - Add a **Google Drive** node named `Get Scripts`.
   - Operation: List files in folder.
   - Set folder ID to expression: `={{ $('Aggregate Inputs').item.json.id_repo_script[0] }}`
   - Use Google Drive OAuth2 credentials.
   - Connect `Aggregate Inputs` to `Get Scripts`.

5. **Add Loop Scripts Node**
   - Add a **SplitInBatches** node named `Loop Scripts`.
   - Default batch size (1).
   - Connect `Get Scripts` to `Loop Scripts`.

6. **Add Download Script Node**
   - Add a **Google Drive** node named `Download Script`.
   - Operation: Download file.
   - File ID set dynamically: `={{ $json.id }}`
   - Use Google Drive OAuth2 credentials.
   - Connect second output of `Loop Scripts` to `Download Script`.

7. **Add Save Script Node**
   - Add a **ReadWriteFile** node named `Save Script`.
   - Operation: Write.
   - File path: `/tmp/script.txt`
   - Data Property Name: `=data` (from download node output).
   - Connect `Download Script` to `Save Script`.

8. **Add Generate WAV Node**
   - Add an **Execute Command** node named `Generate WAV`.
   - Command:
     ```
     =/opt/venv/bin/python /scripts/generate_voice.py /tmp/script.txt /tmp/output_{{ $json.name.replace(/\.txt$/, '') }}.wav
     ```
   - Connect `Save Script` to `Generate WAV`.

9. **Add Read Audio Node**
   - Add a **ReadWriteFile** node named `Read Audio`.
   - Operation: Read.
   - File Selector path:
     ```
     =/tmp/output_{{ $('Loop Scripts').item.json.name.replace(/\.txt$/, '.wav') }}
     ```
   - Connect `Generate WAV` to `Read Audio`.

10. **Add Upload Audio Node**
    - Add a **Google Drive** node named `Upload Audio`.
    - Operation: Upload file.
    - File name: `={{ $json.fileName }}`
    - Folder ID:
      ```
      ={{ $('Aggregate Inputs').item.json.id_repo_audio[0] }}
      ```
    - Drive ID: Select "My Drive" or appropriate drive.
    - Use Google Drive OAuth2 credentials.
    - Connect `Read Audio` to `Upload Audio`.
    - Connect output of `Upload Audio` back to the first output of `Loop Scripts` to continue the batch loop.

11. **Add Workflow Instructions**
    - Add a **Sticky Note** node named `📝 Workflow Instructions`.
    - Paste the workflow description content for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                              |
|----------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| This workflow requires a self-hosted Python script at `/scripts/generate_voice.py` with access to Bark or a similar voice model. | Workflow Instructions sticky note            |
| Script files must be plain text with `.txt` extension for proper processing.                                                     | Workflow Instructions sticky note            |
| Google Drive OAuth2 credentials must have read/write access to the specified folders.                                            | Google Drive nodes credential configuration  |
| Temporary files are written to `/tmp/` directory; ensure the execution environment permits this and has sufficient disk space.   | General workflow environment requirement     |
| The voice synthesis command assumes a Python virtual environment at `/opt/venv/bin/python`. Adjust if different.                | Generate WAV node configuration               |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. It complies fully with applicable content policies and contains no illegal or protected information. All data handled is legal and public.