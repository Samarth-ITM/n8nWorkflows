BBC Radio & Music API Integration Hub for AI Assistants

https://n8nworkflows.xyz/workflows/bbc-radio---music-api-integration-hub-for-ai-assistants-5537


# BBC Radio & Music API Integration Hub for AI Assistants

---
### 1. Workflow Overview

This workflow, titled **"BBC Radio & Music API Integration Hub for AI Assistants"**, functions as a comprehensive MCP (Multi-Client Platform) server for integrating the BBC Radio & Music Services API with AI agents. It exposes 75 distinct API endpoints covering broadcasts, categories, collections, music metadata, personalized user data, radio programmes, podcasts, and networks.

**Target Use Cases:**
- Enable AI assistants and clients to interact programmatically with BBC’s Radio and Music services.
- Provide access to rich metadata and personalized user-centric endpoints.
- Serve as a central integration hub for complex Radio & Music API operations.
- Support advanced users needing granular control over numerous operations.

**Logical Blocks:**

- **1.1 Setup & Overview**
  - Workflow metadata, instructions, warnings, and MCP trigger configuration.
  
- **1.2 Broadcasts**
  - Endpoints concerning radio broadcasts including latest broadcasts and broadcast lookup by PID.
  
- **1.3 Categories**
  - Operations to list categories and retrieve category information by ID.
  
- **1.4 Collections**
  - Retrieve members of collections (playlists or grouped content).
  
- **1.5 Experience**
  - Homepage experience metadata retrieval.
  
- **1.6 Music**
  - A large group of nodes covering popular artists, playlists, tracks, favorites, follows, and detailed popularity metrics.
  
- **1.7 Personalised Categories**
  - User-specific category follow/unfollow operations.
  
- **1.8 Music Export**
  - Endpoints managing music export jobs, preferences, and vendor-specific settings.
  
- **1.9 Personalised Networks**
  - User follow/unfollow operations for networks.
  
- **1.10 Personalised Plays**
  - Write play events (user listening activity).
  
- **1.11 Play Space**
  - Suggested and specific playspace container retrieval.
  
- **1.12 Programmes**
  - Recommended and popular radio programmes and episodes.
  
- **1.13 Radio**
  - User personalized radio favorites, follows, plays, with CRUD operations for episodes, clips, brands, and series.
  
- **1.14 Podcasts**
  - Podcast listings, featured podcasts, podcast details, and episodes.
  
- **1.15 Networks**
  - Retrieval of network metadata.

Each block is implemented as one or more HTTP Request nodes configured to call specific BBC API endpoints, receiving AI-supplied parameters via `$fromAI()` expressions, and returning results directly to the AI agent through the MCP trigger node.

---

### 2. Block-by-Block Analysis

#### 1.1 Setup & Overview

- **Overview:**  
  Provides workflow metadata, usage warnings, setup instructions, and the main MCP trigger node which serves as the server endpoint for AI agent calls.

- **Nodes Involved:**  
  - Advanced Warning (Sticky Note)  
  - Setup Instructions (Sticky Note)  
  - Workflow Overview (Sticky Note)  
  - Radio & Music Services MCP Server (MCP Trigger Node)

- **Node Details:**

  - **Advanced Warning (Sticky Note)**  
    - Serves as a caution to advanced users about the high number of operations (75) and advises on workflow management and performance considerations.  
    - Positioned prominently as a visual warning.
  
  - **Setup Instructions (Sticky Note)**  
    - Provides detailed stepwise instructions for importing, authenticating, activating, and using the MCP server, including notes on parameter auto-population and customization tips.  
    - Includes helpful links to Discord and n8n documentation.
  
  - **Workflow Overview (Sticky Note)**  
    - Summarizes the purpose of the workflow, how it works, and lists all 75 API operations grouped by category.  
    - Helps users understand the scope and organization of the workflow.
  
  - **Radio & Music Services MCP Server (MCP Trigger)**  
    - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
    - Configuration: Webhook path set to `radio-&-music-services-mcp`  
    - Role: Receives AI agent requests, routes to appropriate HTTP Request nodes based on tool calls.  
    - Version: 1  
    - Inputs: None (trigger node)  
    - Outputs: Connected implicitly to all HTTP Request nodes via MCP tool interface.  
    - Failure Modes: Webhook availability, incorrect routing, or malformed requests could cause failures. No auth required on trigger.

---

#### 1.2 Broadcasts

- **Overview:**  
  Handles retrieval of broadcast information including general broadcasts, latest broadcasts, and broadcast lookup by PID.

- **Nodes Involved:**  
  - Sticky Note ("Broadcasts")  
  - Broadcasts (HTTP Request Tool)  
  - Latest Broadcasts (HTTP Request Tool)  
  - Broadcasts by PID (HTTP Request Tool)

- **Node Details:**

  - **Broadcasts**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//broadcasts`  
    - Method: GET  
    - Parameters: offset, limit, service_id, date, sort — all populated via `$fromAI()`  
    - Headers: `X-API-Key` from `$fromAI()`  
    - Role: Fetch paginated broadcasts filtered by parameters.  
    - Failure Modes: API key missing or invalid, parameter errors, network timeouts.
  
  - **Latest Broadcasts**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//broadcasts/latest`  
    - Method: GET  
    - Parameters: offset, limit, service_id, on_air, next, previous, sort via `$fromAI()`  
    - Headers: `X-API-Key`  
    - Role: Fetch latest broadcasts with filters on timing and sorting.  
    - Failure Modes: Same as above.
  
  - **Broadcasts by PID**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//broadcasts/{{pid}}`  
    - Method: GET  
    - Path param: `pid` via `$fromAI()`  
    - Headers: `X-API-Key`  
    - Role: Retrieve broadcast details by unique PID.  
    - Failure Modes: Missing or invalid PID, 404 not found, auth errors.

---

#### 1.3 Categories

- **Overview:**  
  Provides endpoints to list categories and retrieve specific category details by ID.

- **Nodes Involved:**  
  - Sticky Note ("Categories")  
  - List of categories (HTTP Request Tool)  
  - Category by ID (HTTP Request Tool)

- **Node Details:**

  - **List of categories**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//categories`  
    - Method: GET  
    - Query param: `kind` (optional) via `$fromAI()`  
    - Headers: `X-API-Key`  
    - Role: List categories optionally filtered by type.  
    - Failure Modes: Parameter errors, auth failures.
  
  - **Category by ID**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//categories/{{id}}`  
    - Method: GET  
    - Path param: `id` via `$fromAI()`  
    - Headers: `X-API-Key`  
    - Role: Fetch category metadata by ID.  
    - Failure Modes: Invalid ID, 404 errors.

---

#### 1.4 Collections

- **Overview:**  
  Retrieves members of a collection identified by a PID with pagination support.

- **Nodes Involved:**  
  - Sticky Note ("Collections")  
  - Collection Members (HTTP Request Tool)

- **Node Details:**

  - **Collection Members**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//collections/{{pid}}/members`  
    - Method: GET  
    - Path param: `pid` via `$fromAI()`  
    - Query params: offset, limit via `$fromAI()`  
    - Headers: `X-API-Key`  
    - Role: List members (items) of a collection.  
    - Failure Modes: Invalid PID, pagination errors.

---

#### 1.5 Experience

- **Overview:**  
  Retrieves homepage experience metadata.

- **Nodes Involved:**  
  - Sticky Note ("Experience")  
  - Homepage Experience (HTTP Request Tool)

- **Node Details:**

  - **Homepage Experience**  
    - Type: HTTP Request Tool  
    - Endpoint: `https://rms.api.bbc.co.uk//experience/homepage`  
    - Method: GET  
    - Headers: `X-API-Key`  
    - Role: Fetch homepage experience data.  
    - Failure Modes: Auth errors.

---

#### 1.6 Music

- **Overview:**  
  Contains numerous endpoints to retrieve popular artists, playlists, tracks, favorite music, and detailed popularity scores.

- **Nodes Involved:**  
  - Sticky Note ("Music")  
  - Popular Artists  
  - Single Artist Popularity  
  - Popular Playlists  
  - Single Playlist Popularity  
  - Popular Tracks  
  - Single Track Popularity  
  - Favourite Tracks or Clips (GET, POST, PUT)  
  - Favourite Tracks or Clips by Type  
  - Favourite Track or Clip (DELETE, GET, POST, PUT)  
  - Followed Networks, Categories, Artists, Playlists and Genres (GET, POST, PUT)  
  - Followed Networks, Categories, Artists, Playlists and Genres by Type  
  - Followed Network, Category, Artist, Playlist and Genre (DELETE, GET, POST, PUT)

- **Node Details:**  
  Each node is an HTTP Request Tool calling a specific API endpoint on `https://rms.api.bbc.co.uk` with parameters provided dynamically via `$fromAI()`. These nodes require authentication headers including OAuth bearer tokens and API keys. The endpoints support CRUD operations for favorites and follows, and allow detailed querying with pagination, date ranges, and filters.

- **Failure Modes:**  
  - Authentication errors (missing or expired OAuth tokens).  
  - Rate limiting or quota limits.  
  - Invalid parameters (e.g., missing IDs).  
  - Network or API availability issues.

---

#### 1.7 Personalised Categories

- **Overview:**  
  User-specific category follow/unfollow management.

- **Nodes Involved:**  
  - Sticky Note ("Personalised Categories")  
  - Unfollow category (DELETE)  
  - List of followed categories (GET)  
  - Follow category (POST)

- **Node Details:**  
  Similar pattern to Music block with authorization required, query parameters for pagination, and standard HTTP verbs.

---

#### 1.8 Music Export

- **Overview:**  
  Manages music export jobs, preferences, and vendor-specific preferences with full CRUD operations.

- **Nodes Involved:**  
  - Sticky Note ("Music Export")  
  - Music Exports (GET)  
  - Music Export Jobs (GET, POST)  
  - Music Export Tracks (GET)  
  - Music Export Preferences (DELETE, GET, POST)  
  - Music Export Vendor Preferences (DELETE, GET, POST, PUT)

- **Node Details:**  
  Nodes query or modify export jobs and preferences with parameters for pagination and vendor specification. OAuth authorization is consistently required.

---

#### 1.9 Personalised Networks

- **Overview:**  
  User follow/unfollow operations for networks.

- **Nodes Involved:**  
  - Sticky Note ("Personalised Networks")  
  - Unfollow network (DELETE)  
  - List of followed networks (GET)  
  - Follow network (POST)

---

#### 1.10 Personalised Plays

- **Overview:**  
  Allows recording of play events (user listening activity).

- **Nodes Involved:**  
  - Sticky Note ("Personalised Plays")  
  - Write Play Event (POST)

---

#### 1.11 Play Space

- **Overview:**  
  Retrieve suggested playspace containers or playspace container by ID.

- **Nodes Involved:**  
  - Sticky Note ("Play Space")  
  - Suggested Playspace Container (GET)  
  - Playspace Container by ID (GET)  
  - Description - Playspace (Sticky Note)

---

#### 1.12 Programmes

- **Overview:**  
  Access recommended programmes, popular episodes & clips, and radio programme metadata.

- **Nodes Involved:**  
  - Sticky Note ("Programmes")  
  - Recommended Programmes (GET)  
  - Popular Episodes & Clips (GET)  
  - Radio programmes (GET)  
  - Available radio programme by PID (GET)

---

#### 1.13 Radio

- **Overview:**  
  User personalized radio favorites, follows, plays, with endpoints to manage episodes, clips, brands, and series.

- **Nodes Involved:**  
  - Sticky Note ("Radio")  
  - Favourite Episodes and Clips (GET, POST, PUT)  
  - Favourite Episodes and Clips by Type (GET)  
  - Favourite Episode or Clip (DELETE, GET, POST, PUT)  
  - Followed Brands and Series (GET, POST, PUT)  
  - Followed Brands or Series by Type (GET)  
  - Followed Brand or Series (DELETE, GET, POST, PUT)  
  - Played Episode or Clip (GET)

- **Authentication:**  
  Requires OAuth bearer token, API key, and authentication provider headers.

---

#### 1.14 Podcasts

- **Overview:**  
  Fetch lists of podcasts, featured podcasts, podcast details, and episodes.

- **Nodes Involved:**  
  - Sticky Note ("Podcasts")  
  - All Podcasts (GET)  
  - Featured Podcasts (GET)  
  - Podcast (GET)  
  - Podcast Episodes (GET)

---

#### 1.15 Networks

- **Overview:**  
  Retrieve networks metadata, including options for iPlayer Radio responsive navigation and international availability.

- **Nodes Involved:**  
  - Sticky Note ("Networks")  
  - Networks (GET)

---

### 3. Summary Table

| Node Name                                       | Node Type                         | Functional Role                                       | Input Node(s)                  | Output Node(s) | Sticky Note                                                                                 |
|------------------------------------------------|----------------------------------|------------------------------------------------------|-------------------------------|----------------|---------------------------------------------------------------------------------------------|
| Advanced Warning                                | Sticky Note                      | Workflow usage warning and advanced user instructions | None                          | None           | ⚠️ This workflow is for advanced users only! Contains 75 operations, recommended ≤40 tools. |
| Setup Instructions                             | Sticky Note                      | Setup and usage instructions                          | None                          | None           | Setup instructions with links to Discord and n8n docs.                                    |
| Workflow Overview                              | Sticky Note                      | Workflow summary and API operation list               | None                          | None           | Describes all 75 operations by category.                                                   |
| Radio & Music Services MCP Server              | MCP Trigger                     | Main entry point for AI agent requests                | None                          | All HTTP Request nodes |                                                                                             |
| Sticky Note (Broadcasts)                       | Sticky Note                      | Section label for Broadcasts                          | None                          | None           |                                                                                             |
| Broadcasts                                     | HTTP Request Tool                | Fetch broadcasts with filters                         | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Latest Broadcasts                              | HTTP Request Tool                | Fetch latest broadcasts with filters                  | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Broadcasts by PID                              | HTTP Request Tool                | Fetch broadcast by PID                               | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Sticky Note (Categories)                       | Sticky Note                      | Section label for Categories                          | None                          | None           |                                                                                             |
| List of categories                             | HTTP Request Tool                | List categories with optional filter                 | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Category by ID                                 | HTTP Request Tool                | Fetch category details by ID                          | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Sticky Note (Collections)                      | Sticky Note                      | Section label for Collections                         | None                          | None           |                                                                                             |
| Collection Members                             | HTTP Request Tool                | List collection members                              | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Sticky Note (Experience)                       | Sticky Note                      | Section label for Experience                          | None                          | None           |                                                                                             |
| Homepage Experience                            | HTTP Request Tool                | Fetch homepage experience metadata                    | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Sticky Note (Music)                            | Sticky Note                      | Section label for Music operations                    | None                          | None           |                                                                                             |
| Popular Artists                               | HTTP Request Tool                | Fetch popular artists                                | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Single Artist Popularity                       | HTTP Request Tool                | Fetch popularity for a single artist                  | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Popular Playlists                              | HTTP Request Tool                | Fetch popular playlists                              | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Single Playlist Popularity                     | HTTP Request Tool                | Fetch popularity for a single playlist                | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Popular Tracks                                 | HTTP Request Tool                | Fetch popular tracks                                 | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Single Track Popularity                        | HTTP Request Tool                | Fetch popularity for a single track                   | MCP Trigger                   | None           | Requires API key header.                                                                    |
| Favourite Tracks or Clips                      | HTTP Request Tool                | Get user's favorite tracks or clips                   | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Tracks or Clips 1                    | HTTP Request Tool                | Add to user's favorites                               | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Tracks or Clips 2                    | HTTP Request Tool                | Update user's favorites                               | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Tracks or Clips by Type              | HTTP Request Tool                | Get favorites filtered by type                         | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Track or Clip                        | HTTP Request Tool                | Delete a favorite track or clip                        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Track or Clip 1                      | HTTP Request Tool                | Get a favorite track or clip details                   | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Track or Clip 2                      | HTTP Request Tool                | Add a favorite track or clip                           | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Track or Clip 3                      | HTTP Request Tool                | Update a favorite track or clip                        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Networks, Categories, Artists, Playlists and Genres | HTTP Request Tool | Get user's followed entities                           | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Networks, Categories, Artists, Playlists and Genres 1 | HTTP Request Tool | Add to user's followed entities                        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Networks, Categories, Artists, Playlists and Genres 2 | HTTP Request Tool | Update user's followed entities                        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Networks, Categories, Artists, Playlists and Genres 3 | HTTP Request Tool | Get followed entities by type                           | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Network, Category, Artist, Playlist and Genre | HTTP Request Tool | Remove a followed entity by type and id                | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Network, Category, Artist, Playlist and Genre 1 | HTTP Request Tool | Get followed entity by type and id                      | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Network, Category, Artist, Playlist and Genre 2 | HTTP Request Tool | Add a followed entity by type and id                    | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Network, Category, Artist, Playlist and Genre 3 | HTTP Request Tool | Update a followed entity by type and id                 | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Sticky Note (Personalised Categories)         | Sticky Note                      | Section label for Personalised Categories             | None                          | None           |                                                                                             |
| Unfollow category                              | HTTP Request Tool                | Unfollow a category                                   | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| List of followed categories                    | HTTP Request Tool                | List user's followed categories                       | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Follow category                                | HTTP Request Tool                | Follow a category                                    | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Sticky Note (Music Export)                      | Sticky Note                      | Section label for Music Export                         | None                          | None           |                                                                                             |
| Music Exports                                  | HTTP Request Tool                | List music export jobs                                | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Music Export Jobs                              | HTTP Request Tool                | List or create music export jobs                       | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Music Export Tracks                            | HTTP Request Tool                | List tracks for exports                               | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Music Export Preferences                       | HTTP Request Tool                | Manage user music export preferences                   | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Music Export Vendor Preferences                | HTTP Request Tool                | Manage vendor-specific music export preferences        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Sticky Note (Personalised Networks)            | Sticky Note                      | Section label for Personalised Networks               | None                          | None           |                                                                                             |
| Unfollow network                               | HTTP Request Tool                | Unfollow a network                                   | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| List of followed networks                      | HTTP Request Tool                | List user's followed networks                         | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Follow network                                 | HTTP Request Tool                | Follow a network                                    | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Sticky Note (Personalised Plays)                | Sticky Note                      | Section label for Personalised Plays                   | None                          | None           |                                                                                             |
| Write Play Event                               | HTTP Request Tool                | Record a play event                                  | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Sticky Note (Play Space)                        | Sticky Note                      | Section label for Play Space                           | None                          | None           |                                                                                             |
| Suggested Playspace Container                  | HTTP Request Tool                | Get suggested playspace container                     | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Playspace Container by ID                       | HTTP Request Tool                | Get playspace container by ID                         | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Description - Playspace                        | Sticky Note                      | Label and note for Playspace documentation            | None                          | None           |                                                                                             |
| Sticky Note (Programmes)                        | Sticky Note                      | Section label for Programmes                           | None                          | None           |                                                                                             |
| Recommended Programmes                         | HTTP Request Tool                | Get recommended programmes                           | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Popular Episodes & Clips                       | HTTP Request Tool                | Get popular episodes and clips                        | MCP Trigger                   | None           | Requires API key.                                                                          |
| Radio programmes                              | HTTP Request Tool                | Get radio programmes                                 | MCP Trigger                   | None           | Requires API key.                                                                          |
| Available radio programme by Pid              | HTTP Request Tool                | Get radio programme details by PID                   | MCP Trigger                   | None           | Requires API key.                                                                          |
| Sticky Note (Radio)                            | Sticky Note                      | Section label for Radio                                | None                          | None           |                                                                                             |
| Favourite Episodes and Clips                   | HTTP Request Tool                | Manage user's favorite radio episodes and clips       | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episodes and Clips 1                 | HTTP Request Tool                | Add to favorite radio episodes and clips              | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episodes and Clips 2                 | HTTP Request Tool                | Update favorite radio episodes and clips              | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episodes and Clips by Type           | HTTP Request Tool                | List favorite episodes or clips filtered by type      | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episode or Clip                      | HTTP Request Tool                | Delete a favorite episode or clip                      | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episode or Clip 1                    | HTTP Request Tool                | Get details of a favorite episode or clip             | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episode or Clip 2                    | HTTP Request Tool                | Add a favorite episode or clip                         | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Favourite Episode or Clip 3                    | HTTP Request Tool                | Update a favorite episode or clip                      | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brands and Series                      | HTTP Request Tool                | Get followed brands and series                         | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brands and Series 1                    | HTTP Request Tool                | Add followed brands and series                         | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brands and Series 2                    | HTTP Request Tool                | Update followed brands and series                      | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brands or Series by Type               | HTTP Request Tool                | List followed brands or series filtered by type       | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brand or Series                        | HTTP Request Tool                | Delete followed brand or series                        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brand or Series 1                      | HTTP Request Tool                | Get followed brand or series details                   | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brand or Series 2                      | HTTP Request Tool                | Add followed brand or series                           | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Followed Brand or Series 3                      | HTTP Request Tool                | Update followed brand or series                        | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Played Episode or Clip                          | HTTP Request Tool                | List played radio episodes or clips                    | MCP Trigger                   | None           | Requires OAuth token and API key.                                                          |
| Sticky Note (Podcasts)                         | Sticky Note                      | Section label for Podcasts                             | None                          | None           |                                                                                             |
| All Podcasts                                  | HTTP Request Tool                | List all podcasts                                    | MCP Trigger                   | None           | Requires API key.                                                                          |
| Featured Podcasts                             | HTTP Request Tool                | List featured podcasts                               | MCP Trigger                   | None           | Requires API key.                                                                          |
| Podcast                                       | HTTP Request Tool                | Get podcast details by PID                            | MCP Trigger                   | None           | Requires API key.                                                                          |
| Podcast Episodes                              | HTTP Request Tool                | List episodes of a podcast                            | MCP Trigger                   | None           | Requires API key.                                                                          |
| Sticky Note (Networks)                         | Sticky Note                      | Section label for Networks                            | None                          | None           |                                                                                             |
| Networks                                      | HTTP Request Tool                | List networks                                        | MCP Trigger                   | None           | Requires API key.                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Notes for Documentation & Labels**  
   - Add Sticky Notes for:  
     - Advanced Warning (color: 3, large size) with workflow caution content.  
     - Setup Instructions (color: 4) including import and usage steps with helpful links.  
     - Workflow Overview (color: default) summarizing operations.  
     - Section labels for each logical block (Broadcasts, Categories, Collections, etc.) with appropriate colors.

2. **Add MCP Trigger Node**  
   - Node Type: `@n8n/n8n-nodes-langchain.mcpTrigger`  
   - Configure webhook path to "radio-&-music-services-mcp"  
   - No authentication required on trigger.

3. **Add HTTP Request Tool Nodes for Each API Endpoint**  
   For each API endpoint:  
   - Create a node of type `HTTP Request Tool`.  
   - Configure the URL according to the endpoint, e.g. `https://rms.api.bbc.co.uk//broadcasts` or with path parameters like `https://rms.api.bbc.co.uk//categories/{{id}}`.  
   - HTTP method as per endpoint (mostly GET, some POST, PUT, DELETE).  
   - Add query parameters and path parameters using expressions `$fromAI('parameterName', 'description', 'type')`.  
   - Add header parameters: `X-API-Key` (string), and for personalized endpoints also `Authorization` (Bearer token) and `X-Authentication-Provider`. Populate all via `$fromAI()`.  
   - Enable sending of query and headers.  
   - For POST/PUT nodes, configure request body as needed (typically none here, but can be extended).  
   - Provide descriptive `toolDescription` for each node documenting parameters and headers.

4. **Connect Each HTTP Request Node to the MCP Trigger**  
   - Each HTTP Request node acts as an individual tool accessible via the MCP server.  
   - No direct node-to-node wired connections are needed; MCP trigger handles routing internally.

5. **Set Up Credentials**  
   - Create HTTP Header Auth credential(s) including:  
     - `X-API-Key` for API key authentication.  
     - OAuth Bearer Token for user-specific endpoints requiring authorization.  
   - Assign credentials to each HTTP Request node appropriately.

6. **Parameter Defaults & Constraints**  
   - Use `$fromAI()` placeholders to let AI dynamically supply parameters at runtime.  
   - Provide descriptive parameter names and hints for AI to generate correct input.  
   - For pagination parameters (offset, limit), ensure numeric types are specified.  
   - For boolean parameters, specify type as boolean in `$fromAI()`.

7. **Testing & Validation**  
   - Enable the workflow.  
   - Test individual API calls via the MCP URL using AI agents or HTTP clients.  
   - Monitor for authentication failures, rate limits, and parameter validation errors.

8. **Performance & Maintenance Tips**  
   - Disable or delete unused nodes to keep active tool count below 40 for optimal AI client performance.  
   - Group related API endpoints logically for easier management.  
   - Consider splitting into multiple MCP servers if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                       |
|------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow designed for advanced users only due to its large number of tools (75). Recommended maximum is 40 tools for most clients. | Advanced Warning Sticky Note                                                                         |
| Setup instructions include no authentication requirement on MCP trigger and dynamic parameter population via $fromAI().           | Setup Instructions Sticky Note                                                                       |
| For integration help or custom automations, contact via Discord: https://discord.me/cfomodz                                         | Setup Instructions Sticky Note                                                                       |
| Official n8n documentation for MCP nodes: https://docs.n8n.io/integrations/builtin/cluster-nodes/sub-nodes/n8n-nodes-langchain.toolmcp/ | Setup Instructions Sticky Note                                                                       |
| API Key and OAuth tokens are mandatory for most HTTP requests; ensure valid credentials are configured in n8n.                     | General authentication note (implied by HTTP Request nodes configuration)                            |
| Responses from API calls maintain original API structure, enabling flexible post-processing if needed.                             | Workflow Overview Sticky Note                                                                         |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies. All data processed and exposed are legal, public, and free from illegal or offensive material.

---