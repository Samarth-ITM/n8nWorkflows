Export Dialogflow Intents with Priority Levels to Google Sheets via Telegram

https://n8nworkflows.xyz/workflows/export-dialogflow-intents-with-priority-levels-to-google-sheets-via-telegram-7073


# Export Dialogflow Intents with Priority Levels to Google Sheets via Telegram

### 1. Workflow Overview

This workflow automates the export of Dialogflow intents, including their priority levels, into a Google Sheets spreadsheet triggered by a Telegram bot command. It is designed primarily for administrators or authorized users who want to back up or analyze their Dialogflow agent's intents along with priority metadata.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Validation:** Listens for Telegram messages, verifies the user’s identity, and checks for the specific command ("backup").
- **1.2 Dialogflow Intents Retrieval:** Upon valid command, performs an authenticated HTTP request to Dialogflow API to fetch all intents.
- **1.3 Intents Processing and Mapping:** Processes the raw intents JSON, maps priority levels to descriptive text and emojis.
- **1.4 Data Registration:** Appends each processed intent as a new row into a designated Google Sheets spreadsheet, including timestamp and priority data.
- **1.5 Confirmation Response:** Sends a single Telegram message confirming the number of intents successfully recorded.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Validation

**Overview:**  
This block triggers the workflow on receiving any Telegram message, then validates the sender's user ID and the command text to ensure authorized use.

**Nodes Involved:**  
- Telegram Trigger  
- Validación de usuario por ID (User ID Validation)  
- Validación del comando (Command Validation)  
- Mensaje de usuario inválido (Invalid User Message)  
- Mensaje de validación de comando (Invalid Command Message)  
- Sticky Note (explaining validation)

**Node Details:**  

- **Telegram Trigger**  
  - *Type:* Telegram Trigger Node  
  - *Role:* Starts the workflow when any Telegram message is received.  
  - *Config:* Listens for "message" updates only.  
  - *Inputs:* External webhook from Telegram bot.  
  - *Outputs:* Passes message JSON downstream.  
  - *Edge Cases:* Invalid webhook setup, Telegram connectivity issues.

- **Validación de usuario por ID (IF node)**  
  - *Type:* If Node  
  - *Role:* Checks if the Telegram sender ID equals a predefined authorized ID (7894561234, placeholder).  
  - *Config:* Strict number equality check on `$json.message.from.id`.  
  - *Inputs:* Output from Telegram Trigger.  
  - *Outputs:*  
    - True: Passes to command validation.  
    - False: Routes to "Mensaje de usuario inválido".  
  - *Edge Cases:* Missing sender ID, numeric type mismatches.

- **Validación del comando (IF node)**  
  - *Type:* If Node  
  - *Role:* Checks if the received message text is exactly "backup".  
  - *Config:* Case-sensitive string equality check on `$json.message.text`.  
  - *Inputs:* True branch output of user validation.  
  - *Outputs:*  
    - True: Proceeds to fetch Dialogflow intents.  
    - False: Sends invalid command message.  
  - *Edge Cases:* Empty or undefined message text.

- **Mensaje de usuario inválido (Telegram node)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends "Usuario inválido" message to unauthorized users.  
  - *Config:* Uses sender’s chat ID `$json.message.from.id`.  
  - *Inputs:* False branch from user validation.  
  - *Edge Cases:* Telegram API failures.

- **Mensaje de validación de comando (Telegram node)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends "Palabra inválida" message when command is incorrect.  
  - *Config:* Uses sender’s chat ID.  
  - *Inputs:* False branch from command validation.  
  - *Edge Cases:* Telegram API failures.

- **Sticky Note (Validation Explanation)**  
  - *Content:* "Se valida el usuario y comando enviado"  
  - Indicates that this block handles authentication and command validation.

---

#### 2.2 Dialogflow Intents Retrieval

**Overview:**  
Once the user and command are validated, this block queries the Dialogflow API to retrieve all intents from a specified project.

**Nodes Involved:**  
- Obtiene datos de los intents (HTTP Request)

**Node Details:**  

- **Obtiene datos de los intents (HTTP Request node)**  
  - *Type:* HTTP Request  
  - *Role:* Fetches intents from Dialogflow API endpoint `https://dialogflow.googleapis.com/v2/projects/TU_PROJECT_ID/agent/intents`.  
  - *Config:*  
    - Uses predefined Google API OAuth2 credentials for authentication.  
    - No additional query parameters or body.  
  - *Inputs:* True branch from command validation.  
  - *Outputs:* JSON containing intents list.  
  - *Edge Cases:*  
    - Authentication failures (expired token, invalid credentials).  
    - API rate limits.  
    - Network timeouts.  
    - Missing or malformed project ID (`TU_PROJECT_ID` must be replaced with actual).  

- **Sticky Note1**  
  - Content: "Se obtiene la información de los intents y se procesa"  
  - Highlights the retrieval and processing of intents.

---

#### 2.3 Intents Processing and Mapping

**Overview:**  
Transforms raw Dialogflow intents JSON into a structured format, mapping numeric priority levels to descriptive text and emoji labels.

**Nodes Involved:**  
- Mapear intents con su prioridad (Code node)

**Node Details:**  

- **Mapear intents con su prioridad (Code node)**  
  - *Type:* JavaScript Code Node  
  - *Role:* Parses Dialogflow JSON, extracts `displayName` and `priority` from each intent, and assigns a priority emoji and textual label.  
  - *Config:*  
    - Uses `$input.first().json.intents` as input array or empty array fallback.  
    - Defines `obtenerPrioridad(priority)` function mapping numeric priority to:  
      - ≥ 1,000,000: 🔴 Highest  
      - ≥ 750,000: 🟠 High  
      - ≥ 500,000: 🔵 Normal  
      - ≥ 250,000: 🟢 Low  
      - Default: 🚫 Ignore  
    - Outputs array of items, each with:  
      - Nombre (displayName)  
      - prioridad (numeric)  
      - colorPrioridad (emoji)  
      - textoPrioridad (description)  
  - *Inputs:* Output from HTTP Request node.  
  - *Outputs:* Items formatted for Google Sheets.  
  - *Edge Cases:*  
    - Missing `priority` field defaults to 0.  
    - Empty/null intents array.  
    - Expression or code errors.  

---

#### 2.4 Data Registration

**Overview:**  
Appends processed intents as rows to a Google Sheets spreadsheet, recording name, priority details, and timestamp.

**Nodes Involved:**  
- Añadir fila en la hoja (Google Sheets Append)  

**Node Details:**  

- **Añadir fila en la hoja (Google Sheets node)**  
  - *Type:* Google Sheets Node  
  - *Role:* Appends a new row per intent with mapped data into a sheet.  
  - *Config:*  
    - Operation: Append  
    - Document ID: `1v9VHfyEQSWYZxKCMEal0uT5_z6gV1Qx4EIwgr6ShN6E` (Google Sheet ID)  
    - Sheet Name: GID=0 (first sheet)  
    - Columns mapped as:  
      - Nombre: intent displayName  
      - Prioridad: textoPrioridad  
      - Hora de registro: current time in HH:mm:ss  
      - Fecha de registro: current date in dd/MM/yyyy  
      - Color de prioridad: emoji  
      - Valor de prioridad: numeric priority  
  - *Inputs:* Output from Code node.  
  - *Outputs:* Number of rows appended.  
  - *Edge Cases:*  
    - Google API authentication errors.  
    - Sheet not found or permission denied.  
    - API quota exceeded.  

- **Sticky Note2**  
  - Content: "Registro en Sheets y confirmación por Telegram"  
  - Indicates data storage and user notification.

---

#### 2.5 Confirmation Response

**Overview:**  
Sends a Telegram message confirming how many intents have been recorded, ensuring the message is sent only once per execution.

**Nodes Involved:**  
- Mensaje de confirmación (Telegram send message)

**Node Details:**  

- **Mensaje de confirmación (Telegram Node)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends a confirmation message including the count of intents added to Google Sheets.  
  - *Config:*  
    - Text: "Se registraron X intents en Google Sheets." where X is the length of items appended.  
    - Chat ID: same as Telegram Trigger sender ID.  
    - Execute Once: true (message sent once regardless of number of items).  
  - *Inputs:* Output from Google Sheets append node.  
  - *Edge Cases:* Telegram API failures, invalid chat ID.

---

#### 2.6 Additional Notes

- **Sticky Note3** provides a detailed explanation of the entire flow, including the trigger, validation, Dialogflow API call, JSON processing, Google Sheets logging, and Telegram confirmation.

---

### 3. Summary Table

| Node Name                   | Node Type             | Functional Role                         | Input Node(s)              | Output Node(s)                      | Sticky Note                                          |
|-----------------------------|-----------------------|---------------------------------------|----------------------------|-----------------------------------|------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger      | Starts workflow on Telegram message   | -                          | Validación de usuario por ID      |                                                      |
| Validación de usuario por ID| IF Node               | Validates Telegram user ID            | Telegram Trigger           | Validación del comando, Mensaje de usuario inválido | "Se valida el usuario y comando enviado"             |
| Validación del comando      | IF Node               | Validates command text ("backup")     | Validación de usuario por ID | Obtiene datos de los intents, Mensaje de validación de comando | "Se valida el usuario y comando enviado" |
| Mensaje de usuario inválido| Telegram Node         | Sends invalid user message            | Validación de usuario por ID (false branch) | -                                 |                                                      |
| Mensaje de validación de comando | Telegram Node   | Sends invalid command message         | Validación del comando (false branch) | -                                 |                                                      |
| Obtiene datos de los intents| HTTP Request          | Retrieves Dialogflow intents          | Validación del comando (true branch) | Mapear intents con su prioridad   | "Se obtiene la información de los intents y se procesa" |
| Mapear intents con su prioridad| Code Node          | Maps priority levels to emojis/text   | Obtiene datos de los intents| Añadir fila en la hoja             |                                                      |
| Añadir fila en la hoja      | Google Sheets         | Appends intents to Google Sheets      | Mapear intents con su prioridad | Mensaje de confirmación          | "Registro en Sheets y confirmación por Telegram"     |
| Mensaje de confirmación     | Telegram Node         | Sends confirmation message via Telegram| Añadir fila en la hoja     | -                                 |                                                      |
| Sticky Note                 | Sticky Note           | Validation block note                  | -                          | -                                 | "Se valida el usuario y comando enviado"             |
| Sticky Note1                | Sticky Note           | Intent retrieval block note            | -                          | -                                 | "Se obtiene la información de los intents y se procesa" |
| Sticky Note2                | Sticky Note           | Google Sheets registration note        | -                          | -                                 | "Registro en Sheets y confirmación por Telegram"     |
| Sticky Note3                | Sticky Note           | Full flow explanation note             | -                          | -                                 | Detailed workflow explanation                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Set Updates to listen for: `message`  
   - Configure Telegram credentials with your bot token.  
   - This node triggers on any message sent to your bot.

2. **Add IF Node "Validación de usuario por ID":**  
   - Type: If  
   - Condition: Number equals  
   - Left Value: Expression: `$json["message"]["from"]["id"]`  
   - Right Value: Your authorized Telegram user ID (replace placeholder 7894561234).  
   - Connect output of Telegram Trigger → IF node.

3. **Add IF Node "Validación del comando":**  
   - Type: If  
   - Condition: String equals  
   - Left Value: Expression: `$json["message"]["text"]`  
   - Right Value: `backup` (case-sensitive).  
   - Connect True output of user validation node here.

4. **Add Telegram Node "Mensaje de usuario inválido":**  
   - Type: Telegram  
   - Text: `Usuario inválido`  
   - Chat ID: Expression: `$json["message"]["from"]["id"]`  
   - Connect False output of user validation node here.

5. **Add Telegram Node "Mensaje de validación de comando":**  
   - Type: Telegram  
   - Text: `Palabra inválida`  
   - Chat ID: Expression: `$json["message"]["from"]["id"]`  
   - Connect False output of command validation node here.

6. **Add HTTP Request Node "Obtiene datos de los intents":**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://dialogflow.googleapis.com/v2/projects/TU_PROJECT_ID/agent/intents` (replace `TU_PROJECT_ID` with your Dialogflow project ID)  
   - Authentication: OAuth2 with Google API credentials (configured in n8n).  
   - Connect True output of command validation node here.

7. **Add Code Node "Mapear intents con su prioridad":**  
   - Type: Code (JavaScript)  
   - Paste the provided JS code that processes intents JSON, maps priority levels to emojis and text, and prepares output items.  
   - Connect output of HTTP Request node here.

8. **Add Google Sheets Node "Añadir fila en la hoja":**  
   - Type: Google Sheets  
   - Operation: Append  
   - Document ID: Your Google Sheet ID (example: `1v9VHfyEQSWYZxKCMEal0uT5_z6gV1Qx4EIwgr6ShN6E`)  
   - Sheet Name: `gid=0` (or your desired sheet)  
   - Column Mapping:  
     - Nombre: `={{ $json.Nombre }}`  
     - Prioridad: `={{ $json.textoPrioridad }}`  
     - Hora de registro: `={{ $now.format('HH:mm:ss') }}`  
     - Fecha de registro: `={{ $now.format('dd/MM/yyyy') }}`  
     - Color de prioridad: `={{ $json.colorPrioridad }}`  
     - Valor de prioridad: `={{ $json.prioridad }}`  
   - Connect output of Code node here.

9. **Add Telegram Node "Mensaje de confirmación":**  
   - Type: Telegram  
   - Text: `= Se registraron {{$items("Añadir fila en la hoja").length}} intents en Google Sheets.`  
   - Chat ID: Expression: `={{ $("Telegram Trigger").item.json.message.from.id }}`  
   - Set "Execute Once" option to true (message will send only once).  
   - Connect output of Google Sheets append node here.

10. **Ensure all Telegram nodes use proper Telegram credentials.**

11. **Deploy the workflow and test by sending the "backup" command from the authorized Telegram user.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| The workflow requires replacing placeholder `TU_PROJECT_ID` in the HTTP Request node with your actual Dialogflow Google Cloud project ID to function properly.                                                                  | Dialogflow API URL configuration                                                                        |
| Telegram user ID `7894561234` is a placeholder; replace it with the Telegram numeric ID of the authorized user who can trigger this workflow.                                                                                    | User validation setup                                                                                   |
| Google Sheets document ID and sheet GID must match your own spreadsheet where intents will be logged. Ensure your Google API credentials have write permissions for this sheet.                                                    | Google Sheets configuration                                                                              |
| The priority level mapping is customizable in the Code node; adjust emoji or threshold values as needed to fit your project's priority schema.                                                                                   | Priority mapping logic                                                                                   |
| Telegram messages for invalid user or command are sent immediately to inform users of unauthorized access or wrong commands.                                                                                                      | User feedback                                                                                           |
| For more information on Dialogflow intents API: https://cloud.google.com/dialogflow/es/docs/reference/rest/v2/projects.agent.intents/list                                                                                        | Official Dialogflow API Documentation                                                                    |
| For Google Sheets API setup in n8n, see: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets/                                                                                                        | n8n Google Sheets Integration Docs                                                                       |
| Telegram bot setup and webhook configuration: https://core.telegram.org/bots/api#available-methods                                                                                                                              | Telegram Bot API Reference                                                                                |
| The workflow includes detailed sticky notes explaining each block and flow for easier maintenance and onboarding.                                                                                                               | Embedded Sticky Notes                                                                                     |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.