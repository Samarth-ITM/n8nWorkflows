Export all Strava Activity Data to Google Sheets

https://n8nworkflows.xyz/workflows/export-all-strava-activity-data-to-google-sheets-2678


# Export all Strava Activity Data to Google Sheets

### 1. Workflow Overview

This workflow automates the export of Strava activity data into a Google Sheets spreadsheet, ensuring that the spreadsheet stays up-to-date with the latest activities without duplicates. It targets users who want to maintain a chronological, formatted log of their Strava activities for tracking, analysis, or sharing.

The workflow is structured into logical blocks:

- **1.1 Trigger & Input Data Retrieval**: Periodically triggers the workflow and reads existing activities from Google Sheets.
- **1.2 Prepare Existing Data**: Sorts and filters the saved activities to identify the most recent entries.
- **1.3 Retrieve Recent Strava Activities**: Fetches recent activities from Strava, sorts and deduplicates them.
- **1.4 Format Strava Data**: Converts raw Strava activity data into a structured format with appropriate date and time formatting.
- **1.5 Compare & Filter New Activities**: Compares new Strava data against saved data to isolate new activities not yet in the spreadsheet.
- **1.6 Append New Activities**: Adds the filtered new activities to Google Sheets in the proper format.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Input Data Retrieval

**Overview:**  
This block initiates the workflow every two hours and retrieves the current list of activities saved in Google Sheets.

**Nodes Involved:**  
- Schedule Trigger  
- activities

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Starts the workflow every 2 hours.  
  - *Configuration:* Interval set to every 2 hours.  
  - *Inputs:* None  
  - *Outputs:* Triggers the next node `activities`.  
  - *Potential Failures:* None typical; ensure server time zone consistency.  

- **activities**  
  - *Type:* Google Sheets (Read)  
  - *Role:* Reads all rows from the specified Google Sheets spreadsheet and sheet representing saved Strava activities.  
  - *Configuration:*  
    - Document ID linked to the target Google Sheets file.  
    - Sheet ID points to the relevant worksheet.  
  - *Credentials:* Google Sheets OAuth2 credential named "Nik's Google".  
  - *Inputs:* Trigger node  
  - *Outputs:* Provides the saved activities data to `sort_saved`.  
  - *Potential Failures:*  
    - Authentication errors if credentials expire or are revoked.  
    - API rate limits from Google Sheets.  
    - Sheet or document ID changes breaking the connection.  

---

#### 1.2 Prepare Existing Data

**Overview:**  
Sorts the saved activities by their reference ID and limits the dataset to the last 10 entries, then reformats these entries to extract the IDs for comparison.

**Nodes Involved:**  
- sort_saved  
- last_saved  
- saved_last

**Node Details:**

- **sort_saved**  
  - *Type:* Sort  
  - *Role:* Sorts saved activities by the "Ref" field (reference ID).  
  - *Configuration:* Sort ascending (default) on field "Ref".  
  - *Inputs:* `activities` node output  
  - *Outputs:* Sorted data to `last_saved`.  
  - *Potential Failures:* Failures if "Ref" field is missing or inconsistent.  

- **last_saved**  
  - *Type:* Limit  
  - *Role:* Keeps only the last 10 items (most recent saved activities).  
  - *Configuration:* Keep last 10 items.  
  - *Inputs:* Sorted saved activities.  
  - *Outputs:* Limited dataset to `saved_last`.  
  - *Potential Failures:* None significant; empty inputs pass through.  

- **saved_last**  
  - *Type:* Set  
  - *Role:* Extracts and sets the "id" field from the "Ref" column in saved activities, converting it to string for comparison.  
  - *Configuration:* Maps `id` = `{{$json.Ref}}` as string.  
  - *Inputs:* Limited saved activities.  
  - *Outputs:* Outputs a simplified list of saved activity IDs to the `Strava` node input, initiating Strava data retrieval.  
  - *Potential Failures:* Missing "Ref" data would cause undefined IDs.  

---

#### 1.3 Retrieve Recent Strava Activities

**Overview:**  
Fetches up to 10 recent activities from Strava API, sorts them by their ID, removes duplicates, and limits the dataset to the last 10 unique activities.

**Nodes Involved:**  
- Strava  
- sort_strava  
- Remove Duplicates  
- last_strava

**Node Details:**

- **Strava**  
  - *Type:* Strava API node  
  - *Role:* Fetches recent activities from Strava.  
  - *Configuration:*  
    - Operation: Get All Activities  
    - Limit: 10 activities per request  
  - *Credentials:* Selected Strava OAuth2 credential.  
  - *Inputs:* From `saved_last` node (though this is only a data flow trigger; no parameter dependency).  
  - *Outputs:* Raw Strava activity data to `sort_strava`.  
  - *Potential Failures:*  
    - OAuth token expiration or revocation.  
    - API rate limiting by Strava.  
    - Network connectivity issues.  

- **sort_strava**  
  - *Type:* Sort  
  - *Role:* Sorts the Strava activities by their numeric "id" field.  
  - *Configuration:* Sort on field "id".  
  - *Inputs:* Raw Strava data.  
  - *Outputs:* Sorted data to `Remove Duplicates`.  
  - *Potential Failures:* Missing or malformed "id" field causing sorting errors.  

- **Remove Duplicates**  
  - *Type:* Remove Duplicates  
  - *Role:* Eliminates duplicate activities based on the "id" field to ensure unique entries.  
  - *Configuration:* Compare on selected fields; field = "id".  
  - *Inputs:* Sorted Strava data.  
  - *Outputs:* Deduplicated data to `last_strava`.  
  - *Potential Failures:* If "id" field is absent or inconsistent, duplicates may not be removed properly.  

- **last_strava**  
  - *Type:* Limit  
  - *Role:* Limits the dataset to the last 10 unique Strava activities.  
  - *Configuration:* Keep last 10 items.  
  - *Inputs:* Deduplicated Strava activities.  
  - *Outputs:* Limited data to `strava_last` for formatting.  
  - *Potential Failures:* None significant.  

---

#### 1.4 Format Strava Data

**Overview:**  
Transforms raw Strava activity data into a simplified, properly formatted structure including date, distance (km), elevation gain, and formatted moving time, suitable for appending to Google Sheets.

**Nodes Involved:**  
- strava_last

**Node Details:**

- **strava_last**  
  - *Type:* Set  
  - *Role:* Applies formatting to Strava data fields.  
  - *Configuration:*  
    - `id`: numeric ID copied as-is  
    - `fecha`: formatted date string (day/month/year) using `DateTime.fromISO()` and `toFormat('d/M/yyyy')`  
    - `distancia`: distance converted from meters to kilometers, rounded to one decimal place  
    - `elevacion`: total elevation gain rounded to nearest integer  
    - `tiempo`: moving time formatted as `hours:minutes:seconds` string, zero-padded where needed  
  - *Inputs:* Filtered recent Strava activities  
  - *Outputs:* Formatted activities to `Code` node for filtering new entries  
  - *Potential Failures:*  
    - Date formatting errors if `start_date_local` is missing or malformed.  
    - Mathematical errors if distance or elevation is missing or non-numeric.  
    - Time formatting dependent on `moving_time` field presence.  

---

#### 1.5 Compare & Filter New Activities

**Overview:**  
Compares formatted recent Strava activities against the saved activity IDs to filter out those already present in the spreadsheet, outputting only new activities.

**Nodes Involved:**  
- Code  
- sort_results

**Node Details:**

- **Code**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filters the formatted Strava activities to exclude those already saved in Google Sheets.  
  - *Configuration:*  
    - Retrieves all items from `strava_last` (recent activities).  
    - Retrieves all items from `saved_last` (recent saved IDs).  
    - Builds a Set of saved IDs to compare against.  
    - Filters Strava activities to exclude IDs already saved.  
    - Returns filtered list of new activities.  
  - *Inputs:* Formatted Strava data (`strava_last`), saved IDs (`saved_last`)  
  - *Outputs:* Filtered new activities to `sort_results`.  
  - *Potential Failures:*  
    - Expression resolution errors if referenced nodes are renamed or missing.  
    - Type coercion issues if IDs are not consistent strings.  
    - Large datasets may impact performance.  

- **sort_results**  
  - *Type:* Sort  
  - *Role:* Sorts the filtered new activities by "id" before appending.  
  - *Configuration:* Sort on field "id".  
  - *Inputs:* Filtered new activities.  
  - *Outputs:* Sorted new activities to `Google Sheets` append node.  
  - *Potential Failures:* Missing or malformed "id" fields.  

---

#### 1.6 Append New Activities

**Overview:**  
Appends the newly identified Strava activities to the Google Sheets spreadsheet, mapping fields to their corresponding columns with proper formatting and creating clickable links for tracks.

**Nodes Involved:**  
- Google Sheets

**Node Details:**

- **Google Sheets**  
  - *Type:* Google Sheets (Append)  
  - *Role:* Adds new activities as rows to the specified Google Sheets document and sheet.  
  - *Configuration:*  
    - Operation: Append  
    - Document ID and Sheet ID set to target spreadsheet and worksheet.  
    - Columns mapped explicitly:  
      - "Kms" = distance (`distancia`)  
      - "Ref" = activity ID (`id`)  
      - "Fecha" = formatted date (`fecha`)  
      - "Track" = clickable URL to Strava activity page (`http://www.strava.com/activities/{{ id }}`)  
      - "Tiempo" = formatted moving time (`tiempo`)  
      - "Desnivel" = elevation gain (`elevacion`)  
    - Other columns ("Bicicleta", "Terreno") present but not mapped (empty).  
  - *Credentials:* Google Sheets OAuth2 credential "Nik's Google".  
  - *Inputs:* Sorted new activities.  
  - *Outputs:* None (end of workflow).  
  - *Potential Failures:*  
    - Authentication errors if credentials expire.  
    - API rate limits.  
    - Field mapping errors if expected fields are missing.  
    - Spreadsheet permission or quota issues.

---

### 3. Summary Table

| Node Name       | Node Type             | Functional Role                                | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                                  |
|-----------------|-----------------------|------------------------------------------------|-------------------------|------------------------|--------------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Schedule Trigger       | Periodically triggers the workflow every 2 hours | (none)                  | activities             |                                                                                                              |
| activities      | Google Sheets (Read)   | Reads existing saved Strava activities from Sheets | Schedule Trigger        | sort_saved             | In 'activities' node, enter the file and sheet name where data will be saved.                                |
| sort_saved      | Sort                  | Sorts saved activities by Ref (ID)              | activities              | last_saved             |                                                                                                              |
| last_saved      | Limit                 | Limits to last 10 saved activities               | sort_saved              | saved_last             |                                                                                                              |
| saved_last      | Set                   | Extracts saved activity IDs for comparison       | last_saved              | Strava                 |                                                                                                              |
| Strava          | Strava API             | Fetches recent Strava activities                  | saved_last              | sort_strava            | Select corresponding Strava credential.                                                                      |
| sort_strava     | Sort                  | Sorts Strava activities by id                      | Strava                  | Remove Duplicates       |                                                                                                              |
| Remove Duplicates| Remove Duplicates      | Removes duplicate activities by id                | sort_strava             | last_strava            |                                                                                                              |
| last_strava     | Limit                 | Limits to last 10 unique Strava activities        | Remove Duplicates        | strava_last            |                                                                                                              |
| strava_last     | Set                   | Formats Strava data fields (date, distance, time) | last_strava             | Code                   | Adjust format of dates, times, and distances here as needed.                                                |
| Code            | Code (JavaScript)      | Filters new activities not present in saved data  | strava_last, saved_last | sort_results           | Compares IDs to avoid duplicates. Logs filtered results for debugging.                                       |
| sort_results    | Sort                  | Sorts new activities by id before appending        | Code                    | Google Sheets           |                                                                                                              |
| Google Sheets   | Google Sheets (Append) | Appends new activities to Google Sheets            | sort_results            | (none)                 | Credentials needed for Google Sheets. Map columns precisely. Includes clickable Strava activity links.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 2 hours.  

2. **Create Google Sheets node to read saved activities**  
   - Type: Google Sheets (Read)  
   - Set Document ID to your Google Sheets file ID.  
   - Set Sheet ID to the worksheet containing saved Strava activities.  
   - Assign Google Sheets OAuth2 credentials.  
   - Connect Schedule Trigger → this node.  

3. **Create Sort node to sort saved activities**  
   - Type: Sort  
   - Sort by field "Ref" (reference ID).  
   - Connect activities → sort_saved.  

4. **Create Limit node to keep last 10 saved activities**  
   - Type: Limit  
   - Keep last 10 items.  
   - Connect sort_saved → last_saved.  

5. **Create Set node to extract saved IDs**  
   - Type: Set  
   - Map new field `id` = `{{$json.Ref}}` (cast to string).  
   - Connect last_saved → saved_last.  

6. **Create Strava node to fetch recent activities**  
   - Type: Strava  
   - Operation: Get All Activities  
   - Limit: 10  
   - Assign your Strava OAuth2 credential.  
   - Connect saved_last → Strava.  

7. **Create Sort node to sort Strava activities**  
   - Type: Sort  
   - Sort by field "id".  
   - Connect Strava → sort_strava.  

8. **Create Remove Duplicates node**  
   - Type: Remove Duplicates  
   - Compare field "id".  
   - Connect sort_strava → Remove Duplicates.  

9. **Create Limit node to keep last 10 unique Strava activities**  
   - Type: Limit  
   - Keep last 10 items.  
   - Connect Remove Duplicates → last_strava.  

10. **Create Set node to format Strava activity data**  
    - Type: Set  
    - Assign fields:  
      - `id`: `{{$json.id}}` (number)  
      - `fecha`: format `start_date_local` as `d/M/yyyy`  
      - `distancia`: convert `distance` meters to km rounded one decimal  
      - `elevacion`: round `total_elevation_gain`  
      - `tiempo`: format `moving_time` as `H:mm:ss` string  
    - Connect last_strava → strava_last.  

11. **Create Code node to filter new activities**  
    - Type: Code (JavaScript)  
    - Code logic:  
      - Get all items from `strava_last` and `saved_last`.  
      - Build a Set of saved IDs.  
      - Filter Strava items to keep only those whose IDs are not in saved IDs.  
      - Return filtered list.  
    - Connect strava_last and saved_last → Code (both as inputs).  

12. **Create Sort node to sort filtered new activities**  
    - Type: Sort  
    - Sort by field "id".  
    - Connect Code → sort_results.  

13. **Create Google Sheets node to append new activities**  
    - Type: Google Sheets (Append)  
    - Document ID and Sheet ID same as step 2.  
    - Map columns:  
      - "Kms" = `{{$json.distancia}}`  
      - "Ref" = `{{$json.id}}`  
      - "Fecha" = `{{$json.fecha}}`  
      - "Track" = `http://www.strava.com/activities/{{$json.id}}` (URL string)  
      - "Tiempo" = `{{$json.tiempo}}`  
      - "Desnivel" = `{{$json.elevacion}}`  
    - Authenticate with Google Sheets OAuth2.  
    - Connect sort_results → Google Sheets.  

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                             |
|------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Complete the Set up credentials step first: Google Sheets and Strava OAuth2 credentials are required. | Workflow set up instructions.                               |
| Adjust date, time, and distance formats in the 'strava_last' Set node to suit your locale or preference. | Formatting flexibility.                                     |
| The workflow checks for missing activities to avoid duplicates and only appends new data.            | Ensures data integrity and efficient updates.              |
| More detailed explanation and video available at: [Sherblog.es Strava to Google Sheets with n8n](https://sherblog.es/de-strava-a-google-sheets-gracias-a-n8n/) | External resource for deeper understanding and help.       |
| Template originally created on n8n version 1.72.1.                                                    | Version compatibility note.                                 |

---

This documentation fully describes the workflow’s architecture and node-level details, enabling advanced users or automation agents to understand, reproduce, or modify the workflow confidently.