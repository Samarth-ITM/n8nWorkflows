Automated Spotify Playlist Organizer - Sort and Queue Tracks by Popularity

https://n8nworkflows.xyz/workflows/automated-spotify-playlist-organizer---sort-and-queue-tracks-by-popularity-9888


# Automated Spotify Playlist Organizer - Sort and Queue Tracks by Popularity

### 1. Workflow Overview

This workflow, titled **"Automated Spotify Playlist Organizer - Sort and Queue Tracks by Popularity"**, automates the process of retrieving all Spotify playlists of a user, extracting and cleaning the playlists data, organizing the tracks by their popularity, and finally queuing the sorted tracks for playback. It is designed for Spotify users who want to efficiently manage and enjoy their music collections by automatically sorting tracks from a chosen playlist and adding them to the playback queue based on popularity.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Manual trigger to start the workflow.
- **1.2 Retrieve User Playlists**: Fetch all playlists of the authenticated Spotify user.
- **1.3 Clean and Deduplicate Playlists Data**: Process the raw playlists data to extract relevant fields and remove duplicates.
- **1.4 Retrieve Tracks from Target Playlist**: For a selected playlist, fetch all its tracks.
- **1.5 Reorganize Tracks by Popularity**: Deduplicate track entries and sort them ascendingly by popularity.
- **1.6 Queue Tracks for Playback**: Add the sorted tracks sequentially to the Spotify playback queue for listening.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow manually via a trigger node.
  
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’

- **Node Details:**  

  | Node Name                     | Details                                                                                          |
  |-------------------------------|------------------------------------------------------------------------------------------------|
  | When clicking ‘Execute workflow’ | Type: Manual Trigger<br>Role: Starts the workflow manually.<br>Config: Default manual trigger with no parameters.<br>Input: None<br>Output: Triggers next node.<br>Edge cases: None, except user must manually execute to start.|

---

#### 2.2 Retrieve User Playlists

- **Overview:**  
  This block fetches all playlists associated with the authenticated Spotify user.
  
- **Nodes Involved:**  
  - Get a user's playlists  
  - Clean & Deduplicate  
  - Sticky Note (explaining this step)

- **Node Details:**  

  | Node Name             | Details                                                                                                    |
  |-----------------------|------------------------------------------------------------------------------------------------------------|
  | Get a user's playlists | Type: Spotify API Node<br>Role: Retrieves all user playlists.<br>Config: Operation `getUserPlaylists`, returns all playlists without pagination.<br>Input: Trigger node output.<br>Output: Raw playlists data.<br>Requires Spotify OAuth2 credentials.<br>Edge cases: Auth failures, API rate limits, empty playlist collection.<br>Sticky Note: "This node retrieve all your playlist" |
  | Clean & Deduplicate    | Type: Code Node<br>Role: Processes raw playlists data to extract only relevant fields (`name`, `uri`, `total`), filters incomplete entries, and removes duplicates based on URI.<br>Input: Output from Spotify playlists node.<br>Output: Cleaned array of unique playlists.<br>Key Logic: Handles two input formats (array inside first item or multiple playlist items).<br>Edge cases: No playlists returned, malformed data.<br>Sticky Note1: "This node clean and deduplicate the previous output. It displays a list of all your playlists with their ids." |

---

#### 2.3 Retrieve Tracks from Target Playlist

- **Overview:**  
  This block fetches detailed track information for a selected playlist, identified by its URI from previous output.
  
- **Nodes Involved:**  
  - Get a playlist's tracks by URI or ID  
  - Sticky Note (explaining this step)

- **Node Details:**  

  | Node Name                     | Details                                                                                                 |
  |-------------------------------|---------------------------------------------------------------------------------------------------------|
  | Get a playlist's tracks by URI or ID | Type: Spotify API Node<br>Role: Retrieves all tracks from a given playlist using its URI.<br>Config: Uses playlist URI from `Clean & Deduplicate` node; operation `get` on playlist resource.<br>Input: Cleaned playlist array.<br>Output: Playlist tracks data.<br>Requires Spotify OAuth2 credentials.<br>Edge cases: Invalid URI, empty playlist, API errors.<br>Sticky Note2: "This node extract all the song" |

---

#### 2.4 Reorganize Tracks by Popularity

- **Overview:**  
  Processes the playlist tracks to remove duplicates, enrich track info, and sort tracks by their popularity score in ascending order.
  
- **Nodes Involved:**  
  - Playlist reorganizer  
  - Sticky Note (explaining this step)

- **Node Details:**  

  | Node Name           | Details                                                                                                            |
  |---------------------|--------------------------------------------------------------------------------------------------------------------|
  | Playlist reorganizer | Type: Code Node<br>Role: Iterates through tracks of the selected playlist, removes duplicate tracks, and sorts by popularity ascending.<br>Inputs: Playlist tracks data.<br>Outputs: Sorted list of tracks with detailed metadata (`id`, `name`, `artists`, `album`, `popularity`, `added_at`, `url`).<br>Key Logic: Uses sets to track processed playlists and track IDs to avoid duplicates.<br>Edge cases: Tracks without IDs (local or unavailable), missing popularity score (treated as -1).<br>Sticky Note3: "This node reorganize your playlist sorting by popularity" |

---

#### 2.5 Queue Tracks for Playback

- **Overview:**  
  Loops over the sorted tracks to add each song to the Spotify playback queue, enabling the user to listen in order.
  
- **Nodes Involved:**  
  - Loop Over Items  
  - Add a song to a queue  
  - Sticky Note (explaining this step)

- **Node Details:**  

  | Node Name          | Details                                                                                                   |
  |--------------------|-----------------------------------------------------------------------------------------------------------|
  | Loop Over Items    | Type: SplitInBatches Node<br>Role: Iterates over each sorted track one-by-one to process them sequentially.<br>Input: Sorted tracks from `Playlist reorganizer`.<br>Output: Sends individual track data to next node.<br>Edge cases: Large playlists may cause rate limit delays. |
  | Add a song to a queue | Type: Spotify API Node<br>Role: Adds a specified track to the user's playback queue.<br>Config: Uses track ID from incoming JSON to form Spotify URI (`spotify:track:<id>`).<br>Input: Single track item.<br>Output: Confirmation of queued track.<br>Requires Spotify OAuth2 credentials.<br>Edge cases: Playback device not active, track unavailable, API errors.<br>Sticky Note4: "This loop add songs in queue so you can enjoy it for your evening :)" |

---

### 3. Summary Table

| Node Name                     | Node Type            | Functional Role                          | Input Node(s)                      | Output Node(s)                  | Sticky Note                                                                                   |
|-------------------------------|----------------------|----------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Initiate workflow                      | None                             | Get a user's playlists          |                                                                                              |
| Get a user's playlists         | Spotify Node         | Retrieve all user playlists            | When clicking ‘Execute workflow’ | Clean & Deduplicate             | This node retrieve all your playlist                                                        |
| Clean & Deduplicate            | Code Node            | Clean and deduplicate playlists data  | Get a user's playlists           | Get a playlist's tracks by URI or ID | This node clean and deduplicate the previous output. It displays a list of all your playlists with their ids |
| Get a playlist's tracks by URI or ID | Spotify Node         | Fetch tracks from selected playlist    | Clean & Deduplicate              | Playlist reorganizer            | This node extract all the song                                                              |
| Playlist reorganizer           | Code Node            | Deduplicate and sort tracks by popularity | Get a playlist's tracks by URI or ID | Loop Over Items                | This node reorganize your playlist sorting by popularity                                   |
| Loop Over Items               | SplitInBatches Node  | Iterate over sorted tracks             | Playlist reorganizer             | Add a song to a queue           |                                                                                              |
| Add a song to a queue          | Spotify Node         | Add each track to Spotify playback queue | Loop Over Items                 | Loop Over Items                 | This loop add songs in queue so you can enjoy it for your evening :)                        |
| Sticky Note                   | Sticky Note          | Annotation                            | None                             | None                          | ## This node retrieve all your playlist                                                     |
| Sticky Note1                  | Sticky Note          | Annotation                            | None                             | None                          | ## This node clean and deduplicate the previous output. It displays a list of all your playlists with their ids |
| Sticky Note2                  | Sticky Note          | Annotation                            | None                             | None                          | ## This node extract all the song                                                           |
| Sticky Note3                  | Sticky Note          | Annotation                            | None                             | None                          | ## This node reorganize your playlist sorting by popularity                                |
| Sticky Note4                  | Sticky Note          | Annotation                            | None                             | None                          | ## This loop add songs in queue so you can enjoy it for your evening :)                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**
   - Type: Manual Trigger
   - Name: `When clicking ‘Execute workflow’`
   - No parameters needed.
   
2. **Add Spotify Node to Get User Playlists**
   - Type: Spotify
   - Name: `Get a user's playlists`
   - Credentials: Configure Spotify OAuth2 credentials.
   - Parameters:
     - Resource: Playlist
     - Operation: getUserPlaylists
     - Return All: Enabled (true)
   - Connect input from Manual Trigger node.

3. **Add Code Node to Clean & Deduplicate Playlists**
   - Type: Code
   - Name: `Clean & Deduplicate`
   - JavaScript Code:
     ```js
     const source = Array.isArray(items[0]?.json)
       ? items[0].json
       : items.map(i => i.json);

     const minimal = source
       .map(p => ({
         name: p?.name,
         uri: p?.uri,
         total: p?.tracks?.total ?? null,
       }))
       .filter(p => p.name && p.uri);

     const seen = new Set();
     const dedup = minimal.filter(p => (
       seen.has(p.uri) ? false : (seen.add(p.uri), true)
     ));

     return dedup.map(j => ({ json: j }));
     ```
   - Connect input from `Get a user's playlists`.

4. **Add Spotify Node to Get Playlist Tracks by URI/ID**
   - Type: Spotify
   - Name: `Get a playlist's tracks by URI or ID`
   - Credentials: Use existing Spotify OAuth2 credentials.
   - Parameters:
     - Resource: Playlist
     - Operation: get
     - ID: Expression `{{ $json.uri }}`
   - Connect input from `Clean & Deduplicate`.

5. **Add Code Node to Reorganize Playlist Tracks**
   - Type: Code
   - Name: `Playlist reorganizer`
   - JavaScript Code:
     ```js
     const targetId = items[0]?.json?.id;

     const processedPlaylists = new Set();
     const seenTrackIds = new Set();
     const out = [];

     for (const item of items) {
       const pl = item.json;
       if (!pl || pl.id !== targetId) continue;
       if (processedPlaylists.has(pl.id)) continue;
       processedPlaylists.add(pl.id);

       const tracks = pl?.tracks?.items ?? [];
       for (const t of tracks) {
         const tr = t?.track;
         const tid = tr?.id;
         if (!tid) continue;
         if (seenTrackIds.has(tid)) continue;
         seenTrackIds.add(tid);

         out.push({
           playlistId: pl.id,
           playlistName: pl.name,
           id: tid,
           name: tr?.name,
           popularity: tr?.popularity ?? -1,
           artists: (tr?.artists ?? []).map(a => a.name).join(", "),
           album: tr?.album?.name,
           added_at: t.added_at,
           url: tr?.external_urls?.spotify,
         });
       }
     }

     out.sort((a, b) => {
       const pa = a.popularity ?? -1;
       const pb = b.popularity ?? -1;
       if (pa === -1 && pb !== -1) return 1;
       if (pb === -1 && pa !== -1) return -1;
       return pa - pb;
     });

     return out.map(j => ({ json: j }));
     ```
   - Connect input from `Get a playlist's tracks by URI or ID`.

6. **Add SplitInBatches Node for Looping Over Tracks**
   - Type: SplitInBatches
   - Name: `Loop Over Items`
   - Parameters: Use default (batch size 1).
   - Connect input from `Playlist reorganizer`.

7. **Add Spotify Node to Add Song to Queue**
   - Type: Spotify
   - Name: `Add a song to a queue`
   - Credentials: Use Spotify OAuth2 credentials.
   - Parameters:
     - ID: Expression `spotify:track:{{ $json.id }}`
   - Connect input from `Loop Over Items` (second output).

8. **Connect Loop to Continue Processing**
   - Connect the output of `Add a song to a queue` back to the second output of `Loop Over Items` to continue batching until all tracks are queued.

9. **Save and Test Workflow**
   - Ensure Spotify OAuth2 credentials have correct scopes, including playlist-read and playback-modify permissions.
   - Execute workflow manually.
   - Observe the playlists fetched, cleaned, tracks sorted by popularity, and tracks added to queue sequentially.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                    |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Spotify OAuth2 credentials must have scopes: `playlist-read-private`, `playlist-read-collaborative`, and `user-modify-playback-state`. | Spotify Developer Dashboard OAuth2 Setup          |
| Playback device must be active on user account for the queue addition to succeed (e.g., Spotify app open). | Spotify Playback API Documentation                 |
| This workflow assumes all playlists are owned or accessible by the authenticated user.              | Spotify API Documentation                          |
| Sorting tracks ascending by popularity means less popular songs play first, increasing to most popular. | Playlist strategy design                            |

---

**Disclaimer:**  
The text provided derives exclusively from an automated n8n workflow. All data processed is legal and publicly accessible. The workflow respects current content and usage policies.