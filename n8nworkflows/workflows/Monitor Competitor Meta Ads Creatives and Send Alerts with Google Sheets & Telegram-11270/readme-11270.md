Monitor Competitor Meta Ads Creatives and Send Alerts with Google Sheets & Telegram

https://n8nworkflows.xyz/workflows/monitor-competitor-meta-ads-creatives-and-send-alerts-with-google-sheets---telegram-11270


# Monitor Competitor Meta Ads Creatives and Send Alerts with Google Sheets & Telegram

### 1. Workflow Overview

This workflow is designed to monitor competitors' Meta (Facebook) ads creatives continuously and send alerts when new creatives are detected. It targets digital marketers, social media analysts, and competitive intelligence teams who want to track ad creatives from specific Facebook Pages or keyword searches. The workflow fetches ads data from the Meta Ads Library API, compares new creatives against a Google Sheets database, updates the sheet with new entries, and sends notifications through Telegram and Slack.

**Logical Blocks:**

- **1.1 Input Initialization:** Scheduling and setting parameters such as pages, keywords, countries, and API tokens.
- **1.2 Data Retrieval:** Calls to Meta Ads Library API using either page IDs or keywords, including handling pagination.
- **1.3 Data Processing & Filtering:** Extracting ad data, comparing with existing records in Google Sheets, and filtering new creatives.
- **1.4 Data Storage:** Adding new creatives to Google Sheets.
- **1.5 Notification:** Sending alerts about new creatives found to Telegram and Slack.
- **1.6 Control & Looping:** Decision nodes managing flow control and pagination looping.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Initialization

**Overview:**  
This block triggers the workflow on a schedule and initializes parameters including Facebook Ads API token, page IDs, keywords, and countries to search.

**Nodes Involved:**  
- Schedule Trigger  
- Add parameters

**Node Details:**  

- **Schedule Trigger**  
  - Type: Trigger (Schedule)  
  - Config: Runs automatically on a regular interval (default cron-like schedule; exact interval not specified)  
  - Inputs: None (trigger node)  
  - Outputs: Starts the workflow chain to Add parameters  
  - Edge cases: Misconfigured schedule can cause missed or excessive runs.

- **Add parameters**  
  - Type: Set node  
  - Config: Defines fixed parameters like `ad_active_status` = "active", `search_page_ids`, `ad_reached_countries`, `keywords` (empty by default), and `access_token` (placeholder).  
  - Inputs: From Schedule Trigger  
  - Outputs: To Read existing IDs and Page or keywords switch  
  - Edge cases: Missing or invalid access token will cause API failures; empty keywords or page IDs require proper handling downstream.

---

#### 1.2 Data Retrieval

**Overview:**  
Fetches ads from Meta Ads Library API based on whether to search by page IDs or keywords, then handles pagination to fetch all available ads.

**Nodes Involved:**  
- Page or keywords (Switch)  
- Facebook Ads API by page (HTTP Request)  
- Facebook Ads API by keywords (HTTP Request)  
- Check the pagination (Code)  
- If (conditional check for pagination)  
- Set Next URL (Set)  
- Facebook Ads API pagination (HTTP Request)

**Node Details:**  

- **Page or keywords**  
  - Type: Switch  
  - Config: Checks if `search_page_ids` is not empty → routes to page-based API call; else if `keywords` is not empty → routes to keyword-based API call.  
  - Inputs: From Add parameters  
  - Outputs: To either Facebook Ads API by page or Facebook Ads API by keywords  
  - Edge cases: Both inputs empty leads to no data retrieval; case-sensitive strict string validation.

- **Facebook Ads API by page**  
  - Type: HTTP Request  
  - Config: Calls Facebook Ads API with page IDs, countries, fields, and access token.  
  - Authentication: Uses HTTP Header Auth with Meta Ads credentials.  
  - Inputs: From Page or keywords node (pages output)  
  - Outputs: To Check the pagination  
  - Edge cases: API rate limiting, invalid token, malformed parameters.

- **Facebook Ads API by keywords**  
  - Type: HTTP Request  
  - Config: Calls Facebook Ads API with keywords, fixed countries (US, CA), fields, and access token.  
  - Authentication: Same as above.  
  - Inputs: From Page or keywords node (keywords output)  
  - Outputs: To Check the pagination  
  - Edge cases: Same as above; note fixed countries may limit results.

- **Check the pagination**  
  - Type: Code  
  - Config: Extracts ads data and checks if there is a `next` URL for pagination. Returns normalized object with `data` and `next_url`.  
  - Inputs: From Facebook Ads API nodes  
  - Outputs: Two paths: next_url exists → If node; else → Attach existing ids  
  - Edge cases: Unexpected API response formats, empty data arrays.

- **If**  
  - Type: If (conditional)  
  - Config: Checks if `next_url` is not empty to continue pagination.  
  - Inputs: From Check the pagination  
  - Outputs: Next_url exists → Set Next URL; else → Attach existing ids  
  - Edge cases: Infinite loops if next_url never clears; malformed URLs.

- **Set Next URL**  
  - Type: Set  
  - Config: Sets `url` parameter to `next_url` for the next API pagination request.  
  - Inputs: From If node (true path)  
  - Outputs: To Facebook Ads API pagination node  
  - Edge cases: Invalid URL values.

- **Facebook Ads API pagination**  
  - Type: HTTP Request  
  - Config: Calls Facebook Ads API with the `url` from Set Next URL node to fetch next page of ads.  
  - Inputs: From Set Next URL  
  - Outputs: Loops back to Check the pagination  
  - Edge cases: Same API errors as initial calls.

---

#### 1.3 Data Processing & Filtering

**Overview:**  
Processes the retrieved ad data by comparing with existing IDs stored in Google Sheets, filters out duplicates, and prepares only new creatives for storage and notification.

**Nodes Involved:**  
- Read existing IDs (Google Sheets)  
- Collect ID list (Code)  
- Attach existing ids (Merge)  
- Filter new creatives (Code)  
- Split Out

**Node Details:**  

- **Read existing IDs**  
  - Type: Google Sheets (Read)  
  - Config: Reads column J (IDs) from a specified Google Sheet document and sheet.  
  - Credentials: Google Sheets OAuth2  
  - Inputs: From Add parameters  
  - Outputs: To Collect ID list  
  - Edge cases: Authentication errors, sheet range errors, empty sheets.

- **Collect ID list**  
  - Type: Code  
  - Config: Extracts and normalizes IDs to strings from the existing sheet data, removing duplicates.  
  - Inputs: From Read existing IDs  
  - Outputs: To Attach existing ids (merge node)  
  - Edge cases: Missing or malformed IDs.

- **Attach existing ids**  
  - Type: Merge (Combine mode)  
  - Config: Combines API ad data with existing IDs for comparison.  
  - Inputs: From Check the pagination (ads data) and Collect ID list (existing IDs)  
  - Outputs: To Filter new creatives  
  - Edge cases: Data mismatch, empty inputs.

- **Filter new creatives**  
  - Type: Code  
  - Config: Compares the ads from API with existing IDs and filters only the new ads not found before.  
  - Inputs: From Attach existing ids  
  - Outputs: To Split Out  
  - Edge cases: Missing ad IDs, duplicates within the same run.

- **Split Out**  
  - Type: Split Out  
  - Config: Splits array of new ads into individual items for processing one by one.  
  - Inputs: From Filter new creatives  
  - Outputs: To Add to sheet and Count new ads  
  - Edge cases: Empty arrays produce no outputs.

---

#### 1.4 Data Storage

**Overview:**  
Appends new ads to the Google Sheet, updating existing entries or adding new rows.

**Nodes Involved:**  
- Add to sheet

**Node Details:**  

- **Add to sheet**  
  - Type: Google Sheets (Append or Update)  
  - Config: Maps ad fields such as id, page name, title, description, snapshot URL, languages, platforms, creation and delivery times to Google Sheet columns. Uses "appendOrUpdate" to avoid duplicates.  
  - Credentials: Google Sheets OAuth2  
  - Inputs: From Split Out  
  - Outputs: To Count new ads  
  - Edge cases: Column mismatch, API rate limits, improper mapping causing data loss.

---

#### 1.5 Notification

**Overview:**  
Counts new creatives and sends notifications to Telegram and Slack if any new ads are found.

**Nodes Involved:**  
- Count new ads  
- Any new ads? (If)  
- Send a text message (Telegram)  
- Send a message (Slack)

**Node Details:**  

- **Count new ads**  
  - Type: Code  
  - Config: Returns count of new ads found (`newCount`) for conditional checks and messaging.  
  - Inputs: From Add to sheet  
  - Outputs: To Any new ads?  
  - Edge cases: No new ads leads to zero count.

- **Any new ads?**  
  - Type: If  
  - Config: Checks if `newCount` is greater than 0.  
  - Inputs: From Count new ads  
  - Outputs: True → Send message nodes; False → End workflow  
  - Edge cases: False negatives if count is miscalculated.

- **Send a text message (Telegram)**  
  - Type: Telegram node  
  - Config: Sends a text message with the number of new ads and a link to the Google Sheets report. Uses chat ID and credentials for Telegram API.  
  - Credentials: Telegram API credentials  
  - Inputs: From Any new ads? (true path)  
  - Outputs: End  
  - Edge cases: Invalid chat ID, network errors.

- **Send a message (Slack)**  
  - Type: Slack node  
  - Config: Sends a message similar to Telegram with the count and link to the Google Sheet. Channel ID must be configured.  
  - Credentials: Slack connection (not detailed in JSON)  
  - Inputs: From Any new ads? (true path)  
  - Outputs: End  
  - Edge cases: Invalid channel, permission issues.

---

#### 1.6 Control & Looping

**Overview:**  
Manages looping to fetch all pages of ads using pagination, ensuring complete data retrieval.

**Nodes Involved:**  
- If (pagination check)  
- Set Next URL  
- Facebook Ads API pagination  
- Check the pagination

**Node Details:**  
Covered in section 1.2, these nodes loop through paginated API results until no further pages remain.

---

### 3. Summary Table

| Node Name                   | Node Type                | Functional Role                      | Input Node(s)              | Output Node(s)                        | Sticky Note                                                                                                            |
|-----------------------------|--------------------------|------------------------------------|----------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger         | Initiates workflow on schedule     |                            | Add parameters                      |                                                                                                                        |
| Add parameters             | Set                      | Sets initial parameters            | Schedule Trigger           | Read existing IDs, Page or keywords | ## Add your parameters - Creative status, Page IDs, Countries, Keywords, Access token                                  |
| Page or keywords           | Switch                   | Routes to API call by pages or keywords | Add parameters            | Facebook Ads API by page, Facebook Ads API by keywords | This node decides whether to check pages or keywords                                                                   |
| Facebook Ads API by page   | HTTP Request             | Fetch ads by page IDs from API     | Page or keywords (pages)   | Check the pagination                | These nodes make API requests to the Meta Ads Library API                                                              |
| Facebook Ads API by keywords| HTTP Request             | Fetch ads by keywords from API     | Page or keywords (keywords)| Check the pagination                | These nodes make API requests to the Meta Ads Library API                                                              |
| Check the pagination       | Code                     | Extracts ads and pagination info   | Facebook Ads API nodes     | If, Attach existing ids             | The API returns only 25 results per page. This node checks for next pages and loops accordingly.                       |
| If (pagination check)      | If                       | Checks if next page exists         | Check the pagination       | Set Next URL (true), Attach existing ids (false) | The API returns only 25 results per page. This node checks for next pages and loops accordingly.                       |
| Set Next URL               | Set                      | Sets next URL for pagination       | If (pagination check)      | Facebook Ads API pagination         | The API returns only 25 results per page. This node checks for next pages and loops accordingly.                       |
| Facebook Ads API pagination| HTTP Request             | Fetches next page of ads           | Set Next URL               | Check the pagination                | The API returns only 25 results per page. This node checks for next pages and loops accordingly.                       |
| Read existing IDs          | Google Sheets             | Reads stored ad IDs from sheet     | Add parameters             | Collect ID list                    | Check which creative IDs already exist in Google Sheets so we only send a notification when new ones are found         |
| Collect ID list            | Code                     | Extracts and normalizes IDs        | Read existing IDs          | Attach existing ids                | Collect IDs for future matching                                                                                        |
| Attach existing ids        | Merge                    | Combines API data with existing IDs| Check the pagination, Collect ID list | Filter new creatives               | Match creatives from API response with those stored in Google Sheets                                                   |
| Filter new creatives       | Code                     | Filters new ads not in sheet       | Attach existing ids        | Split Out                        | Match creatives from API response with those stored in Google Sheets                                                   |
| Split Out                 | Split Out                 | Splits ads array into individual items | Filter new creatives    | Add to sheet, Count new ads        | Split out array data into individual items                                                                             |
| Add to sheet              | Google Sheets            | Updates sheet with new ads         | Split Out                  | Count new ads                     | Add creatives to Google Sheets. Be careful to keep column order and names in sync.                                     |
| Count new ads             | Code                     | Counts new ads found               | Add to sheet               | Any new ads?                     | Count how many new creatives were found for a compact message. Use {{$json.newCount}} in Slack or Telegram messages.   |
| Any new ads?              | If                       | Checks if there are new ads        | Count new ads              | Send a text message, Send a message | Check if there are any new ads before sending notifications                                                            |
| Send a text message       | Telegram                 | Sends Telegram notification       | Any new ads? (true)        |                                 | Send Telegram notification about new creatives                                                                         |
| Send a message            | Slack                    | Sends Slack notification          | Any new ads? (true)        |                                 | Send Slack notification about new creatives                                                                            |
| Sticky Note - Overview    | Sticky Note              | Documentation note                 |                            |                                     | # Facebook Ads Monitoring Loop - Overview, setup instructions, benefits, author info                                   |
| Sticky Note (Add params)  | Sticky Note              | Documentation note                 |                            |                                     | ## Add your parameters: Creative status, Page IDs, Countries, Keywords, Access token                                   |
| Sticky Note1              | Sticky Note              | Documentation note                 |                            |                                     | This node decides whether to check pages or keywords                                                                   |
| Sticky Note2              | Sticky Note              | Documentation note                 |                            |                                     | These nodes make API requests to the Meta Ads Library API                                                              |
| Sticky Note3              | Sticky Note              | Documentation note                 |                            |                                     | The Meta Ads Library API returns only 25 results per page; pagination handling explained                               |
| Sticky Note4              | Sticky Note              | Documentation note                 |                            |                                     | Check which creative IDs already exist in Google Sheets                                                                |
| Sticky Note5              | Sticky Note              | Documentation note                 |                            |                                     | Collect IDs for future matching                                                                                         |
| Sticky Note6              | Sticky Note              | Documentation note                 |                            |                                     | Match creatives from API response with those already stored in Google Sheets                                            |
| Sticky Note7              | Sticky Note              | Documentation note                 |                            |                                     | Match creatives from API response with those already stored in Google Sheets                                            |
| Sticky Note8              | Sticky Note              | Documentation note                 |                            |                                     | Split out array data into individual items                                                                             |
| Sticky Note9              | Sticky Note              | Documentation note                 |                            |                                     | Add creatives to Google Sheets. Be careful about column order and names                                                |
| Sticky Note10             | Sticky Note              | Documentation note                 |                            |                                     | Count how many new creatives were found. Use {{$json.newCount}} in messages                                            |
| Sticky Note11             | Sticky Note              | Documentation note                 |                            |                                     | Check if there are any new ads before sending notifications                                                            |
| Sticky Note12             | Sticky Note              | Documentation note                 |                            |                                     | Send Slack notification about new creatives                                                                            |
| Sticky Note13             | Sticky Note              | Documentation note                 |                            |                                     | Send Telegram notification about new creatives                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**  
   - Set to run on your desired interval (e.g., daily at a certain time).

3. **Add a Set node ("Add parameters"):**  
   - Create fields:  
     - `ad_active_status` = "active"  
     - `search_page_ids` = "114099778241791,101962412753276" (replace with your target page IDs)  
     - `ad_reached_countries` = "US,AU,NZ,CA"  
     - `keywords` = "" (empty or fill with your keywords)  
     - `access_token` = "put_your_token_here" (your Facebook Marketing API token)

4. **Connect Schedule Trigger → Add parameters.**

5. **Add a Google Sheets node ("Read existing IDs"):**  
   - Operation: Read rows  
   - Sheet: Your Google Sheet document ID and sheet name (e.g., gid=0)  
   - Range: Column J (IDs)  
   - Credentials: Set up OAuth2 credentials for Google Sheets.

6. **Connect Add parameters → Read existing IDs.**

7. **Add a Code node ("Collect ID list"):**  
   - JavaScript to extract and normalize IDs from the sheet data:

   ```js
   const ids = items
     .map(item => item.json.id)
     .map(id => id != null ? String(id) : undefined)
     .filter((id, index, arr) => id && arr.indexOf(id) === index);
   return [{ json: { existingIds: ids } }];
   ```

8. **Connect Read existing IDs → Collect ID list.**

9. **Add a Switch node ("Page or keywords"):**  
   - Two outputs:  
     - Output "pages" when `search_page_ids` is not empty  
     - Output "keywords" when `keywords` is not empty

10. **Connect Add parameters → Page or keywords.**

11. **Add two HTTP Request nodes:**  
    - "Facebook Ads API by page":  
      - Method: GET  
      - URL template:

      ```
      https://graph.facebook.com/v22.0/ads_archive?ad_active_status={{ $json.ad_active_status }}&search_page_ids={{ $json.search_page_ids }}&ad_reached_countries={{ $json.ad_reached_countries }}&fields=ad_creation_time,ad_creative_bodies,ad_creative_link_captions,ad_creative_link_descriptions,ad_creative_link_titles,ad_delivery_start_time,ad_snapshot_url,languages,page_name,publisher_platforms&limit=50&access_token={{ $json.access_token }}
      ```

      - Authentication: HTTP Header Auth using Meta Ads credentials  
      - Connect Page or keywords (pages output) → Facebook Ads API by page

    - "Facebook Ads API by keywords":  
      - Method: GET  
      - URL template similar, but with fixed countries US,CA and using `keywords` parameter if needed  
      - Connect Page or keywords (keywords output) → Facebook Ads API by keywords

12. **Add a Code node ("Check the pagination"):**  
    - JS code to extract data and next page URL:

    ```js
    const response = $input.first().json;
    const ads = response.data || [];
    const nextUrl = response.paging && response.paging.next ? response.paging.next : null;
    return { json: { data: ads, next_url: nextUrl } };
    ```

13. **Connect both Facebook Ads API by page and Facebook Ads API by keywords → Check the pagination.**

14. **Add an If node ("If"):**  
    - Condition: Check if `next_url` is not empty  
    - True output: Continue pagination  
    - False output: Proceed to merging data

15. **Connect Check the pagination → If.**

16. **Add a Set node ("Set Next URL"):**  
    - Assign `url` = `{{$json.next_url}}`

17. **Connect If (true output) → Set Next URL.**

18. **Add an HTTP Request node ("Facebook Ads API pagination"):**  
    - Method: GET  
    - URL: `{{$json.url}}` (dynamic)  
    - Same authentication as previous API calls

19. **Connect Set Next URL → Facebook Ads API pagination → Check the pagination (loop).**

20. **Connect If (false output) → Attach existing ids (next step).**

21. **Add a Merge node ("Attach existing ids"):**  
    - Mode: Combine (combine all)  
    - Inputs:  
      - Data from Check the pagination (ads data)  
      - Data from Collect ID list (existing IDs)

22. **Connect Check the pagination (false output) and Collect ID list → Attach existing ids.**

23. **Add a Code node ("Filter new creatives"):**  
    - JS code to compare and filter new ads:

    ```js
    const ads = $json.data || [];
    const existingIds = ($json.existingIds || []).map(id => id != null ? String(id) : undefined).filter(Boolean);
    const seen = new Set(existingIds);
    const newAds = [];
    for (const ad of ads) {
      if (!ad?.id) continue;
      const adId = String(ad.id);
      if (!seen.has(adId)) {
        newAds.push(ad);
        seen.add(adId);
      }
    }
    return { json: { data: newAds } };
    ```

24. **Connect Attach existing ids → Filter new creatives.**

25. **Add a Split Out node ("Split Out"):**  
    - Field to split: `data`  
    - Include all other fields

26. **Connect Filter new creatives → Split Out.**

27. **Add a Google Sheets node ("Add to sheet"):**  
    - Operation: Append or Update  
    - Map fields: id, page_name, titles, descriptions, snapshot URLs, languages, platforms, creation and delivery times, etc.  
    - Credentials: Google Sheets OAuth2  
    - Sheet and document IDs same as Read existing IDs

28. **Connect Split Out → Add to sheet.**

29. **Add a Code node ("Count new ads"):**  
    - JS code:

    ```js
    return [{ json: { newCount: items.length } }];
    ```

30. **Connect Add to sheet → Count new ads.**

31. **Add an If node ("Any new ads?"):**  
    - Condition: `newCount` > 0

32. **Connect Count new ads → Any new ads?.**

33. **Add Telegram node ("Send a text message"):**  
    - Text:  
      ```
      Hello!
      {{$json.newCount}} of ads were found today!
      You can see full list here — https://docs.google.com/spreadsheets/d/1sLKktslKS1QyC6Hc8amb2d1HBgCUAi_z01YQ_u0Na8Q/edit?usp=sharing
      ```
    - Chat ID: Your Telegram chat ID  
    - Credentials: Telegram API

34. **Add Slack node ("Send a message"):**  
    - Text same as Telegram  
    - Channel: Your Slack channel ID  
    - Credentials: Slack OAuth2

35. **Connect Any new ads? (true output) → Send a text message and Send a message.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                      |
|-------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Create an app on [Facebook Developers](https://developers.facebook.com/apps/), get access to the Marketing API, and generate a token. | Workflow setup instructions                                        |
| Built by Kirill Khatkevich - [LinkedIn Profile](https://www.linkedin.com/in/kirill-khatkevich/)                                 | Author and professional contact                                    |
| The Meta Ads Library API returns only 25 results per page; pagination is handled in the workflow to fetch all results.          | Important API limitation and workflow design note                  |
| Keep Google Sheets columns consistent with the mapped fields in the "Add to sheet" node to avoid data mismatches or loss.       | Data integrity best practice                                       |
| Use {{$json.newCount}} variable in messaging nodes to dynamically show the count of new creatives found.                        | Dynamic message templating                                         |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.