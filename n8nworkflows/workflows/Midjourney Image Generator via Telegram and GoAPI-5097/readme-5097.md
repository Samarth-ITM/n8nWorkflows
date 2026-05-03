Midjourney Image Generator via Telegram and GoAPI

https://n8nworkflows.xyz/workflows/midjourney-image-generator-via-telegram-and-goapi-5097


# Midjourney Image Generator via Telegram and GoAPI

---

### 1. Workflow Overview

This workflow, titled **"Midjourney Image Generator via Telegram and GoAPI"**, orchestrates the generation and upscaling of AI-generated images using the Midjourney model via GoAPI, triggered by user input through Telegram. It is designed for users who want to create images by sending prompts through Telegram and optionally upscale selected images, with status notifications and logs sent back to Telegram and Discord.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation**: Receives user messages from Telegram, filters valid prompts, and prepares session data.
- **1.2 Image Generation Request**: Sends a generation task to GoAPI using the Midjourney model, notifies the user, and polls for task completion.
- **1.3 Image Selection and Upscaling**: Upon generation completion, sends images back to the user for selection, requests upscaling of the chosen image, notifies the user, and polls for upscale completion.
- **1.4 Final Notification and Logging**: Notifies the user with final upscaled image links and logs key information to Discord and Telegram for record-keeping.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

- **Overview:**  
  This block captures incoming Telegram messages, validates the username and message content, and sets up necessary variables for session management.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Get Prompt (IF)  
  - Task (Set)

- **Node Details:**

  - **Telegram Trigger**  
    - *Type & Role:* Telegram Trigger node; entry point capturing any new Telegram messages (`message` updates).  
    - *Configuration:* Listens for message updates only; uses Telegram API credentials linked to a bot.  
    - *Expressions/Variables:* Accesses `$json.message.text` and `$json.message.from.username` for later use.  
    - *Connections:* Outputs to Get Prompt node.  
    - *Edge Cases:* Potential for missing or malformed message objects; bot credential misconfiguration may cause auth errors.

  - **Get Prompt (IF)**  
    - *Type & Role:* Conditional node evaluating if the message is from a specific username and is not the command `/start`.  
    - *Configuration:* Checks if `$json.message.from.username` equals a configured username and message text is not `/start`.  
    - *Connections:* True output proceeds to Task node; false output ends flow.  
    - *Edge Cases:* Username mismatch or empty messages skip processing.

  - **Task (Set)**  
    - *Type & Role:* Set node to assign workflow variables.  
    - *Configuration:* Sets `chatInput` to the user's message text, and `sessionId` to the Telegram username for session tracking.  
    - *Expressions:* `={{ $json.message.text }}` for prompt; `={{ $json.message.from.username }}` for sessionId.  
    - *Connections:* Outputs to Generate Image node.  
    - *Edge Cases:* Missing username or text could cause empty assignments.

---

#### 2.2 Image Generation Request

- **Overview:**  
  Sends the prompt to GoAPI for Midjourney image generation, notifies the user that generation is in progress, and polls GoAPI for task completion.

- **Nodes Involved:**  
  - Generate Image (HTTP Request)  
  - Notify Generation (Telegram)  
  - Get Generation Task (HTTP Request)  
  - Status = complete (IF)  
  - Wait (Wait)

- **Node Details:**

  - **Generate Image (HTTP Request)**  
    - *Type & Role:* HTTP POST request to GoAPI `/task` endpoint to initiate Midjourney image generation.  
    - *Configuration:*  
      - JSON body includes model `midjourney`, task_type `imagine`, prompt from `chatInput`, fixed aspect ratio `12:16`, process mode `fast`, and a bot_id `0`.  
      - Uses HTTP header auth with API key credential named "dhruv21 - GoAPI".  
    - *Expressions:* Prompt injected via `{{ $json.chatInput }}`.  
    - *Connections:* Outputs to Notify Generation.  
    - *Edge Cases:* API authentication failure, invalid prompt causing API rejection, network timeouts.

  - **Notify Generation (Telegram)**  
    - *Type & Role:* Sends a Telegram message notifying the user that image generation is underway.  
    - *Configuration:* Text message: "🖼 Generating Image ...", chat ID taken from the original Telegram trigger message.  
    - *Connections:* Outputs to Get Generation Task node.  
    - *Edge Cases:* Telegram API rate limits or message send failures.

  - **Get Generation Task (HTTP Request)**  
    - *Type & Role:* Polls GoAPI task status endpoint for generation task using task_id from Generate Image node output.  
    - *Configuration:* GET request to `https://api.goapi.ai/api/v1/task/{{task_id}}` with API key auth.  
    - *Connections:* Outputs to Status = complete IF node.  
    - *Edge Cases:* Task not found, API downtime, or delayed response.

  - **Status = complete (IF)**  
    - *Type & Role:* Checks if the generation task status is `completed`.  
    - *Configuration:* Compares `$json.data.status` to string "completed".  
    - *Connections:*  
      - If true, proceeds to Get Index to Upscale node.  
      - If false, loops to Wait node for delay before re-polling Get Generation Task.  
    - *Edge Cases:* Status field missing or unexpected status values.

  - **Wait (Wait)**  
    - *Type & Role:* Pauses workflow execution before re-polling generation task status.  
    - *Configuration:* No custom delay specified, defaults apply (usually a few seconds).  
    - *Connections:* Outputs back to Get Generation Task node.  
    - *Edge Cases:* Excessive wait causing user experience delay.

---

#### 2.3 Image Selection and Upscaling

- **Overview:**  
  After generation completes, sends generated images to user via Telegram with UI to select image index for upscaling, requests upscaling via GoAPI, notifies user, and polls for upscale completion.

- **Nodes Involved:**  
  - Get Index to Upscale (Telegram)  
  - Upscale (HTTP Request)  
  - Notify Upscaling (Telegram)  
  - Get Upscale Task (HTTP Request)  
  - Get Upscale (IF)  
  - Wait1 (Wait)

- **Node Details:**

  - **Get Index to Upscale (Telegram)**  
    - *Type & Role:* Sends message with generated image URL and prompts user to select image index for upscaling via custom form input.  
    - *Configuration:*  
      - Message includes image URL from generation output; provides number input field labeled "Image Index".  
      - Uses Telegram API credentials.  
    - *Connections:* Outputs to Upscale node.  
    - *Edge Cases:* User inputs invalid index, no response, or cancels.

  - **Upscale (HTTP Request)**  
    - *Type & Role:* Sends upscale task request to GoAPI using the selected image index and original generation task ID.  
    - *Configuration:*  
      - POST to GoAPI `/task` endpoint with model `midjourney`, task_type `upscale`, input includes `origin_task_id` from generation and `index` from user input.  
      - Uses API key auth.  
    - *Connections:* Outputs to Notify Upscaling node.  
    - *Edge Cases:* Invalid index, API failures.

  - **Notify Upscaling (Telegram)**  
    - *Type & Role:* Notifies the user that their image is being upscaled.  
    - *Configuration:* Message: "↗ Upscaling Image ..." to the original Telegram chat.  
    - *Connections:* Outputs to Get Upscale Task node.  
    - *Edge Cases:* Telegram API errors.

  - **Get Upscale Task (HTTP Request)**  
    - *Type & Role:* Polls the GoAPI endpoint for the status of the upscale task using the upscale task ID.  
    - *Configuration:* GET request to `/task/{{upscale_task_id}}` with API auth.  
    - *Connections:* Outputs to Get Upscale IF node.  
    - *Edge Cases:* Task not found or API unavailability.

  - **Get Upscale (IF)**  
    - *Type & Role:* Checks if the upscale task status is `completed`.  
    - *Configuration:* Compares `$json.data.status` to `"completed"`.  
    - *Connections:*  
      - If true, outputs to Main Log and Discord - Generation Log nodes.  
      - If false, outputs to Wait1 node to delay before polling again.  
    - *Edge Cases:* Missing status or unexpected values.

  - **Wait1 (Wait)**  
    - *Type & Role:* Waits before re-polling upscale task status to avoid excessive API calls.  
    - *Configuration:* Default wait period.  
    - *Connections:* Loops back to Get Upscale Task.  
    - *Edge Cases:* Excessive wait causing delay.

---

#### 2.4 Final Notification and Logging

- **Overview:**  
  Sends final upscaled image URL to Telegram user and optionally logs detailed generation and upscale info to a Discord channel.

- **Nodes Involved:**  
  - Main Log (Telegram)  
  - Discord - Generation Log (Discord)  
  - Sticky Note1 (Optional log info)

- **Node Details:**

  - **Main Log (Telegram)**  
    - *Type & Role:* Sends the final upscaled image URL back to the Telegram user.  
    - *Configuration:* Sends text containing the final image URL from upscale task output.  
    - *Connections:* Terminal node.  
    - *Edge Cases:* Telegram API failures.

  - **Discord - Generation Log (Discord)**  
    - *Type & Role:* Posts a detailed log message to a Discord channel for record keeping.  
    - *Configuration:*  
      - Message includes all rendered images, individual images, and final upscaled image URLs formatted in markdown.  
      - Posts to a specific guild and channel with flags to suppress embeds and notifications.  
      - Uses Discord bot credentials.  
    - *Connections:* Terminal node.  
    - *Edge Cases:* Discord API rate limits, authentication issues.  
    - *Sticky Note1:* Indicates this node is optional and can be deleted if logging is not desired.

  - **Sticky Note1**  
    - Contains info: "Optional Log. Delete this Node if you don't want to get Logs in Discord."

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                              | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                          |
|---------------------|---------------------|----------------------------------------------|-------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger    | Entry point: receives Telegram messages      | —                       | Get Prompt                      |                                                                                                    |
| Get Prompt           | IF                  | Filters valid username and non-/start msgs  | Telegram Trigger        | Task (true), ends (false)       |                                                                                                    |
| Task                | Set                  | Assigns chatInput and sessionId variables    | Get Prompt              | Generate Image                  |                                                                                                    |
| Generate Image       | HTTP Request         | Sends image generation request to GoAPI      | Task                    | Notify Generation               |                                                                                                    |
| Notify Generation    | Telegram             | Notifies user of generation start             | Generate Image           | Get Generation Task             |                                                                                                    |
| Get Generation Task  | HTTP Request         | Polls GoAPI for generation task status        | Notify Generation        | Status = complete               |                                                                                                    |
| Status = complete    | IF                   | Checks if generation task is completed        | Get Generation Task      | Get Index to Upscale (true), Wait (false) |                                                                                                    |
| Wait                 | Wait                 | Waits before re-polling generation status     | Status = complete (false)| Get Generation Task             |                                                                                                    |
| Get Index to Upscale | Telegram             | Sends images and asks user to select index    | Status = complete (true) | Upscale                       |                                                                                                    |
| Upscale              | HTTP Request         | Sends upscale task request to GoAPI           | Get Index to Upscale     | Notify Upscaling               |                                                                                                    |
| Notify Upscaling     | Telegram             | Notifies user that upscaling started          | Upscale                 | Get Upscale Task               |                                                                                                    |
| Get Upscale Task     | HTTP Request         | Polls GoAPI for upscale task status            | Notify Upscaling         | Get Upscale                   |                                                                                                    |
| Get Upscale          | IF                   | Checks if upscale task is completed            | Get Upscale Task         | Main Log, Discord - Generation Log (true), Wait1 (false) |                                                                                                    |
| Wait1                | Wait                 | Waits before re-polling upscale status         | Get Upscale (false)      | Get Upscale Task               |                                                                                                    |
| Main Log             | Telegram             | Sends final upscaled image URL to user         | Get Upscale (true)       | —                              |                                                                                                    |
| Discord - Generation Log | Discord           | Logs detailed generation info to Discord      | Get Upscale (true)       | —                              | Optional Log. Delete this Node if you don't want to get Logs in Discord.                            |
| Sticky Note          | Sticky Note          | Setup instructions for Telegram and GoAPI     | —                       | —                              | Contains setup instructions and useful links for Telegram Bot token and GoAPI API key setup.       |
| Sticky Note1         | Sticky Note          | Notes about optional Discord logging           | —                       | —                              | Optional Log. Delete this Node if you don't want to get Logs in Discord.                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for `message` updates only  
   - Credentials: Link to your Telegram Bot API credentials  
   - Position: Start node  

2. **Create Get Prompt IF Node**  
   - Type: IF  
   - Conditions:  
     - `$json.message.from.username` equals your target username (replace `"username"` with actual username)  
     - `$json.message.text` not equal to `/start`  
   - Connect input from Telegram Trigger node  

3. **Create Task (Set) Node**  
   - Assign variables:  
     - `chatInput` = `={{ $json.message.text }}`  
     - `sessionId` = `={{ $json.message.from.username }}`  
   - Connect input from Get Prompt IF node’s true output  

4. **Create Generate Image HTTP Request Node**  
   - Method: POST  
   - URL: `https://api.goapi.ai/api/v1/task`  
   - Authentication: HTTP Header Auth with GoAPI API key credential  
   - Headers: Content-Type: application/json  
   - Body (JSON):  
     ```json
     {
       "model": "midjourney",
       "task_type": "imagine",
       "input": {
         "prompt": "{{ $json.chatInput }}",
         "aspect_ratio": "12:16",
         "process_mode": "fast",
         "skip_prompt_check": false,
         "bot_id": 0
       }
     }
     ```  
   - Connect input from Task node  

5. **Create Notify Generation Telegram Node**  
   - Text: "🖼 Generating Image ..."  
   - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
   - Connect input from Generate Image node  

6. **Create Get Generation Task HTTP Request Node**  
   - Method: GET  
   - URL: `https://api.goapi.ai/api/v1/task/{{ $('Generate Image').item.json.data.task_id }}`  
   - Authentication: HTTP Header Auth with GoAPI API key credential  
   - Connect input from Notify Generation node  

7. **Create Status = complete IF Node**  
   - Condition: `$json.data.status` equals `"completed"`  
   - Connect input from Get Generation Task node  

8. **Create Wait Node**  
   - Default wait (typically a few seconds)  
   - Connect input from Status = complete node’s false output  

9. **Connect Wait node output to Get Generation Task node (loop)**  

10. **Create Get Index to Upscale Telegram Node**  
    - Operation: "sendAndWait" with custom form  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Message: `={{ $json.data.output.image_url }}`  
    - Form Fields: Number input labeled "Image Index"  
    - Connect input from Status = complete node’s true output  

11. **Create Upscale HTTP Request Node**  
    - Method: POST  
    - URL: `https://api.goapi.ai/api/v1/task`  
    - Body (JSON):  
      ```json
      {
        "model": "midjourney",
        "task_type": "upscale",
        "input": {
          "origin_task_id": "{{ $('Generate Image').item.json.data.task_id }}",
          "index": "{{ $json.data['Image Index'] }}"
        }
      }
      ```  
    - Authentication: HTTP Header Auth with GoAPI API key credential  
    - Connect input from Get Index to Upscale node  

12. **Create Notify Upscaling Telegram Node**  
    - Text: "↗ Upscaling Image ..."  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Connect input from Upscale node  

13. **Create Get Upscale Task HTTP Request Node**  
    - Method: GET  
    - URL: `https://api.goapi.ai/api/v1/task/{{ $('Upscale').item.json.data.task_id }}`  
    - Authentication: HTTP Header Auth with GoAPI API key credential  
    - Connect input from Notify Upscaling node  

14. **Create Get Upscale IF Node**  
    - Condition: `$json.data.status` equals `"completed"`  
    - Connect input from Get Upscale Task node  

15. **Create Wait1 Node**  
    - Default wait time  
    - Connect input from Get Upscale IF node’s false output  
    - Connect Wait1 output back to Get Upscale Task node (loop)  

16. **Create Main Log Telegram Node**  
    - Text: `={{ $json.data.output.image_url }}`  
    - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - Connect input from Get Upscale IF node’s true output  

17. **(Optional) Create Discord - Generation Log Node**  
    - Type: Discord  
    - Content: Markdown formatted message including:  
      - All rendered images: `{{ $('Get Generation Task').item.json.data.output.image_url }}`  
      - Individual images: `{{ $('Get Generation Task').item.json.data.output.temporary_image_urls.join('\n\n') }}`  
      - Final upscaled image: `{{ $json.data.output.image_url }}`  
    - Guild and channel IDs as per your Discord server setup  
    - Flags: SUPPRESS_EMBEDS, SUPPRESS_NOTIFICATIONS  
    - Credentials: Discord Bot API credentials  
    - Connect input from Get Upscale IF node’s true output  

18. **Create Sticky Notes for Setup Instructions**  
    - Include details about Telegram Bot creation and token acquisition  
    - Instructions for creating GoAPI account and API key, updating credentials in HTTP nodes  

19. **Verify all connections replicate the original flow**  
    - Ensure loops and conditionals match the original logic  

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Setup Instructions: Create Telegram Bot and obtain Token. Create GoAPI account and get API Key. Update credentials in nodes. | See sticky note in workflow; Telegram token help: https://help.zoho.com/portal/en/kb/desk/support-channels/instant-messaging/telegram/articles/telegram-integration-with-zoho-desk#How_to_find_a_token_for_an_existing_Telegram_Bot; GoAPI API key: https://goapi.ai/dashboard/key |
| Discord logging node is optional and can be deleted if logging is not required.                                            | Sticky Note1 attached to Discord - Generation Log node                                                                                         |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---