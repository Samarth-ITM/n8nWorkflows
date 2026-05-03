Dynamic Media Library with On-demand Downloads for Radarr/Sonarr and Plex

https://n8nworkflows.xyz/workflows/dynamic-media-library-with-on-demand-downloads-for-radarr-sonarr-and-plex-6980


# Dynamic Media Library with On-demand Downloads for Radarr/Sonarr and Plex

---

### 1. Workflow Overview

This workflow, titled **"Dynamic Media Library with On-demand Downloads for Radarr/Sonarr and Plex"** (also called "n8n Placeholdarr for Plex"), automates the management of media libraries integrated with Radarr, Sonarr, and Plex. Its main purpose is to create **dummy placeholder media files** for movies and TV shows tagged as `dummy-unprocessed` in Radarr/Sonarr, allowing Plex to display these items without requiring the actual media files to be downloaded initially.

Key functionalities include:

- Detecting new or updated items in Radarr/Sonarr tagged for dummy file creation.
- Creating dummy media files on the server filesystem using FFmpeg via SSH.
- Monitoring playback via Tautulli and triggering real media downloads when dummy files are played.
- Managing Plex collections populated from external sources (JustWatch and Trakt).
- Automated refreshing of Plex libraries and *Arrs monitoring states.
- Removing dummy tags and files when real content is available.
- Handling user requests through Overseerr integration.
- Managing collections dynamically, including adding/removing items based on external lists.

The workflow is logically divided into these blocks:

- **1.1 Webhook Input & External List Handling:** Receives webhooks from *Arrs apps, Tautulli, and external list triggers (JustWatch, Trakt).
- **1.2 Dummy File Creation & Media Monitoring:** Creates dummy files and manages *Arrs monitoring states for movies and series.
- **1.3 Plex Library and Collection Management:** Updates Plex collections based on external list data, including item lookup and collection refresh.
- **1.4 Tautulli Playback Monitoring & Overseerr Requests:** Monitors playback of dummy files and triggers real downloads using Overseerr API.
- **1.5 Utility and Cleanup:** Removes dummy tags/files after processing and handles session termination for dummy playback.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Webhook Input & External List Handling

**Overview:**  
This block handles incoming webhooks from multiple sources: *Arrs applications (Radarr/Sonarr) for dummy file triggers, Tautulli for playback starts, and webhook endpoints for external collection lists from JustWatch and Trakt. It normalizes and formats data for downstream processing.

**Nodes Involved:**  
- Webhook Arrs Dummy file update (webhook)  
- Switch eventType (switch)  
- If Radarr (if)  
- If tag unmonitored exists (if)  
- Remove unmonitored tag (httpRequest)  
- If tag unmonitored exists Sonarr (if)  
- Remove unmonitored tag Sonarr (httpRequest)  
- Webhook Tautulli (webhook)  
- If movie4 (if)  
- Get Radarr information from tmdbId (httpRequest)  
- Get Sonarr information from tvdbId (httpRequest)  
- Webhook Arrs custom list JustWatch (webhook)  
- Call JustWatch API (httpRequest)  
- Format JustWatch results (code)  
- If objectType is movie (if)  
- Respond to Webhook (respondToWebhook)  
- Webhook Arrs custom list Trakt (webhook)  
- Call Trakt API (httpRequest)  
- Format JustWatch results1 (code)  
- Respond to Webhook2 (respondToWebhook)  
- Sonarr serie lookup1 (httpRequest)  
- Format JustWatch results add tvdbId (code)  
- Respond to Webhook1 (respondToWebhook)  
- Check if collection is not empty (if)  
- Merge1 (merge)  
- Set fields for collection (set)  
- Wait 5 minutes before process collection in Plex (wait)

**Node Details:**

- **Webhook Arrs Dummy file update (webhook):**  
  - Entry point for dummy file triggers from Radarr/Sonarr.  
  - Listens to POST requests on a specific path.  
  - Outputs JSON body with eventType and media details.

- **Switch eventType (switch):**  
  - Routes workflow based on `eventType` field in webhook JSON (`Download`, `MovieAdded`, `SeriesAdd`).  
  - Directs to Radarr or Sonarr processing or monitoring nodes.

- **If Radarr (if):**  
  - Checks if the webhook JSON contains a Radarr movie object to separate Radarr from Sonarr workflows.

- **If tag unmonitored exists / Remove unmonitored tag (Radarr):**  
  - Checks if the `dummy-unprocessed` tag exists on the movie.  
  - If yes, removes the tag to indicate processing started.

- **If tag unmonitored exists Sonarr / Remove unmonitored tag Sonarr:**  
  - Similar logic for Sonarr series.

- **Webhook Tautulli (webhook):**  
  - Receives playback start events from Tautulli for dummy files.  
  - Triggers Overseerr request logic downstream.

- **If movie4:**  
  - Checks if the media type received from Tautulli webhook is a movie.

- **Get Radarr information from tmdbId / Get Sonarr information from tvdbId:**  
  - Fetches detailed media info from Radarr/Sonarr API based on TMDB or TVDB IDs.

- **Webhook Arrs custom list JustWatch / Call JustWatch API / Format JustWatch results:**  
  - Receives webhook parameters for JustWatch list queries.  
  - Calls JustWatch GraphQL API with filters and limits.  
  - Formats results extracting TMDB and IMDB IDs.

- **If objectType is movie / Respond to Webhook:**  
  - Branches for movie object type to respond with formatted JustWatch data.

- **Webhook Arrs custom list Trakt / Call Trakt API / Format JustWatch results1 / Respond to Webhook2:**  
  - Similar to JustWatch but for Trakt user lists.  
  - Extracts TMDB and IMDB IDs from Trakt responses.

- **Sonarr serie lookup1 / Format JustWatch results add tvdbId / Respond to Webhook1:**  
  - Performs Sonarr series lookup by IMDB ID to get TVDB IDs.  
  - Formats output adding TVDB IDs for downstream use.

- **Check if collection is not empty / Merge1 / Set fields for collection / Wait 5 minutes before process collection in Plex:**  
  - Validates that the collectionId parameter is present and data is not empty.  
  - Prepares and sets collection data with parameters for later Plex processing.  
  - Wait node delays processing 5 minutes to allow *Arrs to finish import operations.

**Edge Cases / Failure Points:**  
- Missing or malformed webhook JSON input.  
- API rate limiting or failures from JustWatch, Trakt, Radarr, Sonarr.  
- Missing tags or unexpected event types.  
- Network or auth errors on HTTP Requests.  
- Timeout on Wait node if upstream processes are slow.

---

#### 1.2 Dummy File Creation & Media Monitoring

**Overview:**  
This block creates dummy media files on the media server for movies and series tagged as `dummy-unprocessed`. It uses SSH to run FFmpeg commands and manages *Arrs monitoring states to trigger real downloads once playback starts.

**Nodes Involved:**  
- Radarr movie (httpRequest)  
- Sonarr series (httpRequest)  
- Create dummy file (ssh)  
- Create dummy file for movie1 (ssh)  
- Create dummy file for series with runtime (ssh)  
- Refresh movie1 (httpRequest)  
- Refresh Plex (httpRequest)  
- Refresh series (httpRequest)  
- Refresh Plex Series (httpRequest)  
- Monitor movie (httpRequest)  
- Search movie (httpRequest)  
- Monitor series (httpRequest)  
- Monitor all seasons (httpRequest)  
- Monitor all seasons1 (httpRequest)  
- Search season pack (httpRequest)  
- Search season pack1 (httpRequest)  
- Search series2 (httpRequest)  
- Search series season (httpRequest)  
- If deletedFiles exist (if)  
- Remove all dummy files (ssh)  
- Remove unmonitored tag (httpRequest)  
- Remove unmonitored tag Sonarr (httpRequest)  
- If tag unmonitored exists / If tag unmonitored exists Sonarr (if)  
- If Radarr (if)

**Node Details:**

- **Radarr movie / Sonarr series:**  
  - Queries the Radarr or Sonarr API for detailed info about the media item to be processed.

- **Create dummy file / Create dummy file for movie1 / Create dummy file for series with runtime:**  
  - SSH nodes executing FFmpeg commands to create dummy video files with length based on runtime or fixed length.  
  - Handles file permissions and ownership for proper access.  
  - Uses a base dummy file or generates a dummy video stream with overlay.

- **Refresh movie1 / Refresh Plex / Refresh series / Refresh Plex Series:**  
  - Triggers Radarr/Sonarr commands to refresh metadata.  
  - Calls Plex API to refresh library sections for movies or series to detect dummy files.

- **Monitor movie / Monitor series / Monitor all seasons / Monitor all seasons1:**  
  - Updates *Arrs monitoring flags to true to enable downloads when dummy playback occurs.

- **Search movie / Search season pack / Search series2 / Search season pack1 / Search series season:**  
  - Initiates search commands in *Arrs to start actual download processes.

- **If deletedFiles exist:**  
  - Conditional check for existence of deleted dummy files to trigger cleanup or further actions.

- **Remove all dummy files:**  
  - SSH command to delete dummy files from series folders.

- **If tag unmonitored exists / Remove unmonitored tag / Remove unmonitored tag Sonarr:**  
  - Conditional check and removal of dummy tags after processing.

**Edge Cases / Failure Points:**  
- SSH connection failures or unavailable FFmpeg binaries.  
- Permissions errors creating or deleting dummy files.  
- API failures or authentication errors with Radarr, Sonarr, Plex.  
- Race conditions if dummy files are deleted while being played.  
- FFmpeg command failures due to invalid paths or system issues.

---

#### 1.3 Plex Library and Collection Management

**Overview:**  
This block manages Plex collections dynamically by fetching items from Plex collections, adding/removing items based on import lists from JustWatch/Trakt, and ensuring Plex library updates accordingly.

**Nodes Involved:**  
- Get collection1 (httpRequest)  
- Get all items in collection1 (httpRequest)  
- Check if collection contains items1 (if)  
- Split Out Collection Items1 (splitOut)  
- Loop Over Items1 (splitInBatches)  
- Remove item from collection1 (httpRequest)  
- Loop Over Items3 (splitInBatches)  
- Get GUID from imdbId (httpRequest)  
- If found Plex GUID1 (if)  
- Get ratingKey from GUID1 (httpRequest)  
- Add item to collection3 (httpRequest)  
- Add item to collection2 (httpRequest)  
- Move collection item2 (httpRequest)  
- Merge, Merge2, Merge3 (merge)

**Node Details:**

- **Get collection1 / Get all items in collection1:**  
  - Fetches Plex collection metadata and list of items in the collection.

- **Check if collection contains items1:**  
  - Validates if the collection has any items to process.

- **Split Out Collection Items1 / Loop Over Items1 / Loop Over Items3:**  
  - Iterates over collection items in batches for processing.

- **Remove item from collection1:**  
  - Removes specific items from Plex collections based on conditions.

- **Get GUID from imdbId / If found Plex GUID1 / Get ratingKey from GUID1:**  
  - Matches external IMDB IDs to Plex GUIDs and rating keys for item identification.

- **Add item to collection3 / Add item to collection2 / Move collection item2:**  
  - Adds or moves items within Plex collections, handling API permissions and requests.

- **Merge nodes:**  
  - Combines multiple input branches to synchronize processing flows.

**Edge Cases / Failure Points:**  
- Plex API authentication errors or rate limits.  
- Missing or mismatched IMDB/Plex GUID data causing failures to add items.  
- Handling empty collections or items.  
- API responses with unexpected structures.

---

#### 1.4 Tautulli Playback Monitoring & Overseerr Requests

**Overview:**  
This block listens to Tautulli playback start events for dummy files, checks user information against Overseerr, and triggers Overseerr requests to start actual media downloads when dummy playback occurs.

**Nodes Involved:**  
- Webhook Tautulli (webhook)  
- Get Overseerr users (httpRequest)  
- Filter Overseerr user (code)  
- Check if Overseerr user is found (if)  
- Get last 20 Overseerr requests from user (httpRequest)  
- Check if request already exists (code)  
- Check if no existing request for media exist for Overseerr user (if)  
- Make Overseerr request (httpRequest)  
- Get active Tautulli sessions (httpRequest)  
- Get all active sessions for file (code)  
- Terminate Tautulli sessions for dummy (httpRequest)

**Node Details:**

- **Webhook Tautulli:**  
  - Entry point for playback start events from Tautulli.

- **Get Overseerr users / Filter Overseerr user:**  
  - Retrieves the list of Overseerr users and filters to find the matching user by email.

- **Check if Overseerr user is found:**  
  - Validates that the user was found before proceeding.

- **Get last 20 Overseerr requests from user / Check if request already exists / Check if no existing request for media exist for Overseerr user:**  
  - Checks if the media is already requested by the user to avoid duplicates.

- **Make Overseerr request:**  
  - Posts a new request to Overseerr to trigger media download.

- **Get active Tautulli sessions / Get all active sessions for file / Terminate Tautulli sessions for dummy:**  
  - Retrieves current playback sessions for the dummy file and terminates them when real media becomes available, notifying the user.

**Edge Cases / Failure Points:**  
- Overseerr API auth or network failures.  
- Multiple active sessions for the same dummy causing race conditions.  
- User email mismatch or missing user in Overseerr database.  
- Tautulli session termination failures.

---

#### 1.5 Utility and Cleanup

**Overview:**  
This block handles housekeeping tasks such as removing dummy tags, cleaning dummy files from disk after real media arrives, and ensuring Plex and *Arrs states remain consistent.

**Nodes Involved:**  
- Remove unmonitored tag (httpRequest)  
- Remove unmonitored tag Sonarr (httpRequest)  
- Remove all dummy files (ssh)  
- If deletedFiles exist (if)  
- Refresh movie1 (httpRequest)  
- Refresh series (httpRequest)  
- Refresh Plex (httpRequest)  
- Refresh Plex Series (httpRequest)

**Node Details:**

- **Remove unmonitored tag / Remove unmonitored tag Sonarr:**  
  - Removes dummy tags from Radarr/Sonarr items once real media is available.

- **Remove all dummy files:**  
  - Deletes dummy files from the series folder on the media server.

- **If deletedFiles exist:**  
  - Checks if dummy files were deleted to trigger cleanup and refresh logic.

- **Refresh nodes:**  
  - Refreshes *Arrs and Plex to update media library states.

**Edge Cases / Failure Points:**  
- SSH or filesystem permission issues deleting files.  
- Race conditions if cleanup triggers while playback is ongoing.  
- API failures in refreshing libraries.

---

### 3. Summary Table

| Node Name                           | Node Type              | Functional Role                                | Input Node(s)                            | Output Node(s)                                | Sticky Note                                                                                              |
|-----------------------------------|------------------------|-----------------------------------------------|----------------------------------------|----------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Webhook Arrs Dummy file update    | Webhook                | Entry point for *Arrs dummy file triggers     |                                        | Switch eventType                             |                                                                                                         |
| Switch eventType                  | Switch                 | Routes based on eventType from webhook        | Webhook Arrs Dummy file update         | If Radarr, Radarr movie, Sonarr series       |                                                                                                         |
| If Radarr                        | If                     | Checks presence of Radarr movie object        | Switch eventType                       | If tag unmonitored exists, If tag unmonitored exists Sonarr |                                                                                                         |
| If tag unmonitored exists         | If                     | Checks Radarr movie tags for dummy             | If Radarr                             | Remove unmonitored tag, If deletedFiles exist |                                                                                                         |
| Remove unmonitored tag            | HTTP Request           | Removes dummy tag from Radarr movie            | If tag unmonitored exists              | If deletedFiles exist                         |                                                                                                         |
| If tag unmonitored exists Sonarr  | If                     | Checks Sonarr series tags for dummy            | If Radarr                             | Remove unmonitored tag Sonarr, If deletedFiles exist |                                                                                                         |
| Remove unmonitored tag Sonarr     | HTTP Request           | Removes dummy tag from Sonarr series           | If tag unmonitored exists Sonarr       | If deletedFiles exist                         |                                                                                                         |
| Webhook Tautulli                 | Webhook                | Receives Tautulli playback start events       |                                        | If movie4                                    | Sticky Note3: Tautulli webhook setup instructions and sample payload                                   |
| If movie4                       | If                     | Checks if Tautulli event media type is movie  | Webhook Tautulli                      | Get Radarr information from tmdbId            |                                                                                                         |
| Get Radarr information from tmdbId | HTTP Request           | Gets Radarr movie info by TMDB ID              | If movie4                             | Monitor movie                                 |                                                                                                         |
| Get Sonarr information from tvdbId | HTTP Request           | Gets Sonarr series info by TVDB ID             | Switch eventType                     | Monitor series, Code1                         |                                                                                                         |
| Webhook Arrs custom list JustWatch | Webhook                | Receives JustWatch list parameters             |                                        | Call JustWatch API                            | Sticky Note: Arrs import list usage and webhook format instructions                                    |
| Call JustWatch API               | HTTP Request           | Fetches popular titles from JustWatch API     | Webhook Arrs custom list JustWatch     | Format JustWatch results                       |                                                                                                         |
| Format JustWatch results          | Code                   | Formats JustWatch response to IDs              | Call JustWatch API                    | If objectType is movie, Merge1                 |                                                                                                         |
| If objectType is movie            | If                     | Checks if objectType is MOVIE                   | Format JustWatch results              | Respond to Webhook                             |                                                                                                         |
| Respond to Webhook               | Respond to Webhook     | Sends JSON response for JustWatch movie list   | If objectType is movie                |                                              |                                                                                                         |
| Webhook Arrs custom list Trakt    | Webhook                | Receives Trakt list parameters                  |                                        | Call Trakt API                                |                                                                                                         |
| Call Trakt API                  | HTTP Request           | Fetches movie list from Trakt API               | Webhook Arrs custom list Trakt         | Format JustWatch results1                       |                                                                                                         |
| Format JustWatch results1         | Code                   | Formats Trakt API response                      | Call Trakt API                       | Respond to Webhook2, Merge1                     |                                                                                                         |
| Respond to Webhook2              | Respond to Webhook     | Sends JSON response for Trakt movie list       | Format JustWatch results1             |                                              |                                                                                                         |
| Sonarr serie lookup1             | HTTP Request           | Looks up Sonarr series by IMDB ID               | If objectType is movie                | Format JustWatch results add tvdbId             |                                                                                                         |
| Format JustWatch results add tvdbId | Code                   | Adds TVDB ID to formatted results               | Sonarr serie lookup1                 | Respond to Webhook1, Merge1                     |                                                                                                         |
| Respond to Webhook1              | Respond to Webhook     | Sends JSON response with added TVDB ID          | Format JustWatch results add tvdbId  |                                              |                                                                                                         |
| Check if collection is not empty | If                     | Verifies collectionId parameter is not empty   | Respond to Webhook, Respond to Webhook1 | Merge1                                      |                                                                                                         |
| Merge1                         | Merge                  | Merges inputs from JustWatch and Trakt branches| Check if collection is not empty     | Set fields for collection                       |                                                                                                         |
| Set fields for collection        | Set                    | Sets parameters and prepares data array         | Merge1                               | Wait 5 minutes before process collection in Plex |                                                                                                         |
| Wait 5 minutes before process collection in Plex | Wait                   | Delays execution to allow *Arrs to finish import | Set fields for collection             | Get collection1                               |                                                                                                         |
| Radarr movie                    | HTTP Request           | Gets Radarr movie info for dummy file creation  | Switch eventType                     | Create dummy file for movie1                    |                                                                                                         |
| Sonarr series                  | HTTP Request           | Gets Sonarr series info for dummy file creation | Switch eventType                     | Sonarr runtime s01e01                           |                                                                                                         |
| Create dummy file for movie1      | SSH                    | Creates dummy movie file using FFmpeg            | Radarr movie                        | Refresh movie1                                  |                                                                                                         |
| Create dummy file for series with runtime | SSH                    | Creates dummy series file using FFmpeg            | Sonarr runtime s01e01               | Refresh series1                                 |                                                                                                         |
| Refresh movie1                  | HTTP Request           | Refreshes Radarr movie metadata                   | Create dummy file for movie1         | Refresh Plex                                    |                                                                                                         |
| Refresh Plex                   | HTTP Request           | Refreshes Plex movie library section              | Refresh movie1                     |                                              |                                                                                                         |
| Refresh series                 | HTTP Request           | Refreshes Sonarr series metadata                   | Get Sonarr information for series   | Monitor all seasons1                             |                                                                                                         |
| Refresh Plex Series           | HTTP Request           | Refreshes Plex series library section              | Refresh series1                    |                                              |                                                                                                         |
| Monitor movie                 | HTTP Request           | Sets Radarr movie to monitored                      | Get Radarr information from tmdbId | Search movie                                    |                                                                                                         |
| Search movie                 | HTTP Request           | Initiates Radarr movie search/download              | Monitor movie                     | Respond 200                                     |                                                                                                         |
| Monitor series               | HTTP Request           | Sets Sonarr series to monitored                      | Get Sonarr information from tvdbId | Monitor all seasons                              |                                                                                                         |
| Monitor all seasons           | HTTP Request           | Sets monitoring for all seasons in Sonarr            | Monitor series                    | Merge3                                          |                                                                                                         |
| Monitor all seasons1          | HTTP Request           | Alternate node to set monitoring for seasons         | Refresh series                   | Search season pack1                              |                                                                                                         |
| Search season pack             | HTTP Request           | Triggers Sonarr season pack search/download           | Merge3                          | Respond                                          |                                                                                                         |
| Search season pack1            | HTTP Request           | Alternate season search node                          | Monitor all seasons1              |                                              |                                                                                                         |
| Search series2                | HTTP Request           | Triggers Sonarr series search/download                | Monitor series                   |                                              |                                                                                                         |
| Search series season          | HTTP Request           | Triggers Sonarr season search/download                | Monitor all seasons1              |                                              |                                                                                                         |
| Remove all dummy files        | SSH                    | Deletes all dummy files in series folder               | Remove unmonitored tag Sonarr       |                                              |                                                                                                         |
| If deletedFiles exist          | If                     | Checks if dummy files were deleted                      | Remove unmonitored tag, Remove unmonitored tag Sonarr | Get active Tautulli sessions                 |                                                                                                         |
| Get active Tautulli sessions  | HTTP Request           | Retrieves current playback sessions from Tautulli      | If deletedFiles exist              | Get all active sessions for file                |                                                                                                         |
| Get all active sessions for file | Code                   | Filters Tautulli sessions matching dummy file           | Get active Tautulli sessions       | Terminate Tautulli sessions for dummy           |                                                                                                         |
| Terminate Tautulli sessions for dummy | HTTP Request           | Terminates Tautulli sessions for dummy playback          | Get all active sessions for file   |                                              |                                                                                                         |
| Get Overseerr users           | HTTP Request           | Retrieves users from Overseerr API                       | Respond 200, Respond               | Filter Overseerr user                            |                                                                                                         |
| Filter Overseerr user          | Code                   | Filters Overseerr users by email from Tautulli webhook  | Get Overseerr users               | Check if Overseerr user is found                |                                                                                                         |
| Check if Overseerr user is found | If                     | Validates Overseerr user presence                         | Filter Overseerr user             | Get last 20 Overseerr requests from user        |                                                                                                         |
| Get last 20 Overseerr requests from user | HTTP Request           | Gets recent requests by user                              | Check if Overseerr user is found  | Check if request already exists                  |                                                                                                         |
| Check if request already exists | Code                   | Checks if media request already exists                    | Get last 20 Overseerr requests    | Check if no existing request for media exist for Overseerr user |                                                                                                         |
| Check if no existing request for media exist for Overseerr user | If                     | Proceeds only if no existing request found                 | Check if request already exists   | Make Overseerr request                            |                                                                                                         |
| Make Overseerr request        | HTTP Request           | Creates new request for media in Overseerr                  | Check if no existing request      |                                              |                                                                                                         |
| Get collection1               | HTTP Request           | Gets Plex collection metadata                              | Wait 5 minutes before process collection in Plex | Get all items in collection1                    |                                                                                                         |
| Get all items in collection1  | HTTP Request           | Lists all items in Plex collection                         | Get collection1                  | Check if collection contains items1              |                                                                                                         |
| Check if collection contains items1 | If                     | Checks if collection has items                              | Get all items in collection1     | Split Out Collection Items1                       |                                                                                                         |
| Split Out Collection Items1   | Split Out               | Splits collection items array for iteration                | Check if collection contains items1 | Loop Over Items1                                  |                                                                                                         |
| Loop Over Items1              | Split In Batches        | Processes collection items in batches                       | Split Out Collection Items1       | Remove item from collection1, Code                |                                                                                                         |
| Remove item from collection1  | HTTP Request           | Removes items from Plex collection                          | Loop Over Items1                 | Loop Over Items1                                  |                                                                                                         |
| Code                         | Code                   | Processes collection items or data                           | Loop Over Items1                 | Loop Over Items1                                  |                                                                                                         |
| Loop Over Items3              | Split In Batches        | Processes another set of collection items                    | Code                            | Get GUID from imdbId                              |                                                                                                         |
| Get GUID from imdbId          | HTTP Request           | Looks up Plex metadata GUID by IMDB ID                      | Loop Over Items3                 | If found Plex GUID1                                |                                                                                                         |
| If found Plex GUID1           | If                     | Checks if Plex GUID found                                    | Get GUID from imdbId             | Get ratingKey from GUID1                           |                                                                                                         |
| Get ratingKey from GUID1      | HTTP Request           | Gets Plex rating key from GUID                               | If found Plex GUID1              | Add item to collection3, Merge, Merge2             |                                                                                                         |
| Add item to collection3       | HTTP Request           | Adds item to Plex collection                                 | Get ratingKey from GUID1         | Merge                                             |                                                                                                         |
| Add item to collection2       | HTTP Request           | Adds item to Plex collection                                 | Merge                       | Merge2                                            |                                                                                                         |
| Move collection item2         | HTTP Request           | Moves item within Plex collection                             | Merge2                      | Loop Over Items3                                  |                                                                                                         |
| Merge                       | Merge                  | Merges multiple branches                                     | Add item to collection3, Add item to collection2 | Merge2                                            |                                                                                                         |
| Merge2                      | Merge                  | Merges multiple branches                                     | Add item to collection2, Move collection item2 |                                                    |                                                                                                         |
| Merge3                      | Merge                  | Merges monitoring branches                                   | Monitor all seasons, Monitor all seasons1 | Search season pack                                |                                                                                                         |
| Search                       | Various (httpRequest)   | Initiates searches in Radarr/Sonarr                          | Various                        | Various                                           |                                                                                                         |
| Respond 200                  | Respond to Webhook      | Sends 200 OK response                                        | Search movie                    | Get Overseerr users                               |                                                                                                         |
| Respond                     | Respond to Webhook      | Sends 200 OK response                                        | Search season pack              | Get Overseerr users                               |                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Incoming Trigger Events:**
   - *Webhook Arrs Dummy file update*: POST method on path `/bd5102ad-9dc5-4415-b779-89d002c6e448`.
   - *Webhook Tautulli*: POST on `/9cc04992-377c-40fc-a0a9-810c08438224`.
   - *Webhook Arrs custom list JustWatch*: GET/POST on `/:filter/:limit/:objectType/:collectionId`.
   - *Webhook Arrs custom list Trakt*: GET/POST on `/:username/:list/:limit/:objectType/:collectionId`.

2. **Add Switch and If Nodes to Route Based on Event Type:**
   - *Switch eventType*: Route by `eventType` field (`Download`, `MovieAdded`, `SeriesAdd`).
   - *If Radarr*: Check if payload contains Radarr movie.
   - *If tag unmonitored exists* (Radarr and Sonarr branches): Check for `dummy-unprocessed` tag in media tags.

3. **HTTP Request Nodes for Radarr/Sonarr API Calls:**
   - Radarr: *Radarr movie*, *Get Radarr information from tmdbId*, *Monitor movie*, *Search movie*, *Remove unmonitored tag*.
   - Sonarr: *Sonarr series*, *Get Sonarr information from tvdbId*, *Monitor series*, *Monitor all seasons*, *Search season pack*, *Remove unmonitored tag Sonarr*.

4. **SSH Nodes for Dummy File Creation and Cleanup:**
   - Create dummy files with FFmpeg commands for movies and series.
   - Remove dummy files when needed.
   - Use SSH credentials pointing to media server with FFmpeg installed.

5. **Plex API Integration:**
   - Use HTTP Request nodes to refresh Plex libraries.
   - Manage Plex collections by fetching collections, listing items, adding/removing items.
   - Use Plex token with HTTP header authentication.

6. **External List Handling:**
   - Call JustWatch API with GraphQL POST requests.
   - Call Trakt API with appropriate headers and parameters.
   - Format results with Code nodes to extract TMDB, IMDB, and TVDB IDs.

7. **Collection Processing:**
   - Use Wait node to delay processing for 5 minutes after receiving list data.
   - Split collections and iterate with SplitInBatches and SplitOut nodes.
   - Add or remove items from Plex collections based on external list data.

8. **Tautulli Playback Monitoring:**
   - On playback start for dummy files, filter user from Overseerr users list.
   - Check if Overseerr request exists, create new request if not.
   - Terminate Tautulli dummy playback sessions after real media is available.

9. **Utility Nodes:**
   - Merge nodes to combine parallel branches.
   - Set and Code nodes to prepare data structures.
   - Respond to Webhook nodes for proper HTTP responses.

10. **Credential Setup:**
    - HTTP Header Auth credentials for Radarr, Sonarr, Plex, Overseerr API keys.
    - HTTP Query Auth for Tautulli API.
    - SSH credentials for media server with FFmpeg access.

11. **Parameters and Defaults:**
    - Configure URLs with local network IPs or hostnames.
    - Use proper paths for media folders (e.g., `/mnt/user/Media/Plex`).
    - Use correct Plex library section IDs for movies and series.
    - Set tags in *Arrs applications: `dummy`, `dummy-unprocessed`.
    - Disable "search on add" in *Arrs, set monitor to none by default.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| **n8n Placeholdarr for Plex (BETA)** - This flow creates dummy files for Radarr/Sonarr items tagged `dummy-unprocessed`. When a dummy is played, the real media is requested and downloaded automatically. | Sticky Note1 in workflow |
| Use the following webhook URLs format to integrate with *Arrs and external lists: | Sticky Note (12ccd64d-4ea8-4d65-9778-f067866b2c12) |
| JustWatch Webhook format example and parameter explanations included. | Sticky Note (12ccd64d-4ea8-4d65-9778-f067866b2c12) |
| Trakt.TV Webhook format example and parameter explanations included. | Sticky Note (12ccd64d-4ea8-4d65-9778-f067866b2c12) |
| Requirements: SSH with FFmpeg installed, correctly configured API keys and tokens for Radarr, Sonarr, Plex, Overseerr, and Tautulli. | Sticky Note1 and Sticky Note6 |
| Tautulli webhook instructions with sample payload for playback start events with dummy files. | Sticky Note3 |
| To-do: Implement check to ensure no dummy files remain before removing tags. | Sticky Note2 |
| If SSH cannot run FFmpeg, manually copy dummy files to dummy folder as fallback. | Sticky Note4 |

---

**Disclaimer:** The provided workflow is created entirely using n8n automation platform, respecting current content policies and legal constraints. It processes only legal and public data.

---