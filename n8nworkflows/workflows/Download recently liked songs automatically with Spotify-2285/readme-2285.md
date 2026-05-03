Download recently liked songs automatically with Spotify

https://n8nworkflows.xyz/workflows/download-recently-liked-songs-automatically-with-spotify-2285


# Download recently liked songs automatically with Spotify

### 1. Workflow Overview

This workflow automates the management of Spotify playlists to enable offline listening of a limited number of recently liked songs in high quality, optimizing device storage usage.

It consists of the following logical blocks:

- **1.1 Trigger and Global Setup:** Defines the periodic trigger for the workflow execution and sets global parameters like the maximum number of songs to keep.

- **1.2 Playlist Existence Validation:** Retrieves all user playlists, checks if a playlist named "Downloads" exists, and creates it if missing.

- **1.3 Fetch and Filter Playlists:** Identifies the "Downloads" playlist from all playlists for further operations.

- **1.4 Retrieve Liked Tracks:** Gets the user's recently liked tracks from their Spotify library, limited by the defined global parameter.

- **1.5 Filter New Tracks:** Compares liked tracks with those already in the Downloads playlist to identify new tracks to add.

- **1.6 Add New Tracks to Downloads:** Adds the newly liked tracks to the Downloads playlist in batches.

- **1.7 Remove Old Tracks from Downloads:** Removes tracks from the Downloads playlist that are no longer liked to maintain freshness and comply with the limit.

Each block is connected to ensure seamless synchronization between liked songs and the Downloads playlist, enabling automatic offline availability of the latest favorites.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Global Setup

- **Overview:**  
  This block initiates the workflow on a scheduled interval and sets a global variable defining the maximum number of songs to be kept in the Downloads playlist.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Globals  
  - Sticky Note1 (explaining download_limit)  
  - Sticky Note2 (explaining scheduling)

- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow at defined intervals (default: once daily).  
    - *Configuration:* Uses default interval settings (daily).  
    - *Connections:* Output to Globals node.  
    - *Edge Cases:* Misconfiguration could cause unexpected run frequency or no runs.

  - **Globals**  
    - *Type:* Set  
    - *Role:* Defines global variables used throughout the workflow.  
    - *Configuration:* Sets `download_limit` to 50 (maximum number of songs).  
    - *Connections:* Output to Get all Playlists node.  
    - *Edge Cases:* Setting the limit above 50 may cause Spotify API payload errors due to node limitations.  
    - *Sticky Note:* Explains the purpose and max limit of `download_limit`.

  - **Sticky Notes 1 & 2**  
    - Provide contextual information on setting the download limit and scheduling.

---

#### 1.2 Playlist Existence Validation

- **Overview:**  
  Retrieves all user playlists and checks if a playlist named "Downloads" exists; if not, it creates the playlist.

- **Nodes Involved:**  
  - Get all Playlists  
  - If no playlist (checks if playlist list is empty)  
  - If no Downloads Playlist found (checks for absence of "Downloads" playlist)  
  - Create Downloads Playlist  
  - Aggregate  
  - Split Out  
  - Sticky Note3 (explains playlist retrieval and creation)

- **Node Details:**

  - **Get all Playlists**  
    - *Type:* Spotify  
    - *Role:* Fetches all user playlists.  
    - *Configuration:* Retrieves all playlists with `returnAll` enabled.  
    - *Connections:* Output to If no playlist node.  
    - *Edge Cases:* API rate limits or authentication failure.

  - **If no playlist**  
    - *Type:* If  
    - *Role:* Checks if the playlist list is empty (no playlists found).  
    - *Configuration:* Condition tests if the input JSON is empty.  
    - *Connections:* True branch to Create Downloads Playlist; False branch to Aggregate.  
    - *Edge Cases:* Expression failure if input malformed.

  - **If no Downloads Playlist found**  
    - *Type:* If  
    - *Role:* Tests if "Downloads" playlist exists in the fetched playlists.  
    - *Configuration:* Checks that the playlists array does not contain a playlist named "Downloads".  
    - *Connections:* True branch to Create Downloads Playlist; False branch to Split Out.  
    - *Edge Cases:* Could fail if playlist names change or if name case sensitivity differs.

  - **Create Downloads Playlist**  
    - *Type:* Spotify  
    - *Role:* Creates a new playlist named "Downloads".  
    - *Configuration:* Operation set to create playlist with name "Downloads".  
    - *Connections:* Outputs to Get all Playlists node (refresh playlists).  
    - *Credentials:* Requires Spotify OAuth2 credentials.  
    - *Edge Cases:* API quota limits, failure to create due to permissions.

  - **Aggregate**  
    - *Type:* Aggregate  
    - *Role:* Aggregates playlist names and URIs for filtering.  
    - *Configuration:* Aggregates fields `name` and `uri`.  
    - *Connections:* Output to If no Downloads Playlist found.  
    - *Edge Cases:* Empty aggregation if no playlists.

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Extracts playlists matching "Downloads" for further use.  
    - *Configuration:* Splits out fields `name` and `uri`.  
    - *Connections:* Output to Filter out Downloads Playlist node.  
    - *Edge Cases:* Misconfigured field names.

  - **Sticky Note3**  
    - Describes the logic of searching for or creating the Downloads playlist.

---

#### 1.3 Fetch and Filter Playlists

- **Overview:**  
  Isolates the "Downloads" playlist from all playlists for subsequent operations.

- **Nodes Involved:**  
  - Filter out Downloads Playlist  
  - Get Downloads Playlist  
  - Sticky Note4 (explains fetching latest liked songs and comparison)

- **Node Details:**

  - **Filter out Downloads Playlist**  
    - *Type:* Filter  
    - *Role:* Filters playlists to find the one named "Downloads".  
    - *Configuration:* Condition checks playlist `name` equals "Downloads".  
    - *Connections:* Output to Get Downloads Playlist.  
    - *Edge Cases:* Case sensitivity; multiple playlists with same name.

  - **Get Downloads Playlist**  
    - *Type:* Spotify  
    - *Role:* Retrieves detailed information about the "Downloads" playlist.  
    - *Configuration:* Operation `get` using playlist URI from previous node.  
    - *Connections:* Output to Get Liked Tracks node.  
    - *Credentials:* Spotify OAuth2 required.  
    - *Edge Cases:* Playlist deleted externally, permission issues.

---

#### 1.4 Retrieve Liked Tracks

- **Overview:**  
  Retrieves the latest liked tracks up to the download limit defined in globals.

- **Nodes Involved:**  
  - Get Liked Tracks  
  - Sticky Note4

- **Node Details:**

  - **Get Liked Tracks**  
    - *Type:* Spotify  
    - *Role:* Fetches user's liked songs from Spotify library.  
    - *Configuration:* Limit set dynamically from `download_limit` global variable (max 50).  
    - *Connections:* Output to Filter out new tracks node.  
    - *Credentials:* Spotify OAuth2 required.  
    - *Edge Cases:* API limit of 50 tracks; no batching implemented currently.

---

#### 1.5 Filter New Tracks

- **Overview:**  
  Determines which liked tracks are not yet in the Downloads playlist to add only new ones.

- **Nodes Involved:**  
  - Filter out new tracks (Code node)  
  - Loop Over Items  
  - Sticky Note5 (explains adding/removing tracks)

- **Node Details:**

  - **Filter out new tracks**  
    - *Type:* Code (JavaScript)  
    - *Role:* Compares URIs of liked songs with those in Downloads playlist, filtering out duplicates.  
    - *Configuration:* Uses JavaScript to build URI lists and filter accordingly, reversing the result to prioritize newest.  
    - *Connections:* Output to Loop Over Items node.  
    - *Edge Cases:* Empty liked list or downloads list; expression errors.

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Processes tracks one by one or in batches (default batch size).  
    - *Connections:* First output to Get Updated Downloads Playlist; second output to Add tracks to Downloads.  
    - *Edge Cases:* Large batch sizes could hit API limits.

---

#### 1.6 Add New Tracks to Downloads

- **Overview:**  
  Adds filtered new liked tracks to the Downloads playlist.

- **Nodes Involved:**  
  - Add tracks to Downloads  
  - Get Updated Downloads Playlist  
  - Sticky Note5

- **Node Details:**

  - **Add tracks to Downloads**  
    - *Type:* Spotify  
    - *Role:* Adds a track to the Downloads playlist at position 0 (top).  
    - *Configuration:* Uses playlist URI from filtered Downloads playlist; track URI from liked tracks.  
    - *Connections:* Output to Loop Over Items (to continue batch).  
    - *Credentials:* Spotify OAuth2 required.  
    - *Edge Cases:* API rate limits, invalid track URIs.

  - **Get Updated Downloads Playlist**  
    - *Type:* Spotify  
    - *Role:* Retrieves the current state of the Downloads playlist after additions.  
    - *Configuration:* Operation `get` with playlist URI.  
    - *Connections:* Output to Get tracks to remove node.  
    - *Executions:* Set to execute once per batch.  
    - *Edge Cases:* Sync delays in Spotify.

---

#### 1.7 Remove Old Tracks from Downloads

- **Overview:**  
  Removes tracks from the Downloads playlist that are no longer in the liked songs collection, maintaining playlist freshness and respecting the download limit.

- **Nodes Involved:**  
  - Get tracks to remove (Code node)  
  - Remove oldest tracks from Downloads  
  - Sticky Note5

- **Node Details:**

  - **Get tracks to remove**  
    - *Type:* Code (JavaScript)  
    - *Role:* Compares current Downloads playlist tracks with liked tracks and identifies tracks to remove.  
    - *Configuration:* Uses JavaScript to build URI lists and filter accordingly.  
    - *Connections:* Output to Remove oldest tracks from Downloads.  
    - *Execute Once:* Yes (to avoid repeated runs).  
    - *Edge Cases:* Empty inputs, possible race conditions.

  - **Remove oldest tracks from Downloads**  
    - *Type:* Spotify  
    - *Role:* Removes specified tracks from the Downloads playlist.  
    - *Configuration:* Uses playlist URI and track URI to delete tracks.  
    - *Connections:* None (end of removal process).  
    - *Credentials:* Spotify OAuth2 required.  
    - *Edge Cases:* Deletion failures if track no longer exists or permission issues.

---

### 3. Summary Table

| Node Name                     | Node Type                | Functional Role                         | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                          |
|-------------------------------|--------------------------|---------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Schedule Trigger               | Schedule Trigger         | Initiates workflow periodically       | -                                | Globals                           | Explains update interval setup                                                                     |
| Globals                       | Set                      | Sets global download_limit variable   | Schedule Trigger                 | Get all Playlists                 | Defines max 50 songs kept in Downloads playlist                                                    |
| Get all Playlists             | Spotify                  | Retrieves all user playlists           | Globals                         | If no playlist                   |                                                                                                    |
| If no playlist                | If                       | Checks if any playlists exist          | Get all Playlists               | Create Downloads Playlist (true), Aggregate (false) |                                                                                                    |
| Aggregate                    | Aggregate                | Aggregates playlist names and URIs    | If no playlist (false)           | If no Downloads Playlist found   |                                                                                                    |
| If no Downloads Playlist found | If                       | Checks for existence of "Downloads" playlist | Aggregate                     | Create Downloads Playlist (true), Split Out (false) |                                                                                                    |
| Create Downloads Playlist     | Spotify                  | Creates "Downloads" playlist           | If no playlist (true), If no Downloads Playlist found (true) | Get all Playlists               |                                                                                                    |
| Split Out                    | Split Out                | Extracts "Downloads" playlist details  | If no Downloads Playlist found (false) | Filter out Downloads Playlist     |                                                                                                    |
| Filter out Downloads Playlist | Filter                   | Filters playlists to "Downloads"       | Split Out                      | Get Downloads Playlist            |                                                                                                    |
| Get Downloads Playlist        | Spotify                  | Gets detailed "Downloads" playlist info| Filter out Downloads Playlist   | Get Liked Tracks                 |                                                                                                    |
| Get Liked Tracks              | Spotify                  | Retrieves user's liked songs           | Get Downloads Playlist          | Filter out new tracks             |                                                                                                    |
| Filter out new tracks         | Code                     | Filters liked songs not in Downloads   | Get Liked Tracks                | Loop Over Items                  |                                                                                                    |
| Loop Over Items              | Split In Batches          | Processes tracks in batches             | Filter out new tracks           | Get Updated Downloads Playlist (1), Add tracks to Downloads (2) |                                                                                                    |
| Get Updated Downloads Playlist| Spotify                  | Gets updated Downloads playlist info   | Loop Over Items (1)             | Get tracks to remove              |                                                                                                    |
| Add tracks to Downloads       | Spotify                  | Adds tracks to Downloads playlist       | Loop Over Items (2)             | Loop Over Items                  |                                                                                                    |
| Get tracks to remove          | Code                     | Finds tracks to remove from Downloads   | Get Updated Downloads Playlist  | Remove oldest tracks from Downloads |                                                                                                    |
| Remove oldest tracks from Downloads | Spotify                  | Removes old tracks from Downloads playlist | Get tracks to remove           | -                               |                                                                                                    |
| Sticky Note1                 | Sticky Note              | Notes on download_limit                 | -                              | -                               | Defines max 50 songs in Downloads playlist                                                        |
| Sticky Note2                 | Sticky Note              | Notes on scheduling                     | -                              | -                               | Explains update interval                                                                          |
| Sticky Note3                 | Sticky Note              | Notes on playlist retrieval/creation   | -                              | -                               | Explains logic for getting or creating Downloads playlist                                         |
| Sticky Note4                 | Sticky Note              | Notes on liked songs and playlist comparison | -                            | -                               | Explains fetching liked songs and filtering for new ones                                         |
| Sticky Note5                 | Sticky Note              | Notes on adding/removing tracks         | -                              | -                               | Explains adding new tracks and removing old ones to maintain limit                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configuration: Default interval (once daily).  
   - Connect output to Globals node.

2. **Create Globals Set node**  
   - Type: Set  
   - Add a variable `download_limit` of type Number with value `50`.  
   - Connect output to "Get all Playlists" node.

3. **Create Get all Playlists node**  
   - Type: Spotify  
   - Operation: Get User Playlists  
   - Set `returnAll` to true.  
   - Configure with Spotify OAuth2 credentials.  
   - Connect output to "If no playlist" node.

4. **Create If no playlist node**  
   - Type: If  
   - Condition: `$json.isEmpty()` is true (checks if playlist list is empty).  
   - True output connects to "Create Downloads Playlist" node.  
   - False output connects to "Aggregate" node.

5. **Create Aggregate node**  
   - Type: Aggregate  
   - Aggregate fields: `name` and `uri`.  
   - Connect output to "If no Downloads Playlist found" node.

6. **Create If no Downloads Playlist found node**  
   - Type: If  
   - Condition: Check if aggregated playlists do NOT contain a playlist with name "Downloads" (`notContains` operator).  
   - True output connects to "Create Downloads Playlist".  
   - False output connects to "Split Out" node.

7. **Create Create Downloads Playlist node**  
   - Type: Spotify  
   - Operation: Create Playlist  
   - Name: "Downloads"  
   - Credentials: Spotify OAuth2.  
   - Connect output back to "Get all Playlists" node (to refresh playlists).

8. **Create Split Out node**  
   - Type: Split Out  
   - Fields to split: `name`, `uri`.  
   - Connect output to "Filter out Downloads Playlist" node.

9. **Create Filter out Downloads Playlist node**  
   - Type: Filter  
   - Condition: Playlist `name` equals "Downloads".  
   - Connect output to "Get Downloads Playlist" node.

10. **Create Get Downloads Playlist node**  
    - Type: Spotify  
    - Operation: Get Playlist  
    - Playlist ID: Use URI from filtered playlist.  
    - Credentials: Spotify OAuth2.  
    - Connect output to "Get Liked Tracks" node.

11. **Create Get Liked Tracks node**  
    - Type: Spotify  
    - Operation: Get Library (Liked Tracks)  
    - Limit: Use expression `={{ $('Globals').item.json.download_limit }}` to dynamically set limit.  
    - Credentials: Spotify OAuth2.  
    - Connect output to "Filter out new tracks" node.

12. **Create Filter out new tracks node (Code node)**  
    - Type: Code (JavaScript)  
    - Logic:  
      - Extract URIs from current Downloads playlist tracks.  
      - Filter liked tracks to only those not already in Downloads.  
      - Reverse the filtered list so newest tracks are first.  
    - Connect output to "Loop Over Items" node.

13. **Create Loop Over Items node (Split In Batches)**  
    - Type: Split In Batches  
    - Default batch size (can be left default).  
    - First output connects to "Get Updated Downloads Playlist" node.  
    - Second output connects to "Add tracks to Downloads" node.

14. **Create Get Updated Downloads Playlist node**  
    - Type: Spotify  
    - Operation: Get Playlist  
    - Playlist ID: Use Downloads playlist URI.  
    - Credentials: Spotify OAuth2.  
    - Set to execute once per batch.  
    - Connect output to "Get tracks to remove" node.

15. **Create Add tracks to Downloads node**  
    - Type: Spotify  
    - Operation: Add Track to Playlist  
    - Playlist ID: Use Downloads playlist URI.  
    - Track ID: Use current batch item’s track URI.  
    - Position: 0 (add to start).  
    - Credentials: Spotify OAuth2.  
    - Connect output back to Loop Over Items to continue batch processing.

16. **Create Get tracks to remove node (Code node)**  
    - Type: Code (JavaScript)  
    - Logic:  
      - Extract URIs of liked tracks.  
      - From current Downloads playlist tracks, find those not in liked tracks.  
    - Set to execute once.  
    - Connect output to "Remove oldest tracks from Downloads" node.

17. **Create Remove oldest tracks from Downloads node**  
    - Type: Spotify  
    - Operation: Delete Track from Playlist  
    - Playlist ID: Use Downloads playlist URI.  
    - Track ID: Use track URI from code node output.  
    - Credentials: Spotify OAuth2.

18. **Add Sticky Notes** at relevant points to provide user guidance and explanations (see sticky notes content in Summary Table).

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| This workflow supports a maximum of 50 songs due to Spotify API payload limits in the "Get liked songs" node. | Current limitation; implementing batching would allow more songs.                                                |
| After first run, enable automatic downloads on the "Downloads" playlist in the Spotify app to enable offline. | User setup instruction for offline playback.                                                                     |
| Newly created Spotify developer apps may take several minutes before becoming fully functional.                | Important for troubleshooting initial authorization and API access.                                              |
| Workflow automatically manages playlist freshness and storage optimization by only syncing "Downloads" playlist. | Workflow purpose and design rationale.                                                                            |
| For further customization or scaling, consider implementing batching for liked songs fetching and playlist updates. | Suggestion for advanced users to overcome current limitations.                                                   |

---

This documentation provides a complete and detailed understanding of the Spotify "Downloads" playlist synchronization workflow, enabling users and developers to replicate, modify, or troubleshoot it confidently.