Complete Video Management System for AI Agents with api.video

https://n8nworkflows.xyz/workflows/complete-video-management-system-for-ai-agents-with-api-video-5520


# Complete Video Management System for AI Agents with api.video

### 1. Workflow Overview

This workflow is a comprehensive API integration layer designed to expose the full set of 47 operations of the api.video platform as an MCP (Modular Cognitive Process) server for AI agents. It enables AI clients to manage video content, live streams, players, captions, chapters, webhooks, and authentication by invoking dedicated API endpoints via HTTP requests.

**Target Use Cases:**  
- AI agents requiring complete control over video streaming and management via api.video APIs  
- Advanced users needing a detailed, MCP-compatible interface for video content lifecycle including upload, metadata, analytics, and live streaming  
- Developers integrating video management, analytics, and live streaming in AI-powered workflows or chatbot assistants

**Logical Blocks:**

- **1.1 Setup & Metadata:** Sticky notes providing warnings, setup instructions, and workflow overview.  
- **1.2 MCP Trigger:** The entry point for AI agent requests, exposing the MCP server endpoint.  
- **1.3 Account Operations:** Single operation to show account info.  
- **1.4 Analytics:** Retrieval of live stream and video session analytics data.  
- **1.5 Authentication:** API key authentication and token refresh.  
- **1.6 Live Streams:** CRUD and media operations on live streams including thumbnails.  
- **1.7 Players:** CRUD and media operations on video players including logos.  
- **1.8 Videos - Delegated Upload:** Managing upload tokens and performing chunked uploads.  
- **1.9 Videos:** CRUD, upload, status, and thumbnail management on videos.  
- **1.10 Captions:** Caption management for videos (list, delete, show, update, upload).  
- **1.11 Chapters:** Chapter management for videos (list, delete, show, upload).  
- **1.12 Webhooks:** Webhook management (list, create, delete, show).

Each block groups related HTTP request nodes, all connected to a central MCP trigger node that exposes the API surface to AI clients.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Setup & Metadata

- **Overview:** Provides important user guidance, workflow description, and operation breakdowns via sticky notes. These nodes contain no execution logic but serve as documentation within the workflow canvas.

- **Nodes Involved:**  
  - Advanced Warning (sticky note)  
  - Setup Instructions (sticky note)  
  - Workflow Overview (sticky note)  

- **Node Details:**  
  - Type: Sticky Note  
  - Configuration: Text content outlining workflow complexity, usage recommendations, setup instructions, and operation summaries.  
  - Input/Output Connections: None (informational only).  
  - Edge Cases: None.

---

#### 1.2 MCP Trigger

- **Overview:** Acts as the workflow’s single entry point, handling incoming AI agent requests on the configured webhook path. Routes requests to the appropriate API operation nodes.

- **Nodes Involved:**  
  - api.video MCP Server (mcpTrigger)  

- **Node Details:**  
  - Type: MCP Trigger (n8n-nodes-langchain.mcpTrigger)  
  - Configuration: Webhook path set to "api.video-mcp"  
  - Input: Receives HTTP requests from AI agents  
  - Output: Dispatches requests to HTTP Request Tool nodes (API calls) based on requested operation  
  - Edge Cases: Webhook availability, request validation, concurrency limits.

---

#### 1.3 Account Operations

- **Overview:** Provides a single API endpoint to retrieve account information.

- **Nodes Involved:**  
  - Sticky Note "Account"  
  - Show account (HTTP Request Tool)  

- **Node Details:**  
  - Show account:  
    - Type: HTTP Request Tool  
    - Role: Fetch account details via GET https://ws.api.video/account  
    - Authentication: Generic HTTP header authentication (requires API key configured in credentials)  
    - Parameters: No dynamic parameters  
    - Output: Returns raw API JSON response with account info  
    - Edge Cases: Auth failure, network issues.

---

#### 1.4 Analytics

- **Overview:** Retrieves analytics data for live streams, video player sessions, and session events.

- **Nodes Involved:**  
  - Sticky Note "Analytics"  
  - List live stream player sessions  
  - List player session events  
  - List video player sessions  

- **Node Details:**  
  - Each node uses HTTP Request Tool with GET method targeting specific analytics endpoints, e.g.:  
    - Live stream sessions: `/analytics/live-streams/{{ liveStreamId }}` with optional `period` query.  
    - Session events: `/analytics/sessions/{{ sessionId }}/events`  
    - Video player sessions: `/analytics/videos/{{ videoId }}` with `period` and `metadata` filters.  
  - Parameters: AI-populated expressions `$fromAI(...)` to inject query/path parameters dynamically.  
  - Authentication: Generic HTTP header auth  
  - Edge Cases: Invalid IDs, missing parameters, empty results, rate limits.

---

#### 1.5 Authentication

- **Overview:** Enables authentication via API key and supports token refresh operations.

- **Nodes Involved:**  
  - Sticky Note "Authentication"  
  - Authenticate  
  - Refresh token  

- **Node Details:**  
  - Authenticate: POST to `/auth/api-key`  
  - Refresh token: POST to `/auth/refresh`  
  - Both require generic HTTP header authentication credentials  
  - Outputs tokens or authentication status to AI agent  
  - Edge Cases: Invalid credentials, expired tokens, API errors.

---

#### 1.6 Live Streams

- **Overview:** Full CRUD operations on live streams plus thumbnail upload and deletion.

- **Nodes Involved:**  
  - Sticky Note "Live"  
  - List all live streams  
  - Create live stream  
  - Delete a live stream  
  - Show live stream  
  - Update a live stream  
  - Delete a thumbnail  
  - Upload a thumbnail  

- **Node Details:**  
  - Operations use GET, POST, DELETE, PATCH methods on `/live-streams` endpoints  
  - Dynamic path parameters (e.g. liveStreamId) and query parameters use AI expressions  
  - Thumbnail upload/delete via POST/DELETE on `/live-streams/{id}/thumbnail`  
  - Authentication: Generic header auth  
  - Edge Cases: Missing IDs, malformed requests, upload size limits, thumbnail format requirements.

---

#### 1.7 Players

- **Overview:** Manage video players including creation, deletion, update, and logo upload/removal.

- **Nodes Involved:**  
  - Sticky Note "Players"  
  - List all players  
  - Create a player  
  - Delete a player  
  - Show a player  
  - Update a player  
  - Delete logo  
  - Upload a logo  

- **Node Details:**  
  - Standard REST verbs on `/players` endpoints  
  - Dynamic parameters passed from AI via `$fromAI()`  
  - Logo upload/delete on `/players/{playerId}/logo`  
  - Authentication: Generic HTTP header  
  - Edge Cases: Invalid player IDs, file upload limits, auth errors.

---

#### 1.8 Videos - Delegated Upload

- **Overview:** Manage upload tokens for delegated video upload and perform video uploads using these tokens.

- **Nodes Involved:**  
  - Sticky Note "Videos - Delegated Upload"  
  - Upload with an upload token  
  - List all active upload tokens  
  - Generate an upload token  
  - Delete an upload token  
  - Show upload token  

- **Node Details:**  
  - Upload endpoint supports query param `token` and `Content-Range` header for chunked uploads  
  - Upload tokens can be listed, generated, deleted, or inspected via respective endpoints  
  - Authentication: Generic HTTP header  
  - Edge Cases: Token expiration, invalid tokens, partial uploads, network errors.

---

#### 1.9 Videos

- **Overview:** Full video lifecycle management including CRUD, upload, status checking, and thumbnail selection/upload.

- **Nodes Involved:**  
  - Sticky Note "Videos"  
  - List all videos  
  - Create a video  
  - Delete a video  
  - Show a video  
  - Update a video  
  - Upload a video  
  - Show video status  
  - Pick a thumbnail  
  - Upload a thumbnail 1  

- **Node Details:**  
  - Uses standard REST verbs on `/videos` endpoints with dynamic parameters  
  - Uploads support `Content-Range` header for chunked transfers  
  - Thumbnail management via PATCH and POST to `/videos/{videoId}/thumbnail`  
  - Authentication: Generic HTTP header  
  - Edge Cases: Large file uploads, invalid IDs, concurrency, thumbnail format.

---

#### 1.10 Captions

- **Overview:** Manage video captions with list, delete, show, update, and upload operations.

- **Nodes Involved:**  
  - Sticky Note "Captions"  
  - List video captions  
  - Delete a caption  
  - Show a caption  
  - Update caption  
  - Upload a caption  

- **Node Details:**  
  - All operations target `/videos/{videoId}/captions/{language}` endpoints  
  - Language parameter requires valid BCP 47 language code  
  - Methods: GET, DELETE, PATCH, POST accordingly  
  - Authentication: Generic HTTP header  
  - Edge Cases: Language code validation, file format issues, missing captions.

---

#### 1.11 Chapters

- **Overview:** Manage video chapters similarly to captions.

- **Nodes Involved:**  
  - Sticky Note "Chapters"  
  - List video chapters  
  - Delete a chapter  
  - Show a chapter  
  - Upload a chapter  

- **Node Details:**  
  - Operate on `/videos/{videoId}/chapters/{language}` with appropriate methods  
  - Language parameter requires valid BCP 47 code  
  - Authentication: Generic HTTP header  
  - Edge Cases: Invalid chapters, language mismatches.

---

#### 1.12 Webhooks

- **Overview:** Manage webhooks for event notifications.

- **Nodes Involved:**  
  - Sticky Note "Web Hooks"  
  - List all webhooks  
  - Create Webhook  
  - Delete a Webhook  
  - Show Webhook details  

- **Node Details:**  
  - Operate on `/webhooks` endpoints with GET, POST, DELETE  
  - Optional filters for events in list operation  
  - Authentication: Generic HTTP header  
  - Edge Cases: Invalid webhook IDs, event filtering errors.

---

### 3. Summary Table

| Node Name                    | Node Type                 | Functional Role                             | Input Node(s)    | Output Node(s)          | Sticky Note                                                                                                   |
|------------------------------|---------------------------|--------------------------------------------|------------------|-------------------------|--------------------------------------------------------------------------------------------------------------|
| Advanced Warning              | Sticky Note               | User guidance and warnings                  | None             | None                    | # ⚠️ ADVANCED USE ONLY... (Full advanced usage instructions and recommendations)                              |
| Setup Instructions           | Sticky Note               | Setup guidance for users                     | None             | None                    | ### ⚙️ Setup Instructions... (Stepwise setup and usage notes)                                                |
| Workflow Overview            | Sticky Note               | Overview of workflow and operations          | None             | None                    | ## 🛠️ api.video MCP Server... (Operation summary and usage overview)                                         |
| api.video MCP Server         | MCP Trigger               | Entry point for AI agent requests            | None             | All HTTP Request Tools   |                                                                                                              |
| Sticky Note                  | Sticky Note               | Section header: Account                       | None             | None                    | ## Account                                                                                                   |
| Show account                 | HTTP Request Tool         | Show account info                            | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note2                 | Sticky Note               | Section header: Analytics                     | None             | None                    | ## Analytics                                                                                                 |
| List live stream player sessions | HTTP Request Tool     | List analytics of live stream sessions       | MCP Trigger      | MCP Trigger             |                                                                                                              |
| List player session events   | HTTP Request Tool         | List events for a player session              | MCP Trigger      | MCP Trigger             |                                                                                                              |
| List video player sessions   | HTTP Request Tool         | List video player sessions                     | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note3                 | Sticky Note               | Section header: Authentication                | None             | None                    | ## Authentication                                                                                            |
| Authenticate                | HTTP Request Tool         | API key authentication                        | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Refresh token               | HTTP Request Tool         | Refresh authentication token                  | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note4                | Sticky Note               | Section header: Live                           | None             | None                    | ## Live                                                                                                      |
| List all live streams       | HTTP Request Tool         | List live streams                             | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Create live stream          | HTTP Request Tool         | Create a live stream                           | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a live stream        | HTTP Request Tool         | Delete a live stream                           | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show live stream            | HTTP Request Tool         | Show live stream details                       | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Update a live stream        | HTTP Request Tool         | Update live stream info                        | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a thumbnail          | HTTP Request Tool         | Delete live stream thumbnail                   | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Upload a thumbnail          | HTTP Request Tool         | Upload live stream thumbnail                   | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note5                | Sticky Note               | Section header: Players                        | None             | None                    | ## Players                                                                                                   |
| List all players            | HTTP Request Tool         | List all players                              | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Create a player             | HTTP Request Tool         | Create a player                               | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a player             | HTTP Request Tool         | Delete a player                               | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show a player               | HTTP Request Tool         | Show player details                           | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Update a player             | HTTP Request Tool         | Update player info                            | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete logo                 | HTTP Request Tool         | Delete player logo                            | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Upload a logo               | HTTP Request Tool         | Upload player logo                            | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note6                | Sticky Note               | Section header: Videos - Delegated Upload     | None             | None                    | ## Videos - Delegated Upload                                                                                  |
| Upload with an upload token | HTTP Request Tool         | Upload video using upload token               | MCP Trigger      | MCP Trigger             |                                                                                                              |
| List all active upload tokens | HTTP Request Tool       | List active upload tokens                      | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Generate an upload token    | HTTP Request Tool         | Generate a new upload token                    | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete an upload token      | HTTP Request Tool         | Delete an upload token                         | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show upload token           | HTTP Request Tool         | Show details of an upload token                | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note7                | Sticky Note               | Section header: Videos                         | None             | None                    | ## Videos                                                                                                    |
| List all videos             | HTTP Request Tool         | List videos                                   | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Create a video              | HTTP Request Tool         | Create a new video                            | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a video              | HTTP Request Tool         | Delete a video                                | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show a video                | HTTP Request Tool         | Show video details                            | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Update a video              | HTTP Request Tool         | Update video info                             | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Upload a video              | HTTP Request Tool         | Upload video file                             | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show video status           | HTTP Request Tool         | Check video processing status                  | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Pick a thumbnail            | HTTP Request Tool         | Select video thumbnail from video frames       | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Upload a thumbnail 1        | HTTP Request Tool         | Upload video thumbnail                         | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note8                | Sticky Note               | Section header: Captions                       | None             | None                    | ## Captions                                                                                                  |
| List video captions         | HTTP Request Tool         | List captions for a video                      | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a caption            | HTTP Request Tool         | Delete a video caption                         | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show a caption              | HTTP Request Tool         | Show a specific caption                        | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Update caption              | HTTP Request Tool         | Update a video caption                         | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Upload a caption            | HTTP Request Tool         | Upload a new caption                           | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note9                | Sticky Note               | Section header: Chapters                       | None             | None                    | ## Chapters                                                                                                  |
| List video chapters         | HTTP Request Tool         | List chapters for a video                       | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a chapter            | HTTP Request Tool         | Delete a video chapter                         | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show a chapter              | HTTP Request Tool         | Show a specific chapter                        | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Upload a chapter            | HTTP Request Tool         | Upload a new chapter                           | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Sticky Note10               | Sticky Note               | Section header: Web Hooks                       | None             | None                    | ## Web Hooks                                                                                                 |
| List all webhooks           | HTTP Request Tool         | List all webhooks                             | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Create Webhook              | HTTP Request Tool         | Create a new webhook                          | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Delete a Webhook            | HTTP Request Tool         | Delete a webhook                              | MCP Trigger      | MCP Trigger             |                                                                                                              |
| Show Webhook details        | HTTP Request Tool         | Show webhook details                          | MCP Trigger      | MCP Trigger             |                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation:**  
   - Add sticky notes for Advanced Warning, Setup Instructions, Workflow Overview, and each section header (Account, Analytics, Authentication, Live, Players, Videos - Delegated Upload, Videos, Captions, Chapters, Web Hooks).  
   - Populate each with the corresponding descriptive text as per the original workflow.

2. **Add MCP Trigger Node:**  
   - Create an MCP Trigger node named `api.video MCP Server`.  
   - Set webhook path to `api.video-mcp`.  
   - This node will serve as the entry point for AI requests.

3. **Configure Generic HTTP Header Authentication Credential:**  
   - Create a generic credential in n8n for the api.video API key.  
   - Use this credential for all HTTP Request Tool nodes to authenticate via HTTP headers.

4. **Add Account Operation Node:**  
   - Create HTTP Request Tool node named `Show account`.  
   - Set method to GET, URL to `https://ws.api.video/account`.  
   - Use generic HTTP header auth credential.

5. **Add Analytics Nodes:**  
   - `List live stream player sessions`: GET `https://ws.api.video/analytics/live-streams/{{liveStreamId}}`, query param `period`.  
   - `List player session events`: GET `https://ws.api.video/analytics/sessions/{{sessionId}}/events`.  
   - `List video player sessions`: GET `https://ws.api.video/analytics/videos/{{videoId}}`, query params `period`, `metadata`.  
   - Use expressions to inject parameters dynamically from AI input.

6. **Add Authentication Nodes:**  
   - `Authenticate`: POST `https://ws.api.video/auth/api-key`.  
   - `Refresh token`: POST `https://ws.api.video/auth/refresh`.  
   - Use generic HTTP header auth.

7. **Add Live Stream Management Nodes:**  
   - `List all live streams`: GET `https://ws.api.video/live-streams` with query params `streamKey`, `name`, `sortBy`, `sortOrder`.  
   - `Create live stream`: POST same URL.  
   - `Delete a live stream`: DELETE `https://ws.api.video/live-streams/{{liveStreamId}}`.  
   - `Show live stream`: GET same with ID.  
   - `Update a live stream`: PATCH with ID.  
   - `Delete a thumbnail`: DELETE `.../thumbnail` endpoint.  
   - `Upload a thumbnail`: POST `.../thumbnail` endpoint.

8. **Add Player Management Nodes:**  
   - List, create, delete, show, update players using `/players` endpoints.  
   - Delete and upload logo on `/players/{{playerId}}/logo`.

9. **Add Videos Delegated Upload Nodes:**  
   - Upload using token: POST to `/upload` with query param `token` and header `Content-Range`.  
   - List, generate, delete, show upload tokens via `/upload-tokens` endpoints.

10. **Add Videos Management Nodes:**  
    - List, create, delete, show, update videos via `/videos` endpoints with query parameters as applicable.  
    - Upload video source with `Content-Range`.  
    - Show video status.  
    - Pick and upload thumbnails.

11. **Add Captions Nodes:**  
    - List, delete, show, update, and upload captions for videos via `/videos/{{videoId}}/captions/{{language}}`.

12. **Add Chapters Nodes:**  
    - Similar CRUD operations on `/videos/{{videoId}}/chapters/{{language}}`.

13. **Add Webhooks Nodes:**  
    - List, create, delete, show webhooks via `/webhooks`.

14. **Connect all HTTP Request Tool Nodes to the MCP Trigger:**  
    - Each HTTP Request Tool should have its `ai_tool` input connected to the MCP Trigger node's output.

15. **Configure AI Expressions for Inputs:**  
    - Use `$fromAI(parameterName, description, type)` expressions in path parameters, query parameters, and headers to enable AI to supply dynamic input.

16. **Validate and Test:**  
    - Activate the workflow.  
    - Test each operation by invoking the MCP webhook and supplying appropriate parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow contains 47 operations, exceeding the recommended 40 tools for most AI clients. Users should disable or delete unused nodes accordingly. | Advanced usage warning in sticky note "Advanced Warning"                                                    |
| For integration guidance and custom automations, users can ping the author on Discord: https://discord.me/cfomodz                                  | Setup Instructions sticky note                                                                              |
| Official n8n documentation for MCP servers and Langchain MCP triggers: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions sticky note                                                                              |
| api.video official blog about Dynamic Metadata filtering: https://api.video/blog/endpoints/dynamic-metadata                                        | Analytics and Videos nodes that use metadata filtering                                                      |
| Language codes for captions and chapters must comply with BCP 47: https://github.com/libyal/libfwnt/wiki/Language-Code-identifiers                  | Captions and Chapters nodes                                                                                  |

---

This detailed reference fully documents the structure, purpose, and configurations of the api.video MCP server workflow in n8n, enabling advanced users and AI agents to reproduce, understand, and extend the integration reliably.