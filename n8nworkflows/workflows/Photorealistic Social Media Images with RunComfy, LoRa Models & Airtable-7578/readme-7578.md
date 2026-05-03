Photorealistic Social Media Images with RunComfy, LoRa Models & Airtable

https://n8nworkflows.xyz/workflows/photorealistic-social-media-images-with-runcomfy--lora-models---airtable-7578


# Photorealistic Social Media Images with RunComfy, LoRa Models & Airtable

### 1. Workflow Overview

This workflow automates the generation of photorealistic social media images using RunComfy’s Flux Realism LoRa models, integrating with Airtable as a content source and status tracker. It particularly targets content creators or social media managers who want to automatically create styled images for posts based on structured input data, leveraging AI-generated imagery with LoRa models hosted on RunComfy servers.

The workflow is logically divided into these blocks:

- **1.1 Initialization and Input Acquisition:** Trigger initiation and fetching user/workflow identifiers.
- **1.2 Server Management:** Starting a RunComfy server, polling until it is ready, and deleting it after use.
- **1.3 Data Retrieval from Airtable:** Querying social media content data records to drive image generation.
- **1.4 Image Generation Request:** Preparing parameters and submitting image generation prompts to RunComfy.
- **1.5 Status Monitoring:** Polling RunComfy for generation status to determine completion.
- **1.6 Image Download and Airtable Update:** Downloading generated images and updating Airtable records with results.
- **1.7 Notification:** Sending a Telegram message on completion.
- **1.8 Utility and Control Nodes:** Helper nodes for looping, conditional checks, and wait timers.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Input Acquisition

**Overview:**  
This block triggers the workflow manually or via schedule, sets user and workflow version IDs to parameterize subsequent API calls.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Schedule Trigger  
- User ID (Set)  
- User and Workflow Version ID (Set)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point for manual start  
  - Configuration: No parameters  
  - Outputs: User ID node  
  - Failures: None typical

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Entry point for scheduled automation  
  - Configuration: Runs on an interval (default unspecified)  
  - Outputs: User and Workflow Version ID node  
  - Failures: None typical

- **User ID**  
  - Type: Set  
  - Role: Assigns static user ID string for API calls  
  - Configuration: user_id with placeholder `< User ID>`  
  - Outputs: Get RunComfy Workflows node  
  - Failures: None typical

- **User and Workflow Version ID**  
  - Type: Set  
  - Role: Assigns static user_id and workflow_version_id for server start  
  - Configuration: placeholders `< User ID>`, `<Version ID>`  
  - Outputs: RunComfy Start Server node  
  - Failures: Requires valid IDs or API calls will fail  

---

#### 2.2 Server Management

**Overview:**  
Starts a RunComfy server for image generation, polls until server is ready, and deletes the server after generating images to avoid costs.

**Nodes Involved:**  
- RunComfy Start Server (HTTP Request)  
- Wait for Server (Wait)  
- Check Server Status (HTTP Request)  
- Server Ready Check (If)  
- Delete RunComfy Server (HTTP Request)  

**Node Details:**

- **RunComfy Start Server**  
  - Type: HTTP Request  
  - Role: Starts a RunComfy server instance  
  - Configuration: POST to RunComfy API with estimated_duration (600s), server_type (large), workflow_version_id from input  
  - Authentication: HTTP Header Auth with RunComfy API key  
  - Outputs: Wait for Server node  
  - Failures: API errors, auth failures, invalid workflow_version_id

- **Wait for Server**  
  - Type: Wait  
  - Role: Delays next step for 50 seconds before polling server status  
  - Configuration: Fixed 50 seconds wait  
  - Outputs: Check Server Status node  
  - Failures: None typical

- **Check Server Status**  
  - Type: HTTP Request  
  - Role: Gets current status of the server instance  
  - Configuration: GET to RunComfy API using user_id and server_id from previous nodes  
  - Authentication: HTTP Header Auth with RunComfy API key  
  - Outputs: Server Ready Check node  
  - Failures: Network issues, server not found, auth failures

- **Server Ready Check**  
  - Type: If  
  - Role: Checks if server status is ‘Ready’ to proceed  
  - Configuration: Compares current_status field to string “Ready”  
  - Outputs: If ready, proceeds to Airtable data retrieval; else loops back to Wait for Server  
  - Failures: Missing or malformed status field can cause false negatives

- **Delete RunComfy Server**  
  - Type: HTTP Request  
  - Role: Deletes the RunComfy server instance to save costs  
  - Configuration: DELETE request with user_id and server_id, includes an x-api-key header  
  - Authentication: HTTP Header Auth  
  - Outputs: Final message notification node  
  - Failures: API errors, server already deleted, auth errors

---

#### 2.3 Data Retrieval from Airtable

**Overview:**  
Searches the Airtable base/table for social media content records which provide prompts and LoRa model names for image generation.

**Nodes Involved:**  
- Search All Records from Airtable (Airtable node)  
- Code (Code node)  

**Node Details:**

- **Search All Records from Airtable**  
  - Type: Airtable  
  - Role: Queries Airtable “Social Media Content Rep” table in base “appwOUIeUYM3o14eJ”  
  - Configuration: Search operation with no filters (fetches all records)  
  - Authentication: Airtable API token  
  - Outputs: Code node  
  - Failures: API limits, auth errors, no records found

- **Code**  
  - Type: Code (JavaScript)  
  - Role: Adds the server_id obtained from the RunComfy Start Server node to each Airtable record item  
  - Configuration: Obtains server_id from RunComfy Start Server node output, assigns to each item’s JSON  
  - Inputs: Airtable records and RunComfy Start Server output  
  - Outputs: Loop Over Items node  
  - Failures: If RunComfy Start Server node output missing or malformed, code will fail. Defensive coding minimal.

---

#### 2.4 Image Generation Request

**Overview:**  
For each Airtable record, sets RunComfy parameters including prompt and LoRa model, then sends a POST request to RunComfy to generate an image.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Set RunComfy Parameter (Set)  
- RunComfyFluxRealism (HTTP Request)  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes Airtable records one by one or in batches (batch size not specified)  
  - Configuration: Default options  
  - Inputs: Code node output (Airtable records with server_id)  
  - Outputs: Delete RunComfy Server and Set RunComfy Parameter nodes (parallel outputs)  
  - Failures: Large datasets may cause rate-limit issues

- **Set RunComfy Parameter**  
  - Type: Set  
  - Role: Constructs parameters needed for RunComfy image generation including positive prompt, LoRa model name, server_id, model name, and filename  
  - Configuration:  
    - positive_prompt = Airtable 'Prompt' field + pose_1 description from selected record  
    - lora_name = LoRa Name Flux from Airtable record  
    - server_id from Code node  
    - Model = Name field from Airtable  
    - Filename is either first two words of Name or the whole Name if only one word  
  - Outputs: RunComfyFluxRealism node  
  - Failures: Missing Airtable fields cause empty or invalid parameters

- **RunComfyFluxRealism**  
  - Type: HTTP Request  
  - Role: Sends the image generation request to RunComfy Flux Realism API  
  - Configuration:  
    - POST JSON body with nested objects describing model loaders, CLIP encoders, VAE, LoRa configurations (including two LoRa models), image size (832x1216), sampler, scheduler, guidance, noise seed (randomized)  
    - Uses server_id from parameters for the request URL  
    - Timeout set to 300 seconds (5 minutes)  
  - Authentication: HTTP Header Auth  
  - Outputs: Wait for Prompt node  
  - Failures: Timeout, malformed JSON, API errors, invalid server_id, invalid LoRa model names (must be uploaded to RunComfy)

---

#### 2.5 Status Monitoring

**Overview:**  
Waits a fixed time, then queries RunComfy server for image generation status to determine if the image is ready or still processing.

**Nodes Involved:**  
- Wait for Prompt (Wait)  
- ComfyUI Actual Status (HTTP Request)  
- Status Check2 (If)  

**Node Details:**

- **Wait for Prompt**  
  - Type: Wait  
  - Role: Delays 30 seconds before status check  
  - Outputs: ComfyUI Actual Status node  
  - Failures: None typical

- **ComfyUI Actual Status**  
  - Type: HTTP Request  
  - Role: Queries RunComfy API for the status of the image prompt using server_id and prompt_id  
  - Configuration:  
    - GET request to RunComfy history endpoint  
    - Timeout 30 seconds  
  - Authentication: HTTP Header Auth  
  - Outputs: Status Check2 node  
  - Failures: Network errors, prompt_id missing, auth errors

- **Status Check2**  
  - Type: If  
  - Role: Checks if the status string returned equals “success” indicating image generation complete  
  - Outputs:  
    - If success: Download Generated Image node  
    - Else: Wait for Prompt node (loop)  
  - Failures: Status field missing or changed format breaks condition

---

#### 2.6 Image Download and Airtable Update

**Overview:**  
Downloads the generated image file from RunComfy and updates the corresponding Airtable record with the generated image link and status.

**Nodes Involved:**  
- Download Generated Image (HTTP Request)  
- Update Airtable Record (Airtable node)  

**Node Details:**

- **Download Generated Image**  
  - Type: HTTP Request  
  - Role: Downloads the generated image file from RunComfy server using server_id and the image filename extracted from the status response  
  - Configuration:  
    - GET request to RunComfy view endpoint with filename parameter  
    - Response formatted as file  
  - Authentication: HTTP Header Auth  
  - Outputs: Update Airtable Record node  
  - Failures: File not found, network issues, auth errors

- **Update Airtable Record**  
  - Type: Airtable  
  - Role: Updates the Airtable record for the current item with fields such as “Bilder erstellt” (image created status) and drive photo links  
  - Configuration:  
    - Matches Airtable record by id from “Switch6” node item (not present in JSON, assumed external or prior)  
    - Updates multiple fields including pose_1_drive_fotolink with the new image URL  
  - Authentication: Airtable API token  
  - Outputs: Loop Over Items node (to continue next item)  
  - Failures: Airtable API limits, invalid record id, permission errors

---

#### 2.7 Notification

**Overview:**  
Sends a Telegram message notifying that images have been created for the given topic.

**Nodes Involved:**  
- Final message (Telegram)  

**Node Details:**

- **Final message**  
  - Type: Telegram  
  - Role: Sends a chat message to a defined Telegram chat ID  
  - Configuration:  
    - Text message includes the topic field from the Airtable record  
    - Chat ID is fixed (7754721939)  
    - Attribution disabled  
  - Authentication: Telegram API credentials  
  - Outputs: None  
  - Failures: Telegram API issues, invalid chat id

---

#### 2.8 Utility and Control Nodes

**Overview:**  
Nodes that perform control functions like splitting data batches, looping, and providing user instructions.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Sticky Note, Sticky Note1, Sticky Note3 (Documentation nodes)  

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over Airtable records one at a time for processing  
  - Outputs: Parallel paths to Delete RunComfy Server and Set RunComfy Parameter (likely logic to ensure cleanup while generating)  
  - Failures: Large datasets may cause throttling

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Contains textual instructions and tips for users  
  - Content Highlights:  
    - Server node config requirements: workflow server id, runtime length, server size  
    - Airtable linking instructions: required inputs (pose_1 and LoRa name)  
    - Workflow setup notes including RunComfy membership, credentials linking, and contact info for support  
  - Failures: None, purely informational

---

### 3. Summary Table

| Node Name                 | Node Type            | Functional Role                                | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                                                         |
|---------------------------|----------------------|-----------------------------------------------|----------------------------------|---------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Manual workflow start                          | -                                | User ID                                    |                                                                                                                     |
| Schedule Trigger          | Schedule Trigger     | Scheduled workflow start                        | -                                | User and Workflow Version ID                |                                                                                                                     |
| User ID                  | Set                  | Sets user_id for API calls                      | When clicking ‘Execute workflow’ | Get RunComfy Workflows                       |                                                                                                                     |
| User and Workflow Version ID | Set                  | Sets user_id and workflow_version_id            | Schedule Trigger                 | RunComfy Start Server                        |                                                                                                                     |
| Get RunComfy Workflows    | HTTP Request         | Retrieves user workflows                        | User ID                         | -                                           |                                                                                                                     |
| RunComfy Start Server     | HTTP Request         | Starts RunComfy server                          | User and Workflow Version ID     | Wait for Server                             | Server-Node: put workflow server id, time to run, size (min medium, better large or bigger)                         |
| Wait for Server           | Wait                 | Waits 50 seconds before status check           | RunComfy Start Server            | Check Server Status                         | Server-Node: put workflow server id, time to run, size (min medium, better large or bigger)                         |
| Check Server Status       | HTTP Request         | Checks if RunComfy server is ready             | Wait for Server                  | Server Ready Check                          | Server-Node: put workflow server id, time to run, size (min medium, better large or bigger)                         |
| Server Ready Check        | If                   | Conditional to proceed if server status=Ready | Check Server Status              | Search All Records from Airtable / Wait for Server | Server-Node: put workflow server id, time to run, size (min medium, better large or bigger)                   |
| Search All Records from Airtable | Airtable             | Retrieves records from Airtable                 | Server Ready Check               | Code                                        | Link your Airtable database: input pose_1 and exact LoRa Name for Flux Realism (must be uploaded to RunComfy)       |
| Code                     | Code                 | Adds server_id to Airtable records              | Search All Records from Airtable | Loop Over Items                             | Link your Airtable database: input pose_1 and exact LoRa Name for Flux Realism (must be uploaded to RunComfy)       |
| Loop Over Items           | SplitInBatches       | Processes each record batch-wise                | Code                           | Delete RunComfy Server, Set RunComfy Parameter | Link your Airtable database: input pose_1 and exact LoRa Name for Flux Realism (must be uploaded to RunComfy)       |
| Set RunComfy Parameter    | Set                  | Sets parameters for RunComfy prompt             | Loop Over Items                | RunComfyFluxRealism                         | Link your Airtable database: input pose_1 and exact LoRa Name for Flux Realism (must be uploaded to RunComfy)       |
| RunComfyFluxRealism       | HTTP Request         | Sends image generation request to RunComfy       | Set RunComfy Parameter          | Wait for Prompt                            |                                                                                                                     |
| Wait for Prompt           | Wait                 | Waits 30 seconds before checking prompt status | RunComfyFluxRealism             | ComfyUI Actual Status                       |                                                                                                                     |
| ComfyUI Actual Status     | HTTP Request         | Queries the image generation status            | Wait for Prompt                | Status Check2                              |                                                                                                                     |
| Status Check2             | If                   | Checks if generation completed                  | ComfyUI Actual Status          | Download Generated Image / Wait for Prompt |                                                                                                                     |
| Download Generated Image  | HTTP Request         | Downloads generated image file                   | Status Check2                   | Update Airtable Record                      |                                                                                                                     |
| Update Airtable Record    | Airtable             | Updates Airtable record with image info          | Download Generated Image        | Loop Over Items                             |                                                                                                                     |
| Delete RunComfy Server    | HTTP Request         | Deletes RunComfy server to save cost             | Loop Over Items                | Final message                              |                                                                                                                     |
| Final message             | Telegram             | Sends completion notification via Telegram      | Delete RunComfy Server          | -                                           |                                                                                                                     |
| Sticky Note               | Sticky Note          | Instructions on server parameters                | -                              | -                                           | Server-Node: put workflow server id, time to run, size (min medium, better large or bigger)                         |
| Sticky Note1              | Sticky Note          | Airtable linking instructions                     | -                              | -                                           | Link your Airtable database: input pose_1 and exact LoRa Name for Flux Realism (must be uploaded to RunComfy)       |
| Sticky Note3              | Sticky Note          | Workflow setup and support information            | -                              | -                                           | Info RUNCOMFY workflow, setup, credentials, contact hello@saits.ai, RunComfy membership required                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes**  
   - Add a **Manual Trigger** named “When clicking ‘Execute workflow’”. No parameters needed.  
   - Add a **Schedule Trigger** named “Schedule Trigger” with desired interval (e.g. daily).  

2. **Set User IDs**  
   - Add a **Set** node “User ID” with field `user_id` set to your RunComfy user ID string. Connect manual trigger to it.  
   - Add a **Set** node “User and Workflow Version ID” with fields:  
     - `user_id` set to your user ID  
     - `workflow_version_id` set to the RunComfy workflow version ID you want to use  
   Connect schedule trigger to this node.

3. **Get RunComfy Workflows (Optional)**  
   - Add an **HTTP Request** node “Get RunComfy Workflows” with GET URL:  
     `https://beta-api.runcomfy.net/prod/api/users/{{ $json.user_id }}/workflows`  
   - Use HTTP Header Auth with your RunComfy API key. Connect “User ID” node output to this node.

4. **Start RunComfy Server**  
   - Add an **HTTP Request** node “RunComfy Start Server” (POST) to:  
     `https://beta-api.runcomfy.net/prod/api/users/{{ $json.user_id }}/servers`  
   - JSON body parameters:  
     - `estimated_duration`: 600 (seconds or adjust)  
     - `server_type`: “large” (or medium, but large recommended)  
     - `workflow_version_id`: from previous Set node  
   - Use HTTP Header Auth with RunComfy API key. Connect “User and Workflow Version ID” node to this node.

5. **Wait for Server to be Ready**  
   - Add a **Wait** node “Wait for Server” with fixed wait time 50 seconds. Connect from RunComfy Start Server.  
   - Add an **HTTP Request** node “Check Server Status” (GET) to:  
     `https://beta-api.runcomfy.net/prod/api/users/{{ $json.user_id }}/servers/{{ $json.server_id }}`  
   - Use same auth as above. Connect “Wait for Server” to this node.  
   - Add an **If** node “Server Ready Check” to check if JSON field `current_status` equals “Ready”.  
   - If yes, continue to Airtable data retrieval; if no, loop back to “Wait for Server”.

6. **Retrieve Data from Airtable**  
   - Add an **Airtable** node “Search All Records from Airtable” configured to your base and table (e.g., “Social Media Content Rep”). Use search operation without filters to get all records. Authenticate with Airtable API token. Connect “Server Ready Check” true output to this node.

7. **Add Server ID to Each Airtable Record**  
   - Add a **Code** node “Code” with JavaScript to read `server_id` from the “RunComfy Start Server” node and assign to each record’s JSON object. Connect Airtable node output to this node.

8. **Batch Processing Loop**  
   - Add a **SplitInBatches** node “Loop Over Items” to process records one at a time (or set batch size). Connect from “Code” node.

9. **Set Parameters for RunComfy Prompt**  
   - Add a **Set** node “Set RunComfy Parameter” with fields:  
     - `positive_prompt`: Concatenate prompt from Airtable `Prompt` + `pose_1` description  
     - `lora_name`: from Airtable `LoRa Name Flux` field  
     - `server_id`: from “Code” node  
     - `Model`: from Airtable `Name` field  
     - `Filename`: first two words of Name or entire Name  
   Connect “Loop Over Items” output to this node.

10. **Send RunComfy Image Generation Request**  
    - Add **HTTP Request** node “RunComfyFluxRealism” (POST) to URL:  
      `https://{{ $json.server_id }}-comfyui.runcomfy.com/prompt`  
    - Body: JSON with detailed model and sampler configuration, including LoRa models, image size 832x1216, noise seed randomized, and other Flux Realism parameters as per existing workflow.  
    - Set header `Content-Type: application/json`.  
    - Use HTTP Header Auth with RunComfy API key.  
    - Connect “Set RunComfy Parameter” to this node.

11. **Wait and Poll for Prompt Completion**  
    - Add **Wait** node “Wait for Prompt” with 30 seconds wait. Connect from “RunComfyFluxRealism”.  
    - Add **HTTP Request** node “ComfyUI Actual Status” (GET) to:  
      `https://{{ $json.server_id }}-comfyui.runcomfy.com/history/{{ $json.prompt_id }}`  
    - Use same auth. Connect “Wait for Prompt” to this node.  
    - Add **If** node “Status Check2” to check if status string in response equals “success”.  
    - If yes, proceed to download image; else loop back to “Wait for Prompt”.

12. **Download Generated Image**  
    - Add **HTTP Request** node “Download Generated Image” (GET) to:  
      `https://{{ $json.server_id }}-comfyui.runcomfy.com/view?filename={{ filename }}&subfolder=&type=output`  
    - Set Response Format to “File”.  
    - Use HTTP Header Auth. Connect “Status Check2” true output to this node.

13. **Update Airtable Record**  
    - Add **Airtable** node “Update Airtable Record” configured to update the record by id with fields:  
      - “Bilder erstellt” status field (e.g. “Bild 1 erstellt”)  
      - Drive photo links updated with new image URL  
    - Authenticate with Airtable API token. Connect “Download Generated Image” to this node.  
    - Connect “Update Airtable Record” back to “Loop Over Items” to process next record.

14. **Delete RunComfy Server**  
    - Add **HTTP Request** node “Delete RunComfy Server” (DELETE) to:  
      `https://beta-api.runcomfy.net/prod/api/users/{{ user_id }}/servers/{{ server_id }}`  
    - Include header `x-api-key` with API key.  
    - Connect “Loop Over Items” output to this node (ensure deletion happens after processing).  

15. **Send Telegram Notification**  
    - Add **Telegram** node “Final message” to send a message indicating image creation completed for topic.  
    - Configure with chat ID and text using Airtable topic field.  
    - Connect “Delete RunComfy Server” to this node.

16. **Add Sticky Notes**  
    - Add sticky notes with instructions to assist users on server setup, Airtable linking, and workflow context.

17. **Credentials Setup**  
    - Configure credentials for:  
      - RunComfy HTTP Header Authentication (API key)  
      - Airtable API token  
      - Telegram Bot API token

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                            |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Server node requires these body parameters: 1. Workflow server ID, 2. Estimated runtime in seconds, 3. Server size (recommended “large” or bigger).                                                                                                               | Sticky Note near Server nodes                              |
| Airtable must contain fields: `pose_1` describing the image content and exact LoRa model name uploaded to RunComfy to ensure model loading works correctly.                                                                                                       | Sticky Note near Airtable nodes                            |
| Workflow requires RunComfy membership and API credentials; ensure you have valid API keys for RunComfy, Airtable, and Telegram configured in n8n.                                                                                                                | Sticky Note with setup info                                |
| Contact: hello@saits.ai for help with custom RunComfy workflows or multi-LoRa model setups.                                                                                                                                                                        | Sticky Note with contact info                              |
| Use RunComfy Flux Realism model and ensure the rgthree node is included in the RunComfy workflow to avoid crashes.                                                                                                                                                 | Sticky Note with workflow tips                             |
| Be cautious to manually delete RunComfy servers if workflow errors occur to avoid unnecessary costs.                                                                                                                                                                | Sticky Note warning on server management                   |

---

**Disclaimer:**  
The text provided is exclusively extracted from an automated workflow built with n8n, respecting all applicable content policies. It contains only legal and public data.