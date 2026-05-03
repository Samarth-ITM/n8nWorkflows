YouTube Advanced RSS Generator with Telegram Formation

https://n8nworkflows.xyz/workflows/youtube-advanced-rss-generator-with-telegram-formation-2221


# YouTube Advanced RSS Generator with Telegram Formation

### 1. Workflow Overview

This workflow, titled **"[n8n] YouTube Channel Advanced RSS Feeds Generator"**, is designed to generate multiple RSS feed formats from any public YouTube channel or video input without requiring Google API keys or administrative permissions. It leverages third-party services to extract YouTube channel IDs and produce a variety of RSS feed URLs, supporting formats like ATOM, JSON, MRSS, Plaintext, Sfeed, and the native YouTube XML feed.

**Target Use Cases:**

- Aggregating content from public YouTube channels or videos.
- Creating RSS feeds in multiple formats for syndication or further processing.
- Avoiding the need for Google API credentials or complex setup.
- Accepting diverse input types: YouTube channel usernames, IDs, video URLs, or channel URLs.

**Logical Blocks:**

- **1.1 Input Reception & Validation:** Captures user input via a form trigger and validates input types (username, channel ID, video ID, or direct channel URL).
- **1.2 Token Retrieval:** Obtains temporary tokens from a third-party service (commentpicker.com) needed for subsequent API requests.
- **1.3 Channel/Video ID Resolution:** Converts input into a canonical YouTube channel ID, handling various input formats including video IDs.
- **1.4 RSS URL Generation:** Constructs direct YouTube XML feed URLs and other RSS feed URLs using rss-bridge.org for multiple formats.
- **1.5 Aggregation & Response Formatting:** Aggregates all generated RSS URLs and formats them into an HTML table for webhook response.
- **1.6 Webhook Response:** Sends the final formatted HTML table back as a response to the original webhook/form submission.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
  This block captures the user input via a web form and validates it to determine whether the input is a YouTube username, channel ID, video ID, or URL. It normalizes the input for further processing.

- **Nodes Involved:**  
  - `n8n Form Trigger`  
  - `Validation Code`  
  - `Switch`  
  - `Set Channel Username`  
  - `Set Video ID`  
  - `Set XML Feed URL`

- **Node Details:**

  - **n8n Form Trigger**  
    - *Type:* Form Trigger (webhook-based)  
    - *Role:* Entry point for user input; captures YouTube channel username, ID, or video URL.  
    - *Key Configurations:*  
      - Path: `/Youtube`  
      - Form field: "youtube Channel username or ID" (required)  
      - Provides examples of valid inputs in description for user guidance.  
      - Responds via `Respond to Webhook` node.  
    - *Connections:* Output to `Validation Code`.  
    - *Failure Modes:* Missing input, malformed request.

  - **Validation Code**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Parses the input to identify type (channel username, channel ID, video ID) or returns error if invalid.  
    - *Key Expressions:* Uses regex patterns to detect input type and extract values.  
    - *Input:* From Form Trigger  
    - *Output:* JSON with keys `type` and `value` or an error field.  
    - *Failure Modes:* Invalid input formats, undefined input, regex mismatch.

  - **Switch**  
    - *Type:* Switch node  
    - *Role:* Routes workflow based on input type: "channel username", "channel ID", or "video ID".  
    - *Output Connections:*  
      - "Username" → `Set Channel Username`  
      - "Direct" (channel ID) → `Set XML Feed URL`  
      - "Video-ID" → `Set Video ID`  
    - *Failure Modes:* Unmatched input type leads to no route, causing workflow halt.

  - **Set Channel Username**  
    - *Type:* Set node  
    - *Role:* Assigns the extracted username from the switch to a key `channel name` for downstream API calls.  
    - *Input:* From Switch ("Username" path)  
    - *Output:* To `Get Channel ID` node.

  - **Set Video ID**  
    - *Type:* Set node  
    - *Role:* Assigns video ID extracted from input for further API call.  
    - *Input:* From Switch ("Video-ID" path)  
    - *Output:* To `Get Video ID Channel ID`.

  - **Set XML Feed URL**  
    - *Type:* Set node  
    - *Role:* Directly constructs YouTube XML feed URL from a known channel ID input.  
    - *Input:* From Switch ("Direct" path)  
    - *Output:* To `Aggregate`.

#### 2.2 Token Retrieval

- **Overview:**  
  This block retrieves temporary access tokens from commentpicker.com, required by the third-party API to resolve channel and video IDs.

- **Nodes Involved:**  
  - `Get Temporary Token`  
  - `GTT` (Get Temporary Token for video ID path)

- **Node Details:**

  - **Get Temporary Token**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves a temporary token for channel username to channel ID conversion.  
    - *Configuration:*  
      - URL: `https://commentpicker.com/actions/token.php`  
      - Sends headers including cookies and user-agent mimicking a browser to avoid blocks.  
    - *Input:* Receives from Switch "Username" branch before `Get Channel ID`.  
    - *Output:* To `Set Channel Username`.

  - **GTT**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves a temporary token for video ID to channel ID conversion.  
    - *Configuration:* Same as `Get Temporary Token`.  
    - *Input:* Receives from Switch "Video-ID" branch.  
    - *Output:* To `Set Video ID`.

  - *Failure Modes:*  
    - Request failures (network issues, rate limits, blocked cookies)  
    - Token expiration or invalid token returned by the third-party service.

#### 2.3 Channel/Video ID Resolution

- **Overview:**  
  This block uses the tokens to request from commentpicker.com the canonical YouTube channel ID, resolving usernames or video IDs to channel IDs.

- **Nodes Involved:**  
  - `Get Channel ID`  
  - `Get Video ID Channel ID`  
  - `Set XML URL`  
  - `Set XML Feed`

- **Node Details:**

  - **Get Channel ID**  
    - *Type:* HTTP Request  
    - *Role:* Requests YouTube channel details by username using Google API endpoint proxied via commentpicker.com.  
    - *Config:*  
      - Query includes YouTube API URL with `forHandle` parameter set to username.  
      - Uses token from `Get Temporary Token`.  
      - Headers include cookies and user-agent for compatibility.  
    - *Output:* Channel ID extracted from response JSON.  
    - *Next:* To `Set XML URL`.

  - **Get Video ID Channel ID**  
    - *Type:* HTTP Request  
    - *Role:* Retrieves channel ID associated with a given video ID.  
    - *Config:*  
      - Queries YouTube videos endpoint with video ID.  
      - Uses token from `GTT`.  
      - Similar headers as above.  
    - *Output:* Channel ID from video snippet info.  
    - *Next:* To `Set XML Feed`.

  - **Set XML URL**  
    - *Type:* Set node  
    - *Role:* Constructs the standard YouTube feeds XML URL using the resolved channel ID from `Get Channel ID`.  
    - *Output:* To `Aggregate`.

  - **Set XML Feed**  
    - *Type:* Set node  
    - *Role:* Constructs the standard YouTube feeds XML URL using channel ID from video info.  
    - *Output:* To `Aggregate`.

  - *Failure Modes:*  
    - Invalid or expired tokens causing API failures.  
    - Unexpected response formats or missing data.  
    - Rate limiting or blocking by commentpicker.com.

#### 2.4 RSS URL Generation

- **Overview:**  
  Generates 13 different RSS feed URLs in multiple formats (ATOM, JSON, MRSS, Plaintext, Sfeed, HTML, XML) for both channel videos and community posts using rss-bridge.org.

- **Nodes Involved:**  
  - `Aggregate`  
  - `Youtube Channel Videos RSS Formats`  
  - `Youtube Channel Community RSS Formats`

- **Node Details:**

  - **Aggregate**  
    - *Type:* Aggregate node  
    - *Role:* Combines multiple RSS feed URLs into a single dataset for further processing.  
    - *Input:* From `Set XML Feed URL`, `Set XML URL`, and `Set XML Feed`.  
    - *Output:* To both RSS format sets nodes.

  - **Youtube Channel Videos RSS Formats**  
    - *Type:* Set node  
    - *Role:* Creates URLs for six different video RSS formats using rss-bridge.org plus YouTube's direct XML feed.  
    - *Key Expressions:* Extract channel_id from aggregated XML feed URLs using regex.  
    - *Output:* To `Merga Data of Youtube & Community RSS`.

  - **Youtube Channel Community RSS Formats**  
    - *Type:* Set node  
    - *Role:* Creates URLs for six different community post RSS feed formats using rss-bridge.org.  
    - *Output:* To `Merga Data of Youtube & Community RSS`.

  - *Failure Modes:*  
    - Regex extraction failure if aggregated URLs do not contain expected pattern.  
    - rss-bridge.org service downtime or format URL changes.

#### 2.5 Aggregation & Response Formatting

- **Overview:**  
  Merges video and community RSS feed URLs into one dataset and formats them into an HTML table for user-friendly display.

- **Nodes Involved:**  
  - `Merga Data of Youtube & Community RSS` (Merge node)  
  - `Format response as HTML Table` (Code node)

- **Node Details:**

  - **Merga Data of Youtube & Community RSS**  
    - *Type:* Merge node (combine mode with multiplex)  
    - *Role:* Combines RSS URLs from videos and community posts feed sets into a single item list.  
    - *Output:* To `Format response as HTML Table`.

  - **Format response as HTML Table**  
    - *Type:* Code node (JavaScript)  
    - *Role:* Converts the merged RSS feed URLs into a styled HTML table with columns for Type, Format, and URL.  
    - *Output:* JSON with `html` key containing the formatted string.  
    - *Failure Modes:* Incorrect input structure causing runtime errors.

#### 2.6 Webhook Response

- **Overview:**  
  Sends the final HTML table back to the user as a response to the initial form submission.

- **Nodes Involved:**  
  - `Respond to Webhook`

- **Node Details:**

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook node  
    - *Role:* Returns the HTML table as plain text response to the webhook caller (the form).  
    - *Configuration:* Uses expression `{{$json["html"]}}` for response body.  
    - *Failure Modes:* Network issues or webhook misconfiguration causing incomplete responses.

---

### 3. Summary Table

| Node Name                        | Node Type           | Functional Role                              | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                      |
|---------------------------------|---------------------|----------------------------------------------|--------------------------------|---------------------------------------|------------------------------------------------------------------------------------------------|
| n8n Form Trigger                 | Form Trigger        | Receives user input via web form              | (start)                        | Validation Code                       |                                                                                                 |
| Validation Code                 | Code                | Validates and classifies input type           | n8n Form Trigger               | Switch                               |                                                                                                 |
| Switch                         | Switch              | Routes flow based on input type                | Validation Code                | Get Temporary Token, Set XML Feed URL, GTT |                                                                                                 |
| Get Temporary Token             | HTTP Request        | Retrieves token for username-to-channel ID    | Switch (Username path)         | Set Channel Username                  | 3rd party API request                                                                            |
| Set Channel Username            | Set                 | Sets channel username variable                 | Get Temporary Token            | Get Channel ID                       |                                                                                                 |
| Get Channel ID                 | HTTP Request        | Gets channel ID from username                   | Set Channel Username           | Set XML URL                         | 3rd party API request                                                                            |
| Set XML URL                   | Set                 | Builds YouTube XML feed URL from channel ID    | Get Channel ID                 | Aggregate                           | 🤖Generate XML Feed URL                                                                          |
| Set XML Feed URL               | Set                 | Builds YouTube XML feed URL from channel ID    | Switch (Direct path)           | Aggregate                           | 🤖Generate XML Feed URL                                                                          |
| GTT                           | HTTP Request        | Retrieves token for video ID to channel ID     | Switch (Video-ID path)         | Set Video ID                       | 3rd party API request                                                                            |
| Set Video ID                  | Set                 | Sets video ID variable                          | GTT                          | Get Video ID Channel ID              |                                                                                                 |
| Get Video ID Channel ID       | HTTP Request        | Gets channel ID from video ID                   | Set Video ID                  | Set XML Feed                       | 3rd party API request                                                                            |
| Set XML Feed                 | Set                 | Builds YouTube XML feed URL from video channel ID | Get Video ID Channel ID        | Aggregate                           | 🤖Generate XML Feed URL                                                                          |
| Aggregate                   | Aggregate            | Combines multiple RSS URLs into one dataset    | Set XML URL, Set XML Feed URL, Set XML Feed | Youtube Channel Community RSS Formats, Youtube Channel Videos RSS Formats | 🤖Combine results in one                                                                        |
| Youtube Channel Videos RSS Formats | Set                 | Constructs URLs for multiple video RSS formats | Aggregate                    | Merga Data of Youtube & Community RSS | RSS Feed for channel Videos                                                                     |
| Youtube Channel Community RSS Formats | Set                 | Constructs URLs for multiple community RSS formats | Aggregate                    | Merga Data of Youtube & Community RSS | RSS Feed for channel Posts                                                                      |
| Merga Data of Youtube & Community RSS | Merge                | Merges video and community RSS URLs            | Youtube Channel Community RSS Formats, Youtube Channel Videos RSS Formats | Format response as HTML Table            |                                                                                                 |
| Format response as HTML Table | Code                | Formats merged RSS URLs into an HTML table     | Merga Data of Youtube & Community RSS | Respond to Webhook                 |                                                                                                 |
| Respond to Webhook            | Respond to Webhook  | Sends HTML table response to webhook caller    | Format response as HTML Table  | (end)                              | Reply to the webhook request with table                                                        |
| Sticky Note                  | Sticky Note          | Informational overview of workflow and usage   | -                              | -                                   | 🌐 **Generate RSS Feeds for Public Youtube Channel (No API Or Administrator permissions Required 😉)** ... |
| Sticky Note2                 | Sticky Note          | Workarounds and additional context             | -                              | -                                   | ## ℹ️ **Workarounds And Information** ...                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: `Form Trigger`  
   - Path: `/Youtube`  
   - Form Title: "Youtube RSS Generator"  
   - Form Fields: One required text field labeled "youtube Channel username or ID"  
   - Response Mode: `responseNode`  
   - Form Description: Include examples of valid inputs (usernames, channel IDs, video URLs, channel URLs).

2. **Add Code Node "Validation Code":**  
   - Purpose: Parse and classify input into one of: channel username, channel ID, video ID, or error.  
   - Paste JavaScript code that uses regex to analyze the input and returns JSON with `type` and `value`.  
   - Connect output of Form Trigger to this node.

3. **Add Switch Node:**  
   - Configure 3 outputs with rules matching `type` field:  
     - Output "Username" if `type` equals "channel username"  
     - Output "Direct" if `type` equals "channel ID"  
     - Output "Video-ID" if `type` equals "video ID"  
   - Connect output of Validation Code to this Switch.

4. **Create HTTP Request Node "Get Temporary Token":**  
   - URL: `https://commentpicker.com/actions/token.php`  
   - Method: GET  
   - Set required headers: `authority`, `cookie`, `referer`, `user-agent` mimicking a browser.  
   - Connect Switch output "Username" to this node.

5. **Set Node "Set Channel Username":**  
   - Assign variable `channel name` from Switch output value.  
   - Connect `Get Temporary Token` output to this node.

6. **HTTP Request Node "Get Channel ID":**  
   - URL: `https://commentpicker.com/actions/youtube-channel-id.php`  
   - Method: GET  
   - Query parameters:  
     - `url`: YouTube API URL with `forHandle={{channel name}}`  
     - `token`: token from `Get Temporary Token` output  
     - `isPremium`: false  
   - Headers: same as token request.  
   - Connect `Set Channel Username` output to this.

7. **Set Node "Set XML URL":**  
   - Assign variable `rss` = `https://www.youtube.com/feeds/videos.xml?channel_id={{channel ID}}` extracted from `Get Channel ID` response.  
   - Connect `Get Channel ID` output to this node.

8. **Set Node "Set XML Feed URL":**  
   - For direct channel ID input, assign `rss = https://www.youtube.com/feeds/videos.xml?channel_id={{value}}` from Switch "Direct" output.  
   - Connect Switch output "Direct" to this node.

9. **HTTP Request Node "GTT":**  
   - Same as "Get Temporary Token" but connected from Switch "Video-ID" output.

10. **Set Node "Set Video ID":**  
    - Assign variable `Video ID` from Switch "Video-ID" value.  
    - Connect output of `GTT` to this node.

11. **HTTP Request Node "Get Video ID Channel ID":**  
    - URL: `https://commentpicker.com/actions/youtube-channel-id.php`  
    - Query params:  
      - `url`: YouTube videos API endpoint with `id={{Video ID}}`  
      - `token`: from `GTT`  
      - `isPremium`: true  
    - Headers: same as token request.  
    - Connect `Set Video ID` output to this node.

12. **Set Node "Set XML Feed":**  
    - Assign variable `rss` = `https://www.youtube.com/feeds/videos.xml?channel_id={{channelId}}` extracted from video details.  
    - Connect output of `Get Video ID Channel ID` to this node.

13. **Aggregate Node:**  
    - Combines outputs of `Set XML URL`, `Set XML Feed URL`, and `Set XML Feed` nodes into a single dataset.  
    - Connect outputs of these three Set nodes to `Aggregate`.

14. **Set Node "Youtube Channel Videos RSS Formats":**  
    - Use regex on aggregated `rss url` to extract channel_id.  
    - Create keys with URLs for formats: HTML, ATOM, JSON, MRSS, PLAINTEXT, SFEED, plus direct YouTube XML feed.  
    - Connect output of `Aggregate` to this node.

15. **Set Node "Youtube Channel Community RSS Formats":**  
    - Similar to above, build community RSS feed URLs in multiple formats using rss-bridge.org with the extracted channel_id.  
    - Connect output of `Aggregate` to this node.

16. **Merge Node "Merga Data of Youtube & Community RSS":**  
    - Mode: Combine (multiplex)  
    - Connect outputs of both RSS formats Set nodes to this merge node.

17. **Code Node "Format response as HTML Table":**  
    - Takes merged JSON data and builds a styled HTML table listing RSS feed type, format, and clickable URL.  
    - Connect output of merge node to this.

18. **Respond to Webhook Node:**  
    - Sends back the HTML table as plain text response to the form submitter.  
    - Connect output of code node to this.

**Credentials:**  
- No official Google API credentials needed.  
- Uses public third-party service (commentpicker.com) with cookies and user-agent headers set manually.  
- No OAuth or other authentication required.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| No need to acquire Google Cloud API to retrieve channel data. The workflow implements a free workaround method via commentpicker.com.                                                                                                                                                                                                                                                                                                            | Sticky Note2 content                                                                                     |
| The workflow has been tested with various input formats and reliably returns channel IDs from usernames, video URLs, or direct channel IDs.                                                                                                                                                                                                                                                                                                      | Sticky Note2 content                                                                                     |
| The workaround method may become obsolete if the third-party services change their API or policies. Users are encouraged to report issues on the n8n community forum.                                                                                                                                                                                                                                                                              | Sticky Note2 content                                                                                     |
| Uses 3rd party rss-bridge.org to generate multiple RSS feed formats including ATOM, JSON, MRSS, plaintext, sfeed, and native YouTube XML feeds.                                                                                                                                                                                                                                                                                                   | Workflow description and node notes                                                                     |
| The workflow produces 13 RSS feed URLs: 6 community formats, 6 video formats, and 1 native YouTube XML feed.                                                                                                                                                                                                                                                                                                                                       | Sticky Note content                                                                                      |
| Video overview of the workflow is available: [YouTube Channel Advanced RSS Feeds Generator](https://youtu.be/EtzJmrmCiUY)                                                                                                                                                                                                                                                                                                                          | Video link in workflow description                                                                       |
| For support and contributions, visit the [n8n community forum](https://community.n8n.io) and [GitHub repository](https://github.com/n8n-io/n8n).                                                                                                                                                                                                                                                                                                    | Workflow description                                                                                      |

---

This documentation provides a comprehensive understanding and stepwise reproduction guide of the "YouTube Channel Advanced RSS Feeds Generator" workflow, enabling users or automation agents to operate or extend it confidently.