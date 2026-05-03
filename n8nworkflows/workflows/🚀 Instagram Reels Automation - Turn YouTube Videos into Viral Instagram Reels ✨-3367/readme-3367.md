🚀 Instagram Reels Automation - Turn YouTube Videos into Viral Instagram Reels ✨

https://n8nworkflows.xyz/workflows/---instagram-reels-automation---turn-youtube-videos-into-viral-instagram-reels---3367


# 🚀 Instagram Reels Automation - Turn YouTube Videos into Viral Instagram Reels ✨

### 1. Workflow Overview

This workflow automates the transformation of YouTube videos into viral Instagram Reels by leveraging AI-powered editing and scheduling. It is designed for content creators, podcasters, marketers, and brands who want to repurpose long-form videos into engaging short clips optimized for Instagram without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input including the YouTube video URL and clip preferences.
- **1.2 Reels Preparation & Processing:** Sends the input to Spikes Studio API to generate reels, waits for processing completion, and downloads the reels.
- **1.3 Reels Filtering & Structuring:** Organizes the generated reels, filters the best ones for upload, and prepares them for posting.
- **1.4 Scheduled Upload to Instagram:** Uploads reels one at a time to Instagram on a customizable schedule.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving the YouTube video URL and clip length from the user via a form trigger.

**Nodes Involved:**  
- Fill information to Start

**Node Details:**

- **Fill information to Start**  
  - Type: Form Trigger  
  - Role: Entry point capturing user input (YouTube URL, clip length) to start the workflow.  
  - Configuration: Uses a webhook to receive form submissions; expects parameters such as video URL and clip length.  
  - Inputs: External HTTP request (form submission)  
  - Outputs: Passes data to "Prepare Reels" node  
  - Edge Cases: Missing or invalid URL input, malformed clip length values, webhook connectivity issues.

---

#### 1.2 Reels Preparation & Processing

**Overview:**  
This block sends the YouTube video URL and clip preferences to the Spikes Studio API to generate reels, waits for the processing to complete, and downloads the reels when ready.

**Nodes Involved:**  
- Prepare Reels  
- Wait #1  
- Download Reels When Ready  
- Reels Ready?

**Node Details:**

- **Prepare Reels**  
  - Type: HTTP Request  
  - Role: Sends a request to Spikes Studio API to start reel generation based on input parameters.  
  - Configuration: Configured with Spikes Studio API endpoint, includes API key in headers, and sends video URL and clip length in the request body.  
  - Inputs: Data from "Fill information to Start"  
  - Outputs: Triggers "Wait #1" node  
  - Edge Cases: API authentication failure, invalid API response, network timeouts.

- **Wait #1**  
  - Type: Wait  
  - Role: Pauses workflow execution to allow Spikes Studio time to process reels.  
  - Configuration: Default or user-configured wait time (e.g., several minutes).  
  - Inputs: From "Prepare Reels" or "Reels Ready?" (if reels not ready)  
  - Outputs: Triggers "Download Reels When Ready"  
  - Edge Cases: Excessive wait time, premature continuation if processing incomplete.

- **Download Reels When Ready**  
  - Type: HTTP Request  
  - Role: Polls Spikes Studio API to check if reels are ready and downloads metadata or links.  
  - Configuration: Uses API endpoint to check status, includes API key.  
  - Inputs: From "Wait #1"  
  - Outputs: Conditional branch to "Reels Ready?" node  
  - Edge Cases: API rate limits, incomplete data, status errors.

- **Reels Ready?**  
  - Type: If  
  - Role: Checks if reels are ready based on API response.  
  - Configuration: Evaluates a boolean or status field in API response.  
  - Inputs: From "Download Reels When Ready"  
  - Outputs:  
    - True: Proceeds to "Structure Reels"  
    - False: Loops back to "Wait #1" for another wait cycle  
  - Edge Cases: Incorrect status parsing, infinite loop if reels never become ready.

---

#### 1.3 Reels Filtering & Structuring

**Overview:**  
This block structures the reels data, filters the best reels for upload, and prepares them for sequential posting.

**Nodes Involved:**  
- Structure Reels  
- Filter the Best Reels to Upload  
- Send Reels 1 at a time

**Node Details:**

- **Structure Reels**  
  - Type: Split Out  
  - Role: Splits the reels array into individual reel items for processing.  
  - Configuration: Splits JSON array output from previous node into separate items.  
  - Inputs: From "Reels Ready?" (true branch)  
  - Outputs: Passes individual reels to "Filter the Best Reels to Upload"  
  - Edge Cases: Empty or malformed reels array.

- **Filter the Best Reels to Upload**  
  - Type: Code (JavaScript)  
  - Role: Applies custom logic to select the best reels based on criteria (e.g., engagement metrics, length).  
  - Configuration: Contains JavaScript code filtering reels.  
  - Inputs: From "Structure Reels"  
  - Outputs: Passes filtered reels to "Send Reels 1 at a time"  
  - Edge Cases: Code errors, empty filtered results.

- **Send Reels 1 at a time**  
  - Type: Split In Batches  
  - Role: Processes reels sequentially, sending one reel at a time downstream.  
  - Configuration: Batch size set to 1 to ensure sequential processing.  
  - Inputs: From "Filter the Best Reels to Upload"  
  - Outputs:  
    - Main output: Empty (used to control flow)  
    - Second output: Passes current reel to "Download the Reels in the Right Format"  
  - Edge Cases: Batch processing errors, empty input.

---

#### 1.4 Scheduled Upload to Instagram

**Overview:**  
This block downloads the reel in the correct format and posts it to Instagram, then waits before processing the next reel to control upload frequency.

**Nodes Involved:**  
- Download the Reels in the Right Format  
- Post To Instagram  
- Wait #2

**Node Details:**

- **Download the Reels in the Right Format**  
  - Type: HTTP Request  
  - Role: Downloads the video file or reel in the format required by Instagram.  
  - Configuration: Uses URL from reel metadata, handles file download.  
  - Inputs: From "Send Reels 1 at a time" (second output)  
  - Outputs: Passes file to "Post To Instagram"  
  - Edge Cases: Download failures, unsupported formats.

- **Post To Instagram**  
  - Type: HTTP Request  
  - Role: Uploads the reel to Instagram via API (likely Upload-Post API).  
  - Configuration: Includes authentication credentials (OAuth2 or API key), sends video file and metadata (title, description, hashtags).  
  - Inputs: From "Download the Reels in the Right Format"  
  - Outputs: Triggers "Wait #2"  
  - Edge Cases: Authentication errors, API rate limits, upload failures.

- **Wait #2**  
  - Type: Wait  
  - Role: Delays the workflow before processing the next reel to control posting intervals.  
  - Configuration: User-configurable delay to space out posts.  
  - Inputs: From "Post To Instagram"  
  - Outputs: Loops back to "Send Reels 1 at a time" to process next reel batch  
  - Edge Cases: Excessive delay, workflow timeout.

---

### 3. Summary Table

| Node Name                      | Node Type            | Functional Role                          | Input Node(s)               | Output Node(s)                 | Sticky Note |
|--------------------------------|----------------------|----------------------------------------|-----------------------------|-------------------------------|-------------|
| Fill information to Start       | Form Trigger         | Entry point to receive YouTube URL and clip length | -                           | Prepare Reels                  |             |
| Prepare Reels                  | HTTP Request         | Sends request to Spikes Studio API to generate reels | Fill information to Start    | Wait #1                       |             |
| Wait #1                       | Wait                 | Waits for reels processing to complete | Prepare Reels, Reels Ready?  | Download Reels When Ready      |             |
| Download Reels When Ready       | HTTP Request         | Polls API to check if reels are ready  | Wait #1                     | Reels Ready?                  |             |
| Reels Ready?                  | If                   | Checks if reels are ready               | Download Reels When Ready    | Structure Reels (true), Wait #1 (false) |             |
| Structure Reels               | Split Out            | Splits reels array into individual reels | Reels Ready? (true)          | Filter the Best Reels to Upload |             |
| Filter the Best Reels to Upload | Code                 | Filters reels based on custom criteria | Structure Reels             | Send Reels 1 at a time         |             |
| Send Reels 1 at a time         | Split In Batches     | Processes reels sequentially            | Filter the Best Reels to Upload | Download the Reels in the Right Format (second output) |             |
| Download the Reels in the Right Format | HTTP Request         | Downloads reel video in Instagram format | Send Reels 1 at a time (second output) | Post To Instagram             |             |
| Post To Instagram             | HTTP Request         | Uploads reel to Instagram                | Download the Reels in the Right Format | Wait #2                       |             |
| Wait #2                       | Wait                 | Waits between uploads to control schedule | Post To Instagram           | Send Reels 1 at a time         |             |
| Sticky Note                   | Sticky Note          | Visual notes (empty content in this workflow) | -                           | -                             |             |
| Sticky Note1                  | Sticky Note          | Visual notes (empty content)             | -                           | -                             |             |
| Sticky Note3                  | Sticky Note          | Visual notes (empty content)             | -                           | -                             |             |
| Sticky Note4                  | Sticky Note          | Visual notes (empty content)             | -                           | -                             |             |
| Sticky Note5                  | Sticky Note          | Visual notes (empty content)             | -                           | -                             |             |
| Sticky Note6                  | Sticky Note          | Visual notes (empty content)             | -                           | -                             |             |
| Sticky Note7                  | Sticky Note          | Visual notes (empty content)             | -                           | -                             |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Name: Fill information to Start  
   - Type: Form Trigger  
   - Configure webhook to accept inputs:  
     - YouTube video URL (string)  
     - Desired clip length (number or string)  
   - Save and activate webhook.

2. **Create HTTP Request Node for Spikes Studio API**  
   - Name: Prepare Reels  
   - Type: HTTP Request  
   - Method: POST  
   - URL: Spikes Studio API endpoint for reel generation  
   - Authentication: Use API key credential for Spikes Studio  
   - Body: Include YouTube URL and clip length from form trigger data  
   - Output: Always output data  
   - Connect input from "Fill information to Start".

3. **Create Wait Node**  
   - Name: Wait #1  
   - Type: Wait  
   - Configure delay (e.g., 5 minutes or configurable)  
   - Connect input from "Prepare Reels".

4. **Create HTTP Request Node to Poll Reels Status**  
   - Name: Download Reels When Ready  
   - Type: HTTP Request  
   - Method: GET or POST as per API spec  
   - URL: Spikes Studio API endpoint to check reel status  
   - Authentication: Use Spikes Studio API key  
   - Connect input from "Wait #1".

5. **Create If Node to Check Reels Readiness**  
   - Name: Reels Ready?  
   - Type: If  
   - Condition: Check if API response indicates reels are ready (e.g., status == "ready")  
   - True output: Connect to "Structure Reels"  
   - False output: Connect back to "Wait #1" to repeat polling.

6. **Create Split Out Node**  
   - Name: Structure Reels  
   - Type: Split Out  
   - Purpose: Split reels array into individual reel items  
   - Connect input from "Reels Ready?" (true branch).

7. **Create Code Node to Filter Reels**  
   - Name: Filter the Best Reels to Upload  
   - Type: Code (JavaScript)  
   - Logic: Implement filtering criteria (e.g., engagement metrics, clip length)  
   - Connect input from "Structure Reels".

8. **Create Split In Batches Node**  
   - Name: Send Reels 1 at a time  
   - Type: Split In Batches  
   - Batch Size: 1  
   - Connect input from "Filter the Best Reels to Upload".

9. **Create HTTP Request Node to Download Reel Video**  
   - Name: Download the Reels in the Right Format  
   - Type: HTTP Request  
   - Method: GET  
   - URL: Use reel video URL from batch item  
   - Connect input from second output of "Send Reels 1 at a time".

10. **Create HTTP Request Node to Post to Instagram**  
    - Name: Post To Instagram  
    - Type: HTTP Request  
    - Method: POST  
    - URL: Upload-Post or Instagram API endpoint for uploading reels  
    - Authentication: Use Upload-Post API key or OAuth2 credentials  
    - Body: Include video file and metadata (title, description, hashtags)  
    - Connect input from "Download the Reels in the Right Format".

11. **Create Wait Node for Scheduling**  
    - Name: Wait #2  
    - Type: Wait  
    - Configure delay to control posting frequency (e.g., 1 hour)  
    - Connect input from "Post To Instagram".

12. **Loop Back to Process Next Reel**  
    - Connect output of "Wait #2" back to main input of "Send Reels 1 at a time" to continue uploading reels sequentially.

13. **Add Sticky Notes (Optional)**  
    - Add visual sticky notes for documentation or instructions as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                      |
|------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Spikes Studio API is required for reel generation and editing. Free account available at spikes.studio | https://spikes.studio                                |
| Upload-Post API is used for Instagram uploading; free account available at upload-post.com     | https://www.upload-post.com                          |
| Adjust wait times in "Wait #1" and "Wait #2" nodes to optimize processing and posting intervals | Scheduling control                                  |
| Customize reel filtering logic in the "Filter the Best Reels to Upload" code node as needed    | Customizable filtering criteria                      |
| This workflow supports easy extension to other platforms like TikTok, Facebook, or LinkedIn by adding corresponding upload nodes | Multi-platform support                               |

---

This documentation provides a detailed, structured understanding of the Instagram Reels Automation workflow, enabling users and AI agents to comprehend, reproduce, and modify the automation confidently.