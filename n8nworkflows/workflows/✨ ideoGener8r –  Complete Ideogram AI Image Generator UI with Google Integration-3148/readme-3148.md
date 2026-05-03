✨ ideoGener8r –  Complete Ideogram AI Image Generator UI with Google Integration

https://n8nworkflows.xyz/workflows/--ideogener8r----complete-ideogram-ai-image-generator-ui-with-google-integration-3148


# ✨ ideoGener8r –  Complete Ideogram AI Image Generator UI with Google Integration

### 1. Workflow Overview

**Purpose:**  
This workflow, named **ideoGener8r**, provides a fully self-hosted user interface within n8n to interact with the Ideogram AI image generation API. It enables users to generate, upscale, remix, and store AI-generated images seamlessly. The workflow integrates tightly with Google Sheets for metadata logging and Google Drive for image storage, allowing automated management of image assets and their associated data.

**Target Use Cases:**  
- AI image generation on demand via webhooks or UI.  
- Automated upscaling and remixing of generated images.  
- Centralized logging of image metadata in Google Sheets.  
- Secure user authentication with Basic Auth and Bearer token protection.  
- Self-hosted environment for privacy and cost control.

**Logical Blocks:**  
- **1.1 User Authentication & UI Delivery:** Handles login, logout, and serves the HTML UI for image generation.  
- **1.2 Image Generation & Processing:** Manages calls to Ideogram API for image generation, upscaling, and remixing.  
- **1.3 Image Download & Upload:** Downloads images from Ideogram responses and uploads them to Google Drive.  
- **1.4 Metadata Logging:** Logs image generation details into Google Sheets.  
- **1.5 Webhook Management & Routing:** Receives external requests and routes them through appropriate processing paths.  
- **1.6 Error Handling & Response:** Responds to webhook calls with appropriate data or error messages.

---

### 2. Block-by-Block Analysis

#### 1.1 User Authentication & UI Delivery

**Overview:**  
This block manages user login/logout via webhooks secured with Basic Auth, serves the HTML user interface, and sets authorization tokens for subsequent requests.

**Nodes Involved:**  
- Webhook-login  
- HTML-login  
- Respond with UI1  
- Webhook-logout  
- Respond with UI2  
- Webhook-ideogener8r  
- Set Bearer Token  
- HTML-UI  
- Respond with UI

**Node Details:**

- **Webhook-login**  
  - Type: Webhook  
  - Role: Entry point for user login requests.  
  - Configuration: Secured with Basic Auth credentials.  
  - Input: HTTP login requests.  
  - Output: Passes data to HTML-login node.  
  - Failure modes: Auth failure if credentials invalid.

- **HTML-login**  
  - Type: HTML  
  - Role: Serves login page HTML content.  
  - Configuration: Contains login form UI.  
  - Input: From Webhook-login.  
  - Output: To Respond with UI1.  

- **Respond with UI1**  
  - Type: Respond to Webhook  
  - Role: Sends login page HTML response to client.  
  - Input: From HTML-login.  
  - Output: HTTP response.

- **Webhook-logout**  
  - Type: Webhook  
  - Role: Handles logout requests.  
  - Input: HTTP logout requests.  
  - Output: To Respond with UI2.  

- **Respond with UI2**  
  - Type: Respond to Webhook  
  - Role: Sends logout confirmation or redirect UI.  
  - Input: From Webhook-logout.  
  - Output: HTTP response.

- **Webhook-ideogener8r**  
  - Type: Webhook  
  - Role: Entry point for main UI requests after login.  
  - Input: HTTP requests for UI.  
  - Output: To Set Bearer Token.

- **Set Bearer Token**  
  - Type: Set  
  - Role: Injects Bearer token into workflow context for authorization.  
  - Configuration: Sets token variable used in subsequent API calls.  
  - Input: From Webhook-ideogener8r.  
  - Output: To HTML-UI.

- **HTML-UI**  
  - Type: HTML  
  - Role: Serves the main Ideogram AI image generation UI.  
  - Input: From Set Bearer Token.  
  - Output: To Respond with UI.

- **Respond with UI**  
  - Type: Respond to Webhook  
  - Role: Sends the main UI HTML to client.  
  - Input: From HTML-UI.  
  - Output: HTTP response.

**Edge Cases & Failures:**  
- Invalid Basic Auth credentials block login.  
- Missing or invalid Bearer token prevents API calls.  
- HTML serving failures due to malformed content.

---

#### 1.2 Image Generation & Processing

**Overview:**  
This block handles calls to the Ideogram AI API for image generation, upscaling, and remixing. It routes requests based on webhook input and manages API responses.

**Nodes Involved:**  
- Webhook-ideogen  
- Switch  
- IDEOGRAM Image generator  
- ideogram Upscale  
- ideogram Remix  
- Respond to Webhook1  
- Respond to Webhook2  
- Respond to Webhook  
- Respond to Webhook - Generate  
- Respond to Webhook - Generate1  
- Respond to Webhook - Generate2  
- Respond to Webhook - Generate3

**Node Details:**

- **Webhook-ideogen**  
  - Type: Webhook  
  - Role: Receives image generation requests (generate, upscale, remix).  
  - Input: HTTP requests with action parameters.  
  - Output: To Switch node.

- **Switch**  
  - Type: Switch  
  - Role: Routes requests to appropriate API call node based on action type (generate, upscale, remix).  
  - Input: From Webhook-ideogen.  
  - Output: To IDEOGRAM Image generator, ideogram Upscale, or ideogram Remix.

- **IDEOGRAM Image generator**  
  - Type: HTTP Request  
  - Role: Calls Ideogram API `/generate` endpoint.  
  - Configuration: Uses HTTP Header Auth with Bearer token.  
  - Input: From Switch.  
  - Output: To SetImageData and Respond to Webhook1.  
  - Failure modes: API errors, rate limits, invalid parameters.

- **ideogram Upscale**  
  - Type: HTTP Request  
  - Role: Calls Ideogram API `/upscale` endpoint.  
  - Configuration: Uses HTTP Header Auth.  
  - Input: From Switch or Download Image1.  
  - Output: To Set Upload Fields and Respond to Webhook2.  
  - Failure modes: API errors, invalid image references.

- **ideogram Remix**  
  - Type: HTTP Request  
  - Role: Calls Ideogram API `/remix` endpoint.  
  - Configuration: Uses HTTP Header Auth.  
  - Input: From Download Image3.  
  - Output: To Set Upload Fields1 and Respond to Webhook.  
  - Failure modes: API errors, remix parameter issues.

- **Respond to Webhook1, 2, 3, Generate, Generate1, Generate2**  
  - Type: Respond to Webhook  
  - Role: Sends API response data back to the requester.  
  - Input: From respective API call or Google Sheets nodes.  
  - Output: HTTP response.

**Edge Cases & Failures:**  
- API rate limiting or downtime.  
- Invalid or missing API key.  
- Malformed request payloads.  
- Network timeouts.

---

#### 1.3 Image Download & Upload

**Overview:**  
This block downloads images from Ideogram API responses and uploads them to Google Drive folders for storage.

**Nodes Involved:**  
- SetImageData  
- Download Image  
- Upload Image  
- Set Upload Fields  
- Download Upscaled  
- Google Drive Upscale Image  
- Set Upload Fields1  
- Download Upscaled1  
- Google Drive Upscale Image1  
- Download Image1  
- Download Image3

**Node Details:**

- **SetImageData**  
  - Type: Set  
  - Role: Prepares data fields for image download (e.g., URLs).  
  - Input: From IDEOGRAM Image generator.  
  - Output: To Download Image.

- **Download Image**  
  - Type: HTTP Request  
  - Role: Downloads generated image binary data from Ideogram URL.  
  - Input: From SetImageData.  
  - Output: To Upload Image.

- **Upload Image**  
  - Type: Google Drive  
  - Role: Uploads downloaded image to configured Google Drive folder.  
  - Input: From Download Image.  
  - Output: To Add Gen to Sheets.

- **Set Upload Fields**  
  - Type: Set  
  - Role: Prepares metadata for upscaled image upload.  
  - Input: From ideogram Upscale.  
  - Output: To Download Upscaled.

- **Download Upscaled**  
  - Type: HTTP Request  
  - Role: Downloads upscaled image binary data.  
  - Input: From Set Upload Fields.  
  - Output: To Google Drive Upscale Image.

- **Google Drive Upscale Image**  
  - Type: Google Drive  
  - Role: Uploads upscaled image to Drive.  
  - Input: From Download Upscaled.  
  - Output: To Add Gen to Sheets - Upscaled.

- **Set Upload Fields1**  
  - Type: Set  
  - Role: Prepares metadata for remixed image upload.  
  - Input: From ideogram Remix.  
  - Output: To Download Upscaled1.

- **Download Upscaled1**  
  - Type: HTTP Request  
  - Role: Downloads remixed image binary data.  
  - Input: From Set Upload Fields1.  
  - Output: To Google Drive Upscale Image1.

- **Google Drive Upscale Image1**  
  - Type: Google Drive  
  - Role: Uploads remixed image to Drive.  
  - Input: From Download Upscaled1.  
  - Output: To Add Gen to Sheets - Remixed.

- **Download Image1**  
  - Type: HTTP Request  
  - Role: Downloads image for upscaling.  
  - Input: From Switch node (upscale path).  
  - Output: To ideogram Upscale.

- **Download Image3**  
  - Type: HTTP Request  
  - Role: Downloads image for remixing.  
  - Input: From Switch node (remix path).  
  - Output: To ideogram Remix.

**Edge Cases & Failures:**  
- Download failures due to invalid URLs or network issues.  
- Google Drive upload failures due to permission or quota limits.  
- Incorrect folder IDs or missing Drive credentials.

---

#### 1.4 Metadata Logging

**Overview:**  
Logs all generated, upscaled, and remixed image metadata into Google Sheets for tracking and auditing.

**Nodes Involved:**  
- Add Gen to Sheets  
- Add Gen to Sheets - Upscaled  
- Add Gen to Sheets - Remixed  
- Google Sheets1

**Node Details:**

- **Add Gen to Sheets**  
  - Type: Google Sheets  
  - Role: Appends metadata of generated images to Google Sheet.  
  - Input: From Upload Image.  
  - Output: To Respond to Webhook - Generate.

- **Add Gen to Sheets - Upscaled**  
  - Type: Google Sheets  
  - Role: Appends metadata of upscaled images.  
  - Input: From Google Drive Upscale Image.  
  - Output: To Respond to Webhook - Generate1.

- **Add Gen to Sheets - Remixed**  
  - Type: Google Sheets  
  - Role: Appends metadata of remixed images.  
  - Input: From Google Drive Upscale Image1.  
  - Output: To Respond to Webhook - Generate2.

- **Google Sheets1**  
  - Type: Google Sheets  
  - Role: Logs other metadata or history entries.  
  - Input: From Webhook.  
  - Output: To Respond to Webhook - Generate3.

**Edge Cases & Failures:**  
- Google Sheets API quota exceeded.  
- Incorrect sheet or tab configuration.  
- Missing or invalid Google Sheets credentials.

---

#### 1.5 Webhook Management & Routing

**Overview:**  
Manages all incoming webhook requests, routing them to the appropriate processing blocks based on the request type.

**Nodes Involved:**  
- Webhook  
- Webhook-ideogen  
- Webhook-ideogener8r  
- Webhook-login  
- Webhook-logout  
- Switch

**Node Details:**

- **Webhook**  
  - Type: Webhook  
  - Role: Receives generic requests, e.g., for logging or history.  
  - Input: HTTP requests.  
  - Output: To Google Sheets1.

- **Webhook-ideogen**  
  - Type: Webhook  
  - Role: Receives image generation-related requests.  
  - Input: HTTP requests with generation parameters.  
  - Output: To Switch.

- **Webhook-ideogener8r**  
  - Type: Webhook  
  - Role: Receives UI-related requests post-login.  
  - Input: HTTP requests.  
  - Output: To Set Bearer Token.

- **Webhook-login**  
  - Type: Webhook  
  - Role: Receives login requests.  
  - Input: HTTP login requests.  
  - Output: To HTML-login.

- **Webhook-logout**  
  - Type: Webhook  
  - Role: Receives logout requests.  
  - Input: HTTP logout requests.  
  - Output: To Respond with UI2.

- **Switch**  
  - Type: Switch  
  - Role: Routes generation requests to generate, upscale, or remix nodes.  
  - Input: From Webhook-ideogen.  
  - Output: To IDEOGRAM Image generator, ideogram Upscale, ideogram Remix.

**Edge Cases & Failures:**  
- Missing or malformed webhook payloads.  
- Unauthorized access if tokens or auth missing.  
- Incorrect routing due to unexpected parameters.

---

#### 1.6 Error Handling & Response

**Overview:**  
This block ensures that each webhook request receives an appropriate response, including error handling and success confirmation.

**Nodes Involved:**  
- Respond to Webhook (multiple variants)  
- Sticky Notes (for documentation)

**Node Details:**

- **Respond to Webhook** nodes  
  - Type: Respond to Webhook  
  - Role: Send HTTP responses back to clients with success or error data.  
  - Input: From API calls, Google Sheets, or other processing nodes.  
  - Output: HTTP response.

- **Sticky Notes**  
  - Type: Sticky Note  
  - Role: Provide inline documentation and setup notes within the workflow canvas.  
  - Content: Various notes about setup, API keys, Google Sheets columns, and usage tips.

**Edge Cases & Failures:**  
- Failure to respond due to node errors or timeouts.  
- Unhandled exceptions in upstream nodes causing no response.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                          | Input Node(s)                   | Output Node(s)                      | Sticky Note                                                                                      |
|----------------------------|---------------------|----------------------------------------|--------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger      | Manual trigger for testing (disabled)  |                                |                                   |                                                                                                |
| SetImageData               | Set                 | Prepare image download data             | IDEOGRAM Image generator        | Download Image                    |                                                                                                |
| Download Image             | HTTP Request        | Download generated image binary         | SetImageData                   | Upload Image                     |                                                                                                |
| Download Upscaled          | HTTP Request        | Download upscaled image binary          | Set Upload Fields              | Google Drive Upscale Image       |                                                                                                |
| Set Upload Fields          | Set                 | Prepare upload metadata for upscaled    | ideogram Upscale               | Download Upscaled                |                                                                                                |
| Google Drive Upscale Image | Google Drive        | Upload upscaled image to Drive           | Download Upscaled              | Add Gen to Sheets - Upscaled     |                                                                                                |
| Download Upscaled1         | HTTP Request        | Download remixed image binary            | Set Upload Fields1             | Google Drive Upscale Image1      |                                                                                                |
| Google Drive Upscale Image1| Google Drive        | Upload remixed image to Drive             | Download Upscaled1             | Add Gen to Sheets - Remixed      |                                                                                                |
| Sticky Note1               | Sticky Note         | Documentation note                       |                                |                                   |                                                                                                |
| Respond to Webhook - Generate | Respond to Webhook | Respond with generation success          | Add Gen to Sheets              |                                   |                                                                                                |
| Respond to Webhook - Generate1| Respond to Webhook | Respond with upscaled image success      | Add Gen to Sheets - Upscaled   |                                   |                                                                                                |
| Download Image3            | HTTP Request        | Download image for remixing               | Switch                        | ideogram Remix                  |                                                                                                |
| ideogram Remix             | HTTP Request        | Call Ideogram remix API                   | Download Image3               | Set Upload Fields1, Respond to Webhook |                                                                                                |
| Respond to Webhook         | Respond to Webhook  | Respond with remix success                | ideogram Remix                |                                   |                                                                                                |
| IDEOGRAM Image generator   | HTTP Request        | Call Ideogram generate API                | Switch                       | SetImageData, Respond to Webhook1 |                                                                                                |
| Respond to Webhook1        | Respond to Webhook  | Respond with generation API response      | IDEOGRAM Image generator      |                                   |                                                                                                |
| ideogram Upscale           | HTTP Request        | Call Ideogram upscale API                  | Download Image1               | Set Upload Fields, Respond to Webhook2 |                                                                                                |
| Respond to Webhook2        | Respond to Webhook  | Respond with upscale API response          | ideogram Upscale             |                                   |                                                                                                |
| Download Image1            | HTTP Request        | Download image for upscaling               | Switch                       | ideogram Upscale                |                                                                                                |
| Set Upload Fields1         | Set                 | Prepare upload metadata for remix          | ideogram Remix               | Download Upscaled1              |                                                                                                |
| Respond to Webhook - Generate2| Respond to Webhook | Respond with remix success                  | Add Gen to Sheets - Remixed   |                                   |                                                                                                |
| Google Sheets1             | Google Sheets       | Append metadata to Google Sheets           | Webhook                      | Respond to Webhook - Generate3  |                                                                                                |
| HTML-login                 | HTML                | Serve login page HTML                       | Webhook-login                | Respond with UI1                |                                                                                                |
| Webhook-login              | Webhook             | Receive login requests                       |                              | HTML-login                    |                                                                                                |
| Webhook                   | Webhook             | Receive generic webhook requests             |                              | Google Sheets1                |                                                                                                |
| Switch                    | Switch              | Route generation requests                      | Webhook-ideogen              | IDEOGRAM Image generator, Download Image1, Download Image3 |                                                                                                |
| Webhook-ideogen           | Webhook             | Receive image generation requests              |                              | Switch                      |                                                                                                |
| Respond with UI2           | Respond to Webhook  | Respond with logout confirmation UI            | Webhook-logout               |                               |                                                                                                |
| Sticky Note3              | Sticky Note         | Documentation note                             |                              |                               |                                                                                                |
| HTML-UI                   | HTML                | Serve main UI HTML                              | Set Bearer Token             | Respond with UI               |                                                                                                |
| Sticky Note4              | Sticky Note         | Documentation note                             |                              |                               |                                                                                                |
| Set Bearer Token           | Set                 | Set Bearer token for API authorization          | Webhook-ideogener8r          | HTML-UI                      |                                                                                                |
| Webhook-ideogener8r       | Webhook             | Receive UI requests post-login                    |                              | Set Bearer Token             |                                                                                                |
| Webhook-logout            | Webhook             | Receive logout requests                            |                              | Respond with UI2             |                                                                                                |
| Upload Image              | Google Drive        | Upload generated image to Drive                    | Download Image               | Add Gen to Sheets            |                                                                                                |
| Add Gen to Sheets         | Google Sheets       | Append generated image metadata                     | Upload Image                 | Respond to Webhook - Generate |                                                                                                |
| Add Gen to Sheets - Upscaled| Google Sheets      | Append upscaled image metadata                      | Google Drive Upscale Image   | Respond to Webhook - Generate1|                                                                                                |
| Add Gen to Sheets - Remixed| Google Sheets       | Append remixed image metadata                        | Google Drive Upscale Image1  | Respond to Webhook - Generate2|                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook-login Node**  
   - Type: Webhook  
   - Configure with Basic Auth credentials (username/password).  
   - Set path (e.g., `/ideogener8r-login`).  
   - Output to HTML-login node.

2. **Create HTML-login Node**  
   - Type: HTML  
   - Paste login page HTML content (login form).  
   - Connect output to Respond with UI1.

3. **Create Respond with UI1 Node**  
   - Type: Respond to Webhook  
   - Sends login page HTML response.  
   - Connect input from HTML-login.

4. **Create Webhook-logout Node**  
   - Type: Webhook  
   - Set path (e.g., `/ideogener8r-logout`).  
   - Connect output to Respond with UI2.

5. **Create Respond with UI2 Node**  
   - Type: Respond to Webhook  
   - Sends logout confirmation or redirect.  
   - Connect input from Webhook-logout.

6. **Create Webhook-ideogener8r Node**  
   - Type: Webhook  
   - Set path for main UI requests (e.g., `/ideogener8r`).  
   - Connect output to Set Bearer Token.

7. **Create Set Bearer Token Node**  
   - Type: Set  
   - Add field to set Bearer token string (e.g., your secret token).  
   - Connect output to HTML-UI.

8. **Create HTML-UI Node**  
   - Type: HTML  
   - Paste main Ideogram AI UI HTML content.  
   - Connect output to Respond with UI.

9. **Create Respond with UI Node**  
   - Type: Respond to Webhook  
   - Sends main UI HTML response.  
   - Connect input from HTML-UI.

10. **Create Webhook-ideogen Node**  
    - Type: Webhook  
    - Set path for image generation requests (e.g., `/ideogen`).  
    - Connect output to Switch node.

11. **Create Switch Node**  
    - Type: Switch  
    - Configure rules to route based on request action parameter:  
      - `generate` → IDEOGRAM Image generator  
      - `upscale` → Download Image1  
      - `remix` → Download Image3

12. **Create IDEOGRAM Image generator Node**  
    - Type: HTTP Request  
    - Configure to call Ideogram `/generate` endpoint.  
    - Use HTTP Header Auth credential with Bearer token.  
    - Connect output to SetImageData and Respond to Webhook1.

13. **Create SetImageData Node**  
    - Type: Set  
    - Map API response fields to prepare image download URL.  
    - Connect output to Download Image.

14. **Create Download Image Node**  
    - Type: HTTP Request  
    - Download image binary from URL set in SetImageData.  
    - Connect output to Upload Image.

15. **Create Upload Image Node**  
    - Type: Google Drive  
    - Configure with Google Drive credentials and target folder ID.  
    - Upload downloaded image.  
    - Connect output to Add Gen to Sheets.

16. **Create Add Gen to Sheets Node**  
    - Type: Google Sheets  
    - Configure with Google Sheets credentials, target sheet, and tab.  
    - Map metadata fields (image URL, prompt, timestamp, etc.).  
    - Connect output to Respond to Webhook - Generate.

17. **Create Respond to Webhook - Generate Node**  
    - Type: Respond to Webhook  
    - Sends success response with generation metadata.  
    - Connect input from Add Gen to Sheets.

18. **Create Download Image1 Node**  
    - Type: HTTP Request  
    - Downloads image for upscaling (URL from request).  
    - Connect output to ideogram Upscale.

19. **Create ideogram Upscale Node**  
    - Type: HTTP Request  
    - Calls Ideogram `/upscale` endpoint with Bearer token.  
    - Connect output to Set Upload Fields and Respond to Webhook2.

20. **Create Set Upload Fields Node**  
    - Type: Set  
    - Prepare metadata for upscaled image upload.  
    - Connect output to Download Upscaled.

21. **Create Download Upscaled Node**  
    - Type: HTTP Request  
    - Download upscaled image binary.  
    - Connect output to Google Drive Upscale Image.

22. **Create Google Drive Upscale Image Node**  
    - Type: Google Drive  
    - Upload upscaled image to Drive folder.  
    - Connect output to Add Gen to Sheets - Upscaled.

23. **Create Add Gen to Sheets - Upscaled Node**  
    - Type: Google Sheets  
    - Append upscaled image metadata to sheet.  
    - Connect output to Respond to Webhook - Generate1.

24. **Create Respond to Webhook - Generate1 Node**  
    - Type: Respond to Webhook  
    - Sends success response for upscaled image.  
    - Connect input from Add Gen to Sheets - Upscaled.

25. **Create Download Image3 Node**  
    - Type: HTTP Request  
    - Downloads image for remixing.  
    - Connect output to ideogram Remix.

26. **Create ideogram Remix Node**  
    - Type: HTTP Request  
    - Calls Ideogram `/remix` endpoint with Bearer token.  
    - Connect output to Set Upload Fields1 and Respond to Webhook.

27. **Create Set Upload Fields1 Node**  
    - Type: Set  
    - Prepare metadata for remixed image upload.  
    - Connect output to Download Upscaled1.

28. **Create Download Upscaled1 Node**  
    - Type: HTTP Request  
    - Download remixed image binary.  
    - Connect output to Google Drive Upscale Image1.

29. **Create Google Drive Upscale Image1 Node**  
    - Type: Google Drive  
    - Upload remixed image to Drive folder.  
    - Connect output to Add Gen to Sheets - Remixed.

30. **Create Add Gen to Sheets - Remixed Node**  
    - Type: Google Sheets  
    - Append remixed image metadata to sheet.  
    - Connect output to Respond to Webhook - Generate2.

31. **Create Respond to Webhook - Generate2 Node**  
    - Type: Respond to Webhook  
    - Sends success response for remixed image.  
    - Connect input from Add Gen to Sheets - Remixed.

32. **Create Webhook Node**  
    - Type: Webhook  
    - Receives generic requests (e.g., history logging).  
    - Connect output to Google Sheets1.

33. **Create Google Sheets1 Node**  
    - Type: Google Sheets  
    - Append generic metadata or history entries.  
    - Connect output to Respond to Webhook - Generate3.

34. **Create Respond to Webhook - Generate3 Node**  
    - Type: Respond to Webhook  
    - Sends response for generic webhook requests.  
    - Connect input from Google Sheets1.

---

**Credential Setup:**  
- **Ideogram API:** Create HTTP Header Auth credential with `Authorization: Bearer YOUR_IDEOGRAM_API_KEY`.  
- **Google Drive:** OAuth2 credential with access to target Drive folder.  
- **Google Sheets:** OAuth2 credential with access to target Sheet.  
- **Basic Auth:** Username/password credential for login webhook.  
- **Header Auth:** Bearer token credential for generation/remix webhooks.

---

**Default Values & Constraints:**  
- Google Sheets must have required columns for metadata logging (see setup notes).  
- Google Drive folder IDs must be correctly set in upload nodes.  
- API rate limits must be respected; handle errors gracefully.  
- Webhook paths should be unique and secured as described.

---

This detailed documentation enables full understanding, reproduction, and modification of the **ideoGener8r** workflow, ensuring robust integration with Ideogram AI and Google services within n8n.