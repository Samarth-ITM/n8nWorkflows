🛠️ Spotify Tool MCP Server 💪 all 30 operations

https://n8nworkflows.xyz/workflows/----spotify-tool-mcp-server----all-30-operations-5360


# 🛠️ Spotify Tool MCP Server 💪 all 30 operations

### 1. Workflow Overview

This workflow, titled **"Spotify Tool MCP Server"**, serves as a comprehensive multi-command processor (MCP) for Spotify API operations through n8n. It enables execution of all 30 major Spotify API operations available in the Spotify Tool node, acting as a centralized server triggered by an external MCP trigger node.

**Target Use Cases:**  
- Developers or automation architects seeking unified access to Spotify’s extensive API capabilities within a single workflow.  
- Automations that require dynamic interaction with Spotify data such as albums, artists, playlists, tracks, and player controls.  
- Systems integrating Spotify features triggered externally via webhook or other MCP-compatible triggers.

**Logical Blocks:**  
- **1.1 Trigger and Input Reception:** Receives MCP trigger calls to initiate corresponding Spotify API actions.  
- **1.2 Album Operations:** Handles queries related to albums including retrieval, track listing, and search.  
- **1.3 Artist Operations:** Manages artist information retrieval, related artists, top tracks, and search.  
- **1.4 Playlist Operations:** Supports playlist creation, retrieval, modification, and search.  
- **1.5 Track Operations:** Facilitates track retrieval, audio features, liked tracks, and track search.  
- **1.6 Player Controls:** Controls Spotify playback features like play, pause, skip, volume, queue management, etc.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger and Input Reception

- **Overview:**  
  This block acts as the entry point, receiving external MCP trigger requests which specify which Spotify operation to perform.

- **Nodes Involved:**  
  - Spotify Tool MCP Server (MCP Trigger node)

- **Node Details:**

  - **Spotify Tool MCP Server**  
    - *Type:* `@n8n/n8n-nodes-langchain.mcpTrigger` (MCP Trigger node)  
    - *Configuration:* Configured with a webhook for MCP trigger calls. It awaits external commands specifying the Spotify operation to execute.  
    - *Inputs:* None (starts workflow)  
    - *Outputs:* Connects to all Spotify operation nodes (indirectly via ai_tool connections)  
    - *Edge Cases:* Failure may occur if webhook is unreachable, or if incoming commands are malformed or unsupported.  
    - *Notes:* Central orchestrator node for all Spotify operations.

---

#### 2.2 Album Operations

- **Overview:**  
  Covers fetching album details, new releases, album tracks, and searching albums by keyword.

- **Nodes Involved:**  
  - Get an album  
  - Get new album releases  
  - Get an album's tracks by URI or ID  
  - Search albums by keyword  
  - Sticky Note 1 (empty, likely informational placeholder)

- **Node Details:**

  - **Get an album**  
    - *Type:* Spotify Tool  
    - *Role:* Retrieves detailed information about a specific album by ID or URI.  
    - *Configuration:* Parameters to specify album ID or URI (dynamic inputs expected).  
    - *Input:* Triggered by MCP Trigger node  
    - *Output:* Album metadata JSON  
    - *Failures:* Invalid or missing album ID, API rate limits.

  - **Get new album releases**  
    - *Type:* Spotify Tool  
    - *Role:* Fetches the latest album releases globally or regionally.  
    - *Configuration:* Optional country or limit parameters.  
    - *Failures:* Network issues, API quota limits.

  - **Get an album's tracks by URI or ID**  
    - *Type:* Spotify Tool  
    - *Role:* Lists all tracks within a specified album.  
    - *Configuration:* Album ID or URI input required.  
    - *Failures:* Invalid album reference.

  - **Search albums by keyword**  
    - *Type:* Spotify Tool  
    - *Role:* Searches Spotify albums matching a keyword query.  
    - *Configuration:* Search string parameter, optional filters (limit, market).  
    - *Failures:* Empty search strings, API errors.

  - **Sticky Note 1**  
    - Empty content, possibly a placeholder or separator.

---

#### 2.3 Artist Operations

- **Overview:**  
  Enables retrieval of artist information, including albums, related artists, top tracks, and searching artists.

- **Nodes Involved:**  
  - Get an artist  
  - Get an artist's albums by URI or ID  
  - Get an artist's related artists by URI or ID  
  - Get an artist's top tracks by URI or ID  
  - Search artists by keyword  
  - Sticky Note 2 (empty)

- **Node Details:**

  - **Get an artist**  
    - Retrieves artist profile details by ID or URI.  
    - Requires artist identifier as input.

  - **Get an artist's albums by URI or ID**  
    - Lists albums released by the artist.  
    - Accepts artist ID or URI as input.

  - **Get an artist's related artists by URI or ID**  
    - Returns artists related to the specified artist.  
    - Input: artist identifier.

  - **Get an artist's top tracks by URI or ID**  
    - Fetches the top tracks of the artist for a given country or globally.  
    - Parameters: artist ID, optional market.

  - **Search artists by keyword**  
    - Searches Spotify artists by a provided keyword.

  - **Sticky Note 2**  
    - Empty content.

---

#### 2.4 Playlist Operations

- **Overview:**  
  Supports creation, retrieval, modification, and searching of playlists including item addition and removal.

- **Nodes Involved:**  
  - Add an Item to a playlist  
  - Create a playlist  
  - Get a playlist  
  - Get a user's playlists  
  - Get a playlist's tracks by URI or ID  
  - Remove an item from a playlist  
  - Search playlists by keyword  
  - Sticky Note 6 (empty)

- **Node Details:**

  - **Add an Item to a playlist**  
    - Adds tracks or episodes to an existing playlist.  
    - Input: playlist ID and item URIs.

  - **Create a playlist**  
    - Creates a new playlist under the user's account.  
    - Parameters: playlist name, description, public/private status.

  - **Get a playlist**  
    - Retrieves details of a specific playlist.

  - **Get a user's playlists**  
    - Lists playlists owned or followed by a user.

  - **Get a playlist's tracks by URI or ID**  
    - Lists all tracks in a playlist.

  - **Remove an item from a playlist**  
    - Removes specified items from a playlist.

  - **Search playlists by keyword**  
    - Searches for playlists matching keywords.

  - **Sticky Note 6**  
    - Empty content.

---

#### 2.5 Track Operations

- **Overview:**  
  Manages single track retrieval, searching tracks, getting audio features, and liked tracks.

- **Nodes Involved:**  
  - Get a track  
  - Get audio features of a track  
  - Search tracks by keyword  
  - Get liked tracks  
  - Sticky Note 7 (empty)

- **Node Details:**

  - **Get a track**  
    - Retrieves metadata for a single track by ID or URI.

  - **Get audio features of a track**  
    - Retrieves detailed audio analysis and features for a track.

  - **Search tracks by keyword**  
    - Searches tracks matching a keyword string.

  - **Get liked tracks**  
    - Retrieves the user's liked (saved) tracks.

  - **Sticky Note 7**  
    - Empty content.

---

#### 2.6 Player Controls

- **Overview:**  
  Controls Spotify playback functions such as play, pause, skip, volume adjustment, queue management.

- **Nodes Involved:**  
  - Add a song to a queue  
  - Get the currently playing track  
  - Skip to the next track  
  - Pause the player  
  - Skip to the previous song  
  - Get the recently played tracks  
  - Resume the player  
  - Set volume on the player  
  - Start music on the player  
  - Sticky Note 3, Sticky Note 4, Sticky Note 5 (all empty)

- **Node Details:**

  - **Add a song to a queue**  
    - Adds a specific track URI to the user’s playback queue.

  - **Get the currently playing track**  
    - Retrieves information about the user's current playback.

  - **Skip to the next track**  
    - Commands Spotify to skip to the next song in the queue.

  - **Pause the player**  
    - Pauses current playback.

  - **Skip to the previous song**  
    - Commands Spotify to go back to the previous track.

  - **Get the recently played tracks**  
    - Fetches the user's recently played tracks history.

  - **Resume the player**  
    - Resumes paused playback.

  - **Set volume on the player**  
    - Adjusts the playback volume to a specified level.

  - **Start music on the player**  
    - Starts playback on the active device.

  - **Sticky Notes 3, 4, 5**  
    - Empty content.

---

### 3. Summary Table

| Node Name                          | Node Type                       | Functional Role                         | Input Node(s)            | Output Node(s)                | Sticky Note |
|-----------------------------------|--------------------------------|---------------------------------------|--------------------------|------------------------------|-------------|
| Workflow Overview 0               | Sticky Note                    | Informational placeholder              | None                     | None                         |             |
| Spotify Tool MCP Server           | MCP Trigger                   | Entry point, triggers Spotify actions | None                     | All Spotify Tool nodes (ai_tool) |             |
| Get an album                     | Spotify Tool                  | Fetch album details                    | Spotify Tool MCP Server  | None                         |             |
| Get new album releases           | Spotify Tool                  | Fetch new album releases               | Spotify Tool MCP Server  | None                         |             |
| Get an album's tracks by URI or ID | Spotify Tool                | List tracks of an album                | Spotify Tool MCP Server  | None                         |             |
| Search albums by keyword         | Spotify Tool                  | Search albums by keyword               | Spotify Tool MCP Server  | None                         |             |
| Sticky Note 1                   | Sticky Note                   | Placeholder                           | None                     | None                         |             |
| Get an artist                   | Spotify Tool                  | Fetch artist details                   | Spotify Tool MCP Server  | None                         |             |
| Get an artist's albums by URI or ID | Spotify Tool                | List albums of an artist               | Spotify Tool MCP Server  | None                         |             |
| Get an artist's related artists by URI or ID | Spotify Tool         | Fetch related artists                  | Spotify Tool MCP Server  | None                         |             |
| Get an artist's top tracks by URI or ID | Spotify Tool             | Fetch top tracks of an artist          | Spotify Tool MCP Server  | None                         |             |
| Search artists by keyword       | Spotify Tool                  | Search artists by keyword              | Spotify Tool MCP Server  | None                         |             |
| Sticky Note 2                   | Sticky Note                   | Placeholder                           | None                     | None                         |             |
| Add an Item to a playlist       | Spotify Tool                  | Add items to playlist                  | Spotify Tool MCP Server  | None                         |             |
| Create a playlist               | Spotify Tool                  | Create new playlist                    | Spotify Tool MCP Server  | None                         |             |
| Get a playlist                 | Spotify Tool                  | Get playlist details                   | Spotify Tool MCP Server  | None                         |             |
| Get a user's playlists         | Spotify Tool                  | List user's playlists                  | Spotify Tool MCP Server  | None                         |             |
| Get a playlist's tracks by URI or ID | Spotify Tool                | List tracks in a playlist              | Spotify Tool MCP Server  | None                         |             |
| Remove an item from a playlist | Spotify Tool                  | Remove items from playlist             | Spotify Tool MCP Server  | None                         |             |
| Search playlists by keyword    | Spotify Tool                  | Search playlists                       | Spotify Tool MCP Server  | None                         |             |
| Sticky Note 6                 | Sticky Note                   | Placeholder                           | None                     | None                         |             |
| Get a track                   | Spotify Tool                  | Get track details                     | Spotify Tool MCP Server  | None                         |             |
| Get audio features of a track | Spotify Tool                  | Get audio features for track          | Spotify Tool MCP Server  | None                         |             |
| Search tracks by keyword     | Spotify Tool                  | Search tracks                         | Spotify Tool MCP Server  | None                         |             |
| Get liked tracks             | Spotify Tool                  | Get user's liked tracks                | Spotify Tool MCP Server  | None                         |             |
| Sticky Note 7               | Sticky Note                   | Placeholder                           | None                     | None                         |             |
| Add a song to a queue       | Spotify Tool                  | Add track to playback queue            | Spotify Tool MCP Server  | None                         |             |
| Get the currently playing track | Spotify Tool                  | Get current playback info               | Spotify Tool MCP Server  | None                         |             |
| Skip to the next track      | Spotify Tool                  | Skip to next track                    | Spotify Tool MCP Server  | None                         |             |
| Pause the player            | Spotify Tool                  | Pause playback                       | Spotify Tool MCP Server  | None                         |             |
| Skip to the previous song  | Spotify Tool                  | Skip to previous track                 | Spotify Tool MCP Server  | None                         |             |
| Get the recently played tracks | Spotify Tool                  | Get user's recently played tracks     | Spotify Tool MCP Server  | None                         |             |
| Resume the player          | Spotify Tool                  | Resume playback                      | Spotify Tool MCP Server  | None                         |             |
| Set volume on the player   | Spotify Tool                  | Set playback volume                  | Spotify Tool MCP Server  | None                         |             |
| Start music on the player | Spotify Tool                  | Start playback                      | Spotify Tool MCP Server  | None                         |             |
| Sticky Note 3             | Sticky Note                   | Placeholder                         | None                     | None                         |             |
| Sticky Note 4             | Sticky Note                   | Placeholder                         | None                     | None                         |             |
| Sticky Note 5             | Sticky Note                   | Placeholder                         | None                     | None                         |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the MCP Trigger Node:**  
   - Add a node of type `@n8n/n8n-nodes-langchain.mcpTrigger`.  
   - Configure it with a webhook ID or leave default to receive external MCP commands.  
   - This node acts as the entry trigger for all Spotify operations.

2. **Add Spotify Tool Nodes for Album Operations:**  
   - Add nodes of type `Spotify Tool` for:  
     - "Get an album"  
     - "Get new album releases"  
     - "Get an album's tracks by URI or ID"  
     - "Search albums by keyword"  
   - Configure each node with parameters to accept Album ID, URI, or search keywords dynamically from the MCP Trigger.  
   - No static parameters are preset; all inputs come from the MCP trigger payload.

3. **Add Spotify Tool Nodes for Artist Operations:**  
   - Add nodes for:  
     - "Get an artist"  
     - "Get an artist's albums by URI or ID"  
     - "Get an artist's related artists by URI or ID"  
     - "Get an artist's top tracks by URI or ID"  
     - "Search artists by keyword"  
   - Configure to accept artist identifiers or search keywords as input.

4. **Add Spotify Tool Nodes for Playlist Operations:**  
   - Add nodes for:  
     - "Add an Item to a playlist"  
     - "Create a playlist"  
     - "Get a playlist"  
     - "Get a user's playlists"  
     - "Get a playlist's tracks by URI or ID"  
     - "Remove an item from a playlist"  
     - "Search playlists by keyword"  
   - Set these to receive playlist IDs, user IDs, track URIs, or keywords from the MCP trigger.

5. **Add Spotify Tool Nodes for Track Operations:**  
   - Add nodes for:  
     - "Get a track"  
     - "Get audio features of a track"  
     - "Search tracks by keyword"  
     - "Get liked tracks"  
   - Configure to receive track IDs, URIs, or keywords accordingly.

6. **Add Spotify Tool Nodes for Player Control Operations:**  
   - Add nodes for:  
     - "Add a song to a queue"  
     - "Get the currently playing track"  
     - "Skip to the next track"  
     - "Pause the player"  
     - "Skip to the previous song"  
     - "Get the recently played tracks"  
     - "Resume the player"  
     - "Set volume on the player"  
     - "Start music on the player"  
   - Configure inputs for track URIs, volume levels, or device IDs where applicable.

7. **Connect Each Spotify Tool Node's Input to the MCP Trigger Node:**  
   - Use the `ai_tool` connection type or equivalent to route the MCP trigger to each Spotify node so that commands specifying operations are correctly dispatched.

8. **Credential Setup:**  
   - Configure Spotify OAuth2 credentials on each Spotify Tool node to authorize API calls.  
   - Ensure the MCP Trigger node is accessible via webhook with necessary security.

9. **Add Sticky Notes (Optional):**  
   - Add sticky notes as placeholders or documentation blocks for structure clarity.

10. **Testing:**  
    - Test each Spotify Tool node individually via the MCP trigger to ensure API calls work and parameters are correctly received.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                      |
|---------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| The workflow covers all 30 main Spotify API operations supported by the Spotify Tool node in n8n.                              | Workflow description                                |
| MCP Trigger node enables external multi-command control via webhook, ideal for integration scenarios.                          | Node: Spotify Tool MCP Server                       |
| Ensure Spotify OAuth2 credentials have sufficient scopes for all operations (e.g., user-read-playback-state, playlist-modify). | Spotify Developer Dashboard and n8n Credentials    |
| Empty sticky notes are used as visual separators or placeholders, no content provided.                                           | Workflow structure                                  |
| Spotify API rate limits and authorization expiration should be monitored for reliable operation.                               | Integration best practice                           |
| For more information on Spotify API scopes and endpoints, refer to https://developer.spotify.com/documentation/web-api/         | Official Spotify API documentation                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a workflow automation tool. The process strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.