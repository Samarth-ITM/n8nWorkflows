Twitch Auto-Clip-Generator: Fetch from Streamers, Clip & Edit on Autopilot

https://n8nworkflows.xyz/workflows/twitch-auto-clip-generator--fetch-from-streamers--clip---edit-on-autopilot-3521


# Twitch Auto-Clip-Generator: Fetch from Streamers, Clip & Edit on Autopilot

### 1. Workflow Overview

This workflow automates the process of clipping, editing, and uploading Twitch stream highlights. It is designed primarily for Twitch streamers, content creators, and clippers who want to efficiently repurpose stream content for social media platforms without manual intervention.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Date Preparation:** Initiates the workflow daily and prepares the date parameter for fetching clips.
- **1.2 Twitch API Authentication & Clip Retrieval:** Obtains OAuth token and fetches clips from Twitch for the specified date.
- **1.3 Clip Processing & Filtering:** Structures the raw clip data and filters to select the best clips based on predefined criteria.
- **1.4 Clip Looping & Editing:** Iterates over each selected clip, downloads, and edits it into a social-media-ready format.
- **1.5 Uploading Clips:** Uploads the edited clips to Google Drive by default, with placeholders for TikTok, YouTube Shorts, and Instagram Reels uploads.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Date Preparation

**Overview:**  
This block triggers the workflow automatically on a schedule (daily) and prepares the date parameter used to fetch clips from Twitch.

**Nodes Involved:**  
- Schedule Trigger  
- Fetch Date

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow daily (default schedule, likely daily but configurable)  
  - *Configuration:* No custom parameters shown; defaults to daily trigger  
  - *Inputs:* None (start node)  
  - *Outputs:* Connects to Fetch Date node  
  - *Edge Cases:* Misconfiguration could cause no triggers or too frequent runs

- **Fetch Date**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates the date string or object representing the day for which clips should be fetched  
  - *Configuration:* Likely uses current date or previous day to set date parameter for Twitch API calls  
  - *Inputs:* From Schedule Trigger  
  - *Outputs:* To Fetch Streamer Clips  
  - *Edge Cases:* Date formatting errors, timezone issues could cause incorrect clip fetching

---

#### 2.2 Twitch API Authentication & Clip Retrieval

**Overview:**  
This block authenticates with Twitch API using OAuth and fetches clips from streamers for the specified date.

**Nodes Involved:**  
- Obtain your Twitch OAuth Token  
- Fetch Streamer Clips

**Node Details:**

- **Obtain your Twitch OAuth Token**  
  - *Type:* HTTP Request  
  - *Role:* Requests an OAuth token from Twitch API to authorize subsequent API calls  
  - *Configuration:* Uses Twitch client credentials (client ID and secret) to obtain token  
  - *Inputs:* None (triggered indirectly from Fetch Date block)  
  - *Outputs:* Token passed implicitly or stored for use in Fetch Streamer Clips  
  - *Edge Cases:* Authentication failure due to invalid credentials, token expiry, or network errors

- **Fetch Streamer Clips**  
  - *Type:* HTTP Request  
  - *Role:* Calls Twitch API endpoint to retrieve clips from streamers for the given date  
  - *Configuration:* Uses OAuth token for authorization; query parameters include date range and streamer IDs  
  - *Inputs:* Date parameter from Fetch Date, OAuth token from previous node  
  - *Outputs:* Raw clip data to Structure Clips node  
  - *Edge Cases:* API rate limits, invalid token, empty clip results, network timeouts

---

#### 2.3 Clip Processing & Filtering

**Overview:**  
This block structures the raw clip data into manageable items and filters to select only the best clips based on criteria such as view count or clip length.

**Nodes Involved:**  
- Structure Clips  
- Filter Best Clips

**Node Details:**

- **Structure Clips**  
  - *Type:* Split Out  
  - *Role:* Converts the array of clips from Twitch API response into individual items for processing  
  - *Configuration:* Splits the array field containing clips into separate workflow items  
  - *Inputs:* Raw clip data from Fetch Streamer Clips  
  - *Outputs:* Individual clip items to Filter Best Clips  
  - *Edge Cases:* Empty or malformed clip arrays causing no output or errors

- **Filter Best Clips**  
  - *Type:* Limit  
  - *Role:* Limits the number of clips processed further, effectively filtering for top clips  
  - *Configuration:* Default limit set (e.g., top N clips); can be customized based on clip metrics  
  - *Inputs:* Individual clip items from Structure Clips  
  - *Outputs:* Filtered clips to Loop Over Clips  
  - *Edge Cases:* Overly restrictive limits may exclude all clips; no clips to filter

---

#### 2.4 Clip Looping & Editing

**Overview:**  
This block loops over each filtered clip, downloads the clip, and edits it into a social-media-friendly format (e.g., 9:16 aspect ratio).

**Nodes Involved:**  
- Loop Over Clips  
- Download and Edit Clips

**Node Details:**

- **Loop Over Clips**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over each clip item one by one for sequential processing  
  - *Configuration:* Batch size of 1 to process clips individually  
  - *Inputs:* Filtered clips from Filter Best Clips  
  - *Outputs:* Each clip to Download and Edit Clips  
  - *Edge Cases:* Large clip counts may slow workflow; batch size misconfiguration

- **Download and Edit Clips**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the clip file and triggers editing (likely via external API or service)  
  - *Configuration:* Uses clip URL to download; editing parameters set for social media format (e.g., 9:16)  
  - *Inputs:* Single clip data from Loop Over Clips  
  - *Outputs:* Edited clip file to Upload Clips  
  - *Edge Cases:* Download failures, editing service errors, unsupported clip formats

---

#### 2.5 Uploading Clips

**Overview:**  
Uploads the edited clips to Google Drive by default, with nodes prepared for uploading to TikTok, YouTube Shorts, and Instagram Reels (customization required).

**Nodes Involved:**  
- Upload Clips  
- Post To TikTok  
- Upload to your YouTube Account  
- Post To Instagram

**Node Details:**

- **Upload Clips**  
  - *Type:* Google Drive  
  - *Role:* Uploads edited clips to a configured Google Drive folder for storage and review  
  - *Configuration:* Uses Google Drive OAuth2 credentials; folder path configurable  
  - *Inputs:* Edited clip files from Download and Edit Clips  
  - *Outputs:* Connects to Wait node (likely for pacing or delay)  
  - *Edge Cases:* Authentication errors, quota limits, file size restrictions

- **Post To TikTok**  
  - *Type:* HTTP Request  
  - *Role:* Placeholder node for uploading clips directly to TikTok via API (requires customization)  
  - *Configuration:* Empty/default; user must configure API endpoint, authentication, and parameters  
  - *Inputs:* Edited clip files (not connected in current workflow)  
  - *Outputs:* None connected  
  - *Edge Cases:* TikTok API limitations, authentication, video format compliance

- **Upload to your YouTube Account**  
  - *Type:* YouTube  
  - *Role:* Placeholder for uploading clips as YouTube Shorts (requires user setup)  
  - *Configuration:* YouTube OAuth2 credentials needed; video metadata configurable  
  - *Inputs:* Edited clip files (not connected in current workflow)  
  - *Outputs:* None connected  
  - *Edge Cases:* API quota, video format, authentication errors

- **Post To Instagram**  
  - *Type:* HTTP Request  
  - *Role:* Placeholder for posting clips to Instagram Reels (requires customization)  
  - *Configuration:* Empty/default; user must configure API endpoint and authentication  
  - *Inputs:* Edited clip files (not connected in current workflow)  
  - *Outputs:* None connected  
  - *Edge Cases:* Instagram API restrictions, authentication, video format compliance

- **Wait**  
  - *Type:* Wait  
  - *Role:* Adds delay or pacing after uploading clips, possibly to avoid rate limits or batch processing  
  - *Configuration:* Default wait parameters (not specified)  
  - *Inputs:* From Upload Clips  
  - *Outputs:* Loops back to Loop Over Clips for next clip processing  
  - *Edge Cases:* Misconfigured wait times could slow or stall workflow

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                          | Input Node(s)            | Output Node(s)           | Sticky Note                         |
|-----------------------------|---------------------|----------------------------------------|--------------------------|--------------------------|-----------------------------------|
| Schedule Trigger            | Schedule Trigger    | Starts workflow daily                   | None                     | Fetch Date               |                                   |
| Fetch Date                 | Code                | Prepares date parameter for API calls  | Schedule Trigger         | Fetch Streamer Clips     |                                   |
| Obtain your Twitch OAuth Token | HTTP Request        | Gets Twitch OAuth token                 | None (triggered indirectly) | Fetch Streamer Clips (token usage) |                                   |
| Fetch Streamer Clips       | HTTP Request        | Fetches clips from Twitch API           | Fetch Date, OAuth Token  | Structure Clips          |                                   |
| Structure Clips            | Split Out           | Splits clip array into individual items | Fetch Streamer Clips     | Filter Best Clips        |                                   |
| Filter Best Clips          | Limit               | Filters top clips based on criteria     | Structure Clips          | Loop Over Clips          |                                   |
| Loop Over Clips            | Split In Batches    | Iterates over clips one by one          | Filter Best Clips        | Download and Edit Clips (main), none (secondary) |                                   |
| Download and Edit Clips    | HTTP Request        | Downloads and edits clips                | Loop Over Clips          | Upload Clips             |                                   |
| Upload Clips               | Google Drive        | Uploads edited clips to Google Drive    | Download and Edit Clips  | Wait                     |                                   |
| Wait                      | Wait                | Adds delay between clip uploads          | Upload Clips             | Loop Over Clips          |                                   |
| Post To TikTok             | HTTP Request        | Placeholder for TikTok upload            | None                     | None                     |                                   |
| Upload to your YouTube Account | YouTube             | Placeholder for YouTube Shorts upload    | None                     | None                     |                                   |
| Post To Instagram          | HTTP Request        | Placeholder for Instagram Reels upload  | None                     | None                     |                                   |
| Sticky Note9               | Sticky Note         | (Empty content)                          | None                     | None                     |                                   |
| Sticky Note                | Sticky Note         | (Empty content)                          | None                     | None                     |                                   |
| Sticky Note1               | Sticky Note         | (Empty content)                          | None                     | None                     |                                   |
| Sticky Note2               | Sticky Note         | (Empty content)                          | None                     | None                     |                                   |
| Sticky Note3               | Sticky Note         | (Empty content)                          | None                     | None                     |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger daily at a preferred time.

2. **Add a Code node named "Fetch Date"**  
   - Configure to output the date string for the current or previous day in the format required by Twitch API (e.g., ISO 8601).  
   - Connect Schedule Trigger output to this node.

3. **Add an HTTP Request node named "Obtain your Twitch OAuth Token"**  
   - Configure to POST to Twitch OAuth token endpoint (`https://id.twitch.tv/oauth2/token`).  
   - Use client ID and client secret credentials (set up in n8n credentials).  
   - Request client credentials grant type.  
   - No input connection needed; this node can be triggered before fetching clips or integrated as a pre-step.

4. **Add an HTTP Request node named "Fetch Streamer Clips"**  
   - Configure GET request to Twitch API clips endpoint (`https://api.twitch.tv/helix/clips`).  
   - Use OAuth token from previous node in Authorization header.  
   - Use date parameter from "Fetch Date" node to filter clips by broadcast date.  
   - Connect "Fetch Date" output to this node.

5. **Add a Split Out node named "Structure Clips"**  
   - Configure to split the array of clips from the Twitch API response into individual items.  
   - Connect "Fetch Streamer Clips" output to this node.

6. **Add a Limit node named "Filter Best Clips"**  
   - Configure to limit the number of clips processed further (e.g., top 5 or 10).  
   - Optionally, add filtering expressions based on clip view count or length.  
   - Connect "Structure Clips" output to this node.

7. **Add a Split In Batches node named "Loop Over Clips"**  
   - Set batch size to 1 to process clips sequentially.  
   - Connect "Filter Best Clips" output to this node.

8. **Add an HTTP Request node named "Download and Edit Clips"**  
   - Configure to download clip video files using clip URLs.  
   - Integrate with an external video editing API or service to convert clips to 9:16 aspect ratio or other social media formats.  
   - Connect "Loop Over Clips" output to this node.

9. **Add a Google Drive node named "Upload Clips"**  
   - Configure with Google Drive OAuth2 credentials.  
   - Set target folder for storing edited clips.  
   - Connect "Download and Edit Clips" output to this node.

10. **Add a Wait node named "Wait"**  
    - Configure delay as needed (e.g., a few seconds) to pace uploads.  
    - Connect "Upload Clips" output to this node.

11. **Connect "Wait" node output back to "Loop Over Clips"**  
    - This creates a loop to process the next clip after waiting.

12. **Optional: Add HTTP Request nodes for "Post To TikTok", "Post To Instagram" and YouTube node "Upload to your YouTube Account"**  
    - Configure each with appropriate API endpoints and OAuth credentials.  
    - These nodes are placeholders and require user customization.  
    - Connect edited clips output as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is ideal for Twitch streamers and content creators looking to automate clip creation and social media repurposing. | Workflow description and use case summary.                                                      |
| Requires Twitch Developer Console setup for OAuth credentials.                                | Twitch Developer Console: https://dev.twitch.tv/console                                         |
| Google Drive OAuth2 credentials must be configured in n8n for clip storage.                   | Google Drive API documentation: https://developers.google.com/drive/api                         |
| Placeholder nodes for TikTok, YouTube Shorts, and Instagram Reels uploads require user customization and API access. | TikTok API: https://developers.tiktok.com/ ; YouTube API: https://developers.google.com/youtube |
| Consider adding notification integrations (Slack, Discord, Email) for clip readiness alerts.  | Not included but recommended for workflow enhancement.                                          |
| The workflow uses a looping mechanism with a Wait node to pace clip processing and avoid rate limits. | Important for stable execution and API quota management.                                        |

---

This documentation provides a comprehensive understanding of the Twitch Auto-Clip-Generator workflow, enabling reproduction, customization, and troubleshooting.