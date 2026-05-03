Create Playlists and Control Your Spotify Player with GPT-4

https://n8nworkflows.xyz/workflows/create-playlists-and-control-your-spotify-player-with-gpt-4-5923


# Create Playlists and Control Your Spotify Player with GPT-4

### 1. Workflow Overview

This workflow, titled **"Create Playlists and Control Your Spotify Player with GPT-4"**, serves as an AI-powered Spotify assistant. It enables users to interact via chat to control their Spotify player (play, pause, next track) and to create custom playlists based on natural language instructions. The core innovation is leveraging GPT-4 to ideate playlists by generating playlist names and tracklists, then automatically creating these playlists on Spotify, searching for tracks, and adding them.

The workflow is logically divided into the following blocks:

- **1.1 Chat Interface and User Interaction:** Captures user messages and manages conversation context.
- **1.2 AI Agent Processing and Tool Selection:** Uses an AI agent to interpret user intent and decide which Spotify-related action or playlist creation tool to invoke.
- **1.3 Playlist Ideation (Sub-workflow):** When a playlist creation is requested, this sub-workflow uses GPT-4 to generate a playlist name and tracklist.
- **1.4 Spotify Playlist Creation and Track Management:** Creates the playlist on Spotify, searches tracks by artist/title, and adds tracks to the playlist.
- **1.5 Spotify Player Control Tools:** Provides playback control actions such as play, pause, resume, and next track.
- **1.6 Result Cleaning and Feedback:** Extracts and simplifies playlist data to send back to the AI agent for user feedback.

---

### 2. Block-by-Block Analysis

#### 2.1 Chat Interface and User Interaction

- **Overview:**  
  This block receives chat messages from users and maintains conversational memory for context. It acts as the entry point for user requests.

- **Nodes Involved:**  
  - When chat message received  
  - Simple Memory  
  - OpenAI Chat Model

- **Node Details:**

  - **When chat message received**  
    - Type: Chat Trigger (LangChain chatTrigger)  
    - Role: Webhook to receive user chat messages publicly.  
    - Config: Public webhook with initial greeting message: "Hi there! 👋 I'm your Spotify agent. What should I play for you today?"  
    - Connections: Outputs to Spotify Agent node.  
    - Edge cases: Network downtime could disrupt webhook; malformed messages might cause errors.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a sliding window of last 6 chat exchanges to provide context to the AI agent.  
    - Config: Context window length set to 6 messages.  
    - Connections: Memory input to Spotify Agent node.  
    - Edge cases: Context overflow if conversation is very long; memory loss resets context.

  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides general AI language model capabilities with GPT-4.1-mini model.  
    - Config: Uses OpenAI API credentials under "Plan A's OpenAI".  
    - Connections: Linked to Spotify Agent node as language model.  
    - Edge cases: API rate limits, authentication failures, network timeouts.

---

#### 2.2 AI Agent Processing and Tool Selection

- **Overview:**  
  This block interprets user input using an AI agent that decides which Spotify-related tool or playlist creation workflow to invoke. It manages commands like play, pause, next track, create playlist, and more.

- **Nodes Involved:**  
  - Spotify Agent  
  - Create new playlist (toolWorkflow)  
  - Play a playlist (spotifyTool)  
  - Pause player (spotifyTool)  
  - Resume player (spotifyTool)  
  - Next song (spotifyTool)  
  - Get user playlists (spotifyTool)

- **Node Details:**

  - **Spotify Agent**  
    - Type: LangChain Agent  
    - Role: Core AI agent interpreting user commands and orchestrating tool calls.  
    - Config: System message defines the agent as a Spotify assistant with capabilities for playback control and playlist management. It includes detailed instructions on when and how to invoke each tool.  
    - Inputs: Chat messages, AI memory, language model.  
    - Outputs: Calls to various Spotify tools or sub-workflow nodes.  
    - Edge cases: Ambiguous user commands may cause the agent to ask for clarification or fail to execute desired action.

  - **Create new playlist**  
    - Type: LangChain Tool Workflow  
    - Role: Invokes the playlist ideation sub-workflow when user requests playlist creation.  
    - Config: Passes parameters "Playlist guidelines" and "Number of tracks" collected from AI to the sub-workflow. Defaults to 10 tracks if not specified.  
    - Inputs: Parameters from AI agent.  
    - Outputs: Feeds into the ideation sub-workflow.  
    - Edge cases: Failure to parse parameters or sub-workflow failure.

  - **Play a playlist**  
    - Type: Spotify Tool  
    - Role: Plays a Spotify playlist based on its ID as requested by the user.  
    - Config: Requires exact Spotify playlist ID, retrieved or confirmed by the agent.  
    - Edge cases: Invalid or missing playlist ID, playback errors if no active device.

  - **Pause player / Resume player / Next song**  
    - Type: Spotify Tool  
    - Role: Controls playback state on user's Spotify device with respective operations.  
    - Config: Each configured for one operation: pause, resume, or next track.  
    - Edge cases: No active player, network issues.

  - **Get user playlists**  
    - Type: Spotify Tool  
    - Role: Retrieves all user playlists to help agent find playlist IDs.  
    - Config: Returns all playlists, used when user references previously created playlists.  
    - Edge cases: API limits, empty user playlists.

---

#### 2.3 Playlist Ideation (Sub-workflow)

- **Overview:**  
  This sub-workflow, invoked by the "Create new playlist" tool, uses GPT-4 to generate a playlist name and a list of tracks based on user instructions.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - OpenAI Chat Model2  
  - Structured Output Parser  
  - Ideate playlist

- **Node Details:**

  - **When Executed by Another Workflow**  
    - Type: Execute Workflow Trigger  
    - Role: Entry point for sub-workflow triggered by the main workflow.  
    - Inputs: Receives "Playlist guidelines" (string) and "Number of tracks" (number).  
    - Outputs: Forwards inputs to the ideation node.

  - **OpenAI Chat Model2**  
    - Type: LangChain OpenAI Chat Model  
    - Role: AI model used for playlist ideation with GPT-4.1-mini.  
    - Config: Outputs JSON formatted response for further parsing.  
    - Edge cases: API errors, malformed JSON in output.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses the AI JSON response into structured data with playlistName and tracks array.  
    - Config: Uses example schema to validate.  
    - Edge cases: Parsing failures if AI output malformed or incomplete.

  - **Ideate playlist**  
    - Type: LangChain Chain LLM  
    - Role: Core playlist creation prompt logic that instructs the AI to generate a playlist name and tracklist according to given guidelines and track count.  
    - Config: Detailed prompt including instructions for output format, artist/title information, and playlist title constraints.  
    - Inputs: Playlist guidelines and number of tracks.  
    - Outputs: Parsed JSON with playlist name and tracks.  
    - Edge cases: AI hallucinations or inaccuracies in track or artist names.

---

#### 2.4 Spotify Playlist Creation and Track Management

- **Overview:**  
  This block creates the playlist on Spotify, searches for each track to get track IDs, and adds each track to the created playlist.

- **Nodes Involved:**  
  - Create playlist  
  - Get tracks array  
  - Split out tracks  
  - Search the track  
  - Get track IDs  
  - Add track to playlist  
  - Get the final playlist  
  - Clean data to share with agent

- **Node Details:**

  - **Create playlist**  
    - Type: Spotify API Node  
    - Role: Creates a new blank public playlist with the name generated by AI.  
    - Config: Uses playlistName from AI output, public set to true.  
    - Inputs: Playlist name from Ideate playlist node.  
    - Outputs: Playlist ID and metadata.  
    - Edge cases: Spotify API limits, authentication errors.

  - **Get tracks array**  
    - Type: Set Node  
    - Role: Extracts the "tracks" array from AI output into a separate field for iteration.  
    - Config: Assigns tracks from Ideate playlist output.  
    - Edge cases: Missing or empty tracks array.

  - **Split out tracks**  
    - Type: Split Out Node  
    - Role: Splits the tracks array into individual items for sequential processing.  
    - Config: Field to split is "Tracks".  
    - Edge cases: Empty input array.

  - **Search the track**  
    - Type: Spotify API Node  
    - Role: Searches Spotify for the track by artist and title to find the exact Spotify track ID.  
    - Config: Search query is a concatenation of artist and title. Limit set to 1 result.  
    - Edge cases: No search results found, ambiguous track names.

  - **Get track IDs**  
    - Type: Set Node  
    - Role: Extracts the Spotify track ID from search results for use in adding to playlist.  
    - Config: Assigns "Track ID" as the Spotify track ID from search results.  
    - Edge cases: Missing ID if search failed.

  - **Add track to playlist**  
    - Type: Spotify API Node  
    - Role: Adds the found track to the newly created Spotify playlist by ID.  
    - Config: Playlist ID from Create playlist node, track ID from Get track IDs node.  
    - Edge cases: Rate limits, invalid IDs.

  - **Get the final playlist**  
    - Type: Spotify API Node  
    - Role: Retrieves the complete playlist data after all tracks are added.  
    - Config: Uses playlist ID to get full info.  
    - Edge cases: API failures, partial playlist updates.

  - **Clean data to share with agent**  
    - Type: Code Node  
    - Role: Processes the playlist data to produce a simplified summary for the AI agent to use when reporting back to the user.  
    - Config: Extracts playlist ID, name, public URL, and simplified track info (name, artist, album, URI).  
    - Edge cases: Unexpected data structure changes.

---

#### 2.5 Spotify Player Control Tools

- **Overview:**  
  These nodes provide direct playback control on the Spotify player: play a playlist, pause, resume, and skip to next track.

- **Nodes Involved:**  
  - Play a playlist  
  - Pause player  
  - Resume player  
  - Next song

- **Node Details:**

  - **Play a playlist**  
    - Type: Spotify Tool Node  
    - Role: Starts playback of a specified playlist ID.  
    - Config: Requires exact playlist ID as input.  
    - Edge cases: No active device, invalid playlist ID.

  - **Pause player**  
    - Type: Spotify Tool Node  
    - Role: Pauses the current playback.  
    - Config: No additional parameters.  
    - Edge cases: No active playback.

  - **Resume player**  
    - Type: Spotify Tool Node  
    - Role: Resumes paused playback.  
    - Config: No additional parameters.  
    - Edge cases: No paused playback state.

  - **Next song**  
    - Type: Spotify Tool Node  
    - Role: Skips to the next track.  
    - Config: No additional parameters.  
    - Edge cases: No next track available, no active playback.

---

#### 2.6 Result Cleaning and Feedback

- **Overview:**  
  After playlist creation, this block cleans and summarizes the playlist information and sends it back to the AI agent, which can then provide user-friendly feedback.

- **Nodes Involved:**  
  - Clean data to share with agent

- **Node Details:**

  - **Clean data to share with agent**  
    - Type: Code Node  
    - Role: Simplifies the playlist data schema to reduce token usage and improve clarity for the AI agent's follow-up messages.  
    - Code extracts the playlist ID, name, public URL, and an array of tracks with name, artist, album, and URI.  
    - Edge cases: Incomplete data or API changes.

---

### 3. Summary Table

| Node Name                   | Node Type                                   | Functional Role                                           | Input Node(s)                      | Output Node(s)                     | Sticky Note                                                                                                                       |
|-----------------------------|---------------------------------------------|-----------------------------------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| When chat message received  | LangChain Chat Trigger                       | Entry point for user chat messages                         |                                   | Spotify Agent                     | ## Chat trigger and interface                                                                                                    |
| Simple Memory               | LangChain Memory Buffer Window               | Maintains conversation context                             | When chat message received         | Spotify Agent                     | ## Chat trigger and interface                                                                                                    |
| OpenAI Chat Model           | LangChain OpenAI Chat Model                   | Provides AI language model for chat                        |                                   | Spotify Agent                     | ## Chat trigger and interface                                                                                                    |
| Spotify Agent               | LangChain Agent                               | Interprets commands and routes to tools                   | When chat message received, Simple Memory, OpenAI Chat Model | Create new playlist, Play a playlist, Pause player, Resume player, Next song, Get user playlists | ## AI agent tools<br>This agent is the engine of the workflow. Its system instructions define how to call the different tool to interact with Spotify |
| Create new playlist         | LangChain Tool Workflow                       | Invokes playlist ideation sub-workflow                     | Spotify Agent                     | Spotify Agent                     | ## Agent tools                                                                                                                   |
| Play a playlist             | Spotify Tool                                  | Controls Spotify playback to play a playlist               | Spotify Agent                     | Spotify Agent                     | ## Agent tools                                                                                                                   |
| Pause player               | Spotify Tool                                  | Controls Spotify playback to pause                          | Spotify Agent                     | Spotify Agent                     | ## Agent tools                                                                                                                   |
| Resume player              | Spotify Tool                                  | Controls Spotify playback to resume                         | Spotify Agent                     | Spotify Agent                     | ## Agent tools                                                                                                                   |
| Next song                  | Spotify Tool                                  | Controls Spotify playback to skip to next track            | Spotify Agent                     | Spotify Agent                     | ## Agent tools                                                                                                                   |
| Get user playlists         | Spotify Tool                                  | Retrieves user's playlists                                  | Spotify Agent                     | Spotify Agent                     | ## Agent tools                                                                                                                   |
| When Executed by Another Workflow | Execute Workflow Trigger                 | Entry point for playlist ideation sub-workflow             | Create new playlist               | Ideate playlist                  | ## Playlist creation sub-workflow                                                                                                |
| OpenAI Chat Model2          | LangChain OpenAI Chat Model                   | AI model for playlist ideation                             | When Executed by Another Workflow  | Structured Output Parser          | ## Playlist creation sub-workflow                                                                                                |
| Structured Output Parser    | LangChain Structured Output Parser            | Parses AI playlist JSON output                             | OpenAI Chat Model2                | Ideate playlist                  | ## Playlist creation sub-workflow                                                                                                |
| Ideate playlist            | LangChain Chain LLM                            | Generates playlist name and track list                     | When Executed by Another Workflow, Structured Output Parser | Create playlist                 | ## AI step to plan the playlist                                                                                                  |
| Create playlist            | Spotify API Node                              | Creates a blank playlist on Spotify                         | Ideate playlist                  | Get tracks array                 | ## Playlist creation sub-workflow                                                                                                |
| Get tracks array           | Set Node                                      | Extracts tracks array from AI output                        | Create playlist                  | Split out tracks                | ## Playlist creation sub-workflow                                                                                                |
| Split out tracks           | Split Out Node                                | Splits tracks array into individual tracks                 | Get tracks array                 | Search the track               | ## For each track<br>Each track gets searched in Spotify to get their specific IDs to then add them one by one to the playlist. |
| Search the track           | Spotify API Node                              | Searches Spotify for track IDs based on artist and title  | Split out tracks                | Get track IDs                  | ## For each track<br>Each track gets searched in Spotify to get their specific IDs to then add them one by one to the playlist. |
| Get track IDs              | Set Node                                      | Extracts Spotify track ID for adding to playlist           | Search the track                | Add track to playlist          | ## For each track<br>Each track gets searched in Spotify to get their specific IDs to then add them one by one to the playlist. |
| Add track to playlist      | Spotify API Node                              | Adds track to the created Spotify playlist                 | Get track IDs                   | Get the final playlist         | ## Playlist creation sub-workflow                                                                                                |
| Get the final playlist     | Spotify API Node                              | Retrieves the full playlist data                            | Add track to playlist           | Clean data to share with agent | ## Playlist creation sub-workflow                                                                                                |
| Clean data to share with agent | Code Node                                | Simplifies playlist data for AI agent feedback             | Get the final playlist          | Spotify Agent                 | ## Playlist creation sub-workflow                                                                                                |
| Sticky Note                | Sticky Note Node                              | Provides detailed usage instructions and project overview  |                                   |                                   | ## Try It Out!<br>This n8n template provides a powerful AI-powered chatbot that acts as your personal Spotify DJ...             |
| Sticky Note1               | Sticky Note Node                              | Labels chat trigger section                                 |                                   |                                   | ## Chat trigger and interface                                                                                                    |
| Sticky Note2               | Sticky Note Node                              | Labels agent tools section                                  |                                   |                                   | ## Agent tools                                                                                                                   |
| Sticky Note3               | Sticky Note Node                              | Explains AI agent tools                                     |                                   |                                   | ## AI agent tools<br>This agent is the engine of the workflow. Its system instructions define how to call the different tool to interact with Spotify |
| Sticky Note4               | Sticky Note Node                              | Explains AI playlist ideation step                          |                                   |                                   | ## AI step to plan the playlist                                                                                                  |
| Sticky Note5               | Sticky Note Node                              | Labels playlist creation sub-workflow                       |                                   |                                   | ## Playlist creation sub-workflow                                                                                                |
| Sticky Note6               | Sticky Note Node                              | Explains per-track processing in playlist creation         |                                   |                                   | ## For each track<br>Each track gets searched in Spotify to get their specific IDs to then add them one by one to the playlist. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Chat Trigger Node:**  
   - Type: LangChain Chat Trigger  
   - Configure a public webhook with initial message:  
     "Hi there! 👋 I'm your Spotify agent. What should I play for you today?"  
   - No credentials needed.  
   - Position: Entry point of the workflow.

2. **Add Simple Memory Node:**  
   - Type: LangChain Memory Buffer Window  
   - Set context window length to 6 messages.  
   - Connect output of Chat Trigger to input of Simple Memory.

3. **Add OpenAI Chat Model Node (General):**  
   - Type: LangChain OpenAI Chat Model  
   - Model: GPT-4.1-mini  
   - Provide OpenAI API credentials.  
   - Connect Chat Trigger output to this node as language model.

4. **Add Spotify Agent Node:**  
   - Type: LangChain Agent  
   - Configure system message with detailed instructions defining Spotify assistant capabilities (playback control, playlist management).  
   - Connect inputs: Chat Trigger (main), Simple Memory (memory), OpenAI Chat Model (language model).  
   - Configure outputs to various Spotify tools and sub-workflows.

5. **Add Spotify Playback Control Nodes:**  
   - Add Spotify Tool nodes for: Play a playlist, Pause player, Resume player, Next song.  
   - Each configured with respective operation and Spotify OAuth2 credentials.  
   - Connect Spotify Agent output (ai_tool) to each.

6. **Add Get User Playlists Node:**  
   - Spotify Tool node with operation "getUserPlaylists" and returnAll true.  
   - Connect Spotify Agent output (ai_tool) to this node.

7. **Add "Create new playlist" Tool Workflow Node:**  
   - Type: LangChain Tool Workflow  
   - Link to a sub-workflow that builds the playlist.  
   - Define inputs: "Playlist guidelines" (string), "Number of tracks" (number).  
   - Connect Spotify Agent output (ai_tool) to this node.

8. **Build Playlist Ideation Sub-workflow:**  
   - Trigger node: Execute Workflow Trigger with inputs "Playlist guidelines" and "Number of tracks".  
   - OpenAI Chat Model2: GPT-4.1-mini, configured for JSON output.  
   - Structured Output Parser: Use example JSON schema for playlist name and tracks.  
   - Chain LLM node ("Ideate playlist"): Configure detailed prompt instructing AI to create playlist name and track list in specified JSON format.  
   - Connect nodes sequentially: Trigger → OpenAI Chat Model2 → Structured Output Parser → Ideate playlist → Create playlist.

9. **Add Spotify Playlist Creation Nodes:**  
   - Create playlist node: Use Spotify API node to create a playlist with name from AI output, set public to true.  
   - Get tracks array node: Set node to extract "tracks" array from AI output.  
   - Split out tracks node: Split array into individual track items.  
   - Search the track node: Spotify API search node, query format "artist - title", limit 1.  
   - Get track IDs node: Set node to extract track ID from search results.  
   - Add track to playlist node: Spotify API node to add track to playlist by playlist ID and track ID.  
   - Connect nodes sequentially for looping over tracks.

10. **Add Final Playlist Retrieval and Cleaning:**  
    - Get the final playlist node: Spotify API node to fetch full playlist data after all tracks added.  
    - Clean data to share with agent node: Code node to extract and simplify playlist info (ID, name, public URL, simplified tracks).  
    - Connect final playlist retrieval → cleaning → output back to Spotify Agent.

11. **Connect Main Workflow to Sub-workflow:**  
    - Ensure "Create new playlist" tool workflow node calls the sub-workflow and passes parameters correctly.  
    - Connect outputs and inputs accordingly.

12. **Add Credentials:**  
    - Configure OpenAI API credentials under "Plan A's OpenAI" or equivalent.  
    - Configure Spotify OAuth2 credentials for all Spotify nodes and tools.

13. **Add Sticky Notes:**  
    - Add descriptive sticky notes near major blocks for documentation and user guidance, including the introductory note explaining usage and requirements.

14. **Activate Workflow:**  
    - Test webhook with chat interface.  
    - Verify AI agent correctly interprets commands and calls appropriate Spotify actions or creates playlists.  
    - Monitor for errors and adjust retry policies as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                             | Context or Link                                                                                                 |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| This n8n template provides a powerful AI-powered chatbot acting as your personal Spotify DJ. Tell the chatbot your mood and it creates a custom playlist with real tracks on Spotify.    | See Sticky Note at workflow start.                                                                             |
| Requirements: AI Provider Account (e.g., OpenAI), Spotify Account & Developer Credentials.                                                                                                | See Sticky Note at workflow start.                                                                             |
| The AI agent uses GPT-4.1-mini model for efficient and accurate playlist ideation and chat understanding.                                                                               | Node configurations for OpenAI Chat Model and OpenAI Chat Model2.                                              |
| The playlist creation sub-workflow uses strict JSON output formatting to ensure reliable parsing of playlist name and tracks.                                                           | Refer to Structured Output Parser and Ideate playlist node prompts.                                            |
| Playback control tools require an active Spotify player/device for commands like pause, resume, next track, and play playlist to work correctly.                                        | Spotify Tool nodes for playback control.                                                                       |
| The workflow implements retryOnFail on critical Spotify API nodes like Create playlist and Search the track to handle transient API errors gracefully.                                 | See node configurations for retry policies.                                                                    |
| AI agent instructions emphasize user-friendly responses, avoiding technical details like playlist or track IDs in replies, focusing on artist and track names instead.                   | System message in Spotify Agent node.                                                                           |
| For detailed Spotify API information and OAuth setup, visit https://developer.spotify.com/documentation/web-api/                                                                        | Official Spotify Developer documentation.                                                                      |

---

This comprehensive documentation enables advanced users and AI agents to understand, reproduce, and adapt the workflow while anticipating typical edge cases and integration challenges.