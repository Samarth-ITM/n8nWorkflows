🛠️ NASA Tool MCP Server 💪 all 15 operations

https://n8nworkflows.xyz/workflows/----nasa-tool-mcp-server----all-15-operations-5118


# 🛠️ NASA Tool MCP Server 💪 all 15 operations

### 1. Workflow Overview

This workflow, titled **🛠️ NASA Tool MCP Server 💪 all 15 operations**, serves as a comprehensive integration interface to NASA's open data services via n8n's NASA Tool nodes. It exposes 15 distinct NASA data retrieval operations through a single webhook trigger named "NASA Tool MCP Server." This setup allows AI agents or external systems to dynamically invoke any of the available NASA data operations by passing parameters, which the workflow automatically processes and routes.

The workflow is logically structured into the following functional blocks:

- **1.1 Input Reception and Triggering**: A webhook trigger node that receives external requests through a defined endpoint (`nasa-tool-mcp` path), serving as the entry point for all operations.

- **1.2 NASA Data Operations**: Fifteen individual NASA Tool nodes, each dedicated to a specific NASA data resource and operation, grouped by thematic categories:
  - Earth-related data operations (Earth assets, Earth imagery)
  - Asteroid Near Earth Object (NEO) operations (browse, feed, lookup)
  - Astronomy Picture of the Day
  - DONKI space weather event operations (coronal mass ejections, solar flares, radiation belts, interplanetary shocks, etc.)

- **1.3 Documentation and User Guidance**: Sticky Note nodes distributed spatially to provide inline documentation, operation lists, setup instructions, and thematic block labeling for maintainability and user clarity.

Together, these blocks create a zero-configuration-ready NASA data MCP server endpoint that AI agents can use to request any of the 15 supported data operations with parameters automatically populated via `$fromAI()` expressions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Triggering

- **Overview:**  
This block initiates the workflow via an MCP (Modular Chatbot Platform) trigger node configured with a webhook path `nasa-tool-mcp`. It listens for incoming API calls or AI agent requests, which carry parameters for invoking NASA data operations.

- **Nodes Involved:**  
  - NASA Tool MCP Server (MCP Trigger node)

- **Node Details:**  
  - **NASA Tool MCP Server**  
    - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Role: Receives external HTTP requests via webhook and triggers the workflow execution.  
    - Configuration: Webhook path set to `nasa-tool-mcp`.  
    - Inputs: None (trigger node).  
    - Outputs: Passes request data and parameters downstream to NASA Tool nodes via AI tool connection.  
    - Version-specific: Requires n8n version supporting MCP triggers and LangChain integration nodes.  
    - Potential failures: Webhook URL misconfiguration, authentication issues if applicable, malformed requests, or timeout if no response from downstream nodes.  
    - Notes: The webhook URL generated here is the main integration point for AI agents.

---

#### 2.2 NASA Data Operations

- **Overview:**  
This block contains 15 individual NASA Tool nodes, each configured to query a specific NASA API resource. All nodes receive parameters dynamically populated via `$fromAI()` expressions, enabling flexible operation invocation without manual parameter input.

- **Nodes Involved:**  
  - Get many asteroid neos  
  - Get an asteroid neo feed  
  - Get an asteroid neo lookup  
  - Get the astronomy picture of the day  
  - Get a DONKI coronal mass ejection  
  - Get a DONKI high speed stream  
  - Get a DONKI interplanetary shock  
  - Get a DONKI magnetopause crossing  
  - Get a DONKI notifications  
  - Get a DONKI radiation belt enhancement  
  - Get a DONKI solar energetic particle  
  - Get a DONKI solar flare  
  - Get a DONKI wsa enlil simulation  
  - Get Earth assets  
  - Get Earth imagery

- **Node Details:**

  Each NASA Tool node shares these common configuration patterns:

  - Type: `n8n-nodes-base.nasaTool`  
  - Role: Calls the specified NASA API resource with parameters.  
  - Credentials: NASA Tool credentials must be configured once and shared across nodes.  
  - Input Parameters: Dynamically filled via `$fromAI()` expressions where applicable, e.g.:  
    - `asteroidId` for asteroid neo lookup  
    - `lat` and `lon` for Earth assets and imagery  
    - `download` and `binaryPropertyName` for astronomy picture download options  
  - Output: NASA API response data per resource operation.  
  - Version-specific: Requires n8n version with NASA Tool node support.  
  - Potential Failures:  
    - API rate limiting or quota exceeded errors from NASA  
    - Invalid or missing parameters (e.g., missing asteroid ID or coordinates)  
    - Network timeouts or transient NASA API failures  
    - Expression evaluation failures if `$fromAI()` input is not properly supplied  
  - Sub-workflow: None; all nodes directly connected to MCP trigger via "ai_tool" type connection.

  Specific notes on select nodes:

  - **Get an asteroid neo lookup**: Uses expression `{{$fromAI('Asteroid_Id', ``, 'string')}}` to dynamically fetch the asteroid ID from AI input. This enables querying specific near-earth objects flexibly.

  - **Get Earth assets / Get Earth imagery**: Use latitude and longitude parameters dynamically from AI input with expressions like `{{$fromAI('Lat', ``, 'number')}}`. This allows location-based Earth data retrieval.

  - **Get the astronomy picture of the day**: Accepts dynamic boolean for download and string for binary property name, allowing optional media file downloads.

  - All DONKI nodes: Access various space weather event data with no additional parameters configured by default but ready to accept overrides.

---

#### 2.3 Documentation and User Guidance

- **Overview:**  
Sticky Note nodes placed throughout the canvas provide comprehensive documentation, usage instructions, and thematic grouping to enhance workflow maintainability and user onboarding.

- **Nodes Involved:**  
  - Workflow Overview 0  
  - Sticky Note 1 (Earth)  
  - Sticky Note 4 (Astronomy Picture of The Day)  
  - Sticky Note 9 (DONKI)  
  - Sticky Note 11 (Asteroid Neo)

- **Node Details:**  
  - Type: `n8n-nodes-base.stickyNote`  
  - Role: Present human-readable markdown documentation inside the workflow editor.  
  - Content Highlights:  
    - Workflow Overview 0: Lists all 15 operations, setup instructions, and user guidance with links to n8n documentation and Discord support.  
    - Sticky Note 1, 4, 9, 11: Serve as headers grouping related NASA Tool nodes by data domain (Earth, Astronomy Picture, DONKI, Asteroid Neo).  
  - Positioning: Strategically placed for visual clarity around corresponding data operation nodes.  
  - No inputs or outputs; purely documentation.

---

### 3. Summary Table

| Node Name                       | Node Type                          | Functional Role                            | Input Node(s)         | Output Node(s)       | Sticky Note                                                                                          |
|--------------------------------|----------------------------------|--------------------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------------|
| Workflow Overview 0             | Sticky Note                      | Overall workflow documentation and setup   | None                  | None                 | Lists all 15 operations, setup instructions, and help links                                        |
| NASA Tool MCP Server            | MCP Trigger                     | Entry webhook trigger for all NASA ops     | None                  | All NASA Tool nodes  |                                                                                                    |
| Get many asteroid neos          | NASA Tool                       | Retrieve list of many asteroid near-earth objects | NASA Tool MCP Server  | None                 | Grouped under "Asteroid Neo" Sticky Note 11                                                        |
| Get an asteroid neo feed        | NASA Tool                       | Retrieve asteroid near-earth object feed   | NASA Tool MCP Server  | None                 | Grouped under "Asteroid Neo" Sticky Note 11                                                        |
| Get an asteroid neo lookup      | NASA Tool                       | Lookup details for a specific asteroid      | NASA Tool MCP Server  | None                 | Grouped under "Asteroid Neo" Sticky Note 11                                                        |
| Get the astronomy picture of the day | NASA Tool                 | Retrieve NASA's daily astronomy picture    | NASA Tool MCP Server  | None                 | Grouped under "Astronomy Picture of The Day" Sticky Note 4                                         |
| Get a DONKI coronal mass ejection | NASA Tool                   | Retrieve DONKI coronal mass ejection data  | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI high speed stream  | NASA Tool                       | Retrieve DONKI high speed solar wind data  | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI interplanetary shock | NASA Tool                     | Retrieve DONKI interplanetary shock data   | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI magnetopause crossing | NASA Tool                   | Retrieve DONKI magnetopause crossing events | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI notifications      | NASA Tool                       | Retrieve DONKI notifications                | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI radiation belt enhancement | NASA Tool               | Retrieve DONKI radiation belt enhancement data | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI solar energetic particle | NASA Tool                 | Retrieve DONKI solar energetic particle data | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI solar flare        | NASA Tool                       | Retrieve DONKI solar flare data             | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get a DONKI wsa enlil simulation | NASA Tool                     | Retrieve DONKI WSA-Enlil solar wind simulation data | NASA Tool MCP Server  | None                 | Grouped under "DONKI" Sticky Note 9                                                                |
| Get Earth assets               | NASA Tool                       | Retrieve Earth assets at given coordinates  | NASA Tool MCP Server  | None                 | Grouped under "Earth" Sticky Note 1                                                                |
| Get Earth imagery              | NASA Tool                       | Retrieve Earth imagery for given coordinates | NASA Tool MCP Server  | None                 | Grouped under "Earth" Sticky Note 1                                                                |
| Sticky Note 1                  | Sticky Note                    | Label and group Earth-related NASA Tool nodes | None                  | None                 | "Earth"                                                                                            |
| Sticky Note 4                  | Sticky Note                    | Label and group Astronomy Picture node      | None                  | None                 | "Astronomy Picture of The Day"                                                                     |
| Sticky Note 9                  | Sticky Note                    | Label and group DONKI-related NASA Tool nodes | None                  | None                 | "DONKI"                                                                                            |
| Sticky Note 11                 | Sticky Note                    | Label and group Asteroid Near Earth Object nodes | None                  | None                 | "Asteroid Neo"                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a New Workflow**

2. **Add Sticky Note "Workflow Overview 0"**  
   - Type: Sticky Note  
   - Content: Paste the detailed overview and setup instructions as per the original note, including operation list and helpful links.  
   - Size: Width ~420, Height ~1060  
   - Position: Place on left side for overall documentation.

3. **Add MCP Trigger Node "NASA Tool MCP Server"**  
   - Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Parameters:  
     - Path: `nasa-tool-mcp`  
   - Position: Around center-left (e.g., X:-1080, Y:160)  
   - No credentials needed here.  
   - This node serves as the webhook entry point.

4. **Add NASA Tool Nodes for 15 Operations**  
   For each operation below, create a NASA Tool node with the specified resource and parameters:

   - **Earth Block** (Add Sticky Note "Earth" with size 660x200 at approx. X:-2000, Y:560)  
     a. "Get Earth assets"  
        - Resource: `earthAssets`  
        - Parameters:  
          - `lat`: `{{$fromAI('Lat', ``, 'number')}}`  
          - `lon`: `{{$fromAI('Lon', ``, 'number')}}`  
        - Position: X:-1840, Y:600  
     b. "Get Earth imagery"  
        - Resource: `earthImagery`  
        - Parameters:  
          - `lat`: `{{$fromAI('Lat', ``, 'number')}}`  
          - `lon`: `{{$fromAI('Lon', ``, 'number')}}`  
          - `binaryPropertyName`: `{{$fromAI('Binary_Property_Name', ``, 'string')}}`  
        - Position: X:-1560, Y:600  

   - **Asteroid Neo Block** (Add Sticky Note "Asteroid Neo" at X:-800, Y:560)  
     a. "Get many asteroid neos"  
        - Resource: `asteroidNeoBrowse`  
        - Position: X:-560, Y:600  
     b. "Get an asteroid neo feed"  
        - Resource: `asteroidNeoFeed`  
        - Position: X:-60, Y:600  
     c. "Get an asteroid neo lookup"  
        - Resource: `asteroidNeoLookup`  
        - Parameters:  
          - `asteroidId`: `{{$fromAI('Asteroid_Id', ``, 'string')}}`  
        - Position: X:-300, Y:600  

   - **Astronomy Picture of the Day Block** (Add Sticky Note "Astronomy Picture of The Day" at X:-1260, Y:540)  
     a. "Get the astronomy picture of the day"  
        - Resource: `astronomyPictureOfTheDay` (inferred from context)  
        - Parameters:  
          - `download`: `{{$fromAI('Download', ``, 'boolean')}}`  
          - `binaryPropertyName`: `{{$fromAI('Binary_Property_Name', ``, 'string')}}`  
        - Position: X:-1100, Y:600  

   - **DONKI Block** (Add Sticky Note "DONKI" at X:-2000, Y:840)  
     a. "Get a DONKI coronal mass ejection"  
        - Resource: `donkiCoronalMassEjection`  
        - Position: X:-300, Y:880  
     b. "Get a DONKI high speed stream"  
        - Resource: `donkiHighSpeedStream`  
        - Position: X:-1320, Y:880  
     c. "Get a DONKI interplanetary shock"  
        - Resource: `donkiInterplanetaryShock`  
        - Position: X:-60, Y:880  
     d. "Get a DONKI magnetopause crossing"  
        - Resource: `donkiMagnetopauseCrossing`  
        - Position: X:-1840, Y:880  
     e. "Get a DONKI notifications"  
        - Resource: `donkiNotifications`  
        - Position: X:-1040, Y:880  
     f. "Get a DONKI radiation belt enhancement"  
        - Resource: `donkiRadiationBeltEnhancement`  
        - Position: X:180, Y:880  
     g. "Get a DONKI solar energetic particle"  
        - Resource: `donkiSolarEnergeticParticle`  
        - Position: X:-800, Y:880  
     h. "Get a DONKI solar flare"  
        - Resource: `donkiSolarFlare`  
        - Position: X:-560, Y:880  
     i. "Get a DONKI wsa enlil simulation"  
        - Resource: `donkiWsaEnlilSimulation`  
        - Position: X:-1580, Y:880  

5. **Connect all NASA Tool nodes to the MCP Trigger node**  
   - Use the "ai_tool" connection type (if using LangChain MCP node integration) or standard connections if MCP trigger supports multiple outputs.  
   - This setup allows the MCP trigger node to route AI or external requests to the appropriate NASA Tool node.

6. **Add Sticky Notes for thematic grouping**  
   - Add Sticky Notes labeled "Earth", "Asteroid Neo", "Astronomy Picture of The Day", and "DONKI" in proximity to their respective NASA Tool nodes with appropriate colors and sizes matching original workflow.

7. **Configure NASA Tool Credentials**  
   - In the n8n credentials manager, create and configure NASA Tool credentials with required API keys or OAuth tokens as per NASA API requirements.  
   - Link credentials to all NASA Tool nodes for authentication.  
   - Note: Only one credential configuration is needed and can be reused across nodes.

8. **Save and Activate Workflow**  
   - Save workflow.  
   - Activate to enable webhook endpoint.  
   - Copy the webhook URL from the MCP trigger node for external AI agent integration.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                                            |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| Workflow supports 15 NASA data operations pre-built and ready for AI integration.               | Workflow Overview sticky note                                                                                        |
| Setup requires adding NASA Tool credentials once, then opening and closing other nodes to sync. | Workflow Overview sticky note                                                                                        |
| Use the generated webhook URL as MCP server endpoint for AI agent configurations.               | Workflow Overview sticky note                                                                                        |
| Zero configuration needed; AI agents auto-populate parameters via `$fromAI()` expressions.      | Workflow Overview sticky note                                                                                        |
| For MCP integration help, consult [n8n MCP Docs](https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/) or join Discord at [discord.me/cfomodz](https://discord.me/cfomodz) | Workflow Overview sticky note                                                                                        |
| DONKI nodes provide comprehensive solar and space weather event data integration.               | DONKI Sticky Note                                                                                                       |
| Asteroid Neo nodes enable detailed near-earth object data retrieval.                             | Asteroid Neo Sticky Note                                                                                               |
| Earth nodes require latitude and longitude parameters dynamically supplied by AI input.          | Earth Sticky Note                                                                                                       |
| Astronomy Picture of the Day node supports optional download and binary property naming.         | Astronomy Picture of the Day Sticky Note                                                                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow. It complies strictly with prevailing content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public NASA data.