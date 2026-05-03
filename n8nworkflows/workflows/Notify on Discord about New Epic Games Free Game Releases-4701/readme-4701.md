Notify on Discord about New Epic Games Free Game Releases

https://n8nworkflows.xyz/workflows/notify-on-discord-about-new-epic-games-free-game-releases-4701


# Notify on Discord about New Epic Games Free Game Releases

### 1. Workflow Overview

This n8n workflow automates the detection and notification of new free game releases on the Epic Games Store by scraping their free games page and sending alerts to a Discord channel. It is designed to run on a daily schedule, repeatedly attempts the scraping if initial attempts fail, detects changes in game listings, and only sends notifications when new free games appear or the list changes.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling and Retry Management**: Triggers the workflow daily at a specified hour and manages retries for error-prone scraping steps.
- **1.2 Web Scraping and Data Extraction**: Uses a Puppeteer node to scrape the Epic Games Store free games page, extract game offer HTML blocks, and parse game metadata (title, image, URL, and status label).
- **1.3 Change Detection**: Compares the newly scraped data with previously stored data to determine if there are any changes to report.
- **1.4 Notification Preparation and Sending**: Formats the updated game information into Discord embeds and sends a notification message to a configured Discord webhook.
- **1.5 Error Handling and Data Persistence**: Handles failure cases by retrying or sending an error notification, and stores the last known state to avoid duplicate notifications.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduling and Retry Management

**Overview:**  
This block triggers the workflow at 19:00 daily and manages retry counters for the scraping process. It ensures robustness by retrying the scraping up to 5 times if the page content is incomplete or empty.

**Nodes Involved:**  
- Schedule Trigger  
- Set Retries  
- GET Epic Games Free Games page (Puppeteer)  
- If has Results  
- Wait between requests  
- Update Retries  
- If Has Retries  
- Notify on error  
- Sticky Note (retry strategy explanation)

**Node Details:**  

- **Schedule Trigger**  
  - *Type*: Schedule Trigger  
  - *Role*: Initiates workflow execution daily at 19:00  
  - *Config*: Interval trigger at hour 19  
  - *Connections*: Outputs to Set Retries  
  - *Fail Modes*: Minimal; if node fails, workflow won't trigger.

- **Set Retries**  
  - *Type*: Set  
  - *Role*: Initializes or carries forward the retry count (number of attempts)  
  - *Config*: Sets `retries` field to 0 or increments from previous run  
  - *Connections*: Input from Schedule Trigger or If Has Retries; output to GET Epic Games page

- **GET Epic Games Free Games page**  
  - *Type*: Puppeteer  
  - *Role*: Loads the Epic Games Store page for free games, using a headless browser to bypass Cloudflare and dynamic content  
  - *Config*: URL https://store.epicgames.com/en-US/free-games, with browserWSEndpoint configured for stealth and disabled web security (requires n8n-nodes-puppeteer installed)  
  - *Connections*: Input from Set Retries, output to Extract offers  
  - *Failure Modes*: Page load failure, Cloudflare blocking, empty content (common issue)  
  - *Sticky Note*: Explains occasional empty container issue and requirement for Puppeteer node

- **If has Results**  
  - *Type*: If  
  - *Role*: Checks if the extracted HTML container array is not empty (i.e., offers exist)  
  - *Config*: Condition checks `container` field is non-empty array  
  - *Connections*: Input from Extract offers, two outputs: true to Split Out, false to Wait between requests (retry)  
  - *Fail Modes*: False branch triggers retry flow

- **Wait between requests**  
  - *Type*: Wait  
  - *Role*: Delays next retry to avoid rapid repeat requests (10 seconds)  
  - *Connections*: False output from If has Results, outputs to Update Retries

- **Update Retries**  
  - *Type*: Set  
  - *Role*: Increments retry counter by 1  
  - *Config*: `retries = previous retries + 1`  
  - *Connections*: Input from Wait between requests, output to If Has Retries

- **If Has Retries**  
  - *Type*: If  
  - *Role*: Checks if retry count is less than 5; if yes, repeats Set Retries → GET Epic Games; if no, sends error notification  
  - *Config*: Condition `retries < 5`  
  - *Connections*: True → Set Retries; False → Notify on error

- **Notify on error**  
  - *Type*: Discord Webhook  
  - *Role*: Sends a Discord notification indicating failure to check Epic Games  
  - *Config*: Webhook URL with message "Failed to check Epic Games for new content"  
  - *Connections*: Output terminal

- **Sticky Note (Retry explanation)**  
  - *Content*: Explains intermittent empty container issue and Puppeteer requirement

---

#### 2.2 Web Scraping and Data Extraction

**Overview:**  
This block extracts the relevant free game offer blocks from the scraped HTML, splits multiple offers into individual items, and extracts key metadata including the game’s title, image URL, store URL, and release status label.

**Nodes Involved:**  
- Extract offers (HTML)  
- Split Out  
- Extract title and image (HTML)  
- Extract label (Set)  
- Prepare data (Merge)  
- Sticky Note (CSS selector maintenance warning)

**Node Details:**  

- **Extract offers**  
  - *Type*: HTML Extract  
  - *Role*: Parses the raw HTML body to extract all offer cards excluding discovery cards using CSS selectors  
  - *Config*: Selector `[data-component*="OfferCard"]:not([data-component="DiscoverOfferCard"])`, returns array of HTML strings in `container` field  
  - *Connections*: Input from GET Epic Games page, output to If has Results  
  - *Failure Modes*: If Epic Games changes page structure, extraction fails (empty container)  
  - *Sticky Note*: Warns about this node being a common failure point due to page changes

- **Split Out**  
  - *Type*: Split Out  
  - *Role*: Splits the array of HTML containers into separate items for individual processing  
  - *Config*: Splits on `container` field  
  - *Connections*: True output from If has Results, output to Extract title and image and Extract label in parallel

- **Extract title and image**  
  - *Type*: HTML Extract  
  - *Role*: Extracts game title (h6), image URL (from `data-image` attribute of img), and store URL (href attribute of a) from each offer’s HTML snippet  
  - *Config*:  
    - title: `h6` text  
    - image: `img` attribute `data-image`  
    - url: `a` attribute `href`  
  - *Connections*: Input from Split Out, output to Prepare data  
  - *Failure Modes*: Missing or changed selectors can cause empty fields

- **Extract label**  
  - *Type*: Set (with expression)  
  - *Role*: Determines the offer label ("Free Now", "Coming Soon", "Mystery", or empty) based on substring matching in the container HTML content  
  - *Config*: Expression evaluates `$json.container` for keywords  
  - *Connections*: Input from Split Out, output to Prepare data

- **Prepare data**  
  - *Type*: Merge (combineByPosition)  
  - *Role*: Merges the outputs from Extract title and image and Extract label into a single JSON per offer, combining fields by position index  
  - *Connections*: Inputs from Extract title and image and Extract label, outputs to Check if changed

- **Sticky Note (CSS selector warning)**  
  - *Content*: Warns that if the workflow fails, this node’s CSS selectors are likely outdated due to Epic Games site changes

---

#### 2.3 Change Detection

**Overview:**  
This block compares the latest extracted game list with the previously saved list using a hash string to detect if any changes have occurred since the last run. It prevents duplicate notifications if no new free games are found.

**Nodes Involved:**  
- Check if changed (Code)  
- Only when changed (Filter)  
- Wait for conditions (Merge)  
- Sticky Note (hash change explanation)

**Node Details:**  

- **Check if changed**  
  - *Type*: Code  
  - *Role*: Creates a hash string by concatenating all game titles and labels, compares it with saved static data hash; outputs a boolean `value` indicating change  
  - *Config*: Uses `$getWorkflowStaticData('node')` for persistent lastHash; updates stored hash if different  
  - *Connections*: Input from Prepare data, output to Only when changed  
  - *Failure Modes*: Potential for expression or static data access errors; hash string depends on consistent input formatting

- **Only when changed**  
  - *Type*: Filter  
  - *Role*: Filters flow to proceed only if `value` is true (changes detected)  
  - *Config*: Condition `$json.value === true`  
  - *Connections*: Input from Check if changed; outputs: true to Wait for conditions and Success nodes, false terminates flow

- **Wait for conditions**  
  - *Type*: Merge (chooseBranch)  
  - *Role*: Synchronizes two input branches before continuing: one from change detection, one from other logic (success path)  
  - *Connections*: Input from Check if changed (true branch) and Only when changed; outputs to Prepare notification

- **Sticky Note (Hash explanation)**  
  - *Content*: Explains that a stored hash is used to detect changes between executions

---

#### 2.4 Notification Preparation and Sending

**Overview:**  
This block prepares the notification payload with Discord embed objects for each new free game and sends it via Discord webhook. It only executes if changes are detected.

**Nodes Involved:**  
- Prepare notification (Code)  
- Notify Discord (HTTP Request)  
- Success (Merge)  
- Save Static Data (Code)  
- Sticky Notes (Discord notification structure and save logic)

**Node Details:**  

- **Prepare notification**  
  - *Type*: Code  
  - *Role*: Builds the Discord embed payload by iterating over all new game items, embedding thumbnail, title, description, and URL, conforming to Discord embed structure  
  - *Config*: Constructs JSON with username "Epic Games", content "New games detected", avatar URL, and embeds array  
  - *Connections*: Input from Wait for conditions, output to Notify Discord

- **Notify Discord**  
  - *Type*: HTTP Request  
  - *Role*: Sends the prepared JSON payload to the Discord webhook URL via POST  
  - *Config*: URL set to Discord webhook (redacted), sends JSON body, retries on failure  
  - *Connections*: Input from Prepare notification, output to Success

- **Success**  
  - *Type*: Merge (chooseBranch)  
  - *Role*: Synchronizes branches from Only when changed and Notify Discord, coordinates flow to Save Static Data  
  - *Connections*: Input from Notify Discord and Only when changed; output to Save Static Data

- **Save Static Data**  
  - *Type*: Code  
  - *Role*: Updates stored static data with the new hash after successful notification to avoid repeat alerts  
  - *Config*: Saves `lastHash` from input JSON to workflow static data  
  - *Connections*: Input from Success

- **Sticky Note (Notification structure)**  
  - *Content*: Explains the structure of the Discord embeds and the visual presentation of free game notifications

- **Sticky Note (Save logic)**  
  - *Content*: Notes saving the hash only after successful notification to ensure user received the alert

---

### 3. Summary Table

| Node Name                         | Node Type               | Functional Role                                 | Input Node(s)                     | Output Node(s)                           | Sticky Note                                                                                                     |
|----------------------------------|-------------------------|------------------------------------------------|----------------------------------|-----------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger                 | Schedule Trigger        | Triggers workflow daily at 19:00                | —                                | Set Retries                             |                                                                                                                |
| Set Retries                     | Set                     | Initializes or increments retry count           | Schedule Trigger, If Has Retries  | GET Epic Games Free Games page          |                                                                                                                |
| GET Epic Games Free Games page  | Puppeteer               | Loads the Epic Games free games page            | Set Retries                      | Extract offers                          | For some reason sometimes this can return whole page properly but the container with free games is empty. So we repeat few times. Requires `n8n-nodes-puppeteer` to be installed. Otherwise you get blocked by Cloudflare. |
| Extract offers                 | HTML Extract             | Extracts offer cards from page HTML              | GET Epic Games Free Games page    | If has Results                         | If the workflow does not work this is most likely the point of failure. It means that they have changed something on their page and css selector has changed. |
| If has Results                 | If                       | Checks if offers were found                       | Extract offers                   | Split Out (true), Wait between requests (false) |                                                                                                                |
| Split Out                     | Split Out                | Splits offers array into individual items        | If has Results (true)            | Extract title and image, Extract label |                                                                                                                |
| Extract title and image        | HTML Extract             | Extracts title, image URL, and URL per offer     | Split Out                       | Prepare data                          |                                                                                                                |
| Extract label                 | Set                      | Extracts offer label based on content keywords   | Split Out                       | Prepare data                          |                                                                                                                |
| Prepare data                  | Merge                    | Combines title/image data and label per offer    | Extract title and image, Extract label | Check if changed                    |                                                                                                                |
| Check if changed              | Code                     | Detects changes by comparing hash of offers      | Prepare data                    | Only when changed                    | Keeps a 'hash' of changes so that it knows if anything changed between executions.                             |
| Only when changed             | Filter                   | Filters flow to continue only if changes detected | Check if changed                | Wait for conditions, Success          |                                                                                                                |
| Wait for conditions           | Merge                    | Synchronizes branches before notification        | Prepare data, Only when changed  | Prepare notification                   |                                                                                                                |
| Prepare notification          | Code                     | Builds Discord notification payload with embeds | Wait for conditions             | Notify Discord                        | The notification on Discord consists of Embeds. The Template looks like this: Epic Games New games detected... |
| Notify Discord                | HTTP Request             | Posts notification to Discord webhook            | Prepare notification             | Success                             |                                                                                                                |
| Success                      | Merge                    | Synchronizes notification success and filter     | Notify Discord, Only when changed | Save Static Data                     |                                                                                                                |
| Save Static Data              | Code                     | Saves new hash after successful notification     | Success                        | —                                   | Save only when we are sure user got notification.                                                              |
| Wait between requests         | Wait                     | Delays between retries                            | If has Results (false)           | Update Retries                      | This can sometimes fail so it will repeat few times.                                                          |
| Update Retries               | Set                      | Increments retry count                            | Wait between requests            | If Has Retries                     |                                                                                                                |
| If Has Retries               | If                       | Checks if retry count < 5, decides retry or error | Update Retries                  | Set Retries (true), Notify on error (false) |                                                                                                                |
| Notify on error              | Discord Webhook          | Sends error notification on failure              | If Has Retries (false)           | —                                   |                                                                                                                |
| Sticky Note                  | Sticky Note              | Retry explanation                                 | —                              | —                                   | For some reason sometimes this can return whole page properly but the container with free games is empty. So we repeat few times. Requires `n8n-nodes-puppeteer` to be installed. Otherwise you get blocked by Cloudflare. |
| Sticky Note1                 | Sticky Note              | Retry explanation on failure                       | —                              | —                                   | This can sometimes fail so it will repeat few times.                                                           |
| Sticky Note2                 | Sticky Note              | Save logic explanation                             | —                              | —                                   | Save only when we are sure user got notification.                                                              |
| Sticky Note3                 | Sticky Note              | Hash change detection explanation                  | —                              | —                                   | Keeps a 'hash' of changes so that it knows if anything changed between executions.                             |
| Sticky Note4                 | Sticky Note              | CSS selector failure warning                        | —                              | —                                   | If the workflow does not work this is most likely the point of failure. It means that they have changed something on their page and css selector has changed. |
| Sticky Note5                 | Sticky Note              | Discord notification embed structure explanation   | —                              | —                                   | The notification on Discord consists of Embeds. The Template looks like this: Epic Games New games detected... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set to trigger daily at 19:00 (server local time or specify timezone if needed).

2. **Create a Set node named "Set Retries"**  
   - Add field `retries` (number)  
   - Expression: `{{$json.retries || 0}}` (initialize to 0 if undefined)  
   - Connect Schedule Trigger output → Set Retries input.

3. **Create a Puppeteer node named "GET Epic Games Free Games page"**  
   - URL: `https://store.epicgames.com/en-US/free-games`  
   - Set browserWSEndpoint to your Puppeteer WebSocket endpoint with stealth mode and disabled web security.  
   - Enable retry on fail.  
   - Connect Set Retries output → Puppeteer node input.

4. **Create an HTML Extract node "Extract offers"**  
   - Operation: Extract HTML content from property `body` (from Puppeteer)  
   - Extraction Values:  
     - Key: `container`  
     - CSS Selector: `[data-component*="OfferCard"]:not([data-component="DiscoverOfferCard"])`  
     - Return Array: true  
     - Return Value: `html`  
   - Connect Puppeteer output → Extract offers input.

5. **Create an If node "If has Results"**  
   - Condition: Check if `container` field (array) is not empty  
   - Connect Extract offers output → If has Results input.

6. **Create a Split Out node "Split Out"**  
   - Field to split out: `container`  
   - Connect If has Results (true) output → Split Out input.

7. **Create an HTML Extract node "Extract title and image"**  
   - Operation: Extract HTML content from `container` field  
   - Extraction Values:  
     - `title`: CSS selector `h6` (text)  
     - `image`: CSS selector `img`, attribute `data-image`  
     - `url`: CSS selector `a`, attribute `href`  
   - Connect Split Out output → Extract title and image input.

8. **Create a Set node "Extract label"**  
   - Create string field `label` with expression:  
     ```js
     {{$json.container.includes("Free Now") ? "Free Now" : $json.container.includes("Coming Soon") ? "Coming Soon" : $json.container.includes("Mystery") ? "Mystery" : ""}}
     ```  
   - Connect Split Out output → Extract label input.

9. **Create a Merge node "Prepare data"**  
   - Mode: Combine  
   - Combine By: Position  
   - Connect Extract title and image output → Prepare data input 1  
   - Connect Extract label output → Prepare data input 2

10. **Create a Code node "Check if changed"**  
    - JavaScript code:  
      ```js
      const nodeStaticData = $getWorkflowStaticData('node');
      let lastHash = $input.all().map(item => item.json.title + item.json.label).join(',');
      if (nodeStaticData.lastHash == undefined || nodeStaticData.lastHash !== lastHash) {
        nodeStaticData.lastHash = lastHash;
        return {json: {value: true, hash: lastHash}};
      }
      return {json: {value: false}};
      ```  
    - Connect Prepare data output → Check if changed input.

11. **Create a Filter node "Only when changed"**  
    - Condition: Boolean equals `{{$json.value}}` to `true`  
    - Connect Check if changed output → Only when changed input.

12. **Create a Merge node "Wait for conditions"**  
    - Mode: Choose Branch  
    - Connect Prepare data output → Wait for conditions input 1  
    - Connect Only when changed (true) output → Wait for conditions input 2

13. **Create a Code node "Prepare notification"**  
    - JavaScript code:  
      ```js
      let embeds = [];
      for (const item of $input.all()) {
        var content = {
          "thumbnail": {
            "url": item.json.image
          }
        };
        if (item.json.title.length != 0) {
          content["title"] = item.json.title;
          content["url"] = "https://store.epicgames.com" + item.json.url;
          content["description"] = item.json.label;
        } else {
          content["title"] = item.json.label;
        }
        embeds.push(content);
      }
      const payload = {
        username: "Epic Games",
        content: "New games detected",
        avatar_url: "https://upload.wikimedia.org/wikipedia/commons/thumb/3/31/Epic_Games_logo.svg/250px-Epic_Games_logo.svg.png",
        embeds: embeds
      };
      return [{json: {body: payload}}];
      ```  
    - Connect Wait for conditions output → Prepare notification input.

14. **Create an HTTP Request node "Notify Discord"**  
    - Method: POST  
    - URL: Your Discord webhook URL (`https://discord.com/api/webhooks/...`)  
    - Send Body: True  
    - Body Content Type: JSON  
    - JSON Body: `{{$json.body}}`  
    - Retry On Fail: enabled  
    - Connect Prepare notification output → Notify Discord input.

15. **Create a Merge node "Success"**  
    - Mode: Choose Branch  
    - Connect Notify Discord output → Success input 1  
    - Connect Only when changed (false) output → Success input 2 (to ensure flow synchronization)

16. **Create a Code node "Save Static Data"**  
    - JavaScript code:  
      ```js
      const nodeStaticData = $getWorkflowStaticData('node');
      nodeStaticData.lastHash = $input.first().json.hash;
      return {json: {"status": "success"}};
      ```  
    - Connect Success output → Save Static Data input.

17. **Create a Wait node "Wait between requests"**  
    - Amount: 10 seconds  
    - Connect If has Results (false) output → Wait between requests input.

18. **Create a Set node "Update Retries"**  
    - Field `retries` = `{{$node["Set Retries"].json.retries + 1}}`  
    - Connect Wait between requests output → Update Retries input.

19. **Connect Update Retries output → If Has Retries input.**

20. **Create an If node "If Has Retries"**  
    - Condition: Number less than `{{$json.retries}}` < 5  
    - True output → Set Retries input (retry)  
    - False output → Notify on error input

21. **Create a Discord node "Notify on error"**  
    - Authentication: Webhook  
    - Webhook URL: Your Discord error webhook  
    - Content: "Failed to check Epic Games for new content"  
    - Connect If Has Retries (false) output → Notify on error input.

22. **Connect the nodes as per the above flow diagram to ensure proper execution order.**

23. **Credentials Setup:**  
    - Puppeteer: Ensure n8n environment has Puppeteer running with WebSocket endpoint accessible  
    - Discord: Provide Discord webhook URLs for notification and error nodes

24. **Test the workflow manually and adjust selectors or retry counts if needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Requires `n8n-nodes-puppeteer` community node for Puppeteer functionality.                                                | https://github.com/n8n-io/n8n-nodes-puppeteer                                                              |
| Epic Games page structure changes may break CSS selectors used for extraction.                                             | Adjust CSS selectors in "Extract offers" and "Extract title and image" nodes accordingly.                    |
| Discord notification format uses embeds with thumbnails, title, URL, and description to visually highlight free games.   | See Sticky Note5 content for visual template example.                                                      |
| Retry logic helps bypass Cloudflare and intermittent empty content issues by retrying scraping up to 5 times with delays. | Sticky Note and retry nodes explanation.                                                                    |
| Static data storage is used to track last known list hash to avoid duplicate notifications.                                | Node "Save Static Data" and "Check if changed" handle data persistence in workflow static data.             |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly respects current content policies and contains no illegal, offensive, or protected elements. All managed data is legal and public.