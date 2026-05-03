Merge and filter several Rss & send to Telegram

https://n8nworkflows.xyz/workflows/merge-and-filter-several-rss---send-to-telegram-2748


# Merge and filter several Rss & send to Telegram

### 1. Workflow Overview

This workflow is designed to aggregate, curate, and distribute content from multiple RSS feeds to a Telegram channel. It targets users who want to monitor and share updates from several RSS sources in a consolidated, filtered, and formatted manner.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Periodic triggering and RSS feed retrieval from two distinct sources.
- **1.2 Data Curation:** Cleaning and formatting RSS feed items using regular expressions and JavaScript to extract relevant fields (title, link, publication date).
- **1.3 Data Merging and Filtering:** Combining the curated feeds, filtering out older items beyond a specified age threshold, and sorting by publication date.
- **1.4 Output Formatting:** Generating a Markdown-formatted list of items with proper escaping of special characters.
- **1.5 Delivery:** Sending the formatted list as a message to a Telegram channel.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow on a schedule and fetches RSS feeds from two different sources.

**Nodes Involved:**  
- Schedule Trigger  
- RSS Olimpo  
- RSS Torrent

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates the workflow every 8 hours.  
  - *Configuration:* Interval set to 8 hours.  
  - *Inputs:* None (trigger node).  
  - *Outputs:* Connects to both RSS feed nodes.  
  - *Edge Cases:* If the trigger fails or is disabled, the workflow won't run.  
  - *Version:* 1.2

- **RSS Olimpo**  
  - *Type:* RSS Feed Read  
  - *Role:* Reads RSS feed from the Olimpo source.  
  - *Configuration:* URL set to `https://hd-olimpo.club/rss/7335.be0cbfb98c9c4c08ddb3cd459c77967f`. SSL verification enabled (ignoreSSL: false).  
  - *Inputs:* From Schedule Trigger.  
  - *Outputs:* Connects to "Edit Fields" node for Olimpo.  
  - *Edge Cases:* Network errors, invalid RSS format, or URL changes may cause failures.  
  - *Version:* 1.1

- **RSS Torrent**  
  - *Type:* RSS Feed Read  
  - *Role:* Reads RSS feed from the TorrentLand source.  
  - *Configuration:* URL set to `https://torrentland.li/rss/1251.283bddcd8d90ab67e4d36c4e09bc9a21`. Default SSL settings.  
  - *Inputs:* From Schedule Trigger.  
  - *Outputs:* Connects to "Edit Fields1" node for TorrentLand.  
  - *Edge Cases:* Same as above (network, format, URL).  
  - *Version:* 1.1

---

#### 2.2 Data Curation

**Overview:**  
This block processes each RSS feed separately to extract and format key fields: title, link, and publication date. It uses regular expressions and JavaScript string manipulation to clean titles and extract file sizes.

**Nodes Involved:**  
- Edit Fields (for Olimpo)  
- Edit Fields1 (for TorrentLand)  
- Sticky Note2 (instructional note)

**Node Details:**

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Cleans and formats RSS items from Olimpo feed.  
  - *Configuration:*  
    - `title`: Removes "[PACK]" tags and other bracketed text, trims whitespace, appends file size extracted via regex from HTML content.  
    - `link`: Transforms download URLs to a simplified torrent path using regex replacement.  
    - `pubDate`: Converts publication date string to Unix timestamp (milliseconds).  
  - *Expressions:* Uses JavaScript expressions with regex replacements and date parsing.  
  - *Inputs:* From RSS Olimpo.  
  - *Outputs:* To "Merged Rss" node (input 0).  
  - *Edge Cases:*  
    - Regex failures if RSS content structure changes.  
    - Missing or malformed `content` field causing errors in size extraction.  
  - *Version:* 3.4

- **Edit Fields1**  
  - *Type:* Set  
  - *Role:* Similar to "Edit Fields" but for TorrentLand feed.  
  - *Configuration:*  
    - `title`: Removes "[PACK]" prefix and "1080p" suffix, appends file size extracted from HTML content via regex.  
    - `link`: Same URL transformation as above.  
    - `pubDate`: Converts publication date to Unix timestamp.  
  - *Expressions:* JavaScript with regex replacements.  
  - *Inputs:* From RSS Torrent.  
  - *Outputs:* To "Merged Rss" node (input 1).  
  - *Edge Cases:* Same as "Edit Fields".  
  - *Version:* 3.4

- **Sticky Note2**  
  - *Content:* "Curate info\nAdjust the regular expression to achieve the desired result"  
  - *Context:* Positioned near the Edit Fields nodes as a reminder to customize regex patterns for different RSS sources.

---

#### 2.3 Data Merging and Filtering

**Overview:**  
This block merges the two curated RSS feeds, filters out items older than two days, and sorts the remaining items by publication date descending.

**Nodes Involved:**  
- Merged Rss  
- Filter  
- Sort  
- Sticky Note1 (instructional note)

**Node Details:**

- **Merged Rss**  
  - *Type:* Merge  
  - *Role:* Combines the two input streams from the curated RSS feeds into a single list.  
  - *Inputs:* From "Edit Fields" (Olimpo) and "Edit Fields1" (TorrentLand).  
  - *Outputs:* To "Filter" node.  
  - *Edge Cases:* If one input is empty or fails, merge still proceeds with available data.  
  - *Version:* 3

- **Filter**  
  - *Type:* Filter  
  - *Role:* Removes items older than 2 days (48 hours).  
  - *Configuration:*  
    - Condition: `pubDate` > (current timestamp - 2 days in milliseconds).  
    - Uses loose type validation and numeric comparison.  
  - *Inputs:* From "Merged Rss".  
  - *Outputs:* To "Sort".  
  - *Edge Cases:* Items missing `pubDate` will fail the condition and be filtered out.  
  - *Version:* 2.2

- **Sort**  
  - *Type:* Sort  
  - *Role:* Sorts items by `pubDate` descending (most recent first).  
  - *Configuration:* Sort field: `pubDate`, order: descending.  
  - *Inputs:* From "Filter".  
  - *Outputs:* To "Markdown list".  
  - *Edge Cases:* Items without `pubDate` may be sorted unpredictably.  
  - *Version:* 1

- **Sticky Note1**  
  - *Content:* "Items age\nHere the maximum age of the elements that we are going to show is defined"  
  - *Context:* Positioned near Filter and Sort nodes to highlight filtering logic.

---

#### 2.4 Output Formatting

**Overview:**  
This block converts the sorted list of RSS items into a Markdown-formatted message string, escaping special characters to ensure proper rendering in Telegram.

**Nodes Involved:**  
- Markdown list

**Node Details:**

- **Markdown list**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates a Markdown list of links from the RSS items.  
  - *Configuration:*  
    - Defines an `escapeMarkdown` function to escape special Markdown characters (`\, *, _, [, ], ~, `, >, #, +, =, |, !`).  
    - Maps each item to a Markdown link: `[escaped title](link)`.  
    - Joins all entries with newline characters.  
    - Returns a single JSON object with a `message` field containing the formatted list.  
  - *Inputs:* From "Sort".  
  - *Outputs:* To "Telegram".  
  - *Edge Cases:*  
    - Missing `title` or `link` fields replaced with defaults ("No title" or "#").  
    - Large message size may exceed Telegram limits (mitigated by filtering).  
  - *Version:* 2

---

#### 2.5 Delivery

**Overview:**  
This block sends the formatted Markdown message to a specified Telegram channel.

**Nodes Involved:**  
- Telegram

**Node Details:**

- **Telegram**  
  - *Type:* Telegram  
  - *Role:* Sends the Markdown message to a Telegram channel.  
  - *Configuration:*  
    - `text`: Includes a header with separators and the message from previous node.  
    - `chatId`: Set to `-1001216307043` (Telegram channel ID).  
    - `parse_mode`: Markdown (to interpret formatting).  
    - `appendAttribution`: false (no attribution appended).  
  - *Credentials:* Uses configured Telegram API credentials (OAuth2).  
  - *Inputs:* From "Markdown list".  
  - *Outputs:* None (end node).  
  - *Edge Cases:*  
    - Message length exceeding Telegram limits causes failure.  
    - Invalid chat ID or revoked credentials cause errors.  
  - *Version:* 1.2

---

### 3. Summary Table

| Node Name       | Node Type          | Functional Role                      | Input Node(s)          | Output Node(s)       | Sticky Note                                                                                  |
|-----------------|--------------------|------------------------------------|-----------------------|----------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger    | Periodic workflow initiation       | None                  | RSS Olimpo, RSS Torrent|                                                                                              |
| RSS Olimpo      | RSS Feed Read      | Fetch RSS feed from Olimpo source  | Schedule Trigger      | Edit Fields          | ## Rss Nodes In these nodes you have to modify the urls of the rss feeds to be consulted.    |
| RSS Torrent     | RSS Feed Read      | Fetch RSS feed from TorrentLand    | Schedule Trigger      | Edit Fields1         | ## Rss Nodes In these nodes you have to modify the urls of the rss feeds to be consulted.    |
| Edit Fields     | Set                | Curate Olimpo RSS items            | RSS Olimpo            | Merged Rss           | ## Curate info Adjust the regular expression to achieve the desired result                    |
| Edit Fields1    | Set                | Curate TorrentLand RSS items       | RSS Torrent           | Merged Rss           | ## Curate info Adjust the regular expression to achieve the desired result                    |
| Merged Rss      | Merge              | Combine curated RSS feeds           | Edit Fields, Edit Fields1 | Filter             |                                                                                              |
| Filter          | Filter             | Remove items older than 2 days      | Merged Rss            | Sort                 | ## Items age Here the maximum age of the elements that we are going to show is defined        |
| Sort            | Sort               | Sort items by publication date desc| Filter                | Markdown list        | ## Items age Here the maximum age of the elements that we are going to show is defined        |
| Markdown list   | Code                | Format items as Markdown list       | Sort                  | Telegram             |                                                                                              |
| Telegram        | Telegram            | Send formatted message to Telegram | Markdown list         | None                 |                                                                                              |
| Sticky Note     | Sticky Note         | Instructional note on RSS nodes     | None                  | None                 | ## Rss Nodes In these nodes you have to modify the urls of the rss feeds to be consulted.    |
| Sticky Note1    | Sticky Note         | Instructional note on item age      | None                  | None                 | ## Items age Here the maximum age of the elements that we are going to show is defined        |
| Sticky Note2    | Sticky Note         | Instructional note on regex curation| None                  | None                 | ## Curate info Adjust the regular expression to achieve the desired result                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set interval to every 8 hours.  
   - No credentials needed.

2. **Create RSS Feed Read Node for Olimpo**  
   - Type: RSS Feed Read  
   - Set URL to `https://hd-olimpo.club/rss/7335.be0cbfb98c9c4c08ddb3cd459c77967f`  
   - Ensure SSL verification is enabled (ignoreSSL: false).  
   - Connect input from Schedule Trigger.

3. **Create RSS Feed Read Node for TorrentLand**  
   - Type: RSS Feed Read  
   - Set URL to `https://torrentland.li/rss/1251.283bddcd8d90ab67e4d36c4e09bc9a21`  
   - Default SSL settings.  
   - Connect input from Schedule Trigger.

4. **Create Set Node "Edit Fields" for Olimpo**  
   - Type: Set  
   - Add three fields with expressions:  
     - `title`: `={{ $json.title.replace(/\[PACK\].*/, "").replace(/\[.*?\]/g, "").trim() }} ({{$json.content.match(/<strong>Size<\/strong>:\s([\d.]+\s[KMGT]iB)/)[1]}})`  
     - `link`: `={{ $json.link.replace(/\/torrent\/download\/(\d+)\..*/, "/torrents/$1") }}`  
     - `pubDate`: `={{ new Date($json.pubDate).getTime() }}`  
   - Connect input from RSS Olimpo.

5. **Create Set Node "Edit Fields1" for TorrentLand**  
   - Type: Set  
   - Add three fields with expressions:  
     - `title`: `={{$json.title.replace(/^\[PACK\] /, "").replace(/1080p .*/, "")}} ({{$json.content.match(/<strong>Size<\/strong>:\s([\d.]+\s[KMGT]iB)/)[1]}})`  
     - `link`: `={{ $json.link.replace(/\/torrent\/download\/(\d+)\..*/, "/torrents/$1") }}`  
     - `pubDate`: `={{ new Date($json.pubDate).getTime() }}`  
   - Connect input from RSS Torrent.

6. **Create Merge Node "Merged Rss"**  
   - Type: Merge  
   - Connect inputs from both "Edit Fields" and "Edit Fields1" nodes.

7. **Create Filter Node**  
   - Type: Filter  
   - Add condition:  
     - Left value: `={{ $json.pubDate }}`  
     - Operator: greater than  
     - Right value: `={{ Date.now() - 2 * 24 * 60 * 60 * 1000 }}` (2 days in milliseconds)  
   - Connect input from "Merged Rss".

8. **Create Sort Node**  
   - Type: Sort  
   - Sort by field `pubDate` descending.  
   - Connect input from Filter node.

9. **Create Code Node "Markdown list"**  
   - Type: Code (JavaScript)  
   - Paste the following code:

```javascript
function escapeMarkdown(text) {
    return text
        .replace(/\\/g, "\\\\")
        .replace(/\*/g, "\\*")
        .replace(/_/g, "\\_")
        .replace(/\[/g, "\\[")
        .replace(/\]/g, "\\]")
        .replace(/~/g, "\\~")
        .replace(/`/g, "\\`")
        .replace(/>/g, "\\>")
        .replace(/#/g, "\\#")
        .replace(/\+/g, "\\+")
        .replace(/=/g, "\\=")
        .replace(/\|/g, "\\|")
        .replace(/!/g, "\\!");
}

const formattedList = items.map(item => {
    const title = escapeMarkdown(item.json.title || "No title");
    const link = item.json.link || "#";
    return `[${title}](${link})`;
}).join("\n");

return [{ json: { message: formattedList } }];
```

   - Connect input from Sort node.

10. **Create Telegram Node**  
    - Type: Telegram  
    - Set `chatId` to your Telegram channel ID (e.g., `-1001216307043`).  
    - Set `text` to:

```
`----------------------------`
`-- TorrentLand & HDOlimpo --`
`----------------------------`
{{ $json.message }}
```

    - Set `parse_mode` to `Markdown`.  
    - Disable `appendAttribution`.  
    - Configure Telegram API credentials (OAuth2) with your bot token.  
    - Connect input from "Markdown list".

11. **Save and Activate Workflow**  
    - Ensure all nodes are connected as per above.  
    - Test the workflow manually or wait for scheduled trigger.  
    - Monitor for errors such as message length limits or feed parsing issues.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| When opening the workflow for the first time, configure the Telegram credential.                 | Configuration instruction                                                                        |
| In the "RSS" nodes, insert the URLs of the sources to query.                                    | RSS feed customization                                                                          |
| In the "Edit Fields" nodes, adjust the regular expressions to obtain the desired result.        | Regex customization for different RSS feed formats                                             |
| The "Sort" node defines the maximum age of elements to send (default 2 days) to avoid Telegram errors due to message length. | Message length constraint                                                                       |
| Include your Telegram channel ID in the "Telegram" node to direct messages correctly.            | Telegram channel configuration                                                                  |
| Template created in n8n v1.72.1                                                                 | Version compatibility                                                                           |
| Telegram Markdown formatting guide: https://core.telegram.org/bots/api#markdownv2-style         | Useful for customizing message formatting                                                      |

---

This documentation provides a complete, structured understanding of the workflow, enabling reproduction, modification, and troubleshooting by advanced users and automation agents alike.