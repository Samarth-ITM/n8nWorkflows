Sync Upcoming Anime Releases from Jikan API to Google Calendar with Rate Limiting

https://n8nworkflows.xyz/workflows/sync-upcoming-anime-releases-from-jikan-api-to-google-calendar-with-rate-limiting-11126


# Sync Upcoming Anime Releases from Jikan API to Google Calendar with Rate Limiting

### 1. Workflow Overview

This workflow automates the synchronization of upcoming anime release dates for a specific voice actor (or person) with a Google Calendar. It queries the Jikan API to find the voice actor’s ID, fetches their anime roles, filters for anime that have not yet aired, and creates calendar events for those release dates. It includes strict rate limiting to respect the Jikan API’s limits and avoid temporary bans.

Logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow.
- **1.2 Voice Actor Search:** Queries Jikan API to find the voice actor’s ID by name.
- **1.3 Fetch Anime Roles:** Retrieves anime roles associated with the person, respecting API rate limits.
- **1.4 Process Anime Details:** Fetches detailed anime information, filters for unreleased anime.
- **1.5 Calendar Event Creation:** Adds upcoming anime release dates as events in Google Calendar.
- **1.6 Rate Limiting Controls:** Wait nodes inserted to throttle requests and batch processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Initiates the workflow manually.

**Nodes Involved:**  
- Start Workflow

**Node Details:**

- **Start Workflow**  
  - Type: Manual Trigger  
  - Configuration: No parameters; manual start only  
  - Inputs: None  
  - Outputs: 1 (to “Search Voice Actor”)  
  - Failure modes: None typical  
  - Notes: Entry point for user-initiated runs

---

#### 2.2 Voice Actor Search

**Overview:**  
Queries Jikan API’s `/people` endpoint to find the voice actor’s ID by name (default “Mamoru Miyano”).

**Nodes Involved:**  
- Search Voice Actor  
- Rate Limit Protection

**Node Details:**

- **Search Voice Actor**  
  - Type: HTTP Request  
  - Configuration: GET `https://api.jikan.moe/v4/people` with query params:  
    - `q`: voice actor name (default “Mamoru Miyano”)  
    - `limit`: 1 (fetch only the top result)  
  - Expressions: Static query parameters; could be parameterized for dynamic input  
  - Inputs: From Start Workflow  
  - Outputs: 1 (to Rate Limit Protection)  
  - Failures: HTTP errors, no results (empty data array), API rate limits  
  - Notes: Must handle missing or ambiguous results gracefully

- **Rate Limit Protection**  
  - Type: Wait  
  - Configuration: Wait 2 seconds before next node to respect API limits  
  - Inputs: From Search Voice Actor  
  - Outputs: 1 (to Get Anime Roles)  
  - Failures: None typical  
  - Notes: Ensures compliance with Jikan API rate limiting (max 3 req/sec)

---

#### 2.3 Fetch Anime Roles

**Overview:**  
Fetches all anime roles of the voice actor from Jikan API, splitting the array of roles for batch processing.

**Nodes Involved:**  
- Get Anime Roles  
- Split Out  
- Loop Over Items

**Node Details:**

- **Get Anime Roles**  
  - Type: HTTP Request  
  - Configuration: GET `https://api.jikan.moe/v4/people/{{ mal_id }}/anime` where `mal_id` comes from the prior search node  
  - Inputs: From Rate Limit Protection  
  - Outputs: 1 (to Split Out)  
  - Failures: HTTP errors, invalid mal_id, no roles found, API rate limits  
  - Notes: Dynamic URL uses must ensure valid JSON path referencing

- **Split Out**  
  - Type: Split Out  
  - Configuration: Splits the `data` array from prior node into separate items  
  - Inputs: From Get Anime Roles  
  - Outputs: 1 (to Loop Over Items)  
  - Failures: Empty array (no roles), expression failures if data structure changes

- **Loop Over Items**  
  - Type: Split In Batches  
  - Configuration: Processes items in batches (default size) to control flow  
  - Inputs: From Split Out  
  - Outputs: 2 outputs: main and “next batch”  
  - Failures: Batch processing errors, empty batches  
  - Notes: Connects to Batch Rate Limit Wait for pacing

---

#### 2.4 Process Anime Details

**Overview:**  
For each anime role, fetches detailed anime info, checks if it has not yet aired, then passes for calendar event creation.

**Nodes Involved:**  
- Batch Rate Limit Wait  
- Fetch Anime Details  
- Check if Not Aired

**Node Details:**

- **Batch Rate Limit Wait**  
  - Type: Wait  
  - Configuration: Wait for 2 seconds before next request to respect API limits  
  - Inputs: From Loop Over Items (batch continuation output)  
  - Outputs: 1 (to Fetch Anime Details)  
  - Failures: None typical

- **Fetch Anime Details**  
  - Type: HTTP Request  
  - Configuration: GET `https://api.jikan.moe/v4/anime/{{ mal_id }}` where `mal_id` is from the current item’s anime info  
  - Inputs: From Batch Rate Limit Wait  
  - Outputs: 1 (to Check if Not Aired)  
  - Failures: HTTP errors, invalid mal_id, API limits

- **Check if Not Aired**  
  - Type: If  
  - Configuration: Checks if `data.status` equals “Not yet aired”  
  - Inputs: From Fetch Anime Details  
  - Outputs: 1 (true branch to Create an Event), 2 (false branch - no connection)  
  - Failures: JSON path errors, unexpected status strings

---

#### 2.5 Calendar Event Creation

**Overview:**  
Creates Google Calendar events for each upcoming anime release date.

**Nodes Involved:**  
- Create an event

**Node Details:**

- **Create an event**  
  - Type: Google Calendar  
  - Configuration:  
    - Calendar: Primary calendar (credential selected)  
    - Event summary: Anime’s main title (`titles[0].title`)  
    - Start and end date/time: Both set to `aired.from` date from anime data  
  - Inputs: From Check if Not Aired (true branch)  
  - Outputs: 1 (to Loop Over Items input 2 - batch continuation)  
  - Failures: Google OAuth errors, invalid date formats, permission issues  
  - Notes: Uses OAuth2 credential named “Google Calendar account 2” (must be configured)

---

### 3. Summary Table

| Node Name           | Node Type          | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                         |
|---------------------|--------------------|----------------------------------------|------------------------|------------------------|---------------------------------------------------------------------------------------------------|
| Start Workflow      | Manual Trigger     | Entry point to start workflow manually | None                   | Search Voice Actor      | # Sync Upcoming Anime Releases to Google Calendar... setup instructions included                   |
| Search Voice Actor  | HTTP Request       | Search for voice actor by name          | Start Workflow         | Rate Limit Protection   | ## 1. Search Anime - Define voice actor to search                                                   |
| Rate Limit Protection | Wait               | Enforce API rate limit (wait 2 seconds) | Search Voice Actor     | Get Anime Roles        | ## 2. Process & Rate Limit - Fetch roles respecting API limit                                      |
| Get Anime Roles     | HTTP Request       | Get anime roles of the person            | Rate Limit Protection  | Split Out              | ## 2. Process & Rate Limit                                                                           |
| Split Out           | Split Out          | Split anime roles array into individual items | Get Anime Roles        | Loop Over Items        | ## 2. Process & Rate Limit                                                                           |
| Loop Over Items     | Split In Batches   | Process anime roles in batches            | Split Out              | Batch Rate Limit Wait (output 2), Create an event (output 1) | ## 2. Process & Rate Limit                                                                           |
| Batch Rate Limit Wait | Wait               | Wait 2 seconds between batch requests    | Loop Over Items (batch continuation) | Fetch Anime Details     | ## 2. Process & Rate Limit                                                                           |
| Fetch Anime Details | HTTP Request       | Fetch detailed info for each anime       | Batch Rate Limit Wait   | Check if Not Aired      | ## 2. Process & Rate Limit                                                                           |
| Check if Not Aired  | If                 | Filter anime that are not yet aired       | Fetch Anime Details     | Create an event (true)  | ## 3. Add to Calendar - Create Google Calendar event for upcoming anime                             |
| Create an event     | Google Calendar    | Add upcoming anime release as calendar event | Check if Not Aired (true) | Loop Over Items (batch continuation) | ## 3. Add to Calendar                                                                             |
| Sticky Note         | Sticky Note        | Workflow summary and setup instructions  | None                   | None                   | # Sync Upcoming Anime Releases to Google Calendar ... full description                             |
| Sticky Note1        | Sticky Note        | Block 1 description                      | None                   | None                   | ## 1. Search Anime                                                                                  |
| Sticky Note2        | Sticky Note        | Block 2 description                      | None                   | None                   | ## 2. Process & Rate Limit                                                                          |
| Sticky Note3        | Sticky Note        | Block 3 description                      | None                   | None                   | ## 3. Add to Calendar                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named “Start Workflow” with default settings.

2. **Add an HTTP Request node** named “Search Voice Actor”:  
   - Method: GET  
   - URL: `https://api.jikan.moe/v4/people`  
   - Query Parameters:  
     - `q`: Set to the voice actor name you want to track (default “Mamoru Miyano”)  
     - `limit`: 1  
   - Connect “Start Workflow” output to this node input.

3. **Add a Wait node** named “Rate Limit Protection”:  
   - Set amount to 2 seconds (to respect Jikan API rate limits)  
   - Connect “Search Voice Actor” output to this node input.

4. **Add an HTTP Request node** named “Get Anime Roles”:  
   - Method: GET  
   - URL: use expression to dynamically set:  
     `https://api.jikan.moe/v4/people/{{ $json.data[0].mal_id }}/anime`  
   - Connect “Rate Limit Protection” output to this node input.

5. **Add a Split Out node** named “Split Out”:  
   - Field to split out: `data`  
   - Connect “Get Anime Roles” output to this node input.

6. **Add a Split In Batches node** named “Loop Over Items”:  
   - Options: default (custom batch size if desired)  
   - Connect “Split Out” output to this node input.

7. **Add a Wait node** named “Batch Rate Limit Wait”:  
   - Set amount to 2 seconds (throttle batch requests)  
   - Connect the second output (batch continuation) of “Loop Over Items” to this node input.

8. **Add an HTTP Request node** named “Fetch Anime Details”:  
   - Method: GET  
   - URL: use expression:  
     `https://api.jikan.moe/v4/anime/{{ $json.anime.mal_id }}`  
   - Connect “Batch Rate Limit Wait” output to this node input.

9. **Add an If node** named “Check if Not Aired”:  
   - Condition: string equal  
     Left: `{{$json.data.status}}`  
     Right: `Not yet aired`  
   - Connect “Fetch Anime Details” output to this node input.

10. **Add a Google Calendar node** named “Create an event”:  
    - Operation: Create Event  
    - Calendar ID: primary (select your Google Calendar OAuth2 credential)  
    - Event Summary: set to expression `{{$json.data.titles[0].title}}`  
    - Start DateTime: `{{$json.data.aired.from}}`  
    - End DateTime: same as start  
    - Connect “Check if Not Aired” true output to this node input.

11. **Connect “Create an event” output back to the first input of “Loop Over Items”** (to continue batching).

12. **Configure Google Calendar OAuth2 credentials** in n8n under your account, ensuring access to the calendar you wish to update.

13. **Test the workflow** by manually triggering “Start Workflow”.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow strictly respects Jikan API rate limits (max 3 requests per second) by using wait nodes with 2-second delays. | Important to avoid temporary banning from API due to over-requesting.                           |
| Setup instructions and workflow summary are included in sticky notes within the workflow.                                   | Sticky notes provide guidance on usage and configuration.                                      |
| Jikan API docs: https://docs.api.jikan.moe/                                                                                 | Official API documentation for deeper understanding or extension of the workflow.              |
| Google Calendar OAuth2 credentials must be properly configured with calendar access permissions.                            | Required for event creation; otherwise, node will fail with authentication errors.             |
| The voice actor name in “Search Voice Actor” node query parameter can be changed to track any other person supported by Jikan API. | Enables customization for different users or voice actors.                                    |

---

Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.