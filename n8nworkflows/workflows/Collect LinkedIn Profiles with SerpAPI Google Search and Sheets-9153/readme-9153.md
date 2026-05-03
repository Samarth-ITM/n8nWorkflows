Collect LinkedIn Profiles with SerpAPI Google Search and Sheets

https://n8nworkflows.xyz/workflows/collect-linkedin-profiles-with-serpapi-google-search-and-sheets-9153


# Collect LinkedIn Profiles with SerpAPI Google Search and Sheets

### 1. Workflow Overview

This workflow automates the collection of LinkedIn profiles using SerpAPI's Google Search API combined with Google Sheets for data storage. It targets scenarios where a user submits a form specifying keywords and the number of result pages to fetch. The workflow then builds a tailored search query, performs paginated Google searches confined to LinkedIn profile URLs, extracts relevant profile information, and appends this data to a Google Sheet for record-keeping and further analysis.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Captures user input via form submission, including search keywords and pages to fetch.
- **1.2 Query Preparation:** Formats keywords into a SerpAPI-compatible search string and prepares pagination parameters.
- **1.3 Search Execution & Results Validation:** Executes the SerpAPI search queries and checks if results were returned.
- **1.4 Data Extraction & Processing:** Splits search results, extracts profile full names, and formats data.
- **1.5 Data Storage:** Appends processed profile data into a configured Google Sheet.
- **1.6 User Notification:** Sends completion or error responses back to the form submitter.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for form submissions containing user parameters and triggers the workflow.
- **Nodes Involved:** `On form submission`, `Format Keywords`
- **Node Details:**

  - **On form submission**
    - Type: Form Trigger
    - Role: Entry point; receives user input from a web form titled "LinkedIn Search".
    - Config: Two fields — "Keywords (comma separated)" (optional string), and "Pages to fetch" (optional number).
    - Input: Webhook data from form submit.
    - Output: JSON containing raw input including keywords and pages count.
    - Edge Cases: Missing or empty keywords or pages fields; no default enforcement.
  
  - **Format Keywords**
    - Type: Set Node
    - Role: Transforms the raw keyword string into a Google query compatible format.
    - Config: Converts comma-separated keywords into a grouped string like `("keyword1") ("keyword2")` to enhance search accuracy.
    - Key Expression: Uses JavaScript to split, trim, filter empty strings, then join with parentheses and quotes.
    - Input: Output from `On form submission`.
    - Output: JSON with a new property `keywords` formatted for query.
    - Edge Cases: Empty keyword string results in empty query group, possibly causing ineffective searches.

---

#### 2.2 Query Preparation

- **Overview:** Builds a list of pages to query based on requested "Pages to fetch" and calculates corresponding start offsets for SerpAPI pagination.
- **Nodes Involved:** `Build Page List`, `Loop Over Items`
- **Node Details:**

  - **Build Page List**
    - Type: Code Node (JavaScript)
    - Role: Generates an array of items representing each page query request with proper pagination offsets.
    - Config: 
      - Reads `Pages to fetch` (default 1 if missing).
      - Each page corresponds to 10 results (SerpAPI default).
      - Creates JSON objects with page index and offset start values.
    - Input: Uses outputs from `On form submission` and `Format Keywords`.
    - Output: Array of JSON items each representing a search page context.
    - Edge Cases: Non-numeric or zero pages input defaults to 1; potential off-by-one if offset is misread.
  
  - **Loop Over Items**
    - Type: SplitInBatches
    - Role: Iterates over each page item generated to perform individual searches.
    - Config: Default batch size (not explicitly set).
    - Input: Output array from `Build Page List`.
    - Output: Sequentially outputs each page query.
    - Edge Cases: Large page counts may cause long execution times or API rate limits.

---

#### 2.3 Search Execution & Results Validation

- **Overview:** Executes SerpAPI search queries against LinkedIn profiles and routes flow based on presence or absence of results.
- **Nodes Involved:** `SerpAPI Search`, `Check how many results are returned`, `No profiles found`
- **Node Details:**

  - **SerpAPI Search**
    - Type: SerpAPI Node (Google Search)
    - Role: Performs the actual Google search query via SerpAPI restricted to LinkedIn profile URLs.
    - Config:
      - Query: `site:pl.linkedin.com/in/ {{$json.keywordsGrouped}}`
      - Location: Warsaw, Poland (localized search)
      - Pagination controlled via `start` offset.
      - Credentials: SerpApi account OAuth/API key.
    - Input: Page query info from `Loop Over Items`.
    - Output: Search results JSON with total results and organic results.
    - Edge Cases: API errors (auth, quota), zero results, malformed queries.
  
  - **Check how many results are returned**
    - Type: Switch Node
    - Role: Checks `search_information.total_results` to determine if results exist.
    - Config: 
      - Routes to "Empty list" if total_results == 0.
      - Routes to "Not empty list" if total_results > 0.
    - Input: Output from `SerpAPI Search`.
    - Output: Branches flow accordingly.
    - Edge Cases: Missing or malformed `total_results` field.
  
  - **No profiles found**
    - Type: Form Node (Completion)
    - Role: Sends a completion message back to the user if no profiles were found.
    - Config: Message "No profiles found based on params."
    - Input: Triggered when no results.
    - Output: HTTP response to form submitter.
    - Edge Cases: Network issues sending response, user confusion if no profiles appear.

---

#### 2.4 Data Extraction & Processing

- **Overview:** Extracts individual profile results from search output, isolates full names, and prepares data for storage.
- **Nodes Involved:** `Split Out`, `Get Full Name to property of object`
- **Node Details:**

  - **Split Out**
    - Type: SplitOut
    - Role: Splits the array of `organic_results` from SerpAPI into individual items.
    - Config: Field to split is `organic_results`.
    - Input: Output from "Not empty list" route of `Check how many results are returned`.
    - Output: One profile result per item.
    - Edge Cases: Empty or missing `organic_results` field.
  
  - **Get Full Name to property of object**
    - Type: Code Node (JavaScript)
    - Role: Parses the profile title string to extract the full name before the first dash.
    - Config: Splits title by '-' and trims whitespace.
    - Input: Each individual profile item.
    - Output: JSON with new `fullName` property added.
    - Edge Cases: Titles without dashes, empty or malformed titles.

---

#### 2.5 Data Storage

- **Overview:** Appends each processed LinkedIn profile result into a Google Sheet with relevant metadata.
- **Nodes Involved:** `Append profile in sheet`
- **Node Details:**

  - **Append profile in sheet**
    - Type: Google Sheets Node
    - Role: Adds a new row per profile into a predefined Google Sheet.
    - Config:
      - Sheet: Document ID and Sheet name (gid=0) configured.
      - Columns written: Date (timestamp of submission), Profile URL (`link`), Keywords (raw input), Full name (extracted).
      - Append mode: Adds rows without overwriting.
      - Credentials: Google Sheets OAuth2 API for authorized access.
    - Input: Output from `Get Full Name to property of object`.
    - Output: Confirmation of append operation.
    - Edge Cases: Google API rate limits, sheet access permissions, malformed data.

---

#### 2.6 User Notification

- **Overview:** Sends a completion message to the user upon successful data collection.
- **Nodes Involved:** `Form Response`
- **Node Details:**

  - **Form Response**
    - Type: Form Node (Completion)
    - Role: Sends a success message back to the form submitter.
    - Config: Message "Check linked file".
    - Input: Triggered after successful batch processing of page queries.
    - Output: HTTP response to user.
    - Edge Cases: Network delivery issues, premature triggering if partial data.

---

### 3. Summary Table

| Node Name                     | Node Type             | Functional Role                         | Input Node(s)                   | Output Node(s)                    | Sticky Note                                                                                 |
|-------------------------------|-----------------------|---------------------------------------|---------------------------------|----------------------------------|---------------------------------------------------------------------------------------------|
| On form submission             | Form Trigger          | Entry point - receive user input      | —                               | Format Keywords                  |                                                                                             |
| Format Keywords               | Set                   | Format keywords for search query      | On form submission              | Build Page List                 | ## Change form data to query \n```\nelemnt1, element2\n```\nchanged to \n```\n(\"elemnt1\") (\"element2\")\n``` |
| Build Page List               | Code                  | Build pagination query list            | Format Keywords                | Loop Over Items                 |                                                                                             |
| Loop Over Items               | SplitInBatches        | Iterate over each page query           | Build Page List                | SerpAPI Search, Form Response   |                                                                                             |
| SerpAPI Search               | SerpAPI Search         | Perform Google search via SerpAPI      | Loop Over Items                | Check how many results are returned | ## Google query\nSerpAPI used to make a query                                               |
| Check how many results are returned | Switch                | Route flow based on presence of results | SerpAPI Search                | No profiles found, Split Out     |                                                                                             |
| No profiles found             | Form (Completion)      | Inform user no profiles found          | Check how many results are returned | —                             |                                                                                             |
| Split Out                    | SplitOut               | Split results array into individual items | Check how many results are returned | Get Full Name to property of object |                                                                                             |
| Get Full Name to property of object | Code                  | Extract full name from profile title   | Split Out                     | Append profile in sheet          | ## Save profiles in Google Sheet\n\nSetup Google Sheet with the following columns:\n- Date\n- Profile\n- Keywords\n |
| Append profile in sheet       | Google Sheets          | Append profile data into spreadsheet   | Get Full Name to property of object | Loop Over Items                 |                                                                                             |
| Form Response                 | Form (Completion)      | Send success completion message        | Loop Over Items                | —                              |                                                                                             |
| Sticky Note                  | Sticky Note            | Instruction on query formatting        | —                             | —                              | ## Change form data to query \n```\nelemnt1, element2\n```\nchanged to \n```\n(\"elemnt1\") (\"element2\")\n``` |
| Sticky Note1                 | Sticky Note            | Explanation about SerpAPI query        | —                             | —                              | ## Google query\nSerpAPI used to make a query                                               |
| Sticky Note2                 | Sticky Note            | Instructions for Google Sheet setup    | —                             | —                              | ## Save profiles in Google Sheet\n\nSetup Google Sheet with the following columns:\n- Date\n- Profile\n- Keywords\n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the trigger node:**
   - Add a **Form Trigger** node named `On form submission`.
   - Configure the form title as "LinkedIn Search".
   - Add two fields:
     - "Keywords (comma separated)" (string, not required).
     - "Pages to fetch" (number, not required).
   - Set response mode to "lastNode".

2. **Add a Set node to format keywords:**
   - Add a **Set** node named `Format Keywords`.
   - Connect from `On form submission`.
   - Add a string field named `keywords`.
   - Use the following JavaScript expression to convert comma-separated keywords into the grouped format:
     ```javascript
     (() => {
       const keywords = $json["Keywords (comma separated)"]
         .split(',')
         .map(k => k.trim())
         .filter(Boolean);
       return '("' + keywords.join('") ("') + '")';
     })()
     ```

3. **Add a Code node to build page list:**
   - Add a **Code** node named `Build Page List`.
   - Connect from `Format Keywords`.
   - Use JavaScript to generate pagination parameters:
     ```javascript
     const pagesRequested = parseInt($('On form submission').item.json['Pages to fetch'] ?? 1, 10) || 1;
     const perPage = 10;
     const ctx = {
       keywordsGrouped: $('Format Keywords').item.json.keywords,
       rawKeywords: $('On form submission').item.json['Keywords (comma separated)'],
       submittedAt: $('On form submission').item.json.submittedAt
     };
     return Array.from({ length: pagesRequested }, (_, i) => ({
       json: {
         page: i,
         start: i * perPage,
         ...ctx
       }
     }));
     ```

   - Note: Ensure the `start` offset calculation uses the current page index times 10.

4. **Add a SplitInBatches node for paging:**
   - Add **SplitInBatches** node named `Loop Over Items`.
   - Connect from `Build Page List`.
   - Use default batch size or set based on expected API limits.

5. **Add SerpAPI Search node:**
   - Add **SerpAPI** node named `SerpAPI Search`.
   - Connect from `Loop Over Items`.
   - Configure query as:
     ```
     site:pl.linkedin.com/in/ {{$json.keywordsGrouped}}
     ```
   - Set location to "Warsaw,Masovian Voivodeship,Poland".
   - Set additional field `start` to `{{$json.start}}`.
   - Provide valid SerpAPI credentials.

6. **Add Switch node to check search results:**
   - Add **Switch** node named `Check how many results are returned`.
   - Connect from `SerpAPI Search`.
   - Add two rules:
     - "Empty list": when `parseInt($json.search_information.total_results) == 0`.
     - "Not empty list": when `parseInt($json.search_information.total_results) > 0`.

7. **Add Form Completion node for empty results:**
   - Add **Form** node named `No profiles found`.
   - Connect from "Empty list" output of the switch.
   - Configure completion title "Response".
   - Set message "No profiles found based on params.".
   - Use the same webhook ID as the trigger to respond to the form.

8. **Add SplitOut node for organic results:**
   - Add **SplitOut** node named `Split Out`.
   - Connect from "Not empty list" output of the switch.
   - Set field to split out as `organic_results`.

9. **Add Code node to extract full name:**
   - Add **Code** node named `Get Full Name to property of object`.
   - Connect from `Split Out`.
   - Use JavaScript:
     ```javascript
     for (const item of $input.all()) {
       if (typeof item.json.title === 'string') {
         item.json.fullName = item.json.title.split('-')[0].trim();
       }
     }
     return $input.all();
     ```

10. **Add Google Sheets node to append data:**
    - Add **Google Sheets** node named `Append profile in sheet`.
    - Connect from `Get Full Name to property of object`.
    - Configure:
      - Operation: Append
      - Document ID: your target Google Sheet ID
      - Sheet Name: gid=0 (or your sheet name)
      - Map columns:
        - Date: `={{ $('On form submission').item.json.submittedAt }}`
        - Profile: `={{ $json.link }}`
        - Keywords: `={{ $('On form submission').item.json['Keywords (comma separated)'] }}`
        - Full name: `={{ $json.fullName }}`
      - Provide Google Sheets OAuth2 credentials with permissions.

11. **Connect back to Loop Over Items:**
    - Connect output of `Append profile in sheet` back to `Loop Over Items` to continue processing remaining pages.

12. **Add Form Completion node for success:**
    - Add **Form** node named `Form Response`.
    - Connect from `Loop Over Items`.
    - Configure completion title "Response".
    - Set message "Check linked file".
    - Use the same webhook ID as the trigger for response.

13. **Add Sticky Notes (optional):**
    - Add notes to explain query formatting and Google Sheets setup for maintainability.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                       |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| The keywords are transformed from comma-separated values into a grouped query format like `("kw1") ("kw2")`.  | Sticky Note explaining query formatting in the workflow.                                                             |
| SerpAPI is used to perform Google searches restricted to LinkedIn profile URLs with location set to Warsaw.   | Sticky Note highlighting the SerpAPI query configuration.                                                            |
| Google Sheet must contain columns: Date, Profile, Keywords, Full name.                                         | Sticky Note describing required Google Sheet column setup for data appending.                                        |
| Workflow assumes default SerpAPI pagination with 10 results per page and uses offset start accordingly.       | Important for understanding pagination logic in `Build Page List`.                                                   |
| To avoid rate limits, consider limiting the number of pages requested or adding delays in `Loop Over Items`.  | Operational note for large scale deployments.                                                                          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.