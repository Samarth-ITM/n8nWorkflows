Automatically Add Travel Time Blockers to Calendar using Google Directions API

https://n8nworkflows.xyz/workflows/automatically-add-travel-time-blockers-to-calendar-using-google-directions-api-6700


# Automatically Add Travel Time Blockers to Calendar using Google Directions API

### 1. Workflow Overview

This workflow automates the addition of travel time blockers to a Google Calendar to ensure timely arrival at scheduled events. It is designed for users who want to proactively block travel time before appointments, reducing the risk of being late.

The workflow runs daily at 7 AM and performs the following logical blocks:

- **1.1 Initialization and Defaults Setup:** Defines user preferences such as home address, blocker event name, and preferred mode of transportation.
- **1.2 Trigger and AI Orchestration:** Starts the workflow on schedule and uses an AI agent to orchestrate event retrieval, filtering, travel time calculation, and event creation.
- **1.3 Calendar Events Retrieval:** Fetches all calendar events scheduled for the current day.
- **1.4 Travel Time Calculation Sub-Workflow:** Determines travel time between two locations using the Google Directions API.
- **1.5 Travel Time Blocker Creation:** Creates calendar events representing travel time blockers before actual appointments.
- **1.6 Memory and Language Model Integration:** Supports conversational AI operations and state memory for the AI agent.
- **1.7 Credential and Configuration Management:** Manages authentication and API credentials for Google Calendar, Google Directions API, and OpenAI.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Defaults Setup

- **Overview:** Sets default user parameters required throughout the workflow (home address, travel mode, blocker event name).
- **Nodes Involved:** `Set Defaults`
- **Node Details:**
  - Type: Set Node
  - Configuration: Assigns three string variables:
    - `home`: User's default starting address (e.g., "Homestreet 25, 12345 Berlin")
    - `mode`: Mode of transportation (default: "transit")
    - `Blocker name`: The calendar event title for travel time blockers (default: "Travel time")
  - Input: Triggered by the schedule trigger node.
  - Output: Passes values to the AI Agent node.
  - Edge cases: Misconfigured defaults may cause incorrect travel time calculations or event creation.
  - Sticky Note: Provides instructions on setting these defaults including transport modes and blocker name.

#### 2.2 Trigger and AI Orchestration

- **Overview:** Scheduled to run daily at 7 AM and uses an AI Agent to coordinate the entire logic flow.
- **Nodes Involved:** `Schedule Trigger`, `AI Agent`
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Configuration: Runs daily at 7:00 AM.
    - Output: Initiates the workflow by triggering `Set Defaults`.
    - Edge cases: Failure to trigger at set time results in no updates for the day.

  - **AI Agent**
    - Type: LangChain AI Agent Node
    - Configuration: Contains a detailed prompt specifying the workflow’s purpose:
      - Fetch today's calendar events.
      - Filter events with locations.
      - Identify events missing preceding travel blockers.
      - Calculate travel time using the travel directions tool.
      - Create travel time blocker events accordingly.
    - Key Expressions: Uses expressions to dynamically insert parameters from `Set Defaults` and `Schedule Trigger`.
    - Inputs: Receives defaults and memory.
    - Outputs: Calls sub-tools (`get_calendar_events`, `travel_directions`, `create_calendar_event`) as needed.
    - Edge cases: AI prompt misinterpretation or API failures can disrupt logic flow.
    - Sticky Note: Describes the agent’s role clearly for maintainers.

#### 2.3 Calendar Events Retrieval

- **Overview:** Retrieves all events scheduled on the current day from the Google Calendar.
- **Nodes Involved:** `get_calendar_events`
- **Node Details:**
  - Type: Google Calendar Tool Node
  - Configuration: Retrieves all events from the configured Google Calendar email (`k.armbruster91@gmail.com`).
    - Time range is dynamically set by the AI Agent via expressions.
    - Returns all events within the day.
  - Input: Called as a tool by the AI Agent.
  - Output: Provides event data for filtering and processing.
  - Credentials: Requires Google Calendar OAuth2.
  - Edge cases: Auth failure, no events found, or API rate limits.
  - Sticky Note: Advises selection of proper Google Calendar credentials.

#### 2.4 Travel Time Calculation Sub-Workflow

- **Overview:** Calculates travel time between two points using Google Directions API, exposed as a sub-workflow tool.
- **Nodes Involved:** `Sub: travel_directions`, `Call Google Directions API`, `set Travel_time`
- **Node Details:**

  - **Sub: travel_directions**
    - Type: Execute Workflow Trigger
    - Configuration: Defines inputs `origin`, `destination`, and `mode`.
    - Input: Called as a tool by the AI Agent.
    - Output: Passes data to `Call Google Directions API`.
    - Sticky Note: Marks this node as the sub-workflow interface.

  - **Call Google Directions API**
    - Type: HTTP Request Node
    - Configuration:
      - Uses Google Directions API endpoint with dynamic query parameters:
        - `origin`, `destination`, `mode` from the sub-workflow inputs.
      - Authenticates using an HTTP query parameter with API key.
    - Input: Receives parameters from sub-workflow trigger.
    - Output: Returns JSON response with route and duration details.
    - Credentials: Requires Google Directions API key configured as HTTP Query Authentication.
    - Edge cases: API key issues, quota exceeded, invalid locations, or network errors.
    - Sticky Note: Contains instructions and link for API key setup.

  - **set Travel_time**
    - Type: Set Node
    - Configuration: Extracts travel time text (e.g., "15 mins") from the API response (`routes[0].legs[0].duration.text`) and assigns it to `Travel_time`.
    - Input: Receives API response.
    - Output: Sends only `Travel_time` back to the AI Agent for efficiency.
    - Edge cases: Missing or malformed API response data.
    - Sticky Note: Notes token-saving advantage by passing only `Travel_time`.

#### 2.5 Travel Time Blocker Creation

- **Overview:** Creates new calendar events representing travel time blockers before the corresponding appointments.
- **Nodes Involved:** `create_calendar_event`
- **Node Details:**
  - Type: Google Calendar Tool Node
  - Configuration:
    - Creates an event in the specified calendar with:
      - `summary`: The blocker name from defaults (e.g., "Travel time").
      - `start` and `end` times dynamically calculated by the AI Agent:
        - Start = Event start time minus travel time minus 10 minutes buffer.
        - End = Event start time.
  - Input: Called as a tool by the AI Agent.
  - Output: Confirmation of event creation.
  - Credentials: Google Calendar OAuth2 required.
  - Edge cases: Overlapping events, invalid time range, permission issues.
  - Sticky Note: Reminds to select Google Calendar credentials.

#### 2.6 Memory and Language Model Integration

- **Overview:** Provides conversational memory and language model support to the AI Agent.
- **Nodes Involved:** `Simple Memory`, `OpenAI Chat Model`
- **Node Details:**

  - **Simple Memory**
    - Type: Memory Buffer Window (LangChain)
    - Configuration: Uses the timestamp from `Schedule Trigger` as a session key to maintain context.
    - Input: Provides stored session data to AI Agent.
    - Output: Feeds memory to the AI Agent for informed responses.

  - **OpenAI Chat Model**
    - Type: LangChain OpenAI Chat Model Node
    - Configuration: Uses GPT-4.1-mini model.
    - Credentials: OpenAI API key required.
    - Input: Language model input for AI Agent.
    - Output: AI-generated text responses.
    - Edge cases: API limits, network errors, invalid credentials.
    - Sticky Note: Guidance to choose or change AI provider credentials.

#### 2.7 Credential and Configuration Management

- **Overview:** Manages API keys and OAuth credentials for Google Calendar, Directions API, and OpenAI.
- **Nodes Involved:** Credentials are referenced by nodes `get_calendar_events`, `create_calendar_event`, `Call Google Directions API`, and `OpenAI Chat Model`.
- **Details:**
  - Google Calendar OAuth2 credentials must be configured and selected in calendar tool nodes.
  - Google Directions API is authenticated via HTTP Query authentication using an API key.
  - OpenAI API requires an API key credential set in the AI model node.
  - Sticky Notes provide detailed instructions and external links for API key setup and credential connection.

---

### 3. Summary Table

| Node Name                 | Node Type                            | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                                       |
|---------------------------|------------------------------------|--------------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                   | Initiates workflow daily at 7 AM           | —                      | Set Defaults           | Runs daily at 7 am                                                                                                                |
| Set Defaults              | Set                               | Defines default user parameters             | Schedule Trigger       | AI Agent               | Set Default Values: home address, blocker name, and mode of transportation                                                      |
| AI Agent                 | LangChain AI Agent                  | Orchestrates main logic with AI             | Set Defaults, Simple Memory, OpenAI Chat Model, get_calendar_events, travel_directions, create_calendar_event | —                      | The Agent handles the workflow                                                                                                   |
| Simple Memory             | LangChain Memory Buffer Window     | Maintains conversational memory             | Schedule Trigger       | AI Agent               | Select the Credentials for your OpenAI API or change for another provider                                                       |
| OpenAI Chat Model         | LangChain OpenAI Chat Model        | Provides AI language model                    | —                      | AI Agent               | Select the Credentials for your OpenAI API or change for another provider                                                       |
| get_calendar_events       | Google Calendar Tool               | Fetches all events for today                 | AI Agent               | AI Agent               | Select the Credentials for your Google Calendar                                                                                  |
| travel_directions         | Execute Workflow Trigger           | Sub-workflow interface for travel time tool | AI Agent               | Call Google Directions API | This exposes the Sub-workflow as a tool for the Agent                                                                            |
| Call Google Directions API| HTTP Request                      | Calls Google Directions API to get travel time | Sub: travel_directions | set Travel_time         | Select the Credentials for your Google Directions API; instructions and link provided                                           |
| set Travel_time           | Set                               | Extracts travel time from API response       | Call Google Directions API | AI Agent               | This passes on only the Travel_time to the Agent, saving on Tokens                                                               |
| create_calendar_event     | Google Calendar Tool               | Creates travel time blocker calendar event  | AI Agent               | —                      | Select the Credentials for your Google Calendar                                                                                  |
| Sticky Note               | Sticky Note                       | Annotation/Instructional                     | —                      | —                      | Various notes on scheduling, AI agent role, credential setup, and workflow explanation (multiple sticky notes across workflow) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**
   - Type: Schedule Trigger
   - Set to trigger daily at 7:00 AM.

2. **Create a Set node named "Set Defaults":**
   - Define three string fields:
     - `home`: Your home address (e.g., "Homestreet 25, 12345 Berlin").
     - `mode`: Mode of transport (options: "transit", "driving", "walking", "bicycling"). Default: "transit".
     - `Blocker name`: Name for travel time blocker events (default: "Travel time").
   - Connect Schedule Trigger output to this node.

3. **Create an OpenAI Chat Model node:**
   - Select GPT-4.1-mini or preferred model.
   - Configure OpenAI API credentials.
   - No input connections needed.

4. **Create a Simple Memory node (LangChain Memory Buffer):**
   - Set `sessionKey` to the timestamp from Schedule Trigger (`={{ $('Schedule Trigger').item.json.timestamp }}`).
   - Set `sessionIdType` to customKey.
   - Connect Schedule Trigger to this node.

5. **Create the AI Agent node:**
   - Use LangChain Agent node type.
   - Insert a detailed prompt instructing it to:
     - Fetch today's calendar events.
     - Identify events with location.
     - Detect missing travel blockers.
     - Calculate travel times.
     - Create travel blocker events accordingly.
   - Use expressions to reference `Set Defaults` and `Schedule Trigger` data.
   - Connect inputs:
     - From `Set Defaults` main output
     - AI memory input from `Simple Memory`
     - AI language model input from OpenAI Chat Model
   - Configure tools:
     - `get_calendar_events`
     - `travel_directions` (sub-workflow)
     - `create_calendar_event`

6. **Create a Google Calendar Tool node named "get_calendar_events":**
   - Operation: getAll (get all events).
   - Calendar: Select your Google Calendar email.
   - Time range: Set dynamically via AI Agent expressions to cover the current day.
   - Credentials: Connect Google Calendar OAuth2.
   - Connect as AI tool in AI Agent node configuration.

7. **Create a Google Calendar Tool node named "create_calendar_event":**
   - Operation: Create event.
   - Calendar: Same as above.
   - Set `summary` to the blocker name from `Set Defaults`.
   - `start` and `end` times to be set dynamically by AI Agent expressions:
     - Start = Event start time minus travel time minus 10 minutes.
     - End = Event start time.
   - Credentials: Google Calendar OAuth2.
   - Connect as AI tool in AI Agent node.

8. **Create a Sub-Workflow for travel directions:**
   - Create a new workflow named "Travel Time Agent" or similar.
   - Add an Execute Workflow Trigger node with inputs:
     - `origin` (string)
     - `destination` (string)
     - `mode` (string)
   - Add an HTTP Request node:
     - URL: `https://maps.googleapis.com/maps/api/directions/json?origin={{$json.origin}}&destination={{$json.destination}}&mode={{$json.mode}}`
     - Authentication: HTTP Query Auth with Google Directions API key.
   - Add a Set node:
     - Extract travel time from API response (`routes[0].legs[0].duration.text`) and assign to `Travel_time`.
   - Connect nodes in order: Execute Workflow Trigger → HTTP Request → Set node.
   - Configure this sub-workflow as a tool in the main AI Agent node (`travel_directions`).

9. **Connect the main workflow:**
   - `Schedule Trigger` → `Set Defaults` → `AI Agent`
   - `Simple Memory` and `OpenAI Chat Model` feed into `AI Agent`.
   - AI Agent uses the calendar and sub-workflow tools internally.

10. **Set all necessary credentials:**
    - Google Calendar OAuth2 (for event retrieval and creation).
    - OpenAI API key (for AI Agent language model).
    - Google Directions API key (for HTTP Request in sub-workflow).
      - Follow https://g.co/gemini/share/b731be41d4f3 for API key creation.
      - Create HTTP Query Auth credential with key name "key" and your API key value.

11. **Test the workflow:**
    - Run manually or wait for scheduled trigger.
    - Verify travel time blockers are created appropriately on the calendar.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                           | Context or Link                                                                                     |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This bot automatically adds travel time blockers to your calendar to prevent lateness. It runs daily at 7 AM, fetches current day events, calculates travel time using Google Directions API, and creates buffered travel time events accordingly.                      | General Workflow Description                                                                       |
| Instructions for setting up Google Directions API key and creating HTTP Query Auth credentials: [https://g.co/gemini/share/b731be41d4f3](https://g.co/gemini/share/b731be41d4f3)                                                                                          | Google Directions API Setup                                                                        |
| Modes of transportation supported: "transit" (public transport), "driving" (car), "walking" (foot), "bicycling" (bike).                                                                                                                                               | Default Values Setup Sticky Note                                                                   |
| Example of a Travel time blocker event in Google Calendar: ![Travel time blocker in Calendar](https://i.ibb.co/6Rzmqkf9/Calendar.png)                                                                                                                                  | Visual representation of output                                                                    |
| OpenAI GPT-4.1-mini model is used for the AI Agent language processing; credentials must be configured accordingly.                                                                                                                                                     | AI Model Credential Setup                                                                          |
| The workflow is modular and uses a sub-workflow for Google Directions API calls to encapsulate travel time fetching logic, facilitating reuse and token efficiency.                                                                                                    | Sub-workflow design note                                                                           |
| Ensure all credentials are valid and have sufficient quota to avoid runtime failures.                                                                                                                                                                                  | General operational advice                                                                         |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.