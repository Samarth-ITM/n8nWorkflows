Enrich LinkedIn Profiles with Ghost Genius and Google Sheets (No Account Needed)

https://n8nworkflows.xyz/workflows/enrich-linkedin-profiles-with-ghost-genius-and-google-sheets--no-account-needed--5161


# Enrich LinkedIn Profiles with Ghost Genius and Google Sheets (No Account Needed)

### 1. Workflow Overview

This workflow is designed to enrich a list of LinkedIn profile URLs stored in a Google Sheet by fetching detailed profile data from the Ghost Genius API (a cookieless LinkedIn profile data provider). The enriched data is then written back into the Google Sheet, updating the existing entries.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Initiates the workflow manually.
- **1.2 Data Retrieval:** Reads LinkedIn profile URLs from a Google Sheet.
- **1.3 Profile Enrichment:** For each URL, calls the Ghost Genius API to get detailed profile information.
- **1.4 Data Update:** Updates the Google Sheet with the enriched profile data.

Supplementary sticky notes provide resources, setup instructions, and video links.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block triggers the workflow manually to start the profile enrichment process.
- **Nodes Involved:** `Start`
- **Node Details:**

  - **Node Name:** Start  
  - **Type:** Manual Trigger  
  - **Technical Role:** Initiates workflow execution manually on demand.  
  - **Configuration:** Default manual trigger, no parameters set.  
  - **Key Expressions / Variables:** None.  
  - **Connections:** Outputs to `Recover Profiles`.  
  - **Version-Specific Requirements:** None.  
  - **Edge Cases / Failures:** None expected; manual trigger.  

#### 1.2 Data Retrieval

- **Overview:** This block reads the LinkedIn profile URLs from a specified Google Sheet to be enriched.
- **Nodes Involved:** `Recover Profiles`
- **Node Details:**

  - **Node Name:** Recover Profiles  
  - **Type:** Google Sheets  
  - **Technical Role:** Retrieves the list of profile URLs from a Google Sheet document.  
  - **Configuration Choices:**  
    - Reads sheet with ID `1D3aCtCOHUZd2B2KgPmi5_L4a0XyjNLiOiyLH3Jl8PKI`.  
    - Sheet name/gid is `gid=0` (Sheet1).  
    - Reads all rows, no filters applied.  
  - **Key Expressions / Variables:** None; outputs rows with JSON containing `url` field.  
  - **Input Connections:** From `Start`.  
  - **Output Connections:** To `Get Profile Info`.  
  - **Version-Specific Requirements:** Google Sheets node v4.5.  
  - **Edge Cases / Failures:**  
    - Google Sheets API authentication failure.  
    - Empty or malformed rows without valid URLs.  
    - Rate limits from Google Sheets API.  

#### 1.3 Profile Enrichment

- **Overview:** For each LinkedIn profile URL, this block calls the Ghost Genius API to retrieve detailed profile information.
- **Nodes Involved:** `Get Profile Info`
- **Node Details:**

  - **Node Name:** Get Profile Info  
  - **Type:** HTTP Request  
  - **Technical Role:** Sends HTTP GET requests to Ghost Genius API to fetch profile data based on the URL.  
  - **Configuration Choices:**  
    - URL endpoint: `https://api.ghostgenius.fr/v2/profile`.  
    - Query parameter: `url` dynamically set to each profile URL from `Recover Profiles` node (`={{ $json.url }}`).  
    - Authentication: Generic HTTP Header Auth (credentials stored separately).  
    - Batching: Batch size 1, batch interval 2000 ms (throttling requests to avoid API limits).  
    - Retry enabled on failure.  
    - On error: Continue regular output (does not break workflow if one request fails).  
  - **Key Expressions / Variables:**  
    - Query parameter expression: `={{ $json.url }}`  
  - **Input Connections:** From `Recover Profiles`.  
  - **Output Connections:** To `Update`.  
  - **Version-Specific Requirements:** HTTP Request node v4.2.  
  - **Edge Cases / Failures:**  
    - API authentication errors (invalid or expired credentials).  
    - Network timeouts or connectivity issues.  
    - API rate limiting (handled partially by batching and retry).  
    - Empty or incorrect URL causing API to return no data.  
    - Partial or malformed API responses.  

#### 1.4 Data Update

- **Overview:** Updates the original Google Sheet with the enriched profile data obtained from Ghost Genius.
- **Nodes Involved:** `Update`
- **Node Details:**

  - **Node Name:** Update  
  - **Type:** Google Sheets  
  - **Technical Role:** Updates rows in the Google Sheet matching on the LinkedIn profile URL, enriching multiple profile fields.  
  - **Configuration Choices:**  
    - Document ID and Sheet ID same as `Recover Profiles`.  
    - Operation: Update existing rows based on matching column `url`.  
    - Mappings contain detailed multi-field updates including first name, last name, tagline, location, connections, followers, hiring status, open to work status, summary, languages, skills, education (up to 3 entries), experience (up to 5 entries).  
    - Complex expressions format multi-line education and experience data from nested JSON fields.  
  - **Key Expressions / Variables:**  
    - Expressions use optional chaining and array mapping/filtering, e.g.:  
      ```
      ={{ $json.skills?.length > 0 ? $json.skills.join(', ') : '' }}
      ={{ [$json.geo?.location?.name, $json.geo?.country?.name].filter(Boolean).join(', ') }}
      ={{ $json.languages?.map(lang => lang.name + ' (' + lang.level + ')').join(', ') ?? '' }}
      ```
    - Matching column is `url` to properly update existing rows.  
  - **Input Connections:** From `Get Profile Info`.  
  - **Output Connections:** None (end of main flow).  
  - **Version-Specific Requirements:** Google Sheets node v4.5.  
  - **Edge Cases / Failures:**  
    - Matching row not found for a given URL (no update happens for that row).  
    - Google Sheets API limits or authentication issues.  
    - Data formatting errors causing update to fail.  
    - Large payload size issues if profiles have extensive data.  

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                 | Input Node(s)    | Output Node(s)   | Sticky Note                                                                                                                     |
|-------------------|---------------------|--------------------------------|------------------|------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Start             | Manual Trigger      | Initiates workflow manually     | -                | Recover Profiles |                                                                                                                                |
| Recover Profiles  | Google Sheets       | Reads LinkedIn URLs from Sheet  | Start            | Get Profile Info |                                                                                                                                |
| Get Profile Info  | HTTP Request        | Fetches profile data from API   | Recover Profiles | Update           |                                                                                                                                |
| Update            | Google Sheets       | Updates Sheet with enriched data| Get Profile Info  | -                |                                                                                                                                |
| Sticky Note       | Sticky Note         | Workflow title and summary      | -                | -                | ## Enrich a LinkedIn profile list                                                                                             |
| Sticky Note1      | Sticky Note         | Resources and setup links       | -                | -                | ## Resources\n[Video Setup](https://youtu.be/kIOJeMoCfp4)\n\nGoogle Sheet: [Make a copy here](https://docs.google.com/spreadsheets/d/1oiV25COrTMP20nMgNvbtMo2LHcm7ehkqOxbAB-s6PhA/edit?usp=sharing)\n\nAPI LinkedIn (cookieless): [Ghost Genius](https://ghostgenius.fr)\n\nGoogle Sheet Credential Setup: [Video Tutorial](https://www.youtube.com/watch?v=pWGXlZBGu4k) |
| Sticky Note2      | Sticky Note         | Setup video link                | -                | -                | # [Setup Video](https://youtu.be/kIOJeMoCfp4)                                                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: `Start`
   - No special configuration needed.
   - This node will start the workflow manually.

3. **Add a Google Sheets node to read profiles:**
   - Name: `Recover Profiles`
   - Operation: Read
   - Document ID: `1D3aCtCOHUZd2B2KgPmi5_L4a0XyjNLiOiyLH3Jl8PKI` (replace with your own sheet ID)
   - Sheet Name / GID: `gid=0` (Sheet1)
   - Read all rows without filters.
   - Connect `Start` node’s output to this node’s input.

4. **Add an HTTP Request node to get profile info:**
   - Name: `Get Profile Info`
   - HTTP Method: GET
   - URL: `https://api.ghostgenius.fr/v2/profile`
   - Authentication: Set to generic HTTP Header Authentication with your Ghost Genius API key (create credentials in n8n).
   - Query Parameters: Add parameter `url` with value expression: `={{ $json.url }}`
   - Enable batching with Batch Size = 1 and Batch Interval = 2000 ms to avoid hitting API limits.
   - Enable retry on failure.
   - On error: Continue regular output (do not stop workflow).
   - Connect `Recover Profiles` node output to this node input.

5. **Add a Google Sheets node to update profiles:**
   - Name: `Update`
   - Operation: Update
   - Document ID and Sheet Name same as `Recover Profiles` node.
   - Matching Column: `url` (to update rows by URL)
   - Map columns with expressions to populate enriched profile data fields:
     - Example mappings:
       - `url`: `={{ $('Recover Profiles').item.json.url }}`
       - `Firstname`: `={{ $json.first_name }}`
       - `Lastname`: `={{ $json.last_name }}`
       - `Tagline`: `={{ $json.headline }}`
       - `Location`: `={{ [$json.geo?.location?.name, $json.geo?.country?.name].filter(Boolean).join(', ') }}`
       - `Skills`: `={{ $json.skills?.length > 0 ? $json.skills.join(', ') : '' }}`
       - `Languages`: `={{ $json.languages?.map(lang => lang.name + ' (' + lang.level + ')').join(', ') ?? '' }}`
       - Education and Experience fields formatted with multiline strings concatenating relevant details.
       - Boolean fields converted to string (`toString()`).
   - Connect `Get Profile Info` node output to this node input.

6. **Add sticky notes (optional) for documentation inside the n8n editor:**
   - Title note describing workflow purpose.
   - Resources note with links to setup videos, Google Sheet template, and API provider.
   - Setup video note with direct link.

7. **Save and activate the workflow.**

8. **Run manually via the `Start` node to begin enrichment.**

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Video setup guide for the workflow and API usage.                                                                   | [https://youtu.be/kIOJeMoCfp4](https://youtu.be/kIOJeMoCfp4)                                                 |
| Google Sheet template for input and output data.                                                                     | [Make a copy here](https://docs.google.com/spreadsheets/d/1oiV25COrTMP20nMgNvbtMo2LHcm7ehkqOxbAB-s6PhA/edit?usp=sharing) |
| Ghost Genius API for cookieless LinkedIn profile enrichment.                                                        | [https://ghostgenius.fr](https://ghostgenius.fr)                                                             |
| Google Sheets credential setup tutorial for n8n.                                                                     | [https://www.youtube.com/watch?v=pWGXlZBGu4k](https://www.youtube.com/watch?v=pWGXlZBGu4k)                     |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.