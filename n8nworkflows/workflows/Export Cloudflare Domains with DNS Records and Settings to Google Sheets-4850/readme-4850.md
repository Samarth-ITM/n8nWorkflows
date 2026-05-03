Export Cloudflare Domains with DNS Records and Settings to Google Sheets

https://n8nworkflows.xyz/workflows/export-cloudflare-domains-with-dns-records-and-settings-to-google-sheets-4850


# Export Cloudflare Domains with DNS Records and Settings to Google Sheets

### 1. Workflow Overview

This workflow automates the export of domain data from Cloudflare to a Google Sheets spreadsheet. It retrieves detailed information about Cloudflare zones (domains), including DNS records and zone-specific settings, and then consolidates and formats this data for export to a designated Google Sheets document.

**Target Use Cases:**  
- Cloudflare users or administrators who want a centralized, up-to-date export of their domains’ DNS and configuration settings for auditing, reporting, or backup.  
- Teams requiring automated synchronization of Cloudflare domain data into spreadsheets for easier sharing or analysis.

**Logical Blocks:**

- **1.1 Initialization & Trigger:** Manual trigger to start the export and initial page setting for pagination.  
- **1.2 Fetch Cloudflare Zones (TLDs):** Retrieves all Cloudflare zones (domains) with pagination support.  
- **1.3 Process Each Zone:** For each zone, concurrently fetch DNS records and zone settings, then filter and clean data.  
- **1.4 Data Transformation:** Flatten and format retrieved nested JSON data into a tabular structure suitable for Google Sheets.  
- **1.5 Export to Google Sheets:** Append the processed data into a specified Google Sheets worksheet.  
- **1.6 Control Flow & Pagination:** Logic to handle multiple pages of zones by looping with wait nodes and page increments.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Initialization & Trigger

- **Overview:**  
  Starts the workflow manually. Sets initial pagination parameters.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Page

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to execute the workflow on demand.  
    - Configuration: No parameters; triggers workflow manually.  
    - Outputs: Connected to 'Set Page' to initialize pagination.  
    - Failures: None expected.

  - **Set Page**  
    - Type: Set  
    - Role: Initializes or updates the current page number for pagination in Cloudflare API queries.  
    - Configuration: Sets variable `page` to `{{$json["next_page"] || 1}}` (defaults to page 1 if no next page).  
    - Inputs: From Manual Trigger or Increment Page node.  
    - Outputs: To 'Get TLDs' node.  
    - Failures: Expression evaluation errors if `next_page` is malformed.

#### Block 1.2: Fetch Cloudflare Zones (TLDs)

- **Overview:**  
  Retrieves a paginated list of Cloudflare zones (top-level domains) using Cloudflare API.

- **Nodes Involved:**  
  - Get TLDs  
  - Each Host

- **Node Details:**

  - **Get TLDs**  
    - Type: HTTP Request  
    - Role: Calls `https://api.cloudflare.com/client/v4/zones` with pagination query parameter to get zones list.  
    - Configuration:  
      - Auth: Custom HTTP Auth with Cloudflare API token.  
      - Query Parameter: `page` set dynamically from input JSON.  
    - Inputs: From 'Set Page' node.  
    - Outputs: To 'Each Host' node.  
    - Failures: API authentication errors, rate limits, connection timeouts.

  - **Each Host**  
    - Type: Split Out  
    - Role: Splits the zones array (`result`) into individual items to process each zone separately.  
    - Configuration: Field to split: `result`.  
    - Inputs: From 'Get TLDs'.  
    - Outputs: To 'Get Settings', 'Get DNS', and 'Filter TLD' nodes (parallel processing).  
    - Failures: If `result` field is missing or not an array, node will error.

#### Block 1.3: Process Each Zone (DNS & Settings)

- **Overview:**  
  For each zone, fetch DNS records and settings, then clean and filter the data.

- **Nodes Involved:**  
  - Get Settings  
  - Get DNS  
  - Filter TLD  
  - Filter DNS  
  - Filter Settings

- **Node Details:**

  - **Get Settings**  
    - Type: HTTP Request  
    - Role: Retrieves zone settings from `https://api.cloudflare.com/client/v4/zones/{{ $json.id }}/settings`.  
    - Configuration: Uses Cloudflare API token.  
    - Inputs: From 'Each Host'.  
    - Outputs: To 'Filter Settings'.  
    - Failures: API errors if zone ID invalid or auth issues.

  - **Get DNS**  
    - Type: HTTP Request  
    - Role: Retrieves DNS records for a zone from `https://api.cloudflare.com/client/v4/zones/{{ $json.id }}/dns_records`.  
    - Configuration: Uses Cloudflare API token.  
    - Inputs: From 'Each Host'.  
    - Outputs: To 'Filter DNS'.  
    - Failures: Same as Get Settings.

  - **Filter TLD**  
    - Type: Code (JavaScript)  
    - Role: Cleans zone data, retaining only keys: `id`, `name`, `paused`, `name_servers`.  
    - Configuration: Removes all other fields from zone JSON.  
    - Inputs: From 'Each Host'.  
    - Outputs: To 'Merge' node (first input).  
    - Failures: If input JSON malformed, may fail.

  - **Filter DNS**  
    - Type: Code (JavaScript)  
    - Role: Filters each DNS record to keep only `type`, `content`, and `name` fields. Transforms result into a single array under `records` key.  
    - Inputs: From 'Get DNS'.  
    - Outputs: To 'Merge' node (third input).  
    - Failures: Malformed DNS records may cause issues.

  - **Filter Settings**  
    - Type: Code (JavaScript)  
    - Role: Moves `result` array to a new field `resultSETT` and deletes metadata fields (`success`, `errors`, `messages`).  
    - Inputs: From 'Get Settings'.  
    - Outputs: To 'Merge' node (second input).  
    - Failures: If `result` missing, may error.

#### Block 1.4: Data Transformation

- **Overview:**  
  Combines the filtered outputs from zones, settings, and DNS records, then flattens the nested JSON for export.

- **Nodes Involved:**  
  - Merge  
  - More Pages?  
  - Wait  
  - Increment Page  
  - Flatten

- **Node Details:**

  - **Merge**  
    - Type: Merge  
    - Role: Combines the three inputs by position: filtered zone data, filtered settings, filtered DNS records.  
    - Configuration: Mode: Combine, Combine By Position, 3 inputs expected.  
    - Inputs: From Filter TLD, Filter Settings, Filter DNS.  
    - Outputs: To 'More Pages?' node.  
    - Failures: If inputs counts differ or missing, may lead to incomplete merging.

  - **More Pages?**  
    - Type: If  
    - Role: Checks if current page number is less than total pages from API result to decide on pagination continuation.  
    - Configuration: Compares `Set Page.page` vs `Get TLDs.result_info.total_pages`.  
    - Inputs: From Merge.  
    - Outputs: True branch to 'Wait' and 'Flatten'; False branch directly to 'Flatten'.  
    - Failures: If pagination info missing, may cause logic failure.

  - **Wait**  
    - Type: Wait  
    - Role: Delays the workflow (0.3 seconds) before continuing to avoid rate limits or API throttling.  
    - Inputs: True branch from 'More Pages?'.  
    - Outputs: To 'Increment Page'.  
    - Failures: None expected.

  - **Increment Page**  
    - Type: Set  
    - Role: Increments page number by 1 to fetch next API page.  
    - Configuration: Sets `next_page` to `Set Page.page + 1`.  
    - Inputs: From Wait.  
    - Outputs: To Set Page (loop).  
    - Failures: Expression errors if page number missing or invalid.

  - **Flatten**  
    - Type: Code (JavaScript)  
    - Role: Flattens nested JSON objects into a single-level object. Special handling:  
      - DNS records concatenated as formatted strings in `json_records`.  
      - Settings array expanded into individual key-value pairs by setting id.  
    - Inputs: From 'More Pages?'.  
    - Outputs: To 'Export'.  
    - Failures: Complex data structures or missing fields could cause errors.

#### Block 1.5: Export to Google Sheets

- **Overview:**  
  Appends the transformed data rows to a Google Sheets spreadsheet's specified sheet.

- **Nodes Involved:**  
  - Export

- **Node Details:**

  - **Export**  
    - Type: Google Sheets  
    - Role: Writes data to Google Sheets, appending rows with mapped columns matching flattened data keys.  
    - Configuration:  
      - Operation: Append  
      - Document ID: Spreadsheet copied from provided template link.  
      - Sheet ID: Specific sheet tab ID (974904725).  
      - Columns: Explicit mapping to known Cloudflare zone and settings fields, plus DNS records.  
      - Authentication: OAuth2 credential connected for Google Sheets.  
    - Inputs: From Flatten node.  
    - Failures: Authentication issues, API rate limits, sheet permission errors.

#### Block 1.6: Sticky Notes & Documentation

- **Overview:**  
  Provides essential workflow usage notes, author information, and help resources.

- **Nodes Involved:**  
  - Sticky Note2 (Requirements)  
  - Sticky Note5 (Author)  
  - Sticky Note6 (Help)  
  - Sticky Note8 (Empty)  
  - Sticky Note9 (Title)  
  - Note4 (Instructions)

- **Node Details:**  
  - Sticky notes contain no input/output connections; purely informational.  
  - Key content includes required credentials, Google Sheet template link, author contact, and community forum link.

---

### 3. Summary Table

| Node Name                 | Node Type         | Functional Role                            | Input Node(s)                | Output Node(s)            | Sticky Note                                                                                                          |
|---------------------------|-------------------|-------------------------------------------|-----------------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger    | Entry trigger to start workflow           |                             | Set Page                  |                                                                                                                      |
| Set Page                  | Set               | Initialize / update pagination page       | When clicking ‘Test workflow’, Increment Page | Get TLDs                  |                                                                                                                      |
| Get TLDs                  | HTTP Request      | Fetch Cloudflare zones with pagination    | Set Page                    | Each Host                 |                                                                                                                      |
| Each Host                 | Split Out         | Split zones array into individual zones   | Get TLDs                    | Get Settings, Get DNS, Filter TLD |                                                                                                                      |
| Get Settings              | HTTP Request      | Fetch settings for each zone               | Each Host                   | Filter Settings           |                                                                                                                      |
| Get DNS                   | HTTP Request      | Fetch DNS records for each zone            | Each Host                   | Filter DNS                |                                                                                                                      |
| Filter TLD                | Code              | Keep only relevant zone fields             | Each Host                   | Merge (input 1)           |                                                                                                                      |
| Filter DNS                | Code              | Keep only relevant DNS record fields       | Get DNS                     | Merge (input 3)           |                                                                                                                      |
| Filter Settings           | Code              | Clean settings response and prepare data   | Get Settings                | Merge (input 2)           |                                                                                                                      |
| Merge                     | Merge             | Combine zone, settings, and DNS data       | Filter TLD, Filter Settings, Filter DNS | More Pages?               |                                                                                                                      |
| More Pages?               | If                | Determine if more pages need processing    | Merge                       | Wait (true), Flatten (false) |                                                                                                                      |
| Wait                      | Wait              | Delay for rate limit avoidance              | More Pages? (true)          | Increment Page            |                                                                                                                      |
| Increment Page            | Set               | Increase page number for pagination         | Wait                        | Set Page                  |                                                                                                                      |
| Flatten                   | Code              | Flatten nested JSON and format data         | More Pages?                 | Export                    |                                                                                                                      |
| Export                    | Google Sheets     | Append processed data into Google Sheets    | Flatten                     |                           |                                                                                                                      |
| Sticky Note2              | Sticky Note       | Requirements: API key, Google Sheets auth, template link |                             |                           | For storing and processing of data in this flow you will need: Cloudflare API key/token, Google Sheets auth, and copy [my sheet](https://docs.google.com/spreadsheets/d/1jt6od8FMt-Yo7A_CPGuyfqWzL7HJk6SZmQQFO6kkWBo/edit?usp=sharing) as template and match ID in 'Export'. |
| Sticky Note5              | Sticky Note       | Author info and video overview              |                             |                           | Author: Kresimir Pendic (LinkedIn: https://www.linkedin.com/in/mkdizajn/), Video overview: https://youtu.be/sW85-AxMp6g, Other templates: https://n8n.io/creators/kres/ |
| Sticky Note6              | Sticky Note       | Support resources                           |                             |                           | For help, visit community forums: https://community.n8n.io/c/questions/                                              |
| Sticky Note8              | Sticky Note       | Empty content (placeholder)                  |                             |                           |                                                                                                                      |
| Sticky Note9              | Sticky Note       | Title of workflow                           |                             |                           | CloudFlare dump => Google Sheets                                                                                      |
| Note4                     | Sticky Note       | Instructions for fetching data and expanding |                             |                           | Fetch TLDs and DNS/settings, adapt transformations to needs, expand with custom Cloudflare API endpoints: https://developers.cloudflare.com/api/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Name: When clicking ‘Test workflow’  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Set Node to Initialize Page**
   - Name: Set Page  
   - Type: Set  
   - Parameters:  
     - Add a number field named `page`  
     - Value: Expression `={{$json["next_page"] || 1}}`  
   - Connect: Manual Trigger → Set Page  
   - Set "Execute Once" to true.

3. **Create HTTP Request to Get Zones (TLDs)**
   - Name: Get TLDs  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.cloudflare.com/client/v4/zones`  
     - Authentication: Custom HTTP Auth with Cloudflare token  
     - Query Parameter: `page` set to `={{ $json.page }}`  
   - Connect: Set Page → Get TLDs

4. **Create Split Out Node to Process Each Zone Individually**
   - Name: Each Host  
   - Type: Split Out  
   - Parameters:  
     - Field to split out: `result` (from API response)  
   - Connect: Get TLDs → Each Host

5. **Create HTTP Request to Get Zone Settings**
   - Name: Get Settings  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.cloudflare.com/client/v4/zones/{{ $json.id }}/settings`  
     - Authentication: same Cloudflare credentials  
   - Connect: Each Host → Get Settings

6. **Create HTTP Request to Get DNS Records**
   - Name: Get DNS  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://api.cloudflare.com/client/v4/zones/{{ $json.id }}/dns_records`  
     - Authentication: same Cloudflare credentials  
   - Connect: Each Host → Get DNS

7. **Create Code Node to Filter Zone Data**
   - Name: Filter TLD  
   - Type: Code (JavaScript)  
   - Parameters:  
     ```js
     for (const item of $input.all()) {
       let validKeys = [ 'id', 'name', 'paused', 'name_servers' ];
       Object.keys(item.json).forEach((key) => validKeys.includes(key) || delete item.json[key]);
     }
     return $input.all();
     ```
   - Connect: Each Host → Filter TLD

8. **Create Code Node to Filter Settings Data**
   - Name: Filter Settings  
   - Type: Code (JavaScript)  
   - Parameters:  
     ```js
     for (const item of $input.all()) {
       item.json.resultSETT = item.json.result;
       delete item.json.result;
       delete item.json.success;
       delete item.json.errors;
       delete item.json.messages;
     }
     return $input.all();
     ```
   - Connect: Get Settings → Filter Settings

9. **Create Code Node to Filter DNS Records**
   - Name: Filter DNS  
   - Type: Code (JavaScript)  
   - Parameters:  
     ```js
     for (const item of $input.all()) {
       const dnsRecords = item.json.result;
       for (let dnn of dnsRecords) {
         let validKeys = ['type', 'content', 'name'];
         Object.keys(dnn).forEach((key) => validKeys.includes(key) || delete dnn[key]);
       }
       item.json = { records: dnsRecords };
     }
     return $input.all();
     ```
   - Connect: Get DNS → Filter DNS

10. **Create Merge Node to Combine Filtered Data**
    - Name: Merge  
    - Type: Merge  
    - Parameters:  
      - Mode: Combine  
      - Combine By Position  
      - Number of Inputs: 3  
    - Connect:  
      - Filter TLD → Merge (input 1)  
      - Filter Settings → Merge (input 2)  
      - Filter DNS → Merge (input 3)

11. **Create If Node to Check for More Pages**
    - Name: More Pages?  
    - Type: If  
    - Parameters:  
      - Condition: Number equals?  
        - Value1: `={{ $('Set Page').first().json.page }}`  
        - Value2: `={{ $('Get TLDs').first().json.result_info.total_pages }}`  
      - If Equal: False → more pages exist  
    - Connect: Merge → More Pages?

12. **Create Wait Node for Rate Limit Handling**
    - Name: Wait  
    - Type: Wait  
    - Parameters:  
      - Amount: 0.3 seconds  
    - Connect: More Pages? (true branch) → Wait

13. **Create Set Node to Increment Page**
    - Name: Increment Page  
    - Type: Set  
    - Parameters:  
      - Add number field `next_page`  
      - Value: Expression `={{ $('Set Page').first().json.page + 1 }}`  
    - Connect: Wait → Increment Page

14. **Loop Back to Set Page Node**
    - Connect: Increment Page → Set Page

15. **Create Code Node to Flatten and Format Data**
    - Name: Flatten  
    - Type: Code (JavaScript)  
    - Parameters:  
      ```js
      function flatten(obj, prefix = '', res = {}) {
        for (const key in obj) {
          const value = obj[key];
          const prefixedKey = prefix ? `${prefix}_${key}` : key;
          if (typeof value === 'object' && value !== null && !Array.isArray(value)) {
            flatten(value, prefixedKey, res);
          } else {
            if (prefixedKey === "json_records") {
              const formattedRecords = value
                .map(record => `${record.type} ${record.name} ${record.content}`)
                .join('\n');
              res[prefixedKey] = formattedRecords;
              continue;
            }
            if (prefixedKey === "json_resultSETT") {
              value.forEach(setting => {
                res[setting.id] = setting.value;
              });
              continue;
            }
            res[prefixedKey] = value;
          }
        }
        return res;
      }
      return $input.all().map(item => ({ json: flatten(item) }));
      ```
    - Connect: More Pages? (both branches) → Flatten

16. **Create Google Sheets Node to Export Data**
    - Name: Export  
    - Type: Google Sheets  
    - Parameters:  
      - Operation: Append  
      - Document ID: Paste your copied template sheet ID  
      - Sheet Name: Use appropriate sheet tab ID (e.g., 974904725)  
      - Columns: Map fields such as `json_id`, `json_domain`, `json_paused`, `json_name_servers`, and all Cloudflare settings fields plus `json_records`  
    - Credentials: Connect Google Sheets OAuth2 credentials with edit access  
    - Connect: Flatten → Export

17. **Add Sticky Notes** (Optional but recommended for clarity)  
    - Add notes for requirements, author, help, and instructions using Sticky Note nodes with provided content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                        | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| For storing and processing of data in this flow you will need: Cloudflare API key/token (full access), Google Spreadsheet auth connected, and copy [the template sheet](https://docs.google.com/spreadsheets/d/1jt6od8FMt-Yo7A_CPGuyfqWzL7HJk6SZmQQFO6kkWBo/edit?usp=sharing) to your account, then update the Export node with your sheet ID. | Sticky Note2                                                                                   |
| Author: Kresimir Pendic, Senior Backend professional specializing in automation, AI and data analysis. Connect on LinkedIn: https://www.linkedin.com/in/mkdizajn/. Video overview: https://youtu.be/sW85-AxMp6g. Check out other templates: https://n8n.io/creators/kres/. | Sticky Note5                                                                                   |
| Need help? For support, please create a topic on the n8n community forums: https://community.n8n.io/c/questions/                                                                  | Sticky Note6                                                                                   |
| Instructions: Fetch TLDs, DNS and settings, adapt transformations as needed, and expand with custom Cloudflare API endpoints. See Cloudflare API documentation: https://developers.cloudflare.com/api/ | Note4                                                                                         |

---

**Disclaimer:** The provided content is derived exclusively from an automated n8n workflow. It complies with all applicable content policies and contains no illegal, offensive, or protected material. All data handled is public and lawful.