Automatically Add Watermarks to Images with Telegram Bot and ImageKit.io

https://n8nworkflows.xyz/workflows/automatically-add-watermarks-to-images-with-telegram-bot-and-imagekit-io-11229


# Automatically Add Watermarks to Images with Telegram Bot and ImageKit.io

### 1. Workflow Overview

This workflow automates the process of adding watermarks (logos) to images sent via a Telegram bot, using ImageKit.io for image processing and hosting. It is designed for users who wish to receive images through Telegram, automatically apply a watermark overlay, and then get the watermarked image sent back via Telegram.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures incoming Telegram photo messages and validates their type.
- **1.2 Image Retrieval:** Obtains the Telegram file path and downloads the original image.
- **1.3 Image Upload & Processing:** Uploads the downloaded image to ImageKit.io and applies the watermark/logo.
- **1.4 Output Delivery:** Sends the watermarked image back to the Telegram user.
- **1.5 Error Handling:** Sends an error message if the input is not a photo.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives incoming messages via Telegram bot webhook and checks if the message contains a photo. Routes the flow accordingly.

- **Nodes Involved:**  
  - Telegram Trigger1  
  - Check if Photo1  
  - Send Error Message1

- **Node Details:**  

  - **Telegram Trigger1**  
    - Type: Telegram Trigger  
    - Role: Entry point; listens to incoming Telegram messages via webhook.  
    - Configuration: Uses a custom webhook ID to receive updates.  
    - Input: External Telegram message webhook.  
    - Output: Passes message data to the “Check if Photo1” node.  
    - Edge Cases: Telegram webhook misconfiguration; missing webhook ID; no photo data in message.

  - **Check if Photo1**  
    - Type: If node  
    - Role: Conditional check to verify if the incoming message contains a photo.  
    - Configuration: Likely checks if the Telegram message has photo data in its payload (e.g., existence of "photo" array).  
    - Input: Telegram message data.  
    - Output:  
      - True branch: Continues to “Get File Path” node for photo processing.  
      - False branch: Sends an error message to the user.  
    - Edge Cases: Message contains no photo or unsupported media type; expression failure if expected fields missing.

  - **Send Error Message1**  
    - Type: Telegram node  
    - Role: Sends a user-friendly error message when the received message is not a photo.  
    - Configuration: Uses the Telegram bot webhook credentials to send the message.  
    - Input: Triggered by the false branch of “Check if Photo1”.  
    - Output: Ends flow.  
    - Edge Cases: Telegram API rate limits; invalid chat ID; bot permission errors.

---

#### 1.2 Image Retrieval

- **Overview:**  
Fetches the Telegram file path for the photo and downloads the image file for subsequent processing.

- **Nodes Involved:**  
  - Get File Path  
  - Download Image

- **Node Details:**  

  - **Get File Path**  
    - Type: HTTP Request  
    - Role: Calls Telegram API method (likely getFile) to retrieve the file path for the photo.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Telegram API endpoint to get file info using the file_id extracted from the photo message.  
      - Authentication: Telegram Bot token in headers or query params.  
    - Input: Photo file_id from the Telegram message.  
    - Output: Provides file path URL for downloading.  
    - Edge Cases: Invalid file_id; Telegram API errors; request timeouts.

  - **Download Image**  
    - Type: HTTP Request  
    - Role: Downloads the image binary from the Telegram file path.  
    - Configuration:  
      - HTTP Method: GET  
      - URL: Constructed from the file path returned by “Get File Path”.  
      - Response Format: Binary (image data).  
    - Input: File path URL.  
    - Output: Image binary data ready for upload.  
    - Edge Cases: Network failures; invalid URL; large file size causing timeout.

---

#### 1.3 Image Upload & Processing

- **Overview:**  
Uploads the downloaded image to ImageKit.io and applies a watermark (logo) overlay via ImageKit’s image transformation API.

- **Nodes Involved:**  
  - Upload to ImageKit  
  - Add Logo1

- **Node Details:**  

  - **Upload to ImageKit**  
    - Type: HTTP Request  
    - Role: Uploads the raw image binary to ImageKit.io for hosting and further processing.  
    - Configuration:  
      - HTTP Method: POST  
      - URL: ImageKit.io upload endpoint.  
      - Authentication: Uses ImageKit API credentials (private key or token).  
      - Payload: Binary image from “Download Image”.  
    - Input: Image binary data.  
    - Output: URL or ID of uploaded image.  
    - Edge Cases: Authentication errors; upload limit exceeded; network errors.

  - **Add Logo1**  
    - Type: HTTP Request  
    - Role: Applies a watermark/logo overlay on the uploaded image using ImageKit’s URL-based image transformation features.  
    - Configuration:  
      - HTTP Method: GET or POST depending on API usage.  
      - URL: ImageKit transformation URL with parameters specifying the watermark/logo.  
    - Input: ImageKit image URL from “Upload to ImageKit”.  
    - Output: URL of the watermarked image.  
    - Edge Cases: Invalid transformation parameters; API rate limiting; malformed URLs.

---

#### 1.4 Output Delivery

- **Overview:**  
Sends the final watermarked image back to the Telegram user.

- **Nodes Involved:**  
  - Send Watermarked Photo

- **Node Details:**  

  - **Send Watermarked Photo**  
    - Type: Telegram node  
    - Role: Sends the processed image back to the Telegram chat as a photo message.  
    - Configuration:  
      - Uses Telegram bot webhook credentials.  
      - Sends the image URL from the transformed ImageKit image.  
    - Input: Watermarked image URL from “Add Logo1”.  
    - Output: Ends flow with photo sent.  
    - Edge Cases: Invalid chat ID; Telegram API limits; URL accessibility issues.

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                    | Input Node(s)       | Output Node(s)         | Sticky Note                              |
|---------------------|----------------------|----------------------------------|---------------------|------------------------|-----------------------------------------|
| Telegram Trigger1    | Telegram Trigger     | Entry point; receives Telegram messages | -                   | Check if Photo1        |                                         |
| Check if Photo1      | If                   | Checks if incoming message has a photo | Telegram Trigger1   | Get File Path, Send Error Message1 |                                         |
| Send Error Message1  | Telegram             | Sends error if no photo present  | Check if Photo1     | -                      |                                         |
| Get File Path        | HTTP Request         | Retrieves Telegram file path for photo | Check if Photo1     | Download Image         |                                         |
| Download Image       | HTTP Request         | Downloads image binary from Telegram | Get File Path       | Upload to ImageKit     |                                         |
| Upload to ImageKit   | HTTP Request         | Uploads image to ImageKit.io     | Download Image      | Add Logo1              |                                         |
| Add Logo1            | HTTP Request         | Adds watermark/logo overlay      | Upload to ImageKit  | Send Watermarked Photo |                                         |
| Send Watermarked Photo | Telegram             | Sends processed image back to user | Add Logo1          | -                      |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure: Set webhook ID (unique identifier for Telegram updates).  
   - Credentials: Link your Telegram Bot credentials.  
   - Purpose: Receive incoming messages.

2. **Add If Node “Check if Photo1”:**  
   - Type: If Node  
   - Configure: Check if `{{$json["message"]["photo"]}}` exists and is not empty.  
   - True branch: Proceed to next node.  
   - False branch: Send error message.

3. **Add Telegram Node “Send Error Message1”:**  
   - Type: Telegram  
   - Configure: Send message text like “Please send a photo to process.”  
   - Credentials: Telegram Bot credentials.  
   - Connect to False branch of “Check if Photo1”.

4. **Add HTTP Request Node “Get File Path”:**  
   - Type: HTTP Request  
   - Configure:  
     - Method: GET  
     - URL: `https://api.telegram.org/bot<YourBotToken>/getFile?file_id={{$json["message"]["photo"][-1]["file_id"]}}`  
   - Purpose: Get file path for largest photo size.  
   - Connect to True branch of “Check if Photo1”.

5. **Add HTTP Request Node “Download Image”:**  
   - Type: HTTP Request  
   - Configure:  
     - Method: GET  
     - URL: `https://api.telegram.org/file/bot<YourBotToken>/{{$json["result"]["file_path"]}}`  
     - Response Format: File / Binary Data  
   - Input: Output of “Get File Path”.

6. **Add HTTP Request Node “Upload to ImageKit”:**  
   - Type: HTTP Request  
   - Configure:  
     - Method: POST  
     - URL: ImageKit upload API endpoint (e.g., `https://upload.imagekit.io/api/v1/files/upload`)  
     - Authentication: Use ImageKit API credentials (private key).  
     - Payload: Binary image from “Download Image”.  
   - Input: Image binary from previous node.

7. **Add HTTP Request Node “Add Logo1”:**  
   - Type: HTTP Request  
   - Configure:  
     - Method: GET  
     - URL: Use ImageKit URL transformation parameters to add watermark/logo, e.g.:  
       `https://ik.imagekit.io/your_imagekit_id/your_uploaded_image.jpg?tr=w-overlay,oi-logo.png,ox-10,oy-10`  
   - Input: Uploaded image URL from “Upload to ImageKit”.

8. **Add Telegram Node “Send Watermarked Photo”:**  
   - Type: Telegram  
   - Configure:  
     - Send photo message with URL from “Add Logo1”.  
     - Credentials: Telegram Bot credentials.  
   - Connect from “Add Logo1”.

9. **Connect all nodes as per flow:**  
   - Telegram Trigger1 → Check if Photo1  
   - Check if Photo1 True → Get File Path → Download Image → Upload to ImageKit → Add Logo1 → Send Watermarked Photo  
   - Check if Photo1 False → Send Error Message1

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                     |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------|
| This workflow integrates Telegram bot messaging with ImageKit.io for image watermarking.      | Workflow purpose                                    |
| Ensure that Telegram Bot token and ImageKit credentials have correct permissions and scopes.  | Credential requirements                             |
| ImageKit supports URL-based transformations for overlays; consult their docs for parameters. | https://docs.imagekit.io/api-reference/transformations |
| Telegram API limits apply, including message size and rate limits.                            | https://core.telegram.org/bots/api                 |
| Adjust the watermark position (`ox`, `oy`) in ImageKit URL parameters to customize overlay.   | ImageKit URL transformation parameters              |

---

**Disclaimer:** The provided content is exclusively derived from an n8n workflow automation. It complies with all applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.