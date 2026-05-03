Reschedule Google Calendar Appointments with Stream Deck (15/30/60 min)

https://n8nworkflows.xyz/workflows/reschedule-google-calendar-appointments-with-stream-deck--15-30-60-min--4634


# Reschedule Google Calendar Appointments with Stream Deck (15/30/60 min)

### 1. Workflow Overview

This workflow automates the rescheduling of Google Calendar appointments by pushing all remaining events of the day forward by 15, 30, or 60 minutes. It is designed to be triggered remotely via HTTP requests from Elgato Stream Deck buttons, allowing quick adjustment of meeting times without manual calendar editing.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Webhook nodes receive HTTP requests triggered by Stream Deck buttons for each rescheduling interval (15, 30, or 60 minutes).
- **1.2 Event Retrieval:** Google Calendar nodes fetch all calendar events for the rest of the current day corresponding to the requested shift duration.
- **1.3 Event Rescheduling:** Google Calendar nodes update the start and end times of all retrieved events by the specified delay (15, 30, or 60 minutes).
- **1.4 HTTP Response:** Respond nodes send back HTTP 200 OK upon success or HTTP 500 Internal Server Error upon failure to the requester.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for incoming HTTP requests from Stream Deck buttons and triggers the respective rescheduling workflows.

- **Nodes Involved:**  
  - Hit this URL from Streamdeck http request button (30 min)  
  - Hit this URL from Streamdeck http request button 15 (15 min)  
  - Hit this URL from Streamdeck http request button 60 (60 min)

- **Node Details:**  

  1. **Hit this URL from Streamdeck http request button**  
     - Type: Webhook (Trigger node)  
     - Configuration: Default webhook settings with a unique webhook ID, listening for HTTP requests (no specific HTTP method or path constraints).  
     - Role: Entry point for the 30-minute reschedule request.  
     - Outputs: Connected to "Get the Events for the Rest of the Day" node.  
     - Failures: Potential webhook unavailability or request timeout.

  2. **Hit this URL from Streamdeck http request button 15**  
     - Type: Webhook  
     - Role: Entry point for the 15-minute reschedule request.  
     - Outputs: Connected to "Get the Events for the Rest of the Day 15" node.

  3. **Hit this URL from Streamdeck http request button 60**  
     - Type: Webhook  
     - Role: Entry point for the 60-minute reschedule request.  
     - Outputs: Connected to "Get the Events for the Rest of the Day 60" node.

#### 2.2 Event Retrieval

- **Overview:**  
  This block queries Google Calendar for all events remaining in the current day to identify which events need to be rescheduled.

- **Nodes Involved:**  
  - Get the Events for the Rest of the Day (30 min)  
  - Get the Events for the Rest of the Day 15 (15 min)  
  - Get the Events for the Rest of the Day 60 (60 min)

- **Node Details:**  

  1. **Get the Events for the Rest of the Day**  
     - Type: Google Calendar (Read events)  
     - Configuration: Retrieves all calendar events for the remainder of the current day. Uses default calendar or configured calendar credential.  
     - Output: List of events to be processed for rescheduling.  
     - Input: Triggered from "Hit this URL from Streamdeck http request button".  
     - Output Connections:  
       - On success → "Push All Meetings 30 Minutes Later"  
       - On error → "Respond with 500" (error handling)  
     - Failure modes: Authentication issues, API rate limits, network issues.

  2. **Get the Events for the Rest of the Day 15**  
     - Same as above, aligned with 15-minute rescheduling.  
     - Input: Triggered from "Hit this URL from Streamdeck http request button 15".  
     - Output:  
       - Success → "Push All Meetings 15 Minutes Later"  
       - Error → "Respond with 500"

  3. **Get the Events for the Rest of the Day 60**  
     - Same as above, aligned with 60-minute rescheduling.  
     - Input: Triggered from "Hit this URL from Streamdeck http request button 60".  
     - Output:  
       - Success → "Push All Meetings 60 Minutes Later"  
       - Error → "Respond with 500"

#### 2.3 Event Rescheduling

- **Overview:**  
  This block updates all retrieved calendar events by pushing their start and end times forward by the specified delay (15, 30, or 60 minutes).

- **Nodes Involved:**  
  - Push All Meetings 15 Minutes Later  
  - Push All Meetings 30 Minutes Later  
  - Push All Meetings 60 Minutes Later

- **Node Details:**  

  1. **Push All Meetings 15 Minutes Later**  
     - Type: Google Calendar (Update events)  
     - Configuration: Takes events from previous node, modifies their start and end times by adding 15 minutes, and updates them in Google Calendar.  
     - Inputs: Events from "Get the Events for the Rest of the Day 15" node.  
     - Outputs:  
       - On success → "Respond with 200"  
       - On error → "Respond with 500"  
     - Notes: Uses "alwaysOutputData" to ensure output is sent even on error continuation.  
     - Failure modes: Permission errors, API limits, concurrent calendar modifications.

  2. **Push All Meetings 30 Minutes Later**  
     - Same as above, with a 30-minute delay applied to events.  
     - Inputs: From "Get the Events for the Rest of the Day".  
     - Outputs: "Respond with 200" or "Respond with 500".

  3. **Push All Meetings 60 Minutes Later**  
     - Same as above, with 60-minute delay.  
     - Inputs: From "Get the Events for the Rest of the Day 60".  
     - Outputs: "Respond with 200" or "Respond with 500".

#### 2.4 HTTP Response

- **Overview:**  
  Sends back HTTP responses to indicate success or failure of the rescheduling operation.

- **Nodes Involved:**  
  - Respond with 200  
  - Respond with 500

- **Node Details:**  

  1. **Respond with 200**  
     - Type: Respond to Webhook  
     - Configuration: Returns HTTP 200 OK with no additional payload, indicating successful rescheduling.  
     - Inputs: Success paths from all "Push All Meetings ..." nodes.

  2. **Respond with 500**  
     - Type: Respond to Webhook  
     - Configuration: Returns HTTP 500 Internal Server Error, indicating failure somewhere in the workflow.  
     - Inputs: Error paths from "Get the Events ..." and "Push All Meetings ..." nodes.

---

### 3. Summary Table

| Node Name                                    | Node Type             | Functional Role                 | Input Node(s)                          | Output Node(s)                           | Sticky Note                          |
|----------------------------------------------|-----------------------|--------------------------------|--------------------------------------|-----------------------------------------|------------------------------------|
| Hit this URL from Streamdeck http request button          | Webhook               | 30-min reschedule trigger      | (HTTP Request)                       | Get the Events for the Rest of the Day  |                                    |
| Hit this URL from Streamdeck http request button 15       | Webhook               | 15-min reschedule trigger      | (HTTP Request)                       | Get the Events for the Rest of the Day 15 |                                    |
| Hit this URL from Streamdeck http request button 60       | Webhook               | 60-min reschedule trigger      | (HTTP Request)                       | Get the Events for the Rest of the Day 60 |                                    |
| Get the Events for the Rest of the Day                       | Google Calendar       | Fetch events for rest of day (30 min) | Hit this URL from Streamdeck http request button | Push All Meetings 30 Minutes Later<br>Respond with 500 (error) |                                    |
| Get the Events for the Rest of the Day 15                    | Google Calendar       | Fetch events for rest of day (15 min) | Hit this URL from Streamdeck http request button 15 | Push All Meetings 15 Minutes Later<br>Respond with 500 (error) |                                    |
| Get the Events for the Rest of the Day 60                    | Google Calendar       | Fetch events for rest of day (60 min) | Hit this URL from Streamdeck http request button 60 | Push All Meetings 60 Minutes Later<br>Respond with 500 (error) |                                    |
| Push All Meetings 15 Minutes Later                          | Google Calendar       | Reschedule events by +15 min   | Get the Events for the Rest of the Day 15 | Respond with 200<br>Respond with 500    |                                    |
| Push All Meetings 30 Minutes Later                          | Google Calendar       | Reschedule events by +30 min   | Get the Events for the Rest of the Day | Respond with 200<br>Respond with 500    |                                    |
| Push All Meetings 60 Minutes Later                          | Google Calendar       | Reschedule events by +60 min   | Get the Events for the Rest of the Day 60 | Respond with 200<br>Respond with 500    |                                    |
| Respond with 200                                            | Respond to Webhook    | Send HTTP 200 OK response      | Push All Meetings 15/30/60 Minutes Later | None                                    |                                    |
| Respond with 500                                            | Respond to Webhook    | Send HTTP 500 error response   | Get the Events ... / Push All Meetings ... (error branches) | None                                    |                                    |
| Sticky Note                                                | Sticky Note           | (Empty content)                | None                                | None                                    |                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Nodes for Each Reschedule Duration:**  
   - Add three Webhook nodes named:  
     - "Hit this URL from Streamdeck http request button" (for 30 minutes)  
     - "Hit this URL from Streamdeck http request button 15" (for 15 minutes)  
     - "Hit this URL from Streamdeck http request button 60" (for 60 minutes)  
   - Use default webhook settings; each will generate a unique URL.  
   - These nodes act as entry points triggered by HTTP requests.

2. **Create Google Calendar Nodes to Fetch Events (one per duration):**  
   - Add three Google Calendar nodes named:  
     - "Get the Events for the Rest of the Day" (30 min)  
     - "Get the Events for the Rest of the Day 15" (15 min)  
     - "Get the Events for the Rest of the Day 60" (60 min)  
   - Configure each to retrieve all events from now until the end of the day. Use the Google Calendar credentials with appropriate read permissions.  
   - Connect each webhook node’s output to its corresponding event retrieval node.

3. **Create Google Calendar Nodes to Reschedule Events (one per duration):**  
   - Add three Google Calendar nodes named:  
     - "Push All Meetings 30 Minutes Later"  
     - "Push All Meetings 15 Minutes Later"  
     - "Push All Meetings 60 Minutes Later"  
   - Configure each to update the start and end times of all incoming events by adding the respective delay (15, 30, or 60 minutes).  
   - Use Google Calendar credentials with write/update permissions.  
   - Connect each event retrieval node’s success output to its corresponding update node.  
   - Set each update node’s error handling to "Continue on Error" to prevent workflow interruption.  
   - Enable "Always Output Data" to ensure downstream nodes receive data regardless of errors.

4. **Create HTTP Response Nodes:**  
   - Add two Respond to Webhook nodes named:  
     - "Respond with 200" - configured to send HTTP 200 OK.  
     - "Respond with 500" - configured to send HTTP 500 Internal Server Error.  
   - Connect success outputs of all "Push All Meetings ..." nodes to "Respond with 200".  
   - Connect error outputs of all Google Calendar nodes (both fetch and update) to "Respond with 500".

5. **Set Up Connections and Error Handling:**  
   - Webhook → Event Retrieval → Event Rescheduling → Respond 200 (on success)  
   - Event Retrieval / Event Rescheduling (on error) → Respond 500  
   - Ensure that each path properly handles errors and sends an HTTP response to the original request.

6. **Credential Configuration:**  
   - Use Google OAuth2 credentials for Google Calendar nodes with scopes allowing reading and updating calendar events.  
   - No special credentials are required for webhook or respond nodes.

7. **Testing:**  
   - Trigger each webhook URL via HTTP client or Stream Deck HTTP request buttons.  
   - Verify that all calendar events for the rest of the day shift correctly by the specified interval.  
   - Confirm HTTP 200 responses on success, 500 on failure.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| The workflow is controlled remotely via Stream Deck HTTP request buttons, enabling quick calendar shifts. | Useful for professionals managing tight schedules.              |
| Error handling uses "continue on error" to allow partial updates without full workflow failure.            | Prevents workflow halts if a single event update fails.         |
| Google Calendar nodes require OAuth2 credentials with read/write scopes on the target calendar.            | Credential setup is essential for correct operation.            |
| No sticky note content is provided in this workflow.                                                      |                                                                 |
| n8n documentation on Google Calendar node: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleCalendar/ | For advanced configuration and troubleshooting.                  |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.