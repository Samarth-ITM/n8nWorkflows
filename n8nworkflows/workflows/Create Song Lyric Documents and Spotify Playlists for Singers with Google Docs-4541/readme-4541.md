Create Song Lyric Documents and Spotify Playlists for Singers with Google Docs

https://n8nworkflows.xyz/workflows/create-song-lyric-documents-and-spotify-playlists-for-singers-with-google-docs-4541


# Create Song Lyric Documents and Spotify Playlists for Singers with Google Docs

### 1. Workflow Overview

This workflow automates the creation of a song setlist for singers or bands by integrating Google Sheets, Google Docs, Spotify, and AI-powered data verification. It processes a Google Spreadsheet named "Setlist_Manager" containing artists and song titles, then performs the following key logical blocks:

- **1.1 Playlist & Document Initialization:** Creates a dated Spotify playlist and a Google Document to store lyrics.
- **1.2 Input Data Retrieval:** Fetches song and artist data from the Google Sheet.
- **1.3 Data Verification with AI:** Uses an AI model to verify and correct artist and song title information.
- **1.4 Lyrics Fetching:** Retrieves lyrics for each verified song from an external lyrics API.
- **1.5 Lyrics Document Population:** Appends the retrieved lyrics and song information to the Google Document.
- **1.6 Spotify Song Search and Playlist Addition:** Searches for each song in Spotify and adds it to the created playlist.

This modular structure facilitates step-by-step processing, error tolerance (e.g., continuing on failed lyric fetch), and seamless integration between data sources and multimedia platforms for band practice preparation.

---

### 2. Block-by-Block Analysis

#### 2.1 Playlist & Document Initialization

- **Overview:**  
Creates a Spotify playlist and a Google Document named "Setlist - [current date]" to organize songs and their lyrics for the setlist.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Create Playlist  
  - Create Doc

- **Node Details:**

  - **When clicking ‘Test workflow’ (Manual Trigger)**  
    - Type: Manual trigger node; entry point for workflow execution.  
    - Config: No parameters; user manually triggers workflow start.  
    - Inputs: None  
    - Outputs: Connected to "Create Playlist" node  
    - Edge Cases: None; manual start only.

  - **Create Playlist (Spotify Node)**  
    - Type: Spotify API node to create playlists.  
    - Configuration:  
      - Playlist name: "Setlist - [current date]" (dynamic using current date in yyyy-MM-dd format).  
      - Operation: Create playlist.  
      - Credentials: Spotify OAuth2.  
    - Inputs: From manual trigger.  
    - Outputs: Spotify playlist metadata, including playlist URI.  
    - Edge Cases: Spotify API auth errors, rate limits, or playlist creation failures.

  - **Create Doc (Google Docs Node)**  
    - Type: Google Docs API node to create a new document.  
    - Configuration:  
      - Document title: "Setlist - [current date]" (dynamic).  
      - Folder ID: Specified folder for storage.  
      - Credentials: Google Docs OAuth2.  
    - Inputs: From "Create Playlist" node.  
    - Outputs: Google Document metadata including document ID.  
    - Edge Cases: Google API auth failures, quota limits.

---

#### 2.2 Input Data Retrieval

- **Overview:**  
Retrieves the list of artists and song titles from the Google Spreadsheet named "Setlist_Manager".

- **Nodes Involved:**  
  - get data

- **Node Details:**

  - **get data (Google Sheets Node)**  
    - Type: Google Sheets API node to read spreadsheet rows.  
    - Configuration:  
      - Spreadsheet: "Setlist_Manager" (hardcoded document ID).  
      - Sheet: First sheet (gid=0).  
      - Credentials: Google Sheets OAuth2.  
    - Inputs: From "Create Doc" node.  
    - Outputs: Rows containing "Artist" and "SongTitle".  
    - Edge Cases: Sheet access permission errors, empty or malformed rows.

---

#### 2.3 Data Verification with AI

- **Overview:**  
Uses an AI-powered information extractor to verify and correct artist names and song titles.

- **Nodes Involved:**  
  - Information Extractor  
  - OpenAI Chat Model

- **Node Details:**

  - **Information Extractor (Langchain Information Extractor Node)**  
    - Type: AI extraction node using Langchain framework.  
    - Configuration:  
      - Input text template includes artist and song title from Google Sheets.  
      - System prompt: Instructs the model to extract artist and song title only, omitting unknowns.  
      - Attributes extracted: "Artist" and "SongTitle" (required).  
    - Inputs: From "get data" node.  
    - Outputs: Cleaned and verified artist and song title.  
    - Linked Node: Receives language model input from "OpenAI Chat Model".  
    - Edge Cases: AI misinterpretation, missing values, or empty responses.

  - **OpenAI Chat Model (Langchain OpenAI Chat Model Node)**  
    - Type: AI language model node using OpenAI GPT-4o-mini.  
    - Configuration: Default model parameters, no special options.  
    - Inputs: Linked to "Information Extractor" as language model backend.  
    - Outputs: Processed AI responses for extraction.  
    - Credentials: OpenAI API key.  
    - Edge Cases: API rate limits, network errors, malformed responses.

---

#### 2.4 Lyrics Fetching

- **Overview:**  
Fetches lyrics for each verified song using an external lyrics API.

- **Nodes Involved:**  
  - Get Lyrics

- **Node Details:**

  - **Get Lyrics (HTTP Request Node)**  
    - Type: HTTP request to lyrics.ovh API.  
    - Configuration:  
      - URL dynamically constructed as https://api.lyrics.ovh/v1/{Artist}/{SongTitle} using AI-verified data.  
      - On error: Continue workflow (do not fail on missing lyrics).  
    - Inputs: From "Information Extractor" node.  
    - Outputs: JSON response containing lyrics text.  
    - Edge Cases: Missing lyrics, 404 errors, API downtime.

---

#### 2.5 Lyrics Document Population

- **Overview:**  
Appends each song’s title, artist, and lyrics to the Google Document, inserting page breaks between songs.

- **Nodes Involved:**  
  - Populate Doc

- **Node Details:**

  - **Populate Doc (Google Docs Node)**  
    - Type: Google Docs API node to update document content.  
    - Configuration:  
      - Document URL: Uses document ID from "Create Doc".  
      - Actions: Insert song title and artist on a new line, then lyrics, followed by a page break.  
      - Uses expressions to pull data from "Information Extractor" and "Get Lyrics".  
      - Operation: Update document.  
      - Credentials: Google Docs OAuth2.  
    - Inputs: From "Get Lyrics" node.  
    - Outputs: Updated document metadata.  
    - Edge Cases: Google Docs API limits, malformed text insertion.

---

#### 2.6 Spotify Song Search and Playlist Addition

- **Overview:**  
Searches for each song in Spotify by artist and song title, then adds the found track to the created playlist.

- **Nodes Involved:**  
  - Search for Song  
  - Add Song to Playlist

- **Node Details:**

  - **Search for Song (Spotify Node)**  
    - Type: Spotify API node to search tracks.  
    - Configuration:  
      - Search query formatted as "[Artist] by [SongTitle]" from "get data" node (note: this uses the original data, not AI-verified).  
      - Limit: 1 (get top result).  
      - Resource: Track.  
      - Credentials: Spotify OAuth2.  
    - Inputs: From "Populate Doc" node.  
    - Outputs: Track search result including URI.  
    - Edge Cases: No search results, API errors.

  - **Add Song to Playlist (Spotify Node)**  
    - Type: Spotify API node to add track to playlist.  
    - Configuration:  
      - Playlist ID: From "Create Playlist" node (URI).  
      - Track ID: From "Search for Song" node (track URI).  
      - Position: 0 (adds to start).  
      - Credentials: Spotify OAuth2.  
    - Inputs: From "Search for Song" node.  
    - Outputs: Confirmation of addition.  
    - Edge Cases: Invalid playlist or track IDs, API errors, rate limits.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                           | Input Node(s)               | Output Node(s)            | Sticky Note                                                                                                         |
|---------------------------|-----------------------------------------|------------------------------------------|-----------------------------|---------------------------|---------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                         | Entry point for workflow                  | None                        | Create Playlist            |                                                                                                                     |
| Create Playlist           | Spotify                                 | Creates dated Spotify playlist            | When clicking ‘Test workflow’ | Create Doc                 |                                                                                                                     |
| Create Doc                | Google Docs                             | Creates Google Doc for lyrics             | Create Playlist             | get data                   |                                                                                                                     |
| get data                  | Google Sheets                           | Fetches artists and song titles           | Create Doc                  | Information Extractor      |                                                                                                                     |
| Information Extractor     | Langchain Information Extractor        | AI verifies and corrects artist/song info | get data                    | Get Lyrics                 |                                                                                                                     |
| OpenAI Chat Model         | Langchain OpenAI Chat Model             | Provides AI model backend for extraction  | Information Extractor (ai_languageModel) | Information Extractor (ai_languageModel) |                                                                                                                     |
| Get Lyrics                | HTTP Request                           | Fetches lyrics from external API          | Information Extractor       | Populate Doc               |                                                                                                                     |
| Populate Doc              | Google Docs                             | Inserts lyrics and song info into doc     | Get Lyrics                  | Search for Song            |                                                                                                                     |
| Search for Song           | Spotify                                | Searches song on Spotify                   | Populate Doc                | Add Song to Playlist       |                                                                                                                     |
| Add Song to Playlist      | Spotify                                | Adds found song to Spotify playlist       | Search for Song             | None                      |                                                                                                                     |
| Sticky Note               | Sticky Note                            | Describes workflow purpose and steps      | None                        | None                      | ## Setlist Manager\nThis workflow takes a Google spreadsheet called 'Setlist_Manager' with 'Artist' and 'SongTitle' entries and get's Lyrics for each song and creates a playlist for that set of songs.\n\n1. Create Spotify Playlist (naming it 'Setlist - [date of today]')\n2. Create the Google doc that will store the lyrics found. (naming it 'Setlist - [date of today]')\n3. Get the rows of songs from 'Setlist_Manager'.\n4. Use AI to verify the Artist name and song title.\n5. Get the lyrics to the song.\n6. Append the Google Doc with the lyrics.\n7. Search for the song in Spotify.\n8. Add that song to the Spotify Playlist.\n9. Go to band practice and be prepared! =) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Entry point to run the workflow manually.

2. **Add Spotify Node: Create Playlist**  
   - Operation: Create playlist  
   - Name: `"Setlist - {{ $now.format('yyyy-MM-dd') }}"` (dynamic date)  
   - Credentials: Configure Spotify OAuth2 with appropriate scopes (playlist-modify).  
   - Connect input from Manual Trigger.

3. **Add Google Docs Node: Create Doc**  
   - Operation: Create document  
   - Title: `"Setlist - {{ $now.format('yyyy-MM-dd') }}"`  
   - Folder ID: Set to desired folder for storing docs.  
   - Credentials: Configure Google Docs OAuth2.  
   - Connect input from "Create Playlist" node.

4. **Add Google Sheets Node: get data**  
   - Operation: Read rows  
   - Document ID: Use the Google Sheet ID of "Setlist_Manager"  
   - Sheet Name: Default to first sheet (gid=0)  
   - Credentials: Configure Google Sheets OAuth2.  
   - Connect input from "Create Doc" node.

5. **Add Langchain Information Extractor Node**  
   - Input Text Template:  
     ```
     You will be given an artist name and a song title. You'll need to verify the spelling and accuracy of the information.

     artist: {{ $json.Artist }}
     songTitle: {{ $json.SongTitle }}
     ```  
   - System Prompt:  
     ```
     You are an expert extraction algorithm.
     Only extract relevant information from the text.
     If you do not know the value of an attribute asked to extract, you may omit the attribute's value.
     ```  
   - Attributes to Extract:  
     - Artist (required)  
     - SongTitle (required)  
   - Connect input from "get data" node.

6. **Add Langchain OpenAI Chat Model Node**  
   - Model: GPT-4o-mini  
   - Credentials: Configure OpenAI API key.  
   - Connect as language model backend for "Information Extractor" node.

7. **Add HTTP Request Node: Get Lyrics**  
   - Method: GET  
   - URL:  
     ```
     https://api.lyrics.ovh/v1/{{ $json.output.Artist }}/{{ $json.output.SongTitle }}
     ```  
   - On Error: Continue workflow (do not fail on missing lyrics).  
   - Connect input from "Information Extractor" node.

8. **Add Google Docs Node: Populate Doc**  
   - Operation: Update document  
   - Document URL: Use expression to get ID from "Create Doc" node output.  
   - Actions:  
     - Insert text:  
       ```
       {{ $json.output.SongTitle }} - {{ $json.output.Artist }}

       {{ $json.lyrics }}
       ```  
     - Insert page break after each song.  
   - Credentials: Google Docs OAuth2.  
   - Connect input from "Get Lyrics" node.

9. **Add Spotify Node: Search for Song**  
   - Operation: Search track  
   - Query: Format as  
     ```
     {{ $json.Artist }} by {{ $json.SongTitle }}
     ```  
     (Note: currently uses original data, consider updating to AI-verified data if modifying.)  
   - Limit: 1  
   - Credentials: Spotify OAuth2.  
   - Connect input from "Populate Doc" node.

10. **Add Spotify Node: Add Song to Playlist**  
    - Operation: Add track to playlist  
    - Playlist ID: Use URI from "Create Playlist" node output.  
    - Track ID: Use track URI from "Search for Song" node output.  
    - Position: 0 (add at top).  
    - Credentials: Spotify OAuth2.  
    - Connect input from "Search for Song" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                        |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| The workflow automates setlist creation for bands, integrating Google Sheets, Docs, Spotify, and AI for lyrics. | General workflow purpose                                             |
| Spotify OAuth2 credentials must have playlist modification scope enabled.                                      | Spotify OAuth2 setup requirements                                    |
| Google OAuth2 credentials require access to Sheets and Docs APIs.                                              | Google API credential requirements                                   |
| Lyrics are fetched from https://lyrics.ovh/, which may have limited coverage or occasional downtime.          | External lyrics API used                                              |
| AI verification uses Langchain nodes with OpenAI GPT-4o-mini model for improved accuracy of artist/song names. | AI component detail                                                  |
| Error handling is set to continue on missing lyrics to avoid workflow interruption.                            | Workflow robustness note                                             |

---

This completes the detailed reference for the "Create Song Lyric Documents and Spotify Playlists for Singers with Google Docs" workflow. It enables reproduction, modification, and troubleshooting with a clear understanding of data flow, integration points, and potential failure modes.

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow designed with n8n, a tool for integration and automation. The process fully complies with applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.