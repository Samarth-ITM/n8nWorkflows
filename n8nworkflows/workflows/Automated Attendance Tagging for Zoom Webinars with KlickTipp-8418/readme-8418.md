Automated Attendance Tagging for Zoom Webinars with KlickTipp

https://n8nworkflows.xyz/workflows/automated-attendance-tagging-for-zoom-webinars-with-klicktipp-8418


# Automated Attendance Tagging for Zoom Webinars with KlickTipp

### 1. Workflow Overview

This workflow automates attendance tagging for Zoom webinars within KlickTipp using n8n. It listens to Zoom webinar end events, validates webhook requests, retrieves participant and absentee data from Zoom’s API, evaluates attendance duration thresholds, and applies appropriate tags in KlickTipp to segment contacts by attendance level. The workflow supports multiple webinar topics (e.g., Beginner and Expert) and distinguishes internal users (such as hosts) from external participants.

Logical blocks included:

- **1.1 Zoom Webhook Reception and Validation:** Receives Zoom webhook events and handles URL validation.
- **1.2 Zoom Webinar Data Retrieval:** Fetches participant and absentee data from Zoom’s API.
- **1.3 Data Filtering and Segmentation:** Filters internal users and routes participants/absentees by webinar topic.
- **1.4 Attendance Evaluation and Tagging:** Checks attendance percentage thresholds and tags contacts in KlickTipp accordingly.
- **1.5 Response Handling:** Sends appropriate responses back to Zoom for validation and regular events.

---

### 2. Block-by-Block Analysis

#### 1.1 Zoom Webhook Reception and Validation

- **Overview:**  
  Listens for incoming Zoom webhook POST requests, distinguishes between URL validation calls and webinar ended events, and replies accordingly to Zoom’s validation challenge or event notifications.

- **Nodes Involved:**  
  - Listen to ending Zoom webinars (Webhook)  
  - URL Validation check (If)  
  - Crypto  
  - Build Validation Body (Set)  
  - Respond to Webhook (Validation)  
  - Build Normal Event Body (Set)  
  - Respond to Webhook (Events)  
  - Wait 1 second (Wait)

- **Node Details:**

  - **Listen to ending Zoom webinars**  
    - Type: Webhook  
    - Role: Entry point for Zoom webhook events  
    - Configured to listen on `/zoom` path with POST method, responding via a response node  
    - Triggers workflow upon Zoom events including `webinar.ended` and URL validation requests  
    - Possible failure: Missing or malformed webhook calls, authentication errors if webhook secret is mismatched

  - **URL Validation check**  
    - Type: If  
    - Role: Checks if event type is `endpoint.url_validation` indicating Zoom’s webhook URL validation  
    - Uses expression `{{$json.body.event}}` equals `endpoint.url_validation`  
    - Two outputs: True (validation) and False (normal event)  
    - Potential failure: Incorrect event data format or missing `event` field

  - **Crypto**  
    - Type: Crypto  
    - Role: Computes HMAC SHA256 hash of the `plainToken` from Zoom payload using secret key to validate webhook  
    - Inputs: `plainToken` extracted from incoming JSON  
    - Outputs: Encrypted token for response to Zoom validation  
    - Failure cases: Incorrect secret, missing token, expression evaluation errors

  - **Build Validation Body**  
    - Type: Set  
    - Role: Constructs JSON body containing both the plain and encrypted tokens required by Zoom for validation reply  
    - Inputs: `plainToken` and encrypted token from Crypto node  
    - Outputs: Validation response payload

  - **Respond to Webhook (Validation)**  
    - Type: Respond to Webhook  
    - Role: Sends the validation response back to Zoom to confirm webhook URL registration  
    - Connected only from Build Validation Body

  - **Build Normal Event Body**  
    - Type: Set  
    - Role: Prepares a simple JSON response with status "ok" for regular Zoom events  
    - Used when event is not URL validation

  - **Respond to Webhook (Events)**  
    - Type: Respond to Webhook  
    - Role: Sends acknowledgment to Zoom after receiving normal events like webinar ending  
    - Output connected to Wait 1 second node
  
  - **Wait 1 second**  
    - Type: Wait  
    - Role: Delays workflow for 1 second to allow Zoom API data to become available before fetching  
    - Prevents premature API calls that might return incomplete data

---

#### 1.2 Zoom Webinar Data Retrieval

- **Overview:**  
  After receiving a webinar ended event, fetches detailed participant and absentee lists from Zoom’s API using the webinar UUID.

- **Nodes Involved:**  
  - Get past Zoom webinar participants (HTTP Request)  
  - Get past Zoom webinar absentees (HTTP Request)  
  - Split webinar participants (SplitOut)  
  - Split webinar absentees (SplitOut)

- **Node Details:**

  - **Get past Zoom webinar participants**  
    - Type: HTTP Request  
    - Role: Retrieves participant list for the past webinar using Zoom API endpoint `/past_webinars/{uuid}/participants`  
    - Uses OAuth2 credentials for Zoom API authentication  
    - URL dynamically built from webhook JSON path expression referencing webinar UUID  
    - Failure modes: OAuth token expiry, API rate limits, invalid UUID

  - **Get past Zoom webinar absentees**  
    - Type: HTTP Request  
    - Role: Retrieves absentee list for the past webinar using Zoom API endpoint `/past_webinars/{uuid}/absentees`  
    - Same authentication and dynamic URL construction as participants node  
    - Similar failure modes to participants node

  - **Split webinar participants**  
    - Type: SplitOut  
    - Role: Iterates over each participant in the participants array for individual processing  
    - Input field: `participants` array from previous node output  
    - Failure modes: Empty participant lists, malformed data

  - **Split webinar absentees**  
    - Type: SplitOut  
    - Role: Iterates over each absentee in the absentees array for tagging processing  
    - Input field: `registrants` array from absentees node output

---

#### 1.3 Data Filtering and Segmentation

- **Overview:**  
  Filters out internal users (hosts, panelists) from participant processing and routes participants and absentees by webinar topic to apply different tagging logic.

- **Nodes Involved:**  
  - Filter internal users (Filter)  
  - Route by webinar name (Switch)  
  - Route by meeting name1 (Switch)

- **Node Details:**

  - **Filter internal users**  
    - Type: Filter  
    - Role: Excludes participants flagged as `internal_user=true` (e.g., hosts) from further tagging  
    - Condition: Checks boolean field `internal_user` in participant JSON  
    - Failure cases: Missing `internal_user` flag, potential misclassification

  - **Route by webinar name**  
    - Type: Switch  
    - Role: Routes participants by webinar topic string contained in the Zoom payload (e.g., “E-Mail Zustellung für Anfänger” or “E-Mail Zustellung für Experten”)  
    - Uses string contains condition on `$('Listen to ending Zoom webinars').item.json.body.payload.object.topic`  
    - Outputs branch per webinar topic for targeted tagging workflows  
    - Failure modes: Webinar topic string changes or mismatches, case sensitivity issues

  - **Route by meeting name1**  
    - Type: Switch  
    - Role: Routes absentees by webinar meeting name in the same manner as participants for tagging absence tags  
    - Similar configuration and failure modes as Route by webinar name

---

#### 1.4 Attendance Evaluation and Tagging

- **Overview:**  
  Determines attendance level based on duration attended relative to total webinar duration, applies KlickTipp tags for full attendance (≥90%), general attendance (60%-89%), or absence, per webinar type.

- **Nodes Involved:**  
  - Check full attendance (If)  
  - Check full attendance1 (If)  
  - Check general attendance (If)  
  - Check general attendance1 (If)  
  - Tag participant for full attendance  
  - Tag participant with full attendance1  
  - Tag participant for general attendance  
  - Tag participant for general attendance1  
  - Tag absentee for non attendance  
  - Tag absentee for non attendance1

- **Node Details:**

  - **Check full attendance / Check full attendance1**  
    - Type: If  
    - Role: Checks if participant's `duration` is greater than 90% of webinar total duration (converted to seconds)  
    - Expressions compare `$json.duration` to `$('Listen to ending Zoom webinars').item.json.body.payload.object.duration * 60 * 0.9`  
    - `Check full attendance1` also filters out internal users before evaluation  
    - Failure: Missing or invalid duration data, zero webinar duration

  - **Check general attendance / Check general attendance1**  
    - Type: If  
    - Role: Checks if participant’s `duration` is greater than 60% of webinar total duration (but less than 90% as handled by logic connections)  
    - Also ensures participant is not internal user  
    - Similar expression logic as full attendance nodes

  - **Tag participant for full attendance / Tag participant with full attendance1**  
    - Type: Custom KlickTipp Node (community node)  
    - Role: Tags participant contact in KlickTipp with a “full attendance” tag specific to the webinar branch  
    - Uses participant’s email (`$json.user_email`) and webinar-specific tag IDs  
    - Credentials: KlickTipp API with OAuth2 or API key  
    - Failure modes: API errors, invalid email addresses, tag ID mismatches

  - **Tag participant for general attendance / Tag participant for general attendance1**  
    - Type: KlickTipp node  
    - Role: Tags participants who attended between 60% and 89% of webinar duration  
    - Same configuration as full attendance tags, different tag IDs

  - **Tag absentee for non attendance / Tag absentee for non attendance1**  
    - Type: Custom KlickTipp Node  
    - Role: Tags absentees (contacts who registered but did not attend) for respective webinar branch  
    - Uses absentee’s email and webinar-specific "non-attendance" tag IDs

---

#### 1.5 Response Handling

- **Overview:**  
  Sends appropriate HTTP responses back to Zoom for validation and regular event webhook calls.

- **Nodes Involved:**  
  - Respond to Webhook (Validation)  
  - Respond to Webhook (Events)

- **Node Details:**

  - **Respond to Webhook (Validation)**  
    - Sends HMAC validation response to Zoom during webhook registration or validation calls

  - **Respond to Webhook (Events)**  
    - Sends simple `{ "status": "ok" }` JSON response to Zoom on receiving webinar end events to acknowledge receipt

---

### 3. Summary Table

| Node Name                      | Node Type                      | Functional Role                                | Input Node(s)                    | Output Node(s)                     | Sticky Note                                                                                                                  |
|-------------------------------|--------------------------------|-----------------------------------------------|---------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Listen to ending Zoom webinars | Webhook                        | Entry point listening to Zoom webhook events | -                               | URL Validation check             | This node is listening to Zoom events and triggers when a webinar was ended.                                                  |
| URL Validation check           | If                            | Checks if event is URL validation call        | Listen to ending Zoom webinars  | Crypto (true), Build Normal Event Body (false) | This node checks whether the received data was a ending meeting or a URL validation call by Zoom.                             |
| Crypto                        | Crypto                        | Creates HMAC SHA256 hash for validation       | URL Validation check (true)      | Build Validation Body            | This node prepares the response to the Zoom endpoint in order to answer to the validation.                                     |
| Build Validation Body          | Set                           | Builds validation response body                | Crypto                          | Respond to Webhook (Validation) | This node builds the body for the response to the Zoom endpoint in order to answer for validation.                             |
| Respond to Webhook (Validation)| Respond to Webhook             | Sends validation response to Zoom              | Build Validation Body           | -                              | This node sends the response back to the Zoom endpoint.                                                                       |
| Build Normal Event Body        | Set                           | Builds response body for regular events        | URL Validation check (false)    | Respond to Webhook (Events)     | This node builds a body for a response to the endpoint in case of a regular event.                                             |
| Respond to Webhook (Events)    | Respond to Webhook             | Sends acknowledgment for regular events        | Build Normal Event Body         | Wait 1 second                   | This node answers to the endpoint in case of regular Zoom events - such as ending meetings in this case.                       |
| Wait 1 second                 | Wait                          | Delays processing 1 sec to allow data availability | Respond to Webhook (Events)     | Get past Zoom webinar absentees, Get past Zoom webinar participants | This node waits 1 second to avoid calling the past webinar too early before it is available.                                   |
| Get past Zoom webinar absentees| HTTP Request                  | Fetches absentees list from Zoom API            | Wait 1 second                  | Split webinar absentees         | This node pulls data of the webinar absentees in order to process them.                                                        |
| Get past Zoom webinar participants| HTTP Request              | Fetches participants list from Zoom API         | Wait 1 second                  | Split webinar participants      | This node pulls data of the webinar participants in order to process them.                                                     |
| Split webinar absentees        | SplitOut                      | Iterates over absentees array                   | Get past Zoom webinar absentees | Route by meeting name1          | This node iterates through all absentees.                                                                                      |
| Split webinar participants     | SplitOut                      | Iterates over participants array                | Get past Zoom webinar participants | Filter internal users           | This node iterates through all participants.                                                                                   |
| Filter internal users          | Filter                        | Removes internal users (hosts) from participants| Split webinar participants      | Route by webinar name           | This node filters out internal users from the webinar such as the host.                                                        |
| Route by webinar name          | Switch                       | Routes participants by webinar topic            | Filter internal users           | Check full attendance, Check full attendance1 | This node filters for the different webinars via the name in order to process them.                                            |
| Route by meeting name1         | Switch                       | Routes absentees by webinar topic                | Split webinar absentees         | Tag absentee for non attendance, Tag absentee for non attendance1 |                                                                                                                              |
| Check full attendance          | If                            | Checks if attendance ≥ 90%                       | Route by webinar name           | Tag participant for full attendance, Check general attendance | This node checks whether the participants have spent more than 90% in the webinar or not.                                       |
| Check full attendance1         | If                            | Checks if attendance ≥ 90%, filters internal users | Route by webinar name           | Tag participant with full attendance1, Check general attendance1 | This node checks whether the participants have spent more than 90% in the webinar or not.                                       |
| Check general attendance       | If                            | Checks if attendance > 60% but < 90%             | Check full attendance           | Tag participant for general attendance | This node checks whether the participants have spent more than 60% in the webinar or not.                                       |
| Check general attendance1      | If                            | Checks if attendance > 60% but < 90%, filters internal users | Check full attendance1          | Tag participant for general attendance1 | This node checks whether the participants have spent more than 60% in the webinar or not.                                       |
| Tag participant for full attendance | KlickTipp (custom)       | Tags participants with full attendance tag      | Check full attendance           | -                              | This node tags the contacts of your first webinar branch for the full attendance if they have attended more than 90% of the webinar. |
| Tag participant with full attendance1 | KlickTipp (custom)      | Tags participants with full attendance tag      | Check full attendance1          | -                              | This node tags the contacts of your first webinar branch for the full attendance if they have attended more than 90% of the webinar. |
| Tag participant for general attendance | KlickTipp             | Tags participants with general attendance tag   | Check general attendance        | -                              | This node tags the contacts of your first webinar branch for the full attendance if they have attended less than 90% but more than 60%. |
| Tag participant for general attendance1 | KlickTipp            | Tags participants with general attendance tag   | Check general attendance1       | -                              | This node tags the contacts of your first webinar branch for the full attendance if they have attended less than 90% but more than 60%. |
| Tag absentee for non attendance | KlickTipp (custom)          | Tags absentees for non-attendance                 | Route by meeting name1          | -                              | This node tags the contacts of your first webinar branch for their absence.                                                     |
| Tag absentee for non attendance1 | KlickTipp (custom)         | Tags absentees for non-attendance                 | Route by meeting name1          | -                              | This node tags the contacts of your first webinar branch for their absence.                                                     |
| Sticky Note                    | Sticky Note                   | Section label “Webinar data reception”           | -                               | -                              | ## Webinar data reception                                                                                                     |
| Sticky Note1                   | Sticky Note                   | Section label “Pulling webinar data - participants and absentees” | -                               | -                              | ## Pulling webinar data - participants and absentees                                                                           |
| Sticky Note2                   | Sticky Note                   | Section label “Segment all participants and absentees from the past webinar” | -                               | -                              | ## Segment all participants and absentees from the past webinar                                                                |
| Sticky Note3                   | Sticky Note                   | Detailed project and node disclaimer, usage instructions | -                               | -                              | Community Node Disclaimer: This workflow uses KlickTipp community nodes. Full documentation included in note content.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Listen to ending Zoom webinars`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `zoom`  
   - Response Mode: Response Node  
   - Purpose: Receive Zoom webhook events (webinar ended and validation)  

2. **Add If Node for URL Validation Check**  
   - Name: `URL Validation check`  
   - Type: If  
   - Condition:  
     - Expression: `{{$json.body.event}}` equals `endpoint.url_validation` (string equality)  
   - Outputs: Two outputs for validation vs event  

3. **Add Crypto Node for HMAC Validation**  
   - Name: `Crypto`  
   - Type: Crypto  
   - Action: HMAC SHA256  
   - Value: `{{$json.body.payload.plainToken}}`  
   - Secret: Use your Zoom webhook secret (example: `dm4D484mTSKT67HpYQ8U0w`)  
   - Connect True output of URL Validation check to this node  

4. **Add Set Node to Build Validation Response Body**  
   - Name: `Build Validation Body`  
   - Type: Set  
   - Values:  
     - `plainToken`: `{{$json.body.payload.plainToken}}`  
     - `encryptedToken`: `{{$json.data}}` (output from Crypto node)  
   - Keep only these fields  

5. **Add Respond to Webhook Node for Validation Response**  
   - Name: `Respond to Webhook (Validation)`  
   - Type: Respond to Webhook  
   - Connect from Build Validation Body  

6. **Add Set Node to Build Normal Event Response**  
   - Name: `Build Normal Event Body`  
   - Type: Set  
   - Values:  
     - `status`: `"ok"`  
   - Connect False output of URL Validation check here  

7. **Add Respond to Webhook Node for Event Acknowledgement**  
   - Name: `Respond to Webhook (Events)`  
   - Type: Respond to Webhook  
   - Connect from Build Normal Event Body  

8. **Add Wait Node**  
   - Name: `Wait 1 second`  
   - Type: Wait  
   - Duration: 1 second  
   - Connect from Respond to Webhook (Events)  

9. **Add HTTP Request Node to Get Participants**  
   - Name: `Get past Zoom webinar participants`  
   - Type: HTTP Request  
   - URL: `https://api.zoom.us/v2/past_webinars/{{ $json.body.payload.object.uuid }}/participants`  
   - Authentication: Zoom OAuth2 (configure Zoom OAuth2 credentials)  
   - Method: GET  
   - Connect from Wait node  

10. **Add HTTP Request Node to Get Absentees**  
    - Name: `Get past Zoom webinar absentees`  
    - Type: HTTP Request  
    - URL: `https://api.zoom.us/v2/past_webinars/{{ $json.body.payload.object.uuid }}/absentees`  
    - Authentication: Zoom OAuth2  
    - Method: GET  
    - Connect from Wait node  

11. **Add SplitOut Node for Participants**  
    - Name: `Split webinar participants`  
    - Type: SplitOut  
    - Field: `participants`  
    - Connect from Get past Zoom webinar participants  

12. **Add Filter Node to Remove Internal Users**  
    - Name: `Filter internal users`  
    - Type: Filter  
    - Condition: `internal_user` boolean equals true (filter out)  
    - Connect from Split webinar participants  

13. **Add Switch Node to Route Participants by Webinar Name**  
    - Name: `Route by webinar name`  
    - Type: Switch  
    - Condition: Check if webinar topic contains specific strings:  
      - "E-Mail Zustellung für Anfänger"  
      - "E-Mail Zustellung für Experten"  
    - Connect from Filter internal users  

14. **Add If Nodes for Attendance Checks**  
    - `Check full attendance` and `Check full attendance1` for ≥90% duration  
    - `Check general attendance` and `Check general attendance1` for >60% but <90% duration  
    - Use expressions comparing participant duration to `webinar duration * 60 * threshold`  
    - Connect outputs from Route by webinar name  

15. **Add KlickTipp Tagging Nodes for Participants**  
    - For full attendance: `Tag participant for full attendance` and `Tag participant with full attendance1`  
    - For general attendance: `Tag participant for general attendance` and `Tag participant for general attendance1`  
    - Use participant email for tagging  
    - Assign appropriate tag IDs for each webinar branch  
    - Configure KlickTipp API credentials  

16. **Add SplitOut Node for Absentees**  
    - Name: `Split webinar absentees`  
    - Field: `registrants`  
    - Connect from Get past Zoom webinar absentees  

17. **Add Switch Node to Route Absentees by Webinar Name**  
    - Name: `Route by meeting name1`  
    - Conditions same as participant routing but for absentees  
    - Connect from Split webinar absentees  

18. **Add KlickTipp Tagging Nodes for Absentees**  
    - `Tag absentee for non attendance` and `Tag absentee for non attendance1`  
    - Use absentee email for tagging  
    - Assign corresponding “non-attendance” tag IDs per webinar branch  

19. **Connect all nodes as per above connection flow**

20. **Add Sticky Notes**  
    - Add descriptive sticky notes to separate logical blocks and to provide instructions or disclaimers as per workflow requirements.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Community Node Disclaimer: This workflow uses KlickTipp community nodes. <br><br>Introduction and benefits of the workflow, including automated tagging and precise segmentation. <br><br>Setup instructions for Zoom and KlickTipp, including required scopes and custom fields. <br><br>Testing and deployment steps and campaign expansion ideas. <br><br>Customization tips for tag names, thresholds, and error handling.<br><br>Pro Tip: Use test emails and simulate durations to validate tagging logic.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | See Sticky Note3 content inside the workflow JSON for full detailed guidance.                                  |

---

**Disclaimer:** The provided text is derived exclusively from an automated n8n workflow. It complies fully with content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.