Automatic Eventbrite Attendance Tagging in KlickTipp

https://n8nworkflows.xyz/workflows/automatic-eventbrite-attendance-tagging-in-klicktipp-10028


# Automatic Eventbrite Attendance Tagging in KlickTipp

### 1. Workflow Overview

This workflow automates the tagging of Eventbrite event attendees in KlickTipp based on their attendance status. It is designed primarily for event organizers, digital marketers, and KlickTipp users who want to segment their contacts automatically according to whether they participated in an event or not. The workflow fetches the latest attendee data from Eventbrite every 15 minutes, analyzes attendance status, and applies corresponding tags in KlickTipp for segmentation and follow-up campaigns.

The workflow logic is divided into these functional blocks:

- **1.1 Data Reception & Structuring:** Scheduled trigger to fetch attendee data from Eventbrite, then split the list of attendees to process each individually.
- **1.2 Analyzing Attendance:** Check each attendee’s checked-in status and branch logic based on attendance.
- **1.3 Tagging Contacts in KlickTipp:** Apply tags in KlickTipp to mark contacts as “Attended” or “Not attended” based on attendance check.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Reception & Structuring

**Overview:**  
This block initiates the workflow on a schedule and obtains the current attendee list from a specific Eventbrite event. It then splits the attendee list into individual records for processing.

**Nodes Involved:**  
- Trigger every 15min  
- List Evenbrite attendees from event  
- Split attendee list  
- Sticky Note (descriptive)

**Node Details:**

- **Trigger every 15min**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow every 15 minutes to ensure attendee data is updated regularly.  
  - Configuration: Interval set to 15 minutes.  
  - Input: None (trigger node)  
  - Output: Triggers the HTTP request node.  
  - Edge cases: Workflow will not run if n8n instance is down or scheduling conflicts occur.  
  - Notes: Ensures near real-time syncing of attendance data.

- **List Evenbrite attendees from event**  
  - Type: HTTP Request  
  - Role: Retrieves all attendees for a specified Eventbrite event using OAuth2 authentication.  
  - Configuration: URL is set to the Eventbrite API endpoint for attendees of a particular event (`/events/{event_id}/attendees/`). Uses preconfigured OAuth2 credentials for Eventbrite.  
  - Key expressions: None beyond the configured URL and credentials.  
  - Input: Trigger node output  
  - Output: JSON containing a list of attendees in the `attendees` field.  
  - Edge cases: API rate limits, expired OAuth2 tokens, event ID misconfiguration, network failures.  
  - Notes: Must update event ID in URL to target correct event.

- **Split attendee list**  
  - Type: SplitOut  
  - Role: Splits the array of attendees into individual items for separate processing downstream.  
  - Configuration: Splits on the `attendees` field from the HTTP response.  
  - Input: Attendee list JSON  
  - Output: Single attendee JSON objects one by one.  
  - Edge cases: If attendees array is empty or null, no outputs will be generated; handle empty lists gracefully.

- **Sticky Note**  
  - Type: Sticky Note (Informational)  
  - Content: Marks the block as "1. Data reception & structuring" for clarity.

---

#### 1.2 Analyzing Attendance

**Overview:**  
This block evaluates each attendee’s `checked_in` status to determine if they attended the event. It branches the workflow accordingly.

**Nodes Involved:**  
- Attendance check (Switch)  
- Sticky Note1 (descriptive)

**Node Details:**

- **Attendance check**  
  - Type: Switch  
  - Role: Routes each attendee based on whether the `checked_in` property is true or false.  
  - Configuration: Two outputs:  
    - "Attended" if `checked_in` is true  
    - "Not attended" if `checked_in` is false  
  - Key expressions: Evaluates `{{$json.checked_in}}` as boolean.  
  - Input: Individual attendee JSON from Split attendee list  
  - Output: Routes to corresponding tagging nodes for attended or not attended.  
  - Edge cases: Missing or malformed `checked_in` field may cause misrouting or errors; should validate presence of field.  
  - Notes: Strict boolean check ensures accurate segmentation.

- **Sticky Note1**  
  - Type: Sticky Note (Informational)  
  - Content: Marks the block as "2. Analyzing & saving data".

---

#### 1.3 Tagging Contacts in KlickTipp

**Overview:**  
Based on the attendance evaluation, this block tags each contact in KlickTipp with the appropriate tag indicating attendance or non-attendance.

**Nodes Involved:**  
- Tag contact for attendance (KlickTipp node)  
- Tag contact for non attendance (KlickTipp node)  
- Sticky Note4 (descriptive)

**Node Details:**

- **Tag contact for attendance**  
  - Type: KlickTipp  
  - Role: Tags the contact in KlickTipp as having attended the event.  
  - Configuration: Uses the attendee’s email (`{{$json.profile.email}}`) to identify the contact; applies tag ID `13634770` corresponding to "Eventbrite | Participated".  
  - Input: Routed from “Attended” output of Attendance check  
  - Output: None further downstream (end node)  
  - Credentials: KlickTipp API using username/password (preconfigured)  
  - Edge cases: Email not found in KlickTipp, API errors, invalid tag ID, network issues.  
  - Notes: Tag IDs must be customized to the user’s KlickTipp setup.

- **Tag contact for non attendance**  
  - Type: KlickTipp  
  - Role: Tags the contact in KlickTipp as having not attended the event.  
  - Configuration: Uses the attendee’s email (`{{$json.profile.email}}`); applies tag ID `13634786` corresponding to "Eventbrite | Not participated".  
  - Input: Routed from “Not attended” output of Attendance check  
  - Output: None further downstream (end node)  
  - Credentials: Same KlickTipp API credentials as above  
  - Edge cases: Same as attendance tag node.  
  - Notes: User must ensure tag ID corresponds to non-attendance tag in KlickTipp.

- **Sticky Note4**  
  - Type: Sticky Note (Informational)  
  - Content: Marks the block as "2. Analyze & Tag" to clarify tagging actions.

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                                | Input Node(s)               | Output Node(s)                   | Sticky Note                                                                                                                       |
|----------------------------|--------------------------|-----------------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Trigger every 15min         | Schedule Trigger         | Initiates workflow every 15 minutes            | None                        | List Evenbrite attendees from event | This node triggers the flow in the defined frequency.                                                                            |
| List Evenbrite attendees from event | HTTP Request            | Fetches attendees list from Eventbrite API     | Trigger every 15min          | Split attendee list              | This node gets all attendees of the defined event.                                                                               |
| Split attendee list         | SplitOut                 | Splits attendee list into individual records   | List Evenbrite attendees from event | Attendance check               | This node splits the list of attendees.                                                                                          |
| Attendance check            | Switch                   | Routes attendees based on checked_in status    | Split attendee list          | Tag contact for attendance, Tag contact for non attendance | This node checks whether the attendees were checked in at the event.                                                             |
| Tag contact for attendance  | KlickTipp                | Tags contacts as attended                       | Attendance check (Attended)  | None                            | This node tags the contact for their attendance at the event.                                                                    |
| Tag contact for non attendance | KlickTipp                | Tags contacts as not attended                   | Attendance check (Not attended) | None                            | This node tags the contact for their non attendance at the event.                                                                |
| Sticky Note                 | Sticky Note              | Informational, block label                      | None                        | None                            | ## 1. Data reception & structuring                                                                                               |
| Sticky Note1                | Sticky Note              | Informational, block label                      | None                        | None                            | ## 2. Analyzing & saving data                                                                                                    |
| Sticky Note2                | Sticky Note              | Informational, overview and instructions       | None                        | None                            | ## Who’s it for ... (Detailed project description and setup notes)                                                               |
| Sticky Note3                | Sticky Note              | Informational, timing and sync frequency       | None                        | None                            | ## 1. Data Reception\nTriggered every **15 min** to fetch the latest Eventbrite attendees. Keeps your participant data synced automatically. 💡 Adjust timing for real-time or daily sync. |
| Sticky Note4                | Sticky Note              | Informational, tagging summary                  | None                        | None                            | ## 2. Analyze & Tag\nChecks who **attended vs. missed** the event using `checked_in`. Tags participants in KlickTipp for instant segmentation. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set interval to trigger every 15 minutes.

2. **Create an HTTP Request node**  
   - Name: List Evenbrite attendees from event  
   - Method: GET  
   - URL: `https://www.eventbriteapi.com/v3/events/{your_event_id}/attendees/` (replace `{your_event_id}` with your actual Eventbrite event ID)  
   - Authentication: OAuth2 (select or create Eventbrite OAuth2 credentials with attendee read scope)  
   - Connect Schedule Trigger output to this node’s input.

3. **Create a SplitOut node**  
   - Name: Split attendee list  
   - Set the field to split: `attendees`  
   - Connect HTTP Request output to this node.

4. **Create a Switch node**  
   - Name: Attendance check  
   - Add two rules:  
     - Output "Attended": Condition where `{{$json.checked_in}}` is boolean true  
     - Output "Not attended": Condition where `{{$json.checked_in}}` is boolean false  
   - Connect SplitOut output to this node.

5. **Create KlickTipp node for attendance tagging**  
   - Name: Tag contact for attendance  
   - Operation: contact-tagging  
   - Email: Expression `{{$json.profile.email}}`  
   - Tag ID: Enter your KlickTipp tag ID for attended contacts (e.g., `13634770`)  
   - Credentials: Select your KlickTipp API credentials (username/password)  
   - Connect “Attended” output of Switch node to this node.

6. **Create KlickTipp node for non-attendance tagging**  
   - Name: Tag contact for non attendance  
   - Operation: contact-tagging  
   - Email: Expression `{{$json.profile.email}}`  
   - Tag ID: Enter your KlickTipp tag ID for non-attended contacts (e.g., `13634786`)  
   - Credentials: Reuse KlickTipp API credentials  
   - Connect “Not attended” output of Switch node to this node.

7. **Add Sticky Notes (optional but recommended for documentation clarity)**  
   - Add sticky notes above/beside the nodes to label blocks:  
     - "1. Data reception & structuring" near Trigger, HTTP Request, and Split nodes  
     - "2. Analyzing & saving data" near Switch and KlickTipp tagging nodes  
     - Optionally add overview notes describing the workflow purpose and setup instructions.

8. **Configure Credentials**  
   - Set up OAuth2 credentials for Eventbrite with attendee read permissions.  
   - Set up KlickTipp API credentials using username and password with API access enabled.

9. **Test the workflow manually**  
   - Run once manually to verify data retrieval and tagging.  
   - Check KlickTipp to verify contacts are tagged correctly based on attendance status.

10. **Activate the workflow**  
   - Once tested successfully, activate to run automatically every 15 minutes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                                                                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Designed for **event organizers**, **digital marketers**, and **KlickTipp users** who already sync their Eventbrite registrants to KlickTipp and want to automatically track event attendance.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | Project description embedded in Sticky Note2.                                                                                                                                                                                                     |
| Prerequisite: Contacts must already exist in KlickTipp. Use the related workflow **“Subscribe Eventbrite orders to KlickTipp”** to import registrants before running this workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Workflow dependency and prerequisite note.                                                                                                                                                                                                         |
| Important: Eventbrite must record attendee check-ins via the **Eventbrite Organizer App** or barcode scanning to enable accurate attendance tagging.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Operational requirement for attendance accuracy.                                                                                                                                                                                                   |
| Customization ideas: Adjust schedule frequency, handle multiple events, tag by ticket class or refund status, and combine with order/refund workflows for full funnel tracking.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Suggested workflow expansions.                                                                                                                                                                                                                      |
| Campaign expansion ideas: Refund tagging, post-event automation (thank-you or replay campaigns), and performance insights using KlickTipp segmentation.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Marketing and automation use cases from tagged data.                                                                                                                                                                                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.