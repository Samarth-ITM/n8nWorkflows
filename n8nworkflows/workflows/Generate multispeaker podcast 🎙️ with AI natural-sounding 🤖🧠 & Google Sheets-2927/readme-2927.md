Generate multispeaker podcast 🎙️ with AI natural-sounding 🤖🧠 & Google Sheets

https://n8nworkflows.xyz/workflows/generate-multispeaker-podcast-----with-ai-natural-sounding--------google-sheets-2927


# Generate multispeaker podcast 🎙️ with AI natural-sounding 🤖🧠 & Google Sheets

### 1. Workflow Overview

This workflow automates the creation of multi-speaker podcasts by leveraging AI-powered text-to-speech technology integrated with Google Sheets and Google Drive. It is designed for content creators, marketers, and businesses that want to streamline podcast production by automatically converting a script into a natural-sounding audio file with distinct voices for each speaker.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Data Retrieval**: Manual trigger initiates the workflow and retrieves the podcast script from Google Sheets.
- **1.2 Data Aggregation and Formatting**: Aggregates rows from the sheet and formats the script into a single text string with speaker labels.
- **1.3 Audio Generation and Polling**: Sends the formatted text to an AI text-to-speech API to generate audio, then polls the API until the audio generation is complete.
- **1.4 Audio Download and Storage**: Retrieves the generated audio file URL, downloads the audio, and uploads it to Google Drive for storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Retrieval

- **Overview**: This block starts the workflow manually and retrieves the podcast script from Google Sheets.
- **Nodes Involved**:  
  - When clicking ‘Test workflow’  
  - Get Podcast text

- **Node Details**:

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow manually on user command.  
    - Configuration: Default manual trigger node, no parameters.  
    - Inputs: None  
    - Outputs: Triggers "Get Podcast text" node.  
    - Edge Cases: None typical; manual trigger requires user interaction.

  - **Get Podcast text**  
    - Type: Google Sheets  
    - Role: Retrieves the podcast script data from a specified Google Sheets document.  
    - Configuration: Configured to read rows from a Google Sheet containing columns for speaker names and their respective text lines.  
    - Expressions/Variables: Likely uses sheet ID and range parameters (not explicitly shown).  
    - Inputs: Trigger from manual node.  
    - Outputs: Passes data to "Get all rows" node.  
    - Edge Cases: Authentication errors with Google Sheets; empty or malformed data; API rate limits.

#### 2.2 Data Aggregation and Formatting

- **Overview**: Aggregates all rows from the sheet and formats the combined text into a single string with speaker labels for multi-speaker audio generation.
- **Nodes Involved**:  
  - Get all rows  
  - Full Podcast Text

- **Node Details**:

  - **Get all rows**  
    - Type: Aggregate  
    - Role: Combines multiple rows of speaker and text data into a single dataset for processing.  
    - Configuration: Aggregates data from the previous Google Sheets node.  
    - Inputs: Data from "Get Podcast text".  
    - Outputs: Passes aggregated data to "Full Podcast Text".  
    - Edge Cases: Empty datasets; aggregation failures.

  - **Full Podcast Text**  
    - Type: Code (JavaScript)  
    - Role: Processes aggregated data to format the podcast script into one string, prefixing each line with the speaker's name.  
    - Configuration: Custom JavaScript code that loops through rows, concatenates speaker names and their lines into a formatted string.  
    - Expressions/Variables: Uses input data fields for speaker and text columns; outputs a single formatted text string.  
    - Inputs: Aggregated rows from "Get all rows".  
    - Outputs: Passes formatted text to "Create Audio".  
    - Edge Cases: Missing speaker names or text; code errors; unexpected data formats.

#### 2.3 Audio Generation and Polling

- **Overview**: Sends the formatted text to an AI text-to-speech API to generate multi-speaker audio, then polls the API until the audio is ready.
- **Nodes Involved**:  
  - Create Audio  
  - Wait 60 sec.  
  - Get status  
  - Completed?

- **Node Details**:

  - **Create Audio**  
    - Type: HTTP Request  
    - Role: Sends a request to the AI API to generate the podcast audio with multiple voices.  
    - Configuration: POST request with payload including formatted text and voice parameters for each speaker.  
    - Inputs: Formatted text from "Full Podcast Text".  
    - Outputs: Passes response (likely including a job ID) to "Wait 60 sec."  
    - Edge Cases: API authentication errors; request timeouts; invalid payloads; insufficient API credits.

  - **Wait 60 sec.**  
    - Type: Wait  
    - Role: Pauses workflow execution for 60 seconds before polling the API status again.  
    - Configuration: Fixed 60-second wait.  
    - Inputs: Triggered after "Create Audio" and after each status check if not completed.  
    - Outputs: Triggers "Get status".  
    - Edge Cases: Workflow timeout if polling takes too long.

  - **Get status**  
    - Type: HTTP Request  
    - Role: Queries the AI API for the current status of the audio generation job.  
    - Configuration: GET request using job ID from "Create Audio" response.  
    - Inputs: Triggered after wait.  
    - Outputs: Passes status response to "Completed?" node.  
    - Edge Cases: API errors; invalid job ID; network issues.

  - **Completed?**  
    - Type: If  
    - Role: Checks if the audio generation status is "COMPLETED".  
    - Configuration: Condition evaluates the status field from "Get status" response.  
    - Inputs: Status data from "Get status".  
    - Outputs:  
      - If true: proceeds to "Get Url Audio".  
      - If false: loops back to "Wait 60 sec." for another polling cycle.  
    - Edge Cases: Status field missing or unexpected; infinite loops if job never completes.

#### 2.4 Audio Download and Storage

- **Overview**: Retrieves the URL of the generated audio, downloads the audio file, and uploads it to Google Drive.
- **Nodes Involved**:  
  - Get Url Audio  
  - Get File Audio  
  - Upload Audio

- **Node Details**:

  - **Get Url Audio**  
    - Type: HTTP Request  
    - Role: Retrieves the direct URL of the generated audio file from the API.  
    - Configuration: GET request using job ID or relevant identifier.  
    - Inputs: Triggered after "Completed?" node confirms completion.  
    - Outputs: Passes audio URL to "Get File Audio".  
    - Edge Cases: API errors; missing URL in response.

  - **Get File Audio**  
    - Type: HTTP Request  
    - Role: Downloads the audio file from the provided URL.  
    - Configuration: GET request with binary response expected.  
    - Inputs: Audio URL from "Get Url Audio".  
    - Outputs: Passes binary audio data to "Upload Audio".  
    - Edge Cases: Network errors; file not found; large file size causing timeouts.

  - **Upload Audio**  
    - Type: Google Drive  
    - Role: Uploads the downloaded audio file to a specified Google Drive folder.  
    - Configuration: Configured with Google Drive credentials and target folder ID.  
    - Inputs: Binary audio file from "Get File Audio".  
    - Outputs: None (end of workflow).  
    - Edge Cases: Google Drive authentication errors; insufficient permissions; quota limits.

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                          | Input Node(s)                 | Output Node(s)              | Sticky Note                                                                                      |
|-------------------------|-------------------|----------------------------------------|------------------------------|-----------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Starts the workflow manually            | None                         | Get Podcast text             |                                                                                                |
| Get Podcast text        | Google Sheets     | Retrieves podcast script from Google Sheets | When clicking ‘Test workflow’ | Get all rows                |                                                                                                |
| Get all rows            | Aggregate         | Aggregates rows from Google Sheets      | Get Podcast text              | Full Podcast Text            |                                                                                                |
| Full Podcast Text       | Code              | Formats aggregated data into single text string | Get all rows                 | Create Audio                |                                                                                                |
| Create Audio            | HTTP Request      | Sends text to AI API to generate audio | Full Podcast Text             | Wait 60 sec.                 | When you register for the API service you will get 1$ for free. For continuous work add API credits to your account. |
| Wait 60 sec.            | Wait              | Waits 60 seconds before polling status  | Create Audio, Completed?      | Get status                  |                                                                                                |
| Get status              | HTTP Request      | Checks audio generation status          | Wait 60 sec.                  | Completed?                  |                                                                                                |
| Completed?              | If                | Checks if audio generation is complete  | Get status                   | Get Url Audio (if yes), Wait 60 sec. (if no) |                                                                                                |
| Get Url Audio           | HTTP Request      | Retrieves URL of generated audio file   | Completed?                   | Get File Audio              |                                                                                                |
| Get File Audio          | HTTP Request      | Downloads audio file from URL            | Get Url Audio                | Upload Audio                |                                                                                                |
| Upload Audio            | Google Drive      | Uploads audio file to Google Drive       | Get File Audio               | None                       |                                                                                                |
| Sticky Note3            | Sticky Note       |                                        | None                         | None                       |                                                                                                |
| Sticky Note4            | Sticky Note       |                                        | None                         | None                       |                                                                                                |
| Sticky Note5            | Sticky Note       |                                        | None                         | None                       |                                                                                                |
| Sticky Note6            | Sticky Note       |                                        | None                         | None                       |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named "When clicking ‘Test workflow’".  
   - No special configuration needed.

2. **Add Google Sheets Node to Retrieve Podcast Script**  
   - Add a **Google Sheets** node named "Get Podcast text".  
   - Configure credentials for Google Sheets OAuth2.  
   - Set operation to "Read Rows".  
   - Specify the spreadsheet ID and the sheet name or range containing the podcast script.  
   - Ensure columns include speaker names and their corresponding text lines.  
   - Connect output of "When clicking ‘Test workflow’" to this node.

3. **Add Aggregate Node to Combine Rows**  
   - Add an **Aggregate** node named "Get all rows".  
   - Configure to aggregate all rows retrieved from Google Sheets into a single dataset.  
   - Connect output of "Get Podcast text" to this node.

4. **Add Code Node to Format Podcast Text**  
   - Add a **Code** node named "Full Podcast Text".  
   - Use JavaScript to iterate over aggregated rows and concatenate each speaker's name and text into a formatted string.  
   - Example logic: For each row, append `"SpeakerName: Text\n"` to a string.  
   - Output the concatenated string as a single field.  
   - Connect output of "Get all rows" to this node.

5. **Add HTTP Request Node to Create Audio**  
   - Add an **HTTP Request** node named "Create Audio".  
   - Configure as POST request to the AI text-to-speech API endpoint.  
   - Set authentication headers or API key as required.  
   - Include the formatted text from "Full Podcast Text" in the request body, specifying voice parameters for each speaker.  
   - Connect output of "Full Podcast Text" to this node.

6. **Add Wait Node to Pause Before Polling**  
   - Add a **Wait** node named "Wait 60 sec."  
   - Configure to wait for 60 seconds.  
   - Connect output of "Create Audio" to this node.

7. **Add HTTP Request Node to Get Status**  
   - Add an **HTTP Request** node named "Get status".  
   - Configure as GET request to the API endpoint that returns the status of the audio generation job.  
   - Use job ID from "Create Audio" response in the request URL or parameters.  
   - Connect output of "Wait 60 sec." to this node.

8. **Add If Node to Check Completion**  
   - Add an **If** node named "Completed?".  
   - Configure condition to check if the status field in "Get status" response equals "COMPLETED".  
   - Connect output of "Get status" to this node.

9. **Configure If Node Outputs**  
   - On **true** branch (audio generation completed), connect to "Get Url Audio".  
   - On **false** branch (not completed), connect back to "Wait 60 sec." to continue polling.

10. **Add HTTP Request Node to Get Audio URL**  
    - Add an **HTTP Request** node named "Get Url Audio".  
    - Configure as GET request to retrieve the audio file URL from the API, using job ID or relevant identifier.  
    - Connect true output of "Completed?" node to this node.

11. **Add HTTP Request Node to Download Audio File**  
    - Add an **HTTP Request** node named "Get File Audio".  
    - Configure as GET request to download the audio file from the URL obtained in "Get Url Audio".  
    - Set response format to binary.  
    - Connect output of "Get Url Audio" to this node.

12. **Add Google Drive Node to Upload Audio**  
    - Add a **Google Drive** node named "Upload Audio".  
    - Configure credentials for Google Drive OAuth2.  
    - Set operation to "Upload File".  
    - Specify the target folder ID in Google Drive.  
    - Use binary data from "Get File Audio" as the file content.  
    - Connect output of "Get File Audio" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| When you register for the API service you will get 1$ for free. For continuous work add API credits to your account. | Important for managing API usage and avoiding interruptions in audio generation.                 |
| This workflow is ideal for automating multi-speaker podcast production using AI text-to-speech. | Use case context for content creators, marketers, and businesses.                                |
| Google Sheets integration allows easy script management and updates.                             | Facilitates content editing without modifying the workflow.                                     |
| Google Drive storage ensures generated audio files are accessible and shareable.                 | Enables centralized storage and easy distribution of podcast audio files.                        |

---

This documentation provides a detailed, stepwise understanding of the workflow, enabling users and AI agents to reproduce, modify, and troubleshoot the multi-speaker podcast generation process effectively.