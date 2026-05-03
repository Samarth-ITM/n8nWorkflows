🚚 Automate Delivery Confirmation with Telegram Bot, Google Drive and Gmail

https://n8nworkflows.xyz/workflows/---automate-delivery-confirmation-with-telegram-bot--google-drive-and-gmail-3204


# 🚚 Automate Delivery Confirmation with Telegram Bot, Google Drive and Gmail

### 1. Workflow Overview

This workflow automates delivery confirmation for logistics operations using a Telegram bot, Google Drive, Google Sheets, and Gmail. It targets small and medium-sized businesses lacking a Transportation Management System (TMS) to record proofs of delivery efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Initialization Block:** Sets up global static data to track user states and store shipment-related data.
- **1.2 Workflow Trigger and Command Parsing:** Listens for Telegram messages, identifies commands, and routes the flow accordingly.
- **1.3 Driver Input Command Block:** Handles commands from drivers to initiate shipment tracking steps or send confirmation.
- **1.4 Driver Input Collection Block:** Based on the driver's current state, collects shipment number, GPS location, and shipment photo.
- **1.5 Data Storage and Confirmation Block:** Saves collected data to Google Drive and Google Sheets, sends confirmation messages to drivers, and notifies the logistics team via email.
- **1.6 Cleanup Block:** Clears stored states after delivery confirmation is sent.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization Block

- **Overview:** Initializes global static data to store Telegram user states and shipment data. This setup is required once before activating the workflow.
- **Nodes Involved:**  
  - Initiate Workflow Data  
  - Sticky Note (Initialization instructions)

- **Node Details:**

  - **Initiate Workflow Data**  
    - Type: Code Node  
    - Role: Creates a global static object `telegramStates` to track each chat's state flags.  
    - Configuration: Checks if `telegramStates` exists; if not, initializes it as an empty object.  
    - Inputs: None (manual run recommended)  
    - Outputs: Returns the initialized static data.  
    - Edge Cases: None significant; intended to run once.

---

#### 1.2 Workflow Trigger and Command Parsing

- **Overview:** Listens for incoming Telegram messages and determines if the message is a command to route the flow accordingly.
- **Nodes Involved:**  
  - Telegram Trigger  
  - Command? (If Node)  
  - Switch Command  
  - Sticky Note (Trigger instructions)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger Node  
    - Role: Starts the workflow when a Telegram message is received.  
    - Configuration: Listens for "message" updates only.  
    - Inputs: Telegram messages  
    - Outputs: Message JSON with chat and message details.  
    - Edge Cases: Authentication errors if bot token invalid; webhook setup required.

  - **Command?**  
    - Type: If Node  
    - Role: Checks if the incoming message contains the "/start" command entity.  
    - Configuration: Checks existence of `message.entities[0]` and if it equals "/start".  
    - Inputs: Telegram Trigger output  
    - Outputs: True (command detected) or False (not a command)  
    - Edge Cases: Messages without entities or malformed messages.

  - **Switch Command**  
    - Type: Switch Node  
    - Role: Routes the flow based on the exact text command from the user.  
    - Configuration: Checks for commands: `/addShipment`, `/addGPS`, `/sendPhoto`, `/sendConfirmation`.  
    - Inputs: Output from Command? node or Telegram Trigger  
    - Outputs: One of five outputs: each command or fallback "extra".  
    - Edge Cases: Unknown commands routed to fallback.

---

#### 1.3 Driver Input Command Block

- **Overview:** Sets state flags to indicate what input the workflow expects next from the driver and sends Telegram messages prompting the driver for input.
- **Nodes Involved:**  
  - waitingShipmentNumber (Code)  
  - waitingGPS (Code)  
  - waitingPhoto (Code)  
  - addShipmentNumber (Telegram Message)  
  - addGPS (Telegram Message)  
  - sendPhoto (Telegram Message)  
  - Welcome Message (Telegram Message)  
  - Sticky Note (Command block instructions)

- **Node Details:**

  - **waitingShipmentNumber**  
    - Type: Code Node  
    - Role: Sets the global static data state for the chat ID to `waitingShipmentNumber: true`.  
    - Configuration: Updates `telegramStates` object keyed by chat ID.  
    - Inputs: Switch Command output for `/addShipment`  
    - Outputs: Passes input forward.  
    - Edge Cases: Concurrent chats handled by separate keys.

  - **waitingGPS**  
    - Type: Code Node  
    - Role: Sets state to `waitingGPS: true` for the chat ID.  
    - Inputs: Switch Command output for `/addGPS`  
    - Outputs: Passes input forward.

  - **waitingPhoto**  
    - Type: Code Node  
    - Role: Sets state to `waitingPhoto: true` for the chat ID.  
    - Inputs: Switch Command output for `/sendPhoto`  
    - Outputs: Passes input forward.

  - **addShipmentNumber**  
    - Type: Telegram Message Node  
    - Role: Prompts driver to enter the shipment number with forced reply.  
    - Configuration: Sends message "Please enter the delivery number for this shipment." with force reply enabled.  
    - Inputs: waitingShipmentNumber node  
    - Outputs: Telegram message sent confirmation.

  - **addGPS**  
    - Type: Telegram Message Node  
    - Role: Requests GPS location from driver with forced reply.  
    - Configuration: Message "Please share your GPS location by clicking the attachment button." with force reply.  
    - Inputs: waitingGPS node

  - **sendPhoto**  
    - Type: Telegram Message Node  
    - Role: Requests driver to upload a photo of the shipment with forced reply.  
    - Inputs: waitingPhoto node

  - **Welcome Message**  
    - Type: Telegram Message Node  
    - Role: Sends introductory message with available commands when no recognized command is given.  
    - Inputs: Switch Command fallback output  
    - Outputs: Message sent to driver.

---

#### 1.4 Driver Input Collection Block

- **Overview:** Processes the driver's input based on the current state, stores the data in global static data, and sends confirmation messages with instructions for the next step.
- **Nodes Involved:**  
  - Waiting Conditions (Code)  
  - Check State (Switch)  
  - Shipment Number (Set)  
  - Store Shipment (Code)  
  - addShipmentNumber result (Telegram Message)  
  - Store GPS Location (Set)  
  - Store GPS (Code)  
  - addGPS result (Telegram Message)  
  - Get Picture (Telegram)  
  - Upload Picture (Google Drive)  
  - Share Picture (Google Drive)  
  - Save Public Image Link (Code)  
  - addPhoto result (Telegram Message)  
  - Sticky Note (Input collection instructions)

- **Node Details:**

  - **Waiting Conditions**  
    - Type: Code Node  
    - Role: Reads the global static data `telegramStates` for the chat ID to determine the current expected input state (`waitingShipmentNumber`, `waitingGPS`, or `waitingPhoto`).  
    - Outputs: JSON with `state` key indicating current expected input.  
    - Inputs: Telegram Trigger output.

  - **Check State**  
    - Type: Switch Node  
    - Role: Routes flow based on the `state` value from Waiting Conditions node.  
    - Outputs: Four outputs for each state or fallback.

  - **Shipment Number**  
    - Type: Set Node  
    - Role: Extracts shipment number from the Telegram message text and sets it as `shipmentNumber` in JSON.  
    - Inputs: Check State output for `waitingShipmentNumber`.

  - **Store Shipment**  
    - Type: Code Node  
    - Role: Saves the shipment number into global static data under `shipmentNumber`.  
    - Inputs: Shipment Number node.

  - **addShipmentNumber result**  
    - Type: Telegram Message Node  
    - Role: Confirms shipment number recorded and prompts next step `/addGPS`.  
    - Inputs: Shipment Number node.

  - **Store GPS Location**  
    - Type: Set Node  
    - Role: Extracts latitude and longitude from Telegram message location object.  
    - Inputs: Check State output for `waitingGPS`.

  - **Store GPS**  
    - Type: Code Node  
    - Role: Saves latitude and longitude into global static data under `gpsLatitude` and `gpsLongitude`.  
    - Inputs: Store GPS Location node.

  - **addGPS result**  
    - Type: Telegram Message Node  
    - Role: Confirms GPS coordinates recorded and prompts next step `/sendPhoto`.  
    - Inputs: Store GPS Location node.

  - **Get Picture**  
    - Type: Telegram Node (Get File)  
    - Role: Retrieves the photo file from Telegram using the file ID from the message.  
    - Inputs: Check State output for `waitingPhoto`.

  - **Upload Picture**  
    - Type: Google Drive Node  
    - Role: Uploads the photo file to a specified Google Drive folder.  
    - Inputs: Get Picture node.

  - **Share Picture**  
    - Type: Google Drive Node  
    - Role: Sets sharing permissions on the uploaded file to "anyone with link can view".  
    - Inputs: Upload Picture node.

  - **Save Public Image Link**  
    - Type: Code Node  
    - Role: Extracts the public shareable link and file ID from Google Drive response, stores it and the current timestamp (`deliveryTime`) in global static data.  
    - Inputs: Share Picture node.

  - **addPhoto result**  
    - Type: Telegram Message Node  
    - Role: Confirms photo saved and prompts next step `/sendConfirmation`.  
    - Inputs: Save Public Image Link node.

---

#### 1.5 Data Storage and Confirmation Block

- **Overview:** Loads all collected shipment data from global static data, appends it to a Google Sheet, sends a confirmation message to the driver, and emails the logistics team with delivery details and photo.
- **Nodes Involved:**  
  - Load Workspace Data (Code)  
  - Load Delivery Information (Google Sheets)  
  - Confirmation Driver (Telegram Message)  
  - Distribution Team Confirmation (Gmail)  
  - Clear State (Code)  
  - Sticky Note (Confirmation block instructions)

- **Node Details:**

  - **Load Workspace Data**  
    - Type: Code Node  
    - Role: Retrieves all stored shipment data (shipment number, GPS coordinates, public image link, delivery time) from global static data for the current chat ID.  
    - Outputs: JSON with all delivery info.  
    - Inputs: Switch Command output for `/sendConfirmation`.

  - **Load Delivery Information**  
    - Type: Google Sheets Node  
    - Role: Appends a new row with the shipment data into a specified Google Sheet.  
    - Configuration: Maps columns `shipmentNumber`, `recordTime` (current timestamp), `gpsLatitude`, `gpsLongitude`, `cargoPicture` (public image link), and `deliveryTime`.  
    - Inputs: Load Workspace Data node.

  - **Confirmation Driver**  
    - Type: Telegram Message Node  
    - Role: Sends a summary message to the driver with shipment details and a link to the uploaded photo.  
    - Inputs: Load Workspace Data node.

  - **Distribution Team Confirmation**  
    - Type: Gmail Node  
    - Role: Sends an HTML email to the logistics team with shipment number, GPS coordinates, delivery time, and embedded shipment photo.  
    - Configuration: Uses Gmail API credentials, sets sender name to "LogiGreenTrack Solution", recipient email configured.  
    - Inputs: Load Workspace Data node.

  - **Clear State**  
    - Type: Code Node  
    - Role: Deletes the stored state for the chat ID from global static data to reset the workflow for the next shipment.  
    - Inputs: Distribution Team Confirmation node.

---

#### 1.6 Cleanup Block

- **Overview:** Resets the user's state after delivery confirmation is sent to allow new shipment tracking sessions.
- **Nodes Involved:**  
  - Clear State (Code)

- **Node Details:**

  - **Clear State**  
    - See above in 1.5.

---

### 3. Summary Table

| Node Name                  | Node Type             | Functional Role                                  | Input Node(s)                  | Output Node(s)                          | Sticky Note                                                                                         |
|----------------------------|-----------------------|-------------------------------------------------|-------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------|
| Initiate Workflow Data      | Code                  | Initialize global static data for states         | None                          | None                                   | ### 0. Initiate Workplace Static Data: Run once before activating workflow                         |
| Telegram Trigger            | Telegram Trigger      | Trigger workflow on Telegram message             | None                          | Command?                               | ### 1. Workflow Trigger with Telegram Message                                                     |
| Command?                   | If                    | Check if message is "/start" command              | Telegram Trigger              | Switch Command, Waiting Conditions     |                                                                                                   |
| Switch Command             | Switch                | Route flow based on Telegram command              | Command?                      | waitingShipmentNumber, waitingGPS, waitingPhoto, Load Workspace Data, Welcome Message | ### 2. Driver's Input Command Block: Routes commands and sets state flags                          |
| waitingShipmentNumber       | Code                  | Set state to wait for shipment number input       | Switch Command                | addShipmentNumber                      |                                                                                                   |
| waitingGPS                 | Code                  | Set state to wait for GPS location input          | Switch Command                | addGPS                                |                                                                                                   |
| waitingPhoto               | Code                  | Set state to wait for shipment photo input        | Switch Command                | sendPhoto                             |                                                                                                   |
| addShipmentNumber           | Telegram Message      | Ask driver to enter shipment number               | waitingShipmentNumber          | None                                  |                                                                                                   |
| addGPS                     | Telegram Message      | Ask driver to share GPS location                   | waitingGPS                    | None                                  |                                                                                                   |
| sendPhoto                  | Telegram Message      | Ask driver to upload shipment photo                | waitingPhoto                  | None                                  |                                                                                                   |
| Welcome Message            | Telegram Message      | Send welcome and instructions for unknown commands | Switch Command (fallback)     | None                                  |                                                                                                   |
| Waiting Conditions          | Code                  | Determine current expected input state            | Telegram Trigger              | Check State                           |                                                                                                   |
| Check State                | Switch                | Route flow based on current input state           | Waiting Conditions            | Shipment Number, Store GPS Location, Get Picture, Instructions | ### 3. Driver's Input Collection Block: Processes input based on state                            |
| Shipment Number            | Set                   | Extract shipment number from message               | Check State                   | addShipmentNumber result, Store Shipment |                                                                                                   |
| Store Shipment             | Code                  | Save shipment number in global static data         | Shipment Number               | None                                  |                                                                                                   |
| addShipmentNumber result    | Telegram Message      | Confirm shipment number recorded and prompt next  | Shipment Number               | None                                  |                                                                                                   |
| Store GPS Location         | Set                   | Extract GPS latitude and longitude                  | Check State                   | addGPS result, Store GPS               |                                                                                                   |
| Store GPS                  | Code                  | Save GPS coordinates in global static data         | Store GPS Location            | None                                  |                                                                                                   |
| addGPS result              | Telegram Message      | Confirm GPS recorded and prompt next step          | Store GPS Location            | None                                  |                                                                                                   |
| Get Picture                | Telegram              | Retrieve photo file from Telegram                    | Check State                   | Upload Picture                       |                                                                                                   |
| Upload Picture             | Google Drive          | Upload photo to Google Drive folder                  | Get Picture                  | Share Picture                        |                                                                                                   |
| Share Picture              | Google Drive          | Set sharing permissions to public                    | Upload Picture               | Save Public Image Link               |                                                                                                   |
| Save Public Image Link     | Code                  | Save public image link and delivery time             | Share Picture                | addPhoto result                      |                                                                                                   |
| addPhoto result            | Telegram Message      | Confirm photo saved and prompt next step             | Save Public Image Link        | None                                  |                                                                                                   |
| Load Workspace Data        | Code                  | Load all stored shipment data for confirmation      | Switch Command               | Load Delivery Information, Confirmation Driver, Distribution Team Confirmation |                                                                                                   |
| Load Delivery Information  | Google Sheets         | Append shipment data to Google Sheet                  | Load Workspace Data          | None                                  |                                                                                                   |
| Confirmation Driver        | Telegram Message      | Send shipment summary to driver                       | Load Workspace Data          | None                                  |                                                                                                   |
| Distribution Team Confirmation | Gmail               | Send delivery confirmation email to logistics team   | Load Workspace Data          | Clear State                         |                                                                                                   |
| Clear State                | Code                  | Clear stored state for chat ID after confirmation    | Distribution Team Confirmation | None                                  |                                                                                                   |
| Instructions              | Telegram Message      | Send instructions if input state is unknown          | Check State                  | None                                  |                                                                                                   |
| Sticky Note3              | Sticky Note           | Link to detailed tutorial video                        | None                        | None                                  | ### 5. Do you need more details? Find a step-by-step guide in this tutorial [🎥 Watch My Tutorial](https://youtu.be/9NS4RYaOwJ8) |
| Sticky Note5              | Sticky Note           | Instructions for Driver's Input Collection Block       | None                        | None                                  | ### 3. Driver's Input Collection Block: Setup instructions for Telegram and Google Drive nodes    |
| Sticky Note6              | Sticky Note           | Instructions for Driver's Input Command Block          | None                        | None                                  | ### 2. Driver's Input Command Block: Setup instructions for Telegram and Gmail nodes              |
| Sticky Note1              | Sticky Note           | Instructions for Workflow Trigger                       | None                        | None                                  | ### 1. Workflow Trigger with Telegram Message                                                     |
| Sticky Note               | Sticky Note           | Instructions for Initialization Block                   | None                        | None                                  | ### 0. Initiate Workplace Static Data                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Initialization Code Node**  
   - Name: Initiate Workflow Data  
   - Type: Code  
   - Code: Initialize `telegramStates` in global static data if not present.  
   - Run this node once manually before activating the workflow.

2. **Create Telegram Trigger Node**  
   - Name: Telegram Trigger  
   - Type: Telegram Trigger  
   - Configure with your Telegram Bot API token.  
   - Set to listen for "message" updates.

3. **Create If Node to Detect /start Command**  
   - Name: Command?  
   - Type: If  
   - Condition: Check if `message.entities[0]` exists and equals "/start".

4. **Create Switch Node to Route Commands**  
   - Name: Switch Command  
   - Type: Switch  
   - Conditions for message text equal to:  
     - `/addShipment` → Output 1  
     - `/addGPS` → Output 2  
     - `/sendPhoto` → Output 3  
     - `/sendConfirmation` → Output 4  
     - Else → fallback output

5. **Create Code Nodes to Set States**  
   - waitingShipmentNumber: sets `telegramStates[chatId] = {waitingShipmentNumber: true}`  
   - waitingGPS: sets `telegramStates[chatId] = {waitingGPS: true}`  
   - waitingPhoto: sets `telegramStates[chatId] = {waitingPhoto: true}`

6. **Create Telegram Message Nodes to Prompt Driver**  
   - addShipmentNumber: message asking for shipment number with force reply.  
   - addGPS: message asking for GPS location with force reply.  
   - sendPhoto: message asking for shipment photo with force reply.  
   - Welcome Message: default message listing commands.

7. **Create Code Node to Determine Current Input State**  
   - Name: Waiting Conditions  
   - Reads `telegramStates[chatId]` and returns current state as `state`.

8. **Create Switch Node to Route Based on State**  
   - Name: Check State  
   - Routes based on `state` value:  
     - `waitingShipmentNumber` → Shipment Number flow  
     - `waitingGPS` → GPS Location flow  
     - `waitingPhoto` → Photo flow  
     - Else → Instructions message

9. **Shipment Number Flow**  
   - Set Node: Extract shipment number from message text.  
   - Code Node: Store shipment number in global static data.  
   - Telegram Message Node: Confirm shipment number recorded and prompt `/addGPS`.

10. **GPS Location Flow**  
    - Set Node: Extract latitude and longitude from message location.  
    - Code Node: Store GPS coordinates in global static data.  
    - Telegram Message Node: Confirm GPS recorded and prompt `/sendPhoto`.

11. **Photo Flow**  
    - Telegram Node: Get Picture file using photo file ID from message.  
    - Google Drive Node: Upload picture to a specified folder.  
    - Google Drive Node: Share picture with public read permissions.  
    - Code Node: Save public image link and delivery time in global static data.  
    - Telegram Message Node: Confirm photo saved and prompt `/sendConfirmation`.

12. **Load Workspace Data**  
    - Code Node: Retrieve all stored shipment data from global static data.

13. **Google Sheets Node**  
    - Append a new row with shipment data (shipment number, record time, GPS coordinates, photo link, delivery time) to a configured Google Sheet.

14. **Telegram Message Node**  
    - Send confirmation message to driver with shipment details and photo link.

15. **Gmail Node**  
    - Send delivery confirmation email to logistics team with shipment details and embedded photo.  
    - Configure with Gmail API credentials and recipient email.

16. **Clear State Code Node**  
    - Remove stored state for chat ID from global static data to reset workflow.

17. **Instructions Telegram Message Node**  
    - Send instructions message if user input state is unknown.

18. **Connect all nodes as per the logical flow:**  
    - Telegram Trigger → Command? → Switch Command → State-setting nodes → Prompt messages  
    - Telegram Trigger → Waiting Conditions → Check State → Input processing flows  
    - `/sendConfirmation` command → Load Workspace Data → Google Sheets, Telegram Confirmation, Gmail → Clear State

19. **Credential Setup:**  
    - Telegram Bot API token for Telegram nodes.  
    - Google Drive API credentials for Drive nodes.  
    - Google Sheets API credentials for Sheets node.  
    - Gmail API credentials for Gmail node.

20. **Google Drive Folder and Google Sheet Preparation:**  
    - Create a Google Drive folder for shipment photos.  
    - Create a Google Sheet with columns: shipmentNumber, recordTime, gpsLatitude, gpsLongitude, cargoPicture, deliveryTime.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                         |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow can be adapted to add more functionalities; see the tutorial video for details.                        | [Tutorial Video](https://youtu.be/9NS4RYaOwJ8)                                                        |
| The Telegram bot can handle multiple drivers simultaneously by tracking states per chat ID.                         |                                                                                                       |
| Original Python-based tool inspiration and blog article about Telegram shipment tracking bot.                        | [Blog Article](https://www.samirsaci.com/build-a-shipment-tracking-tool-using-a-telegram-bot/)         |
| Workflow created with n8n version 1.82.1, submitted March 17th, 2025.                                               |                                                                                                       |
| Branding and consulting services by Samir, founder of LogiGreen Consulting, specializing in logistics automation.   | [LogiGreen Consulting](https://www.logi-green.com/)                                                   |
| Video tutorial and detailed step-by-step guide available for setup and customization.                                | ![Guide](https://www.samirsaci.com/content/images/2025/04/Telegram-Shipment-Tracking.png)              |
| Connect with Samir on LinkedIn for more logistics & supply chain automation insights.                                | [LinkedIn](https://www.linkedin.com/in/samir-saci)                                                    |

---

This document provides a complete, structured reference to understand, reproduce, and modify the "Automate Delivery Confirmation with Telegram Bot, Google Drive and Gmail" workflow. It anticipates potential errors such as authentication failures, missing inputs, or invalid commands, and guides credential and environment setup for seamless integration.