Enrich domain URLs with Hunter.io and export contacts to Google Sheets

https://n8nworkflows.xyz/workflows/enrich-domain-urls-with-hunter-io-and-export-contacts-to-google-sheets-14453


# Enrich domain URLs with Hunter.io and export contacts to Google Sheets

# 1. Workflow Overview

This workflow manually enriches domain data from Google Sheets with contact information from Hunter.io, then writes the enriched results back into another sheet tab.

Its primary use case is lead enrichment: starting from a list of domains, it retrieves contact records associated with those domains and exports selected fields into a structured Google Sheets tab for downstream sales, outreach, or research operations.

## 1.1 Input Reception

The workflow begins with a manual trigger. A user launches it directly from the n8n editor.

## 1.2 Source Data Retrieval

After execution starts, the workflow reads rows from a Google Sheets document, specifically from a tab intended to hold domains.

## 1.3 Domain Enrichment with Hunter.io

Each input row is passed to a Hunter node. This block is intended to perform domain-based email/contact enrichment using Hunter.io.

## 1.4 Export to Destination Sheet

The results returned by Hunter.io are appended into another Google Sheets tab called “Exported data”, using a predefined column mapping.

## 1.5 Documentation / In-Canvas Guidance

Several sticky notes document setup expectations, provide the sheet template link, explain the workflow behavior, and include branding/contact information.

---

# 2. Block-by-Block Analysis

## 2.1 Workflow Documentation and Setup Notes

### Overview
This block contains only sticky notes. These notes explain the workflow’s purpose, setup steps, sheet template, processing stages, and external branding/contact details. They do not affect execution but are important for reproduction and maintenance.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note16
- Sticky Note17

### Node Details

#### Sticky Note
- **Type and technical role:** Sticky Note; visual documentation node.
- **Configuration choices:** Contains a Google Sheets template reference.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None beyond sticky note support in n8n.
- **Edge cases or potential failure types:** None at runtime.
- **Sub-workflow reference:** None.
- **Content:**  
  Google sheets Template  
  https://docs.google.com/spreadsheets/d/180ca70GWcm-BpM8U2ygS6jqhCTE_avx93Kd4d0USe5A

#### Sticky Note1
- **Type and technical role:** Sticky Note; visual explanation of the trigger.
- **Configuration choices:** Describes manual workflow start behavior.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.
- **Content:**  
  Manual workflow trigger  
  Starts the workflow when the user manually clicks 'Execute workflow' in the n8n editor.

#### Sticky Note2
- **Type and technical role:** Sticky Note; explains the source-read and enrichment block.
- **Configuration choices:** Documents the Sheets → Hunter stage.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.
- **Content:**  
  Fetch rows and enrich with Hunter  
  Reads rows from a source Google Sheet, then sends each row to Hunter.io to look up email addresses or domain contact data.

#### Sticky Note3
- **Type and technical role:** Sticky Note; explains the export block.
- **Configuration choices:** Documents appending data to the destination sheet.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.
- **Content:**  
  Save enriched results to sheet  
  Appends the Hunter-enriched contact data as new rows into a destination Google Sheet.

#### Sticky Note4
- **Type and technical role:** Sticky Note; main workflow description and setup guide.
- **Configuration choices:** Provides operating summary, setup checklist, and customization ideas.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.
- **Content summary:**  
  - Manual execution
  - Read rows from Google Sheets
  - Enrich through Hunter.io
  - Append results back to Google Sheets
  - Setup checklist for credentials and node configuration
  - Suggestion to replace manual trigger with a schedule
  - Suggestion to add a filter before export

#### Sticky Note16
- **Type and technical role:** Sticky Note; branding/image note.
- **Configuration choices:** Embeds a logo image.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** Requires n8n sticky note markdown/image rendering.
- **Edge cases or potential failure types:** External image may fail to load visually; no workflow impact.
- **Sub-workflow reference:** None.
- **Content:**  
  Logo Growth AI image link.

#### Sticky Note17
- **Type and technical role:** Sticky Note; branding/contact note.
- **Configuration choices:** Includes business promotion and LinkedIn contact links.
- **Key expressions or variables used:** None.
- **Input and output connections:** None.
- **Version-specific requirements:** None.
- **Edge cases or potential failure types:** None.
- **Sub-workflow reference:** None.
- **Content:**  
  Need more advanced automation solutions? Contact us for custom enterprise workflows!  
  Growth-AI.fr  
  https://www.linkedin.com/in/allanvaccarizi/  
  https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/

---

## 2.2 Manual Execution Entry Point

### Overview
This block defines how the workflow starts. It uses a manual trigger, meaning the flow runs only when explicitly launched from the n8n editor.

### Nodes Involved
- When Manually Triggered

### Node Details

#### When Manually Triggered
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; workflow entry node for manual execution.
- **Configuration choices:** No custom parameters are set.
- **Key expressions or variables used:** None.
- **Input and output connections:**  
  - **Input:** None
  - **Output:** Read Domains from Sheets
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - Cannot run automatically on a schedule unless replaced with another trigger node.
  - Requires a user to execute it in the n8n UI.
- **Sub-workflow reference:** None.

---

## 2.3 Source Sheet Read

### Overview
This block reads the source rows from Google Sheets. It is intended to return records containing domain values that can be sent to Hunter.io for enrichment.

### Nodes Involved
- Read Domains from Sheets

### Node Details

#### Read Domains from Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`; reads data from a Google Sheets tab.
- **Configuration choices:**  
  - Uses a Google Sheets document by URL:  
    `https://docs.google.com/spreadsheets/d/180ca70GWcm-BpM8U2ygS6jqhCTE_avx93Kd4d0USe5A`
  - Reads from sheet/tab: `gid=0`
  - No advanced options are configured.
- **Key expressions or variables used:** None are visible in the JSON.
- **Input and output connections:**  
  - **Input:** When Manually Triggered
  - **Output:** Find Emails via Hunter
- **Version-specific requirements:** Google Sheets node type version 4.7.
- **Edge cases or potential failure types:**  
  - OAuth2 credential failure or expired Google token
  - Wrong spreadsheet permissions
  - Wrong tab selection or renamed sheet
  - Empty sheet or rows without a usable domain column
  - Schema/header mismatch affecting downstream node expectations
- **Sub-workflow reference:** None.

**Important implementation note:**  
The JSON does not explicitly show a column mapping or an expression indicating which field contains the domain. Therefore, successful operation depends on the output structure of this node matching what the Hunter node expects, or on additional manual configuration in the UI.

---

## 2.4 Hunter.io Enrichment

### Overview
This block sends each row from Google Sheets to Hunter.io to find domain-related contacts or emails. It is the enrichment core of the workflow.

### Nodes Involved
- Find Emails via Hunter

### Node Details

#### Find Emails via Hunter
- **Type and technical role:** `n8n-nodes-base.hunter`; sends requests to Hunter.io using API credentials.
- **Configuration choices:**  
  - Limit is set to `10`
  - `filters` is present but empty
  - `domain` is configured as `=`
- **Key expressions or variables used:**  
  - The `domain` parameter is currently set to `=`, which strongly suggests an incomplete or broken expression.
- **Input and output connections:**  
  - **Input:** Read Domains from Sheets
  - **Output:** Append Exported Data to Sheets
- **Version-specific requirements:** Type version 1.
- **Edge cases or potential failure types:**  
  - Invalid or incomplete domain expression
  - Missing input field from Google Sheets
  - Hunter API authentication issues
  - API quota/rate-limit exhaustion
  - Invalid domain values
  - No contacts found for a given domain
  - Response shape may not match the append node’s expected schema
- **Sub-workflow reference:** None.

**Critical note:**  
The `domain` field is not properly configured in the provided JSON. In a functional workflow, this should reference a domain field from the previous node, for example an expression like `{{$json.domain}}` or the actual source column name. Without this correction, the node is likely to fail or return no useful results.

---

## 2.5 Export to Google Sheets

### Overview
This block appends the Hunter.io output into a destination Google Sheets tab. It uses a fixed destination sheet and explicit column definitions.

### Nodes Involved
- Append Exported Data to Sheets

### Node Details

#### Append Exported Data to Sheets
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends rows to Google Sheets.
- **Configuration choices:**  
  - Operation: `append`
  - Spreadsheet URL:  
    `https://docs.google.com/spreadsheets/d/180ca70GWcm-BpM8U2ygS6jqhCTE_avx93Kd4d0USe5A`
  - Destination sheet/tab: `Exported data` (`gid=762290939`)
  - Mapping mode: define below
  - Explicit output columns defined:
    - Mail adress
    - Confidence
    - First name
    - Last name
    - Phone number
    - Linkedin
    - Position
    - Position Raw
    - Seniority
    - Departement
    - Twitter
  - `attemptToConvertTypes`: false
  - `convertFieldsToString`: false
- **Key expressions or variables used:**  
  The configured values for all mapped columns are set to either `=` or, in one case, `='` for Phone number. This indicates that the mapping expressions are incomplete or blank.
- **Input and output connections:**  
  - **Input:** Find Emails via Hunter
  - **Output:** None
- **Version-specific requirements:** Google Sheets node type version 4.7.
- **Edge cases or potential failure types:**  
  - OAuth2 authentication failure
  - Destination sheet missing or inaccessible
  - Incomplete column expressions causing blank rows or expression errors
  - Hunter output may be nested rather than flat, requiring transformation before append
  - Column names in the sheet may differ from node schema
  - Appending arrays of contacts may need item splitting first
- **Sub-workflow reference:** None.

**Critical note:**  
The column mappings are not actually completed in the provided JSON. Each column should normally map to a Hunter response field via expressions such as `{{$json.first_name}}`, `{{$json.last_name}}`, etc. As provided, this node will not reliably export meaningful data until the mappings are corrected.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | Sticky Note | Provides the Google Sheets template link |  |  | # Google sheets Template  /  https://docs.google.com/spreadsheets/d/180ca70GWcm-BpM8U2ygS6jqhCTE_avx93Kd4d0USe5A |
| Sticky Note1 | Sticky Note | Documents the manual trigger behavior |  |  | ## Manual workflow trigger / Starts the workflow when the user manually clicks 'Execute workflow' in the n8n editor. |
| Sticky Note2 | Sticky Note | Documents the read-and-enrich stage |  |  | ## Fetch rows and enrich with Hunter / Reads rows from a source Google Sheet, then sends each row to Hunter.io to look up email addresses or domain contact data. |
| Sticky Note3 | Sticky Note | Documents the export stage |  |  | ## Save enriched results to sheet / Appends the Hunter-enriched contact data as new rows into a destination Google Sheet. |
| When Manually Triggered | Manual Trigger | Starts the workflow manually from the editor |  | Read Domains from Sheets | ## Manual workflow trigger / Starts the workflow when the user manually clicks 'Execute workflow' in the n8n editor. |
| Read Domains from Sheets | Google Sheets | Reads source domain rows from the spreadsheet | When Manually Triggered | Find Emails via Hunter | ## Fetch rows and enrich with Hunter / Reads rows from a source Google Sheet, then sends each row to Hunter.io to look up email addresses or domain contact data. |
| Find Emails via Hunter | Hunter | Enriches each domain with Hunter.io contact/email data | Read Domains from Sheets | Append Exported Data to Sheets | ## Fetch rows and enrich with Hunter / Reads rows from a source Google Sheet, then sends each row to Hunter.io to look up email addresses or domain contact data. |
| Append Exported Data to Sheets | Google Sheets | Appends enriched contact rows to the destination sheet | Find Emails via Hunter |  | ## Save enriched results to sheet / Appends the Hunter-enriched contact data as new rows into a destination Google Sheet. |
| Sticky Note4 | Sticky Note | Main in-canvas workflow explanation and setup guidance |  |  | ## Domain url to exported data / ### How it works / 1. The workflow is triggered manually by clicking 'Execute workflow'. / 2. It reads rows from a Google Sheet to retrieve a list of domains or companies to enrich. / 3. Each row is passed to Hunter.io to find associated email addresses or contact data. / 4. The enriched results are appended as new rows back into a Google Sheet. / ### Setup steps / Connect your Google Sheets credentials to both the 'Get row(s) in sheet' and 'Append row in sheet' nodes. / Configure the source Google Sheet URL and sheet name in the 'Get row(s) in sheet' node. / Connect your Hunter.io API credentials to the 'Hunter' node and configure the lookup type (e.g., domain search or email finder). / Configure the destination Google Sheet URL and column mapping in the 'Append row in sheet' node. / ### Customization / You can swap the manual trigger for a scheduled trigger (e.g., daily) to run enrichment automatically. You can also add a filter node between Hunter and the append step to only save rows where valid emails were found. |
| Sticky Note16 | Sticky Note | Branding/logo display |  |  | ![Logo Growth AI](https://cdn.prod.website-files.com/6825df5b20329ba581df4914/68d413c43f8729fa336568a6_Logo_horizontal.png) |
| Sticky Note17 | Sticky Note | Branding and contact links |  |  | ## Need more advanced automation solutions? Contact us for custom enterprise workflows! / # Growth-AI.fr / https://www.linkedin.com/in/allanvaccarizi/ / https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/ |

---

# 4. Reproducing the Workflow from Scratch

Below is the exact rebuild process, including the corrections that are required for the workflow to function properly.

## 4.1 Prepare the Google Sheet
1. Create or duplicate a Google Sheets file.
2. Add a source tab for domains, for example:
   - Tab name: `Domain`
   - Include at least one header column such as `domain`
3. Add a destination tab for exports:
   - Tab name: `Exported data`
4. In `Exported data`, create these headers:
   - `Mail adress`
   - `Confidence`
   - `First name`
   - `Last name`
   - `Phone number`
   - `Linkedin`
   - `Position`
   - `Position Raw`
   - `Seniority`
   - `Departement`
   - `Twitter`

## 4.2 Create the Trigger
1. Add a node: **Manual Trigger**
2. Name it: **When Manually Triggered**
3. Leave default settings unchanged.

## 4.3 Create the Source Google Sheets Node
1. Add a node: **Google Sheets**
2. Name it: **Read Domains from Sheets**
3. Connect it after **When Manually Triggered**
4. Configure Google Sheets credentials:
   - Use **Google Sheets OAuth2**
   - Authenticate the Google account that can access the spreadsheet
5. Set the spreadsheet:
   - Document ID / URL: the target Google Sheets file
6. Select the source sheet:
   - Choose the domain tab, such as `Domain`
7. Configure the node to read rows from the sheet.
8. Ensure the output includes the domain column, for example `domain`.

## 4.4 Create the Hunter.io Node
1. Add a node: **Hunter**
2. Name it: **Find Emails via Hunter**
3. Connect it after **Read Domains from Sheets**
4. Configure Hunter credentials:
   - Use your Hunter API credential
5. Set the lookup mode according to the node UI. This workflow is intended for domain-based search.
6. Set **Limit** to `10`
7. Set the **Domain** field to an expression referencing the source row’s domain field, for example:
   - `{{$json.domain}}`
   - If your source column has another name, use that exact field name instead
8. Leave filters empty unless you want to restrict results.

## 4.5 Validate Hunter Output Shape
1. Run the workflow once with a test domain.
2. Inspect the Hunter node output.
3. Confirm whether:
   - Each contact is already emitted as a separate item, or
   - The node returns an array nested inside one item
4. If the result is nested, add an intermediate node such as:
   - **Item Lists** or **Code**
   - Split the contacts array into one item per contact before writing to Sheets

This step is important because the append node expects row-like items.

## 4.6 Create the Destination Google Sheets Append Node
1. Add another **Google Sheets** node
2. Name it: **Append Exported Data to Sheets**
3. Connect it after **Find Emails via Hunter**  
   - Or after your flatten/split node if you added one
4. Configure the same Google Sheets OAuth2 credential
5. Set:
   - Operation: **Append**
   - Document URL: same spreadsheet
   - Sheet: `Exported data`
6. Set mapping mode to **Define Below**
7. Map each output column to the corresponding Hunter field. The exact expressions depend on the Hunter output format, but a typical mapping would be:
   - `Mail adress` → email field
   - `Confidence` → confidence field
   - `First name` → first name field
   - `Last name` → last name field
   - `Phone number` → phone field
   - `Linkedin` → linkedin field
   - `Position` → position field
   - `Position Raw` → raw position field
   - `Seniority` → seniority field
   - `Departement` → department field
   - `Twitter` → twitter field
8. If needed, use expressions like:
   - `{{$json.email}}`
   - `{{$json.confidence}}`
   - `{{$json.first_name}}`
   - `{{$json.last_name}}`
   - `{{$json.phone_number}}`
   - `{{$json.linkedin}}`
   - `{{$json.position}}`
   - `{{$json.position_raw}}`
   - `{{$json.seniority}}`
   - `{{$json.department}}`
   - `{{$json.twitter}}`

## 4.7 Add the Visual Notes
To match the original canvas, add these sticky notes:

1. A note with the Google Sheets template link
2. A note describing the manual trigger
3. A note describing the Sheets-to-Hunter block
4. A note describing the append-to-sheet block
5. A larger note describing:
   - how the workflow works
   - setup steps
   - customization suggestions
6. A logo note using the Growth AI image
7. A contact note with:
   - Growth-AI.fr
   - LinkedIn links

## 4.8 Final Connection Order
Build the core execution chain in this order:

1. **When Manually Triggered**
2. **Read Domains from Sheets**
3. **Find Emails via Hunter**
4. **Append Exported Data to Sheets**

Optional improved chain if Hunter returns nested contacts:

1. **When Manually Triggered**
2. **Read Domains from Sheets**
3. **Find Emails via Hunter**
4. **Split / Flatten Contacts**
5. **Append Exported Data to Sheets**

## 4.9 Required Credentials
You need:
- **Google Sheets OAuth2 API**
  - Must have access to the spreadsheet
- **Hunter API**
  - Must be valid and within quota limits

## 4.10 Important Functional Corrections
The provided workflow JSON is not fully operational as-is. To make it work:
1. Replace the Hunter `domain` value `=` with a real expression referencing the sheet’s domain field.
2. Replace the append node column values (`=`, `='`) with real expressions referencing Hunter output fields.
3. If Hunter returns arrays, add a transformation/splitting step before append.
4. Verify source headers and destination headers exactly match your intended field mapping.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Google Sheets template | https://docs.google.com/spreadsheets/d/180ca70GWcm-BpM8U2ygS6jqhCTE_avx93Kd4d0USe5A |
| Growth AI logo image | https://cdn.prod.website-files.com/6825df5b20329ba581df4914/68d413c43f8729fa336568a6_Logo_horizontal.png |
| Growth-AI.fr contact/branding note | In-canvas branding |
| Allan Vaccarizi LinkedIn | https://www.linkedin.com/in/allanvaccarizi/ |
| Hugo Marinier LinkedIn | https://www.linkedin.com/in/hugo-marinier-%F0%9F%A7%B2-6537b633/ |
| Suggested enhancement from notes: replace manual trigger with scheduled trigger | Use Cron or Schedule Trigger in n8n |
| Suggested enhancement from notes: add a filter before append | Use IF or Filter logic to save only valid Hunter results |

## Additional Technical Note
There are no sub-workflows, Execute Workflow nodes, webhook entry points, or multiple runtime entry points in this workflow. The only execution entry point is the Manual Trigger node.