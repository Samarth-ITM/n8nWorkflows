Smart Shipping Prioritization with Google Gemini and Google Sheets

https://n8nworkflows.xyz/workflows/smart-shipping-prioritization-with-google-gemini-and-google-sheets-7296


# Smart Shipping Prioritization with Google Gemini and Google Sheets

### 1. Workflow Overview

This workflow, titled **Smart Shipping Prioritization with Google Gemini and Google Sheets**, automates the process of selecting and prioritizing shipments for dispatch based on order data stored in a Google Sheets document. It leverages Google Gemini AI models alongside LangChain agents to interpret shipment priorities dynamically and update the shipment status accordingly.

The workflow is designed for logistics or fulfillment teams looking to automate shipment scheduling by combining AI-driven decision-making with real-time spreadsheet data. It reads shipment orders, applies AI logic to identify the highest priority shipment based on order date and specific customer details, updates the shipment status in Google Sheets, and responds with the selected shipment information.

**Logical Blocks:**

- **1.1 Input Reception:** Receives external triggers via webhook.
- **1.2 AI Processing:** Uses a LangChain AI Agent integrated with Google Gemini and other tools to analyze shipment data and determine priority.
- **1.3 Data Access and Update:** Reads from and updates Google Sheets that maintain shipment records.
- **1.4 Output Response:** Sends the prioritized shipment information back to the requester.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block handles incoming requests to trigger the workflow. It acts as the entry point where shipment prioritization requests are received.

**Nodes Involved:**  
- Button (Webhook)

**Node Details:**  
- **Button**  
  - *Type:* Webhook  
  - *Role:* Receives HTTP requests to start the workflow.  
  - *Configuration:*  
    - Webhook path and ID placeholder values must be replaced (`REPLACE_WITH_WEBHOOK_ID`, `REPLACE_WITH_PATH`).  
    - Response mode set to "responseNode" to use a downstream node for response.  
  - *Inputs:* None (start node)  
  - *Outputs:* Connects to AI Agent node.  
  - *Edge Cases:*  
    - Missing or incorrect webhook configuration will prevent external triggering.  
    - Unauthorized or malformed requests could cause failures if no validation is added externally.  

#### 1.2 AI Processing

**Overview:**  
This block uses AI to analyze shipment data and select the highest priority shipment. It integrates LangChain’s AI Agent with Google Gemini language models and auxiliary tools to process data and output structured JSON.

**Nodes Involved:**  
- AI Agent  
- Gemini Flash 2.5 (Google Gemini model)  
- Date & Time (AI tool)  
- Shipping (Google Sheets Tool)  
- Structured Output Parser  

**Node Details:**  

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Central AI logic executor that orchestrates multiple AI tools to determine shipment priority.  
  - *Configuration:*  
    - Prompt instructs the agent to select priority shipping and respond with JSON fields: `idEnvio`, `nombre`, `direccion`, and `detalle`.  
    - System message clarifies the mission: prioritize by oldest `fechaOrden` unless `detalle` indicates potential cancellation, which increases priority.  
    - Uses multiple tools: Date & Time, Shipping sheet (Google Sheets Tool), Gemini language model, and structured output parser.  
    - Output parser enabled to enforce JSON output structure.  
  - *Inputs:* Receives trigger from Button node and auxiliary data from Date & Time, Shipping, Gemini Flash 2.5, and Structured Output Parser nodes through AI tool interfaces.  
  - *Outputs:* Passes structured JSON output to Update and Screen nodes.  
  - *Edge Cases:*  
    - AI model response delay/timeouts.  
    - Errors in tool integration (e.g., Google Sheets access, model unavailability).  
    - Malformed AI outputs causing parsing failures.  
  - *Version Notes:* Requires LangChain nodes v2 or higher for agent capabilities.  

- **Gemini Flash 2.5**  
  - *Type:* Google Gemini Chat Model  
  - *Role:* Provides advanced language understanding and generation as part of the AI agent’s reasoning.  
  - *Configuration:* Uses the `models/gemini-2.5-flash-lite` model.  
  - *Inputs:* Invoked by AI Agent as language model.  
  - *Outputs:* Feeds language model output back to AI Agent.  
  - *Edge Cases:* API access or credential issues; model version deprecation.  

- **Date & Time**  
  - *Type:* DateTime Tool  
  - *Role:* Supplies current time/date information to AI Agent to support prioritization logic.  
  - *Inputs:* Invoked by AI Agent as an AI tool.  
  - *Outputs:* Provides date/time data back to AI Agent.  
  - *Edge Cases:* Timezone mismatches or incorrect date formatting.  

- **Shipping**  
  - *Type:* Google Sheets Tool  
  - *Role:* Reads shipment data from a Google Sheets document.  
  - *Configuration:*  
    - Requires document ID and sheet name placeholders to be replaced.  
    - Sheet contains columns: `idEnvio`, `fechaOrden`, `nombre`, `direccion`, `detalle`, `enviado`.  
  - *Inputs:* Invoked by AI Agent as an AI tool.  
  - *Outputs:* Delivers shipment data for AI analysis.  
  - *Edge Cases:*  
    - Access permission or credential errors.  
    - Sheet structure mismatch or missing data.  

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Ensures AI Agent output conforms to expected JSON schema.  
  - *Configuration:* Uses example JSON schema with fields: `idEnvio`, `nombre`, `direccion`, and `detalle`.  
  - *Inputs:* Invoked by AI Agent to parse model output.  
  - *Outputs:* Clean JSON passed back to AI Agent.  
  - *Edge Cases:* Parsing errors if AI output deviates from schema.

#### 1.3 Data Access and Update

**Overview:**  
This block updates the shipment status in the Google Sheets document based on AI-selected priority shipment.

**Nodes Involved:**  
- Update (Google Sheets)

**Node Details:**  

- **Update**  
  - *Type:* Google Sheets  
  - *Role:* Updates the `enviado` column to "X" for the prioritized shipment row identified by `idEnvio`.  
  - *Configuration:*  
    - Document ID and sheet name placeholders must be filled.  
    - Update operation with defined mapping: match rows by `idEnvio`, update `enviado` column to "X".  
    - Uses dynamic expression to retrieve `idEnvio` from AI Agent output.  
  - *Inputs:* Receives data from AI Agent node.  
  - *Outputs:* Sends updated data downstream to Screen node.  
  - *Edge Cases:*  
    - Credential or permission errors.  
    - Failure to find matching `idEnvio`.  
    - Google Sheets API rate limits or update conflicts.  

#### 1.4 Output Response

**Overview:**  
This block responds to the original webhook request with the AI-selected shipment data, completing the request cycle.

**Nodes Involved:**  
- Screen (Respond to Webhook)

**Node Details:**  

- **Screen**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends all incoming items (from Update node) as the HTTP response to the initial webhook call.  
  - *Configuration:* Set to respond with all incoming items, providing full shipment update confirmation.  
  - *Inputs:* Receives data from AI Agent and Update nodes.  
  - *Outputs:* None (end node).  
  - *Edge Cases:*  
    - Response timeout if workflow execution is slow.  
    - Potential incomplete data if upstream nodes fail.

#### Additional Node

- **Sticky Note**  
  - *Type:* Sticky Note  
  - *Role:* Documentation within the workflow canvas describing the workflow purpose and required sheet structure.  
  - *Content:* "## n8n Terminal  
      Read shipping sheet, determine priority with AI and update external screen. Connect a Shipments sheet with idEnvio, fechaOrden, nombre, direccion, detalle and enviado"

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                        | Input Node(s)         | Output Node(s)         | Sticky Note                                                                                      |
|------------------------|------------------------------------|-------------------------------------|-----------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| Button                 | Webhook                            | Entry point; receives external trigger | None                  | AI Agent              | ## n8n Terminal<br>Read shipping sheet, determine priority with AI and update external screen. Connect a Shipments sheet with idEnvio, fechaOrden, nombre, direccion, detalle and enviado |
| AI Agent               | LangChain Agent                    | AI decision-making on shipment priority | Button, Date & Time, Shipping, Gemini Flash 2.5, Structured Output Parser | Update, Screen         | ## n8n Terminal<br>Read shipping sheet, determine priority with AI and update external screen. Connect a Shipments sheet with idEnvio, fechaOrden, nombre, direccion, detalle and enviado |
| Date & Time            | DateTime Tool                     | Provides current date/time to AI Agent | None                  | AI Agent              |                                                                                                 |
| Shipping               | Google Sheets Tool                 | Reads shipments data from Google Sheets | None                  | AI Agent              |                                                                                                 |
| Gemini Flash 2.5       | Google Gemini Chat Model           | Provides language model input to AI Agent | None                  | AI Agent              |                                                                                                 |
| Structured Output Parser| LangChain Structured Output Parser| Parses AI output into JSON           | None                  | AI Agent              |                                                                                                 |
| Update                 | Google Sheets                     | Updates shipment status in Google Sheets | AI Agent              | Screen                |                                                                                                 |
| Screen                 | Respond to Webhook                 | Sends final response back to requester | AI Agent, Update      | None                  |                                                                                                 |
| Sticky Note            | Sticky Note                       | Documentation on workflow purpose    | None                  | None                  | ## n8n Terminal<br>Read shipping sheet, determine priority with AI and update external screen. Connect a Shipments sheet with idEnvio, fechaOrden, nombre, direccion, detalle and enviado |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node ("Button"):**  
   - Type: Webhook  
   - Set webhook path (e.g., `/shipping-prioritization`) and webhook ID (auto-generated or custom).  
   - Configure response mode to "responseNode".  

2. **Add LangChain AI Agent Node ("AI Agent"):**  
   - Type: LangChain Agent (must have LangChain nodes installed)  
   - Configure prompt type as "define".  
   - Set prompt text: "Select priority shipping. Answer with Json with idEnvio, nombre, direccion and detalle".  
   - Add system message:  
     ```
     You are an assistant with access to these tools:
     Date and Time
     Shipping sheet

     Your mission is to select one sheet row for next shipment.

     Priority order is oldest fechaOrden, but if the customer says that it could cancel in detalle column, that is priority.
     ```  
   - Enable output parser with structured JSON schema.  
   - Configure AI tools for this agent as follows (see next steps).  
   - Connect input from "Button" node.

3. **Create Date & Time Node:**  
   - Type: DateTime Tool  
   - No parameters needed.  
   - Connect as AI tool input to AI Agent.

4. **Create Shipping Node:**  
   - Type: Google Sheets Tool  
   - Set Google Sheets credentials.  
   - Enter Google Sheets Document ID (replace placeholder).  
   - Enter Sheet Name containing shipment data (replace placeholder).  
   - This sheet must include columns: `idEnvio`, `fechaOrden`, `nombre`, `direccion`, `detalle`, `enviado`.  
   - Connect as AI tool input to AI Agent.

5. **Create Gemini Flash 2.5 Node:**  
   - Type: LangChain Google Gemini Chat Model  
   - Choose model: `models/gemini-2.5-flash-lite`.  
   - Configure credentials for Google Gemini API access.  
   - Connect as AI language model input to AI Agent.

6. **Create Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser  
   - Provide example JSON schema:  
     ```json
     {
       "idEnvio": "1",
       "nombre": "John Doe",
       "direccion": "8345 NW 55 st",
       "detalle": "I need it today"
     }
     ```  
   - Connect as AI output parser input to AI Agent.

7. **Create Update Node:**  
   - Type: Google Sheets  
   - Use same Google Sheets credentials as Shipping node.  
   - Set document ID and sheet name (same as Shipping).  
   - Operation: Update  
   - Mapping Mode: Define Below  
   - Matching Columns: `idEnvio`  
   - Columns to update:  
     - `enviado`: static value `"X"`  
     - `idEnvio`: expression referencing AI Agent output: `{{$node["AI Agent"].json["output"]["idEnvio"]}}`  
   - Connect input from AI Agent node.

8. **Create Screen Node:**  
   - Type: Respond to Webhook  
   - Set to respond with all incoming items.  
   - Connect inputs from AI Agent and Update nodes.

9. **Link Connections:**  
   - Button → AI Agent  
   - Date & Time → AI Agent (as AI tool)  
   - Shipping → AI Agent (as AI tool)  
   - Gemini Flash 2.5 → AI Agent (as language model)  
   - Structured Output Parser → AI Agent (as output parser)  
   - AI Agent → Update  
   - AI Agent → Screen  
   - Update → Screen

10. **Create Sticky Note:**  
    - Add a sticky note with text:  
      ```
      ## n8n Terminal
      Read shipping sheet, determine priority with AI and update external screen. Connect a Shipments sheet with idEnvio, fechaOrden, nombre, direccion, detalle and enviado
      ```  
    - Place near entry node for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                             |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow uses Google Gemini AI via LangChain nodes to enhance AI decision-making capabilities. | Requires valid Google Gemini API credentials and LangChain setup. |
| Google Sheets must have columns: `idEnvio`, `fechaOrden`, `nombre`, `direccion`, `detalle`, `enviado`. | Sheet structure is critical for correct operation.          |
| Replace all placeholders (`REPLACE_WITH_WEBHOOK_ID`, `REPLACE_WITH_PATH`, `REPLACE_WITH_DOC_ID`, `REPLACE_WITH_SHEET_NAME`) before deployment. | Configuration step before workflow activation.               |
| AI priority logic: prioritize oldest order date unless "detalle" mentions possible cancellation, which raises priority. | Ensures urgent or at-risk shipments are dispatched first.    |
| LangChain nodes version 2+ required for Agent functionality; Google Sheets nodes version 4.6 used. | Compatibility with these versions recommended.                |

---

*Disclaimer:* The provided content is derived exclusively from an automated workflow created with n8n, strictly adhering to current content policies and containing no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.