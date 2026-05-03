Fetch latest RSS articles and store non-duplicates in Google Sheets

https://n8nworkflows.xyz/workflows/fetch-latest-rss-articles-and-store-non-duplicates-in-google-sheets-14704


# Fetch latest RSS articles and store non-duplicates in Google Sheets

# 1. Workflow Overview

This workflow reads a list of website URLs from Google Sheets, detects which of those websites expose an RSS or Atom feed, fetches the latest articles from the valid feeds, filters articles to only those published in the current UTC hour, and stores only non-duplicate articles in another Google Sheet.

Its main use cases are:

- Monitoring a list of websites for newly published RSS content
- Building a lightweight article ingestion pipeline into Google Sheets
- Avoiding duplicate storage by comparing fetched article titles against previously stored titles

The workflow is organized into the following logical blocks.

## 1.1 Trigger and URL Source Intake

The workflow starts manually and loads the list of candidate website URLs from a Google Sheet.

## 1.2 Website Inspection and RSS Feed Discovery

Each source URL is fetched via HTTP, scanned for RSS/Atom feed links in the HTML, and split into valid-feed vs no-feed paths.

## 1.3 RSS Reading and Latest-Article Filtering

For each discovered RSS feed URL, the workflow reads feed items and keeps only items whose publication hour matches the current UTC hour. If no matching article is found, execution stops.

## 1.4 Article Normalization

The selected feed item data is reduced to a consistent structure: `title`, `link`, and `article`.

## 1.5 Existing Article Lookup

Previously stored rows are read from the destination Google Sheet and checked to determine whether the sheet is empty or already contains titles.

## 1.6 Duplicate Detection and Branching

Existing titles are aggregated into a list, compared against the newly fetched titles, and the workflow decides whether:
- all fetched titles already exist,
- or some titles are new and should be appended.

## 1.7 Storage and Status Output

New rows are appended to Google Sheets. If the destination sheet is empty, all fetched items are inserted. If all titles already exist, a message node produces a simple status output.

---

# 2. Block-by-Block Analysis

## 2.1 Trigger and URL Source Intake

### Overview
This block initializes the workflow manually and loads the source websites from a Google Sheet. It provides the list of candidate URLs that will be inspected for RSS availability.

### Nodes Involved
- When clicking ‘Execute workflow’
- RSS URLs
- Sticky Note
- Sticky Note3

### Node Details

#### When clicking ‘Execute workflow’
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for testing or on-demand execution.
- **Configuration choices:** No custom parameters.
- **Key expressions or variables used:** None.
- **Input and output connections:** No input; outputs to `RSS URLs`.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None at runtime; only starts when manually executed.
- **Sub-workflow reference:** None.

#### RSS URLs
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads source URL rows from a Google Sheet.
- **Configuration choices:** Uses a specific Google Sheets document named **RSS URLs**, sheet **Sheet1** (`gid=0`). No explicit operation is shown, so this behaves as a read/list operation with default output of rows.
- **Key expressions or variables used:** Expects a column named `URLs`, because downstream code references `$json.URLs`.
- **Input and output connections:** Input from `When clicking ‘Execute workflow’`; output to `Fetch Each URL`.
- **Version-specific requirements:** Type version 4.7; requires valid Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - OAuth credential errors
  - Spreadsheet not found / sheet renamed
  - Missing `URLs` column
  - Empty sheet causing no downstream items
- **Sub-workflow reference:** None.

#### Sticky Note
- **Type and technical role:** `stickyNote`; visual documentation.
- **Configuration choices:** Content: “List of Urls to check for RSS feed”
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note3
- **Type and technical role:** `stickyNote`; high-level workflow explanation.
- **Configuration choices:** Contains a full description of how the process works and setup steps.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.2 Website Inspection and RSS Feed Discovery

### Overview
This block iterates over each source URL, downloads the page HTML, attempts to discover an RSS/Atom feed link via regex, and routes only valid feed URLs forward.

### Nodes Involved
- Fetch Each URL
- Get Website Data
- Check RSS URLs
- Fetch RSS URLs
- Each RSS URL
- Stop if no Url found
- Sticky Note1
- Sticky Note2

### Node Details

#### Fetch Each URL
- **Type and technical role:** `code`; reshapes each input row into a normalized field for HTTP fetching.
- **Configuration choices:** Runs once for each item. Returns:
  - `websiteUrl: $json.URLs`
- **Key expressions or variables used:** `$json.URLs`
- **Input and output connections:** Input from `RSS URLs`; output to `Get Website Data`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - If `URLs` is missing or empty, downstream HTTP request fails
  - Variable `websites` is declared but unused
- **Sub-workflow reference:** None.

#### Get Website Data
- **Type and technical role:** `httpRequest`; fetches webpage content for inspection.
- **Configuration choices:** URL is dynamic: `={{ $json.websiteUrl }}`. No extra request options are set.
- **Key expressions or variables used:** `$json.websiteUrl`
- **Input and output connections:** Input from `Fetch Each URL`; output to `Check RSS URLs`.
- **Version-specific requirements:** Type version 4.3.
- **Edge cases or potential failure types:**
  - Invalid URL
  - DNS/network failures
  - SSL/TLS issues
  - Redirect behavior depending on defaults
  - Non-HTML responses
  - Sites blocking bots or requiring headers/user-agent
- **Sub-workflow reference:** None.

#### Check RSS URLs
- **Type and technical role:** `code`; parses raw HTML response to detect RSS/Atom links.
- **Configuration choices:** Runs once for each item. Looks in `$json["data"]` for HTML, uses regex:
  - `<link rel="alternate" type="application/rss+xml|application/atom+xml" href="...">`
- **Key expressions or variables used:**
  - `$json["data"]`
  - `$('Fetch Each URL').item.json.websiteUrl`
- **Input and output connections:** Input from `Get Website Data`; output to `Fetch RSS URLs`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Assumes HTTP Request output body is in `data`; depending on node settings/response format, this may differ
  - Regex may miss feeds if attributes are ordered differently or HTML is non-standard
  - Relative feed URL handling is simplistic: `websiteUrl + relativePath` may produce malformed URLs
  - Pages with multiple RSS links only capture the first match
  - Non-HTML pages return `rssFeedFound: false`
- **Sub-workflow reference:** None.

#### Fetch RSS URLs
- **Type and technical role:** `if`; filters discovered results based on `rssFeedFound`.
- **Configuration choices:** Condition checks `={{ $json.rssFeedFound }}` equals `true`.
- **Key expressions or variables used:** `$json.rssFeedFound`
- **Input and output connections:**
  - Input from `Check RSS URLs`
  - True output to `Each RSS URL`
  - False output to `Stop if no Url found`
- **Version-specific requirements:** Type version 2.2.
- **Edge cases or potential failure types:**
  - Strict boolean comparison means string `"true"` would fail
- **Sub-workflow reference:** None.

#### Each RSS URL
- **Type and technical role:** `code`; normalizes discovered feed URL into `RssUrl`.
- **Configuration choices:** Runs once per item. Returns:
  - `RssUrl: $json.rssFeedUrl`
- **Key expressions or variables used:** `$json.rssFeedUrl`
- **Input and output connections:** Input from `Fetch RSS URLs` true branch; output to `RSS Read`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Variable `websites` is declared but unused
  - Missing `rssFeedUrl` breaks downstream RSS reader
- **Sub-workflow reference:** None.

#### Stop if no Url found
- **Type and technical role:** `stopAndError`; terminates the branch when no RSS feed was found.
- **Configuration choices:** Error message: `RSS field not found for this URL`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Fetch RSS URLs` false branch; no output.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - In multi-item execution, one bad source URL may stop execution for that branch and potentially the run depending on workflow settings
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `stickyNote`; visual comment.
- **Configuration choices:** Content: “Get Each one using this code”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `stickyNote`; visual documentation.
- **Configuration choices:** Describes feed detection and latest-article retrieval logic.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.3 RSS Reading and Latest-Article Filtering

### Overview
This block reads items from each valid RSS feed and keeps only those published during the current UTC hour. If no such article exists, the workflow stops for that branch.

### Nodes Involved
- RSS Read
- Latest Uploaded Articles
- Stop if no Article found

### Node Details

#### RSS Read
- **Type and technical role:** `rssFeedRead`; fetches and parses feed items from an RSS/Atom endpoint.
- **Configuration choices:** URL is `={{ $json.RssUrl }}`.
- **Key expressions or variables used:** `$json.RssUrl`
- **Input and output connections:** Input from `Each RSS URL`; output to `Latest Uploaded Articles`.
- **Version-specific requirements:** Type version 1.2.
- **Edge cases or potential failure types:**
  - Invalid or inaccessible feed URL
  - Unsupported feed format
  - Empty feed
  - Missing `pubDate`, `link`, `title`, or `contentSnippet` fields
- **Sub-workflow reference:** None.

#### Latest Uploaded Articles
- **Type and technical role:** `if`; filters feed items to the current UTC hour.
- **Configuration choices:** Compares:
  - `DateTime.fromISO($json.pubDate).toUTC().toFormat('HH')`
  - `new Date().getUTCHours().toString()`
- **Key expressions or variables used:** `$json.pubDate`
- **Input and output connections:**
  - Input from `RSS Read`
  - True output to `Fetched Article`
  - False output to `Stop if no Article found`
- **Version-specific requirements:** Type version 2.2. Relies on Luxon `DateTime` support in n8n expressions.
- **Edge cases or potential failure types:**
  - `pubDate` may not be ISO-formatted; many RSS feeds use RFC822 dates, so `DateTime.fromISO(...)` may fail or return invalid values
  - Hour-only filtering ignores date, so an old article from the same UTC hour on a different day could pass if present
  - String comparison may fail if hour formatting differs (`"08"` vs `"8"`)
- **Sub-workflow reference:** None.

#### Stop if no Article found
- **Type and technical role:** `stopAndError`; stops when no article matches the time filter.
- **Configuration choices:** Error message: `No latest Article is found`
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Latest Uploaded Articles` false branch.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**
  - May be overly aggressive if feeds publish less frequently than hourly
- **Sub-workflow reference:** None.

---

## 2.4 Article Normalization

### Overview
This block standardizes the selected feed item fields into a compact shape used by all later duplicate-check and storage nodes.

### Nodes Involved
- Fetched Article
- Sticky Note4

### Node Details

#### Fetched Article
- **Type and technical role:** `set`; maps RSS item fields into destination schema.
- **Configuration choices:** Assigns:
  - `link = $json.link`
  - `title = $json.title`
  - `article = $json.contentSnippet`
- **Key expressions or variables used:**
  - `$json.link`
  - `$json.title`
  - `$json.contentSnippet`
- **Input and output connections:** Input from `Latest Uploaded Articles` true branch; output to `Fetch title`.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - If `contentSnippet` is absent, `article` may be empty
  - HTML content may be truncated or plain text depending on feed parser output
- **Sub-workflow reference:** None.

#### Sticky Note4
- **Type and technical role:** `stickyNote`; visual note.
- **Configuration choices:** Content: “Format Data”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.5 Existing Article Lookup

### Overview
This block reads previously stored entries from the destination Google Sheet and branches depending on whether the sheet already contains data.

### Nodes Involved
- Fetch title
- Check if data is empty
- Get Each item
- Add data in sheets
- Sticky Note5
- Sticky Note6

### Node Details

#### Fetch title
- **Type and technical role:** `googleSheets`; reads existing stored article rows from the destination sheet.
- **Configuration choices:** Uses the **RSS Fetched Feed** spreadsheet, `Sheet1` (`gid=0`). `alwaysOutputData` is enabled, so the node returns output even if nothing is found.
- **Key expressions or variables used:** None directly.
- **Input and output connections:** Input from `Fetched Article`; output to `Check if data is empty`.
- **Version-specific requirements:** Type version 4.7; requires Google Sheets OAuth2 credentials.
- **Edge cases or potential failure types:**
  - Auth issues
  - Missing spreadsheet or sheet
  - Empty sheet handling depends on node behavior and output structure
- **Sub-workflow reference:** None.

#### Check if data is empty
- **Type and technical role:** `if`; determines whether the fetched destination-sheet payload is empty.
- **Configuration choices:** Tests whether `={{ $json }}` is an empty object.
- **Key expressions or variables used:** `$json`
- **Input and output connections:**
  - Input from `Fetch title`
  - True output to `Get Each item`
  - False output to `Get Titles List`
- **Version-specific requirements:** Type version 2.2.
- **Edge cases or potential failure types:**
  - This depends on exactly how the Google Sheets node represents “no rows”
  - If the node returns metadata instead of an empty object, this check may not behave as intended
- **Sub-workflow reference:** None.

#### Get Each item
- **Type and technical role:** `code`; prepares fetched article items for first-time sheet insertion.
- **Configuration choices:** Reads all items from `Fetched Article`, converts each to `{title, link, article}` items.
- **Key expressions or variables used:** `$('Fetched Article').all()`
- **Input and output connections:** Input from `Check if data is empty` true branch; output to `Add data in sheets`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Ignores current input and depends entirely on `Fetched Article`
  - If `Fetched Article` contains multiple items from multiple feeds, all are appended
- **Sub-workflow reference:** None.

#### Add data in sheets
- **Type and technical role:** `googleSheets`; appends articles when destination sheet is considered empty.
- **Configuration choices:** Appends `link`, `title`, `article` to spreadsheet **RSS Fetched Feed**, `Sheet1`.
- **Key expressions or variables used:**
  - `$json.link`
  - `$json.title`
  - `$json.article`
- **Input and output connections:** Input from `Get Each item`; no downstream connection.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases or potential failure types:**
  - Auth or quota issues
  - Schema mismatch if target columns are renamed
  - Duplicates are not checked in this branch because it assumes sheet is empty
- **Sub-workflow reference:** None.

#### Sticky Note5
- **Type and technical role:** `stickyNote`; visual comment.
- **Configuration choices:** Content: “Fetch existing title from sheets and check that it exists”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note6
- **Type and technical role:** `stickyNote`; visual comment.
- **Configuration choices:** Content: “Get each item and store in sheet.”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.6 Duplicate Detection and Branching

### Overview
This block builds a unique list of existing titles, compares incoming article titles against it, and routes the workflow either to a “data already exists” status or to a filtered append path for only new titles.

### Nodes Involved
- Get Titles List
- Check if title exits in sheet
- Check if matches
- Get matching titles
- Message
- Sticky Note7
- Sticky Note8
- Sticky Note9

### Node Details

#### Get Titles List
- **Type and technical role:** `code`; aggregates unique existing titles from destination-sheet rows.
- **Configuration choices:** Iterates through all input items and builds `titlesList`.
- **Key expressions or variables used:** `items[i].json.title`
- **Input and output connections:** Input from `Check if data is empty` false branch; output to `Check if title exits in sheet`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Rows without `title` create `undefined` entries
  - Case-sensitive duplicate handling
  - Entire existing title set is collapsed into one item
- **Sub-workflow reference:** None.

#### Check if title exits in sheet
- **Type and technical role:** `code`; compares fetched article titles against the existing `titlesList`.
- **Configuration choices:** Reads:
  - all new titles from `Fetched Article`
  - existing titles from current item `titlesList`
  Returns:
  - `allTitlesFound`
  - `notFoundTitles`
- **Key expressions or variables used:**
  - `$('Fetched Article').all().map(item => item.json.title)`
  - `$json["titlesList"]`
- **Input and output connections:** Input from `Get Titles List`; output to `Check if matches`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Title comparison is exact and case-sensitive
  - Whitespace or formatting changes can bypass duplicate detection
  - If multiple new items exist and at least one is missing, `allTitlesFound` becomes false
- **Sub-workflow reference:** None.

#### Check if matches
- **Type and technical role:** `if`; routes execution based on whether all fetched titles already exist.
- **Configuration choices:** Checks whether `={{ $json.allTitlesFound }}` is true.
- **Key expressions or variables used:** `$json.allTitlesFound`
- **Input and output connections:**
  - Input from `Check if title exits in sheet`
  - True output to `Message`
  - False output to `Get matching titles`
- **Version-specific requirements:** Type version 2.2.
- **Edge cases or potential failure types:**
  - The configured right-side expression is irrelevant for a unary true check, but present in JSON
- **Sub-workflow reference:** None.

#### Message
- **Type and technical role:** `set`; outputs a status payload when all titles already exist.
- **Configuration choices:** Sets `message = "Data already exists"`.
- **Key expressions or variables used:** None.
- **Input and output connections:** Input from `Check if matches` true branch; no downstream node.
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Get matching titles
- **Type and technical role:** `code`; selects only fetched articles whose titles are listed in `notFoundTitles`.
- **Configuration choices:** Cross-references `notFoundTitles` against all items from `Fetched Article`, returns only matching article objects.
- **Key expressions or variables used:**
  - `$json["notFoundTitles"]`
  - `$('Fetched Article').all()`
- **Input and output connections:** Input from `Check if matches` false branch; output to `Append row in sheet`.
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Exact title match only
  - If `notFoundTitles` is empty despite false branch, no rows will be returned
- **Sub-workflow reference:** None.

#### Sticky Note7
- **Type and technical role:** `stickyNote`; visual comment.
- **Configuration choices:** Content describes storing titles in a list and filtering new ones.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note8
- **Type and technical role:** `stickyNote`; visual comment.
- **Configuration choices:** Content: “New titles matching with fetched title and store in sheet.”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

#### Sticky Note9
- **Type and technical role:** `stickyNote`; visual comment.
- **Configuration choices:** Content: “Add message that data already exits.”
- **Input and output connections:** None.
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.

---

## 2.7 Storage and Final Output

### Overview
This block appends only the newly detected articles to the destination Google Sheet.

### Nodes Involved
- Append row in sheet

### Node Details

#### Append row in sheet
- **Type and technical role:** `googleSheets`; appends filtered non-duplicate article rows.
- **Configuration choices:** Operation is `append`; writes `link`, `title`, and `article` into spreadsheet **RSS Fetched Feed**, `Sheet1`.
- **Key expressions or variables used:**
  - `$json.link`
  - `$json.title`
  - `$json.article`
- **Input and output connections:** Input from `Get matching titles`; no downstream node.
- **Version-specific requirements:** Type version 4.7.
- **Edge cases or potential failure types:**
  - Auth issues
  - Quota/rate limits
  - Column schema drift
  - If duplicates slip through due to title variations, they will still be appended
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| When clicking ‘Execute workflow’ | manualTrigger | Manual workflow start |  | RSS URLs | ## How it works  1.Checking each URL from a list.  2.Identifying which URLs contain a valid RSS feed.  3.Fetching the latest articles from those identified URLs.  4.Stopping the workflow if no valid article is found.  ## Setup Steps  1.Trigger: Workflow starts when you click "Execute Workflow."  2.Fetch RSS URLs: Retrieves URLs from Google Sheets.  3.Check for RSS Feed: Validates whether each URL contains a valid RSS feed.  4.Fetch Latest Articles: For valid feeds, the workflow fetches the latest articles.  5.Fetch Existing Titles: Pulls existing titles from Google Sheets to compare against new titles.  6.Check for Duplicates: Compares new titles with existing ones. If duplicate, skips; if not, proceeds.  7.Store Non-Duplicate Titles: Adds non-duplicate articles to Google Sheets.  8.Message Node: Optionally logs duplicate data (e.g., "Data already exists").  9.Stop if No Articles: Stops the workflow if no new articles are found. |
| RSS URLs | googleSheets | Read source website URLs from Google Sheets | When clicking ‘Execute workflow’ | Fetch Each URL | List of Urls to check for RSS feed |
| Sticky Note | stickyNote | Visual annotation |  |  |  |
| Sticky Note2 | stickyNote | Visual annotation |  |  |  |
| Sticky Note1 | stickyNote | Visual annotation |  |  |  |
| Fetch Each URL | code | Normalize source row into `websiteUrl` | RSS URLs | Get Website Data | Get Each one using this code |
| Fetched Article | set | Normalize feed item fields | Latest Uploaded Articles | Fetch title | Format Data |
| Stop if no Article found | stopAndError | Stop when no current-hour article is found | Latest Uploaded Articles |  | Format Data |
| Sticky Note3 | stickyNote | Visual annotation |  |  |  |
| RSS URLs | googleSheets | Read source website URLs from Google Sheets | When clicking ‘Execute workflow’ | Fetch Each URL | ## How it works  1.Checking each URL from a list.  2.Identifying which URLs contain a valid RSS feed.  3.Fetching the latest articles from those identified URLs.  4.Stopping the workflow if no valid article is found.  ## Setup Steps  1.Trigger: Workflow starts when you click "Execute Workflow."  2.Fetch RSS URLs: Retrieves URLs from Google Sheets.  3.Check for RSS Feed: Validates whether each URL contains a valid RSS feed.  4.Fetch Latest Articles: For valid feeds, the workflow fetches the latest articles.  5.Fetch Existing Titles: Pulls existing titles from Google Sheets to compare against new titles.  6.Check for Duplicates: Compares new titles with existing ones. If duplicate, skips; if not, proceeds.  7.Store Non-Duplicate Titles: Adds non-duplicate articles to Google Sheets.  8.Message Node: Optionally logs duplicate data (e.g., "Data already exists").  9.Stop if No Articles: Stops the workflow if no new articles are found. |
| Get Website Data | httpRequest | Fetch website HTML for RSS discovery | Fetch Each URL | Check RSS URLs | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |
| Check RSS URLs | code | Detect RSS/Atom feed URL in HTML | Get Website Data | Fetch RSS URLs | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |
| Fetch RSS URLs | if | Branch on whether a feed URL was found | Check RSS URLs | Each RSS URL; Stop if no Url found | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |
| Latest Uploaded Articles | if | Keep only current-hour articles | RSS Read | Fetched Article; Stop if no Article found | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |
| Add data in sheets | googleSheets | Append items when destination sheet is empty | Get Each item |  | Get each item and store in sheet. |
| Sticky Note4 | stickyNote | Visual annotation |  |  |  |
| Fetch title | googleSheets | Read existing stored article rows | Fetched Article | Check if data is empty | Fetch existing title from sheets and check that it exists |
| Get Titles List | code | Build unique list of existing titles | Check if data is empty | Check if title exits in sheet | Store existing title in a list and check that if any new titles are exiting if not then store new title in a seperate list. |
| Check if data is empty | if | Decide whether destination sheet already has rows | Fetch title | Get Each item; Get Titles List | Fetch existing title from sheets and check that it exists |
| Get Each item | code | Convert fetched articles into append-ready rows | Check if data is empty | Add data in sheets | Get each item and store in sheet. |
| Append row in sheet | googleSheets | Append only non-duplicate new articles | Get matching titles |  | New titles matching with fetched title and store in sheet. |
| Check if title exits in sheet | code | Compare fetched titles against existing title list | Get Titles List | Check if matches | Store existing title in a list and check that if any new titles are exiting if not then store new title in a seperate list. |
| Stop if no Url found | stopAndError | Stop when no RSS feed is found for a URL | Fetch RSS URLs |  | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |
| Each RSS URL | code | Normalize discovered feed URL into `RssUrl` | Fetch RSS URLs | RSS Read | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |
| Get matching titles | code | Filter fetched articles to only new titles | Check if matches | Append row in sheet | New titles matching with fetched title and store in sheet. |
| Message | set | Output duplicate-status message | Check if matches |  | Add message that data already exits. |
| Check if matches | if | Branch on whether all fetched titles already exist | Check if title exits in sheet | Message; Get matching titles | Store existing title in a list and check that if any new titles are exiting if not then store new title in a seperate list. |
| Sticky Note5 | stickyNote | Visual annotation |  |  |  |
| Sticky Note6 | stickyNote | Visual annotation |  |  |  |
| Sticky Note7 | stickyNote | Visual annotation |  |  |  |
| Sticky Note8 | stickyNote | Visual annotation |  |  |  |
| Sticky Note9 | stickyNote | Visual annotation |  |  |  |
| RSS Read | rssFeedRead | Read articles from discovered RSS feed | Each RSS URL | Latest Uploaded Articles | Fetch data from a specific URL, most likely to retrieve the RSS feed,check which urls have RSS feed and returns true then pass only those urls whose RSS flag is true. Then check for the latest articles published if not found then stop the workflow. |

Note: the workflow contains two Google Sheets nodes named differently but targeting distinct spreadsheets:
- `RSS URLs` reads source URLs
- `Fetch title`, `Add data in sheets`, and `Append row in sheet` target the destination article sheet

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `RSS Feed Latest Article Fetcher`.

2. **Add a Manual Trigger node**
   - Node type: **Manual Trigger**
   - Keep default configuration.
   - This will be the entry point.

3. **Prepare the source Google Sheet**
   - Create a spreadsheet for source websites.
   - Add a sheet, typically `Sheet1`.
   - Ensure it contains a column named exactly: `URLs`
   - Each row should contain one website homepage URL.

4. **Add a Google Sheets node to read source URLs**
   - Node type: **Google Sheets**
   - Name: `RSS URLs`
   - Credential: configure **Google Sheets OAuth2**
   - Select the source spreadsheet
   - Select the source sheet (`Sheet1`)
   - Set the node to read rows
   - Connect `Manual Trigger` → `RSS URLs`

5. **Add a Code node to normalize source URLs**
   - Node type: **Code**
   - Name: `Fetch Each URL`
   - Mode: **Run Once for Each Item**
   - Use this logic:
     - read `$json.URLs`
     - output `{ websiteUrl: $json.URLs }`
   - Connect `RSS URLs` → `Fetch Each URL`

6. **Add an HTTP Request node to fetch each website**
   - Node type: **HTTP Request**
   - Name: `Get Website Data`
   - URL: `={{ $json.websiteUrl }}`
   - Keep default method `GET`
   - Keep response format compatible with downstream code expecting page content
   - Connect `Fetch Each URL` → `Get Website Data`

7. **Add a Code node to detect RSS feed URLs**
   - Node type: **Code**
   - Name: `Check RSS URLs`
   - Mode: **Run Once for Each Item**
   - Configure logic to:
     - read raw HTML content from the HTTP node output
     - test it with a regex for:
       - `application/rss+xml`
       - `application/atom+xml`
     - if found, output:
       - `rssFeedFound: true`
       - `rssFeedUrl: <resolved url>`
     - otherwise output:
       - `rssFeedFound: false`
   - Also reference the original website URL for relative feed paths.
   - Connect `Get Website Data` → `Check RSS URLs`

8. **Add an IF node to branch on feed presence**
   - Node type: **IF**
   - Name: `Fetch RSS URLs`
   - Condition:
     - Boolean
     - `={{ $json.rssFeedFound }}`
     - Equals `true`
   - Connect `Check RSS URLs` → `Fetch RSS URLs`

9. **Add a Stop and Error node for websites without RSS**
   - Node type: **Stop And Error**
   - Name: `Stop if no Url found`
   - Error message: `RSS field not found for this URL`
   - Connect `Fetch RSS URLs` false output → `Stop if no Url found`

10. **Add a Code node to normalize discovered RSS URLs**
    - Node type: **Code**
    - Name: `Each RSS URL`
    - Mode: **Run Once for Each Item**
    - Output:
      - `RssUrl: $json.rssFeedUrl`
    - Connect `Fetch RSS URLs` true output → `Each RSS URL`

11. **Add an RSS Feed Read node**
    - Node type: **RSS Feed Read**
    - Name: `RSS Read`
    - URL: `={{ $json.RssUrl }}`
    - Connect `Each RSS URL` → `RSS Read`

12. **Add an IF node to keep only latest-hour articles**
    - Node type: **IF**
    - Name: `Latest Uploaded Articles`
    - Add a string comparison condition:
      - Left: `={{ DateTime.fromISO($json.pubDate).toUTC().toFormat('HH') }}`
      - Right: `={{ new Date().getUTCHours().toString() }}`
    - Connect `RSS Read` → `Latest Uploaded Articles`

13. **Add a Stop and Error node for no latest article**
    - Node type: **Stop And Error**
    - Name: `Stop if no Article found`
    - Error message: `No latest Article is found`
    - Connect `Latest Uploaded Articles` false output → `Stop if no Article found`

14. **Add a Set node to format article data**
    - Node type: **Set**
    - Name: `Fetched Article`
    - Add fields:
      - `link` = `={{ $json.link }}`
      - `title` = `={{ $json.title }}`
      - `article` = `={{ $json.contentSnippet }}`
    - Connect `Latest Uploaded Articles` true output → `Fetched Article`

15. **Prepare the destination Google Sheet**
    - Create another spreadsheet for stored article records.
    - Add columns:
      - `title`
      - `link`
      - `article`
    - Keep column names consistent with append mappings.

16. **Add a Google Sheets node to read existing destination rows**
    - Node type: **Google Sheets**
    - Name: `Fetch title`
    - Credential: same or another **Google Sheets OAuth2** credential
    - Select destination spreadsheet
    - Select destination sheet
    - Enable **Always Output Data**
    - Set it to read rows
    - Connect `Fetched Article` → `Fetch title`

17. **Add an IF node to detect whether the sheet data is empty**
    - Node type: **IF**
    - Name: `Check if data is empty`
    - Condition:
      - Object
      - `={{ $json }}`
      - Is empty
    - Connect `Fetch title` → `Check if data is empty`

18. **Add a Code node for first-time insertion when the destination sheet is empty**
    - Node type: **Code**
    - Name: `Get Each item`
    - Logic:
      - collect all items from `Fetched Article`
      - emit one item per article with:
        - `title`
        - `link`
        - `article`
    - Connect `Check if data is empty` true output → `Get Each item`

19. **Add a Google Sheets append node for empty-sheet writes**
    - Node type: **Google Sheets**
    - Name: `Add data in sheets`
    - Operation: **Append**
    - Target: destination spreadsheet and sheet
    - Map columns:
      - `link` = `={{ $json.link }}`
      - `title` = `={{ $json.title }}`
      - `article` = `={{ $json.article }}`
    - Connect `Get Each item` → `Add data in sheets`

20. **Add a Code node to aggregate existing titles**
    - Node type: **Code**
    - Name: `Get Titles List`
    - Logic:
      - iterate over all incoming sheet rows
      - extract `json.title`
      - deduplicate titles
      - return one item containing `titlesList`
    - Connect `Check if data is empty` false output → `Get Titles List`

21. **Add a Code node to compare fetched titles with existing titles**
    - Node type: **Code**
    - Name: `Check if title exits in sheet`
    - Logic:
      - read all titles from `Fetched Article`
      - read existing `titlesList`
      - compute:
        - `allTitlesFound`
        - `notFoundTitles`
    - Connect `Get Titles List` → `Check if title exits in sheet`

22. **Add an IF node for duplicate handling**
    - Node type: **IF**
    - Name: `Check if matches`
    - Condition:
      - Boolean
      - `={{ $json.allTitlesFound }}`
      - Is true
    - Connect `Check if title exits in sheet` → `Check if matches`

23. **Add a Set node for duplicate message output**
    - Node type: **Set**
    - Name: `Message`
    - Add field:
      - `message` = `Data already exists`
    - Connect `Check if matches` true output → `Message`

24. **Add a Code node to keep only new titles**
    - Node type: **Code**
    - Name: `Get matching titles`
    - Logic:
      - read `notFoundTitles`
      - compare them with all items from `Fetched Article`
      - output only matching article items with:
        - `link`
        - `title`
        - `article`
    - Connect `Check if matches` false output → `Get matching titles`

25. **Add a final Google Sheets append node**
    - Node type: **Google Sheets**
    - Name: `Append row in sheet`
    - Operation: **Append**
    - Target: destination spreadsheet and sheet
    - Map:
      - `link` = `={{ $json.link }}`
      - `title` = `={{ $json.title }}`
      - `article` = `={{ $json.article }}`
    - Connect `Get matching titles` → `Append row in sheet`

26. **Add sticky notes if desired**
    - Optional visual notes used in the original workflow:
      - list of URLs
      - RSS detection explanation
      - formatting explanation
      - duplicate checking explanation
      - append explanation
      - status message explanation

27. **Configure credentials**
    - For every Google Sheets node:
      - create or select a **Google Sheets OAuth2 API** credential
      - ensure the account has read/write access to both spreadsheets
    - No credential is required for RSS Feed Read unless the feed is private.
    - No credential is required for HTTP Request in this workflow unless the target websites require auth.

28. **Validate source and destination schemas**
    - Source sheet must contain `URLs`
    - Destination sheet should contain `title`, `link`, `article`
    - Keep names exact to avoid mapping and code issues

29. **Test with one URL first**
    - Use a website known to expose an RSS feed
    - Verify the HTTP Request output field name; if the HTML is not in `data`, update the `Check RSS URLs` code accordingly
    - Verify the feed exposes `pubDate`, `title`, `link`, and `contentSnippet`

30. **Harden the workflow before production use**
    - Improve relative RSS URL resolution with a proper URL join function
    - Replace `DateTime.fromISO($json.pubDate)` with a parser that supports RSS date formats
    - Consider using article `link` as a stronger deduplication key than `title`
    - Replace stop nodes with graceful skip logic if you want the workflow to continue across mixed-validity URLs

### Sub-workflow setup
This workflow does **not** use sub-workflows and does not invoke other workflows.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow uses two separate Google Sheets files: one as the source list of website URLs, and one as the destination store of fetched articles. | Source intake and article storage design |
| Duplicate detection is based only on article title equality. This is simple but can miss near-duplicates or allow duplicates when titles vary slightly. | Data quality / deduplication behavior |
| The current “latest article” filter compares only UTC hour, not full timestamp recency. | Time filtering logic |
| The RSS discovery code assumes website HTML is available in the HTTP Request output field named `data`. | Important implementation assumption |
| The workflow contains several Stop and Error nodes, so one failure branch may interrupt expected bulk-processing behavior. | Execution behavior |
| Sticky note content includes a high-level operating description and setup sequence directly inside the canvas. | Embedded project documentation |