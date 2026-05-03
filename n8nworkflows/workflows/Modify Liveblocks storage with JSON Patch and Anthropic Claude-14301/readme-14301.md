Modify Liveblocks storage with JSON Patch and Anthropic Claude

https://n8nworkflows.xyz/workflows/modify-liveblocks-storage-with-json-patch-and-anthropic-claude-14301


# Modify Liveblocks storage with JSON Patch and Anthropic Claude

# 1. Workflow Overview

This workflow demonstrates how to modify a Liveblocks collaborative document by generating JSON Patch operations with Anthropic Claude and applying them back to Liveblocks Storage.

Its main use case is AI-assisted editing of multiplayer application state, such as a shared canvas or design document. In this example, the document contains a `/shapes` array, and the AI receives the current state plus a natural-language change request: add a blue circle and make the square orange.

The workflow is organized into these logical blocks:

## 1.1 Workflow Entry and Room Creation
The workflow starts manually, creates a new Liveblocks room using the current execution ID as the room identifier, and prepares an empty collaborative context.

## 1.2 Initial Storage Seeding
A first JSON Patch operation initializes the room storage by adding a `/shapes` array containing sample shapes.

## 1.3 Storage Retrieval and AI Presence Start
The workflow fetches the current storage document and, in parallel, starts showing an AI presence indicator in the room.

## 1.4 AI JSON Patch Generation
The current shapes document is passed to an AI Agent backed by Anthropic Claude. The model is instructed to return only JSON Patch operations, and a structured output parser constrains the response to a JSON Patch-like schema.

## 1.5 Applying AI Changes and Presence Expiration
The generated patch is applied to Liveblocks Storage, and the AI presence is updated to expire shortly after.

## 1.6 Final Verification
The workflow retrieves the updated storage document so the final room state can be inspected.

---

# 2. Block-by-Block Analysis

## 2.1 Workflow Entry and Room Creation

**Overview:**  
This block initializes the run. A manual trigger starts execution, and a new Liveblocks room is created using the execution ID so the workflow has an isolated room to work with.

**Nodes Involved:**  
- When clicking ‘Execute workflow’
- Create a room

### Node: When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Manual entry point for testing or ad hoc execution.
- **Configuration choices:**  
  No parameters are configured; it simply starts the workflow when executed from the editor.
- **Key expressions or variables used:**  
  None in this node.
- **Input and output connections:**  
  - Input: none
  - Output: `Create a room`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - No runtime failure in normal use
  - Only available for manual/editor execution, so unsuitable as-is for production automation
- **Sub-workflow reference:**  
  None

### Node: Create a room
- **Type and technical role:** `CUSTOM.liveblocks`  
  Calls the Liveblocks API to create a room.
- **Configuration choices:**  
  - Operation: `createRoom`
  - Room ID: uses `{{$execution.id}}`
  - Default accesses: empty array `[]`
- **Key expressions or variables used:**  
  - `={{ $execution.id }}` as the room identifier
  - `={{ [] }}` for default access definitions
- **Input and output connections:**  
  - Input: `When clicking ‘Execute workflow’`
  - Output: `Patch room storage`
- **Version-specific requirements:**  
  Type version `1` of the custom Liveblocks node
- **Edge cases or potential failure types:**  
  - Liveblocks authentication failure
  - Room creation conflict if the same execution ID were somehow reused
  - API validation failure if access format expectations change in the custom node
- **Sub-workflow reference:**  
  None

---

## 2.2 Initial Storage Seeding

**Overview:**  
This block initializes the room’s collaborative document with a `/shapes` array containing two example objects. It ensures the AI has meaningful document state to modify later.

**Nodes Involved:**  
- Patch room storage

### Node: Patch room storage
- **Type and technical role:** `CUSTOM.liveblocks`  
  Applies a JSON Patch payload to the room storage document.
- **Configuration choices:**  
  - Resource: `storage`
  - Operation: `patchStorageDocument`
  - Room ID comes from the previously created room: `{{$json.id}}`
  - Patch body adds `/shapes` with two entries:
    - `rect-1`, type `rectangle`, red
    - `circle-1`, type `circle`, green
- **Key expressions or variables used:**  
  - `={{ $json.id }}` to use the room ID returned by the room creation node
- **Input and output connections:**  
  - Input: `Create a room`
  - Output: `Get room storage`
- **Version-specific requirements:**  
  Type version `1` of the custom Liveblocks node
- **Edge cases or potential failure types:**  
  - Invalid patch syntax
  - Patch failure if Liveblocks expects the storage root or path to exist in a different form
  - API rejection if the payload does not match JSON Patch requirements
- **Sub-workflow reference:**  
  None

---

## 2.3 Storage Retrieval and AI Presence Start

**Overview:**  
This block reads the current document state from Liveblocks and also shows AI presence in the room. The retrieval result feeds the AI step, while presence gives front-end users a visible “AI assistant is working” indicator.

**Nodes Involved:**  
- Get room storage
- Set presence in a room

### Node: Get room storage
- **Type and technical role:** `CUSTOM.liveblocks`  
  Fetches the current Liveblocks storage document as JSON.
- **Configuration choices:**  
  - Resource: `storage`
  - Operation: `getStorageDocument`
  - Room ID: `{{$execution.id}}`
  - Format: `json`
- **Key expressions or variables used:**  
  - `={{ $execution.id }}`
- **Input and output connections:**  
  - Input: `Patch room storage`
  - Outputs:
    - `AI Agent`
    - `Set presence in a room`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Storage not found if room creation or initialization failed
  - API auth failure
  - Unexpected output shape could break downstream expression access such as `$json.shapes`
- **Sub-workflow reference:**  
  None

### Node: Set presence in a room
- **Type and technical role:** `CUSTOM.liveblocks`  
  Sets a temporary user presence entry in the room so the AI appears as an active participant.
- **Configuration choices:**  
  - Operation: `setPresence`
  - Room ID: `{{$execution.id}}`
  - User ID: `__AI_AGENT`
  - User info JSON:
    - name: `AI Assistant`
    - avatar: `https://liveblocks.io/api/avatar?u=__AI_AGENT&agent=true`
  - TTL: `null`, meaning no expiration is defined at this stage
- **Key expressions or variables used:**  
  - `={{ $execution.id }}`
- **Input and output connections:**  
  - Input: `Get room storage`
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Auth/API failures
  - Presence payload formatting problems if the custom node expects strict JSON parsing
  - Since this branch is independent, failure here may not stop AI patch generation depending on n8n error handling settings
- **Sub-workflow reference:**  
  None

---

## 2.4 AI JSON Patch Generation

**Overview:**  
This block converts the current room storage into AI instructions and constrains the model to emit JSON Patch operations only. Anthropic Claude serves as the language model, while a structured parser enforces an array-of-operations schema.

**Nodes Involved:**  
- AI Agent
- Anthropic Chat Model
- Structured Output Parser

### Node: AI Agent
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`  
  Orchestrates an LLM call using a fixed prompt and structured output parser.
- **Configuration choices:**  
  - Prompt type: `define`
  - Output parser enabled
  - Prompt content includes:
    - A role instruction: generate JSON Patch operations only
    - The current document rendered with `JSON.stringify($json.shapes, null, 2)`
    - A natural-language request: “Add a blue circle, and make the square orange.”
    - An instruction to apply changes to `/shapes`
- **Key expressions or variables used:**  
  - `{{ JSON.stringify($json.shapes, null, 2) }}`
- **Input and output connections:**  
  - Main input: `Get room storage`
  - Main output: `Patch room storage1`
  - AI language model input: `Anthropic Chat Model`
  - AI output parser input: `Structured Output Parser`
- **Version-specific requirements:**  
  Type version `3.1`
- **Edge cases or potential failure types:**  
  - Prompt typo risk: the text says “JSON patch generatgor” and “JSPN”; harmless unless the model misinterprets it
  - If `$json.shapes` is missing or not serializable as expected, the prompt content may be wrong
  - Model may still output semantically incorrect patches even when syntactically valid
  - If the model returns operations outside the allowed schema, parser validation may fail
  - Ambiguity: the request says “square,” but the document contains a rectangle; the model must infer that `rect-1` is the intended target
- **Sub-workflow reference:**  
  None

### Node: Anthropic Chat Model
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatAnthropic`  
  Provides the Claude model used by the AI Agent.
- **Configuration choices:**  
  - Model: `claude-sonnet-4-6`
  - No custom options set
- **Key expressions or variables used:**  
  None
- **Input and output connections:**  
  - AI language model output to: `AI Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Anthropic authentication failure
  - Model availability or quota issues
  - Timeout/rate limit
  - Regional or account access limitations for the selected model
- **Sub-workflow reference:**  
  None

### Node: Structured Output Parser
- **Type and technical role:** `@n8n/n8n-nodes-langchain.outputParserStructured`  
  Enforces that the LLM output conforms to a JSON schema.
- **Configuration choices:**  
  - Manual schema
  - Expects an array of objects with:
    - `op`: string, one of `add`, `remove`, `replace`
    - `path`: string
    - `value`: unrestricted
  - Required fields: `op`, `path`
- **Key expressions or variables used:**  
  The schema is provided as an expression string.
- **Input and output connections:**  
  - AI output parser output to: `AI Agent`
- **Version-specific requirements:**  
  Type version `1.3`
- **Edge cases or potential failure types:**  
  - Parser failure if the model outputs plain text, markdown, or malformed JSON
  - Schema is permissive on `value`, so semantic correctness is not guaranteed
  - Missing support for JSON Patch ops like `move`, `copy`, `test`; only three operations are allowed
- **Sub-workflow reference:**  
  None

---

## 2.5 Applying AI Changes and Presence Expiration

**Overview:**  
This block applies the AI-generated patch operations to the room storage and then updates AI presence so it disappears shortly after the work completes.

**Nodes Involved:**  
- Patch room storage1
- Set presence in a room1

### Node: Patch room storage1
- **Type and technical role:** `CUSTOM.liveblocks`  
  Applies the AI-generated JSON Patch operations to Liveblocks Storage.
- **Configuration choices:**  
  - Resource: `storage`
  - Operation: `patchStorageDocument`
  - Room ID: `{{$execution.id}}`
  - Body: `{{$json.output}}`, expected to contain the parsed array of patch operations returned by the AI Agent
- **Key expressions or variables used:**  
  - `={{ $json.output }}`
  - `={{ $execution.id }}`
- **Input and output connections:**  
  - Input: `AI Agent`
  - Outputs:
    - `Get room storage1`
    - `Set presence in a room1`
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - `$json.output` may not be shaped exactly as the Liveblocks API expects
  - Generated path may be wrong, such as replacing the wrong element in `/shapes`
  - Patch may fail if it references non-existent indexes or invalid structure
- **Sub-workflow reference:**  
  None

### Node: Set presence in a room1
- **Type and technical role:** `CUSTOM.liveblocks`  
  Updates the AI presence with a short TTL so it disappears after the operation.
- **Configuration choices:**  
  - Operation: `setPresence`
  - Room ID: `{{$execution.id}}`
  - User ID: `__AI_AGENT`
  - Same user info as earlier
  - TTL: `2` seconds
- **Key expressions or variables used:**  
  - `={{ $execution.id }}`
- **Input and output connections:**  
  - Input: `Patch room storage1`
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Presence API failure
  - If earlier presence was never set, this may still succeed or simply create an expiring presence depending on Liveblocks behavior
- **Sub-workflow reference:**  
  None

---

## 2.6 Final Verification

**Overview:**  
This block retrieves the final storage document after the AI-generated patch has been applied. It is used to verify that the collaborative state changed as intended.

**Nodes Involved:**  
- Get room storage1

### Node: Get room storage1
- **Type and technical role:** `CUSTOM.liveblocks`  
  Reads the updated room storage document in JSON format.
- **Configuration choices:**  
  - Resource: `storage`
  - Operation: `getStorageDocument`
  - Room ID: `{{$execution.id}}`
  - Format: `json`
- **Key expressions or variables used:**  
  - `={{ $execution.id }}`
- **Input and output connections:**  
  - Input: `Patch room storage1`
  - Output: none
- **Version-specific requirements:**  
  Type version `1`
- **Edge cases or potential failure types:**  
  - Read-after-write timing issues are unlikely but possible depending on backend consistency
  - Auth/API failure
- **Sub-workflow reference:**  
  None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | n8n-nodes-base.manualTrigger | Manual workflow entry point |  | Create a room | ## Modify Liveblocks Storage with JSON Patch<br><br>### How it works<br>This example uses [Liveblocks Storage](https://liveblocks.io/docs/ready-made-features/multiplayer/sync-engine/liveblocks-storage), a sync engine created by [Liveblocks](https://liveblocks.io) that allows you to create collaborative applications like Figma, Pitch, and Spline. When we fetch the Storage value for a [room](https://liveblocks.io/docs/concepts#Rooms), we're fetching the state of the multiplayer document which users are collaborating on.<br>In this workflow example, our document holds a list of shapes, like a drawing tool. Here's a rectangle, for example:<br><br>`{ "id": "rect-1", "type": "rectangle", "x": 100, "y": 150, "width": 200, "height": 100, "color": "#ff0000" }`<br><br>Picture this hooked up to a design tool like Figma, with the user asking AI to edit their document.<br><br>In these nodes, to generates a [JSON Patch](https://liveblocks.io/docs/guides/modifying-storage-via-rest-api-with-json-patch) operation from the user's request ("Add a blue circle, and make the square orange") and applies it to the collaborative document.<br><br>As soon as the JSON Patch operation has run, each user's design tool in their web browser will update with the changes in real time.<br><br>Additionally, we're setting presence in the room, which means that the AI will appear in the document's live avatar stacks while it works, before disappearing shortly after.<br><br>### Setup<br>No setup required. Replace the trigger with any you like to use this workflow. |
| Create a room | CUSTOM.liveblocks | Create a Liveblocks room | When clicking ‘Execute workflow’ | Patch room storage | ## Create a room<br>Create a multiplayer room and initialize it with a Storage value. In a real app, this'll already be created. |
| Patch room storage | CUSTOM.liveblocks | Seed initial storage with shapes | Create a room | Get room storage | ## Create a room<br>Create a multiplayer room and initialize it with a Storage value. In a real app, this'll already be created. |
| Get room storage | CUSTOM.liveblocks | Fetch current storage document | Patch room storage | AI Agent; Set presence in a room | ## Patching Storage with AI<br>Getting the full Storage value, passing it to AI, and asking it to run a user query on the data. The AI is told to return JSON Patch operations, and is then applied to Storage. On the front end, the room updates in real time. |
| AI Agent | @n8n/n8n-nodes-langchain.agent | Generate JSON Patch operations from natural language | Get room storage | Patch room storage1 | ## Patching Storage with AI<br>Getting the full Storage value, passing it to AI, and asking it to run a user query on the data. The AI is told to return JSON Patch operations, and is then applied to Storage. On the front end, the room updates in real time. |
| Anthropic Chat Model | @n8n/n8n-nodes-langchain.lmChatAnthropic | Claude model backing the AI Agent |  | AI Agent | ## Patching Storage with AI<br>Getting the full Storage value, passing it to AI, and asking it to run a user query on the data. The AI is told to return JSON Patch operations, and is then applied to Storage. On the front end, the room updates in real time. |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Enforce JSON Patch output schema |  | AI Agent | ## Patching Storage with AI<br>Getting the full Storage value, passing it to AI, and asking it to run a user query on the data. The AI is told to return JSON Patch operations, and is then applied to Storage. On the front end, the room updates in real time. |
| Patch room storage1 | CUSTOM.liveblocks | Apply AI-generated patch to storage | AI Agent | Get room storage1; Set presence in a room1 | ## Patching Storage with AI<br>Getting the full Storage value, passing it to AI, and asking it to run a user query on the data. The AI is told to return JSON Patch operations, and is then applied to Storage. On the front end, the room updates in real time. |
| Get room storage1 | CUSTOM.liveblocks | Fetch final storage for verification | Patch room storage1 |  | ## Check value<br>Checking the final value after applying the operation. |
| Set presence in a room | CUSTOM.liveblocks | Show AI presence in the room | Get room storage |  | ## Show presence<br>Show AI presence in the app's avatar stack. |
| Set presence in a room1 | CUSTOM.liveblocks | Expire AI presence after work completes | Patch room storage1 |  | ## Hide presence<br>Tell the AI presence to expire in 2 seconds. |
| Sticky Note | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note1 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note2 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note3 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note4 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |
| Sticky Note6 | n8n-nodes-base.stickyNote | Documentation note |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Modify Liveblocks Storage with JSON Patch`.
   - Keep it inactive while building.

2. **Add the manual trigger**
   - Create a `Manual Trigger` node.
   - Leave it with default settings.
   - This will act as the entry point for test runs.

3. **Add a Liveblocks node to create a room**
   - Add a `Liveblocks` node using the custom `CUSTOM.liveblocks` integration available in your n8n instance.
   - Set:
     - **Operation**: `createRoom`
     - **Room ID**: `={{ $execution.id }}`
     - **Default accesses**: `={{ [] }}`
   - Configure Liveblocks credentials:
     - Create/select a `liveblocksApi` credential
     - Provide the API key/token required by your Liveblocks setup
   - Connect:
     - `Manual Trigger` → `Create a room`

4. **Add a Liveblocks node to initialize storage**
   - Add another `CUSTOM.liveblocks` node.
   - Name it `Patch room storage`.
   - Set:
     - **Resource**: `storage`
     - **Operation**: `patchStorageDocument`
     - **Room ID**: `={{ $json.id }}`
     - **Body**: a JSON Patch array that adds `/shapes`
   - Use this body:
     ```json
     [
       {
         "op": "add",
         "path": "/shapes",
         "value": [
           {
             "id": "rect-1",
             "type": "rectangle",
             "x": 100,
             "y": 150,
             "width": 200,
             "height": 100,
             "color": "#ff0000"
           },
           {
             "id": "circle-1",
             "type": "circle",
             "x": 300,
             "y": 200,
             "radius": 50,
             "color": "#00ff00"
           }
         ]
       }
     ]
     ```
   - Connect:
     - `Create a room` → `Patch room storage`

5. **Add a Liveblocks node to read the storage**
   - Add another `CUSTOM.liveblocks` node.
   - Name it `Get room storage`.
   - Set:
     - **Resource**: `storage`
     - **Operation**: `getStorageDocument`
     - **Room ID**: `={{ $execution.id }}`
     - **Format**: `json`
   - Connect:
     - `Patch room storage` → `Get room storage`

6. **Add the AI presence start node**
   - Add another `CUSTOM.liveblocks` node.
   - Name it `Set presence in a room`.
   - Set:
     - **Operation**: `setPresence`
     - **Room ID**: `={{ $execution.id }}`
     - **User ID**: `__AI_AGENT`
     - **User Info**:
       ```json
       {
         "name": "AI Assistant",
         "avatar": "https://liveblocks.io/api/avatar?u=__AI_AGENT&agent=true"
       }
       ```
     - **TTL**: leave null / no expiration
   - Connect:
     - `Get room storage` → `Set presence in a room`

7. **Add the AI Agent node**
   - Add an `AI Agent` node from the LangChain-enabled n8n nodes.
   - Set:
     - **Prompt Type**: `Define`
     - **Has Output Parser**: enabled
   - In the text/prompt field, insert:
     ```text
     =You are a JSON patch generatgor. All you return is JSPN patch operations. Here is the current document:

     ```
     {{ JSON.stringify($json.shapes, null, 2) }}
     ```

     Here is how you must modify the document:

     """
     Add a blue circle, and make the square orange. 
     """

     Apply this change to `/shapes`.
     ```
   - Connect:
     - `Get room storage` → `AI Agent`

8. **Add the Anthropic model node**
   - Add an `Anthropic Chat Model` node.
   - Set:
     - **Model**: `claude-sonnet-4-6`
   - Configure Anthropic credentials:
     - Create/select an `anthropicApi` credential
     - Provide a valid Anthropic API key
   - Connect its AI output to the AI Agent:
     - `Anthropic Chat Model` → `AI Agent` on the `ai_languageModel` connection

9. **Add the structured output parser**
   - Add a `Structured Output Parser` node.
   - Set:
     - **Schema Type**: `Manual`
   - Use this schema:
     ```json
     {
       "type": "array",
       "items": {
         "type": "object",
         "properties": {
           "op": {
             "type": "string",
             "enum": ["add", "remove", "replace"]
           },
           "path": {
             "type": "string"
           },
           "value": {}
         },
         "required": ["op", "path"]
       }
     }
     ```
   - Connect it to the AI Agent using the parser connection:
     - `Structured Output Parser` → `AI Agent` on the `ai_outputParser` connection

10. **Add the Liveblocks patch node for AI output**
    - Add another `CUSTOM.liveblocks` node.
    - Name it `Patch room storage1`.
    - Set:
      - **Resource**: `storage`
      - **Operation**: `patchStorageDocument`
      - **Room ID**: `={{ $execution.id }}`
      - **Body**: `={{ $json.output }}`
    - Connect:
      - `AI Agent` → `Patch room storage1`

11. **Add the AI presence expiration node**
    - Add another `CUSTOM.liveblocks` node.
    - Name it `Set presence in a room1`.
    - Set:
      - **Operation**: `setPresence`
      - **Room ID**: `={{ $execution.id }}`
      - **User ID**: `__AI_AGENT`
      - **User Info**:
        ```json
        {
          "name": "AI Assistant",
          "avatar": "https://liveblocks.io/api/avatar?u=__AI_AGENT&agent=true"
        }
        ```
      - **TTL**: `2`
    - Connect:
      - `Patch room storage1` → `Set presence in a room1`

12. **Add the final verification node**
    - Add another `CUSTOM.liveblocks` node.
    - Name it `Get room storage1`.
    - Set:
      - **Resource**: `storage`
      - **Operation**: `getStorageDocument`
      - **Room ID**: `={{ $execution.id }}`
      - **Format**: `json`
    - Connect:
      - `Patch room storage1` → `Get room storage1`

13. **Optional: add visual documentation notes**
    - Add sticky notes matching these sections if desired:
      - General workflow explanation and setup note
      - Create room
      - Patching storage with AI
      - Show presence
      - Hide presence
      - Check value

14. **Verify the branching structure**
    - From `Get room storage`, there should be two outgoing connections:
      - to `AI Agent`
      - to `Set presence in a room`
    - From `Patch room storage1`, there should be two outgoing connections:
      - to `Get room storage1`
      - to `Set presence in a room1`

15. **Test the workflow**
    - Execute manually.
    - Confirm:
      - A room is created
      - Initial shapes are added
      - Current storage is fetched
      - AI returns valid JSON Patch operations
      - The patch is applied successfully
      - Final storage shows the requested modifications

16. **Expected AI behavior**
    - The model should likely:
      - add a new blue circle to `/shapes`
      - replace the `color` value of the rectangle-like shape with orange
    - Because the request refers to a “square” while the data contains a rectangle, validate the model’s output carefully.

17. **Production adaptation notes**
    - Replace the manual trigger with a webhook, app event trigger, chat trigger, or another real input source.
    - Replace the hard-coded instruction with a user-provided prompt.
    - Consider storing the room ID externally instead of generating a fresh one for every execution.
    - Add validation or fallback logic before applying AI-generated patches.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Liveblocks Storage is presented as a sync engine for collaborative applications similar to Figma, Pitch, and Spline. | https://liveblocks.io/docs/ready-made-features/multiplayer/sync-engine/liveblocks-storage |
| The workflow uses Liveblocks rooms as the multiplayer document container. | https://liveblocks.io/docs/concepts#Rooms |
| The AI-generated modifications are based on JSON Patch operations applied through the Liveblocks API. | https://liveblocks.io/docs/guides/modifying-storage-via-rest-api-with-json-patch |
| The example assumes no setup beyond credentials and suggests replacing the manual trigger with any preferred trigger. | General workflow note |
| AI presence is shown in the room avatar stack and then configured to expire shortly after completion. | Based on the presence nodes in the workflow |