Send a personalized WhatsApp morning brief with GPT-4o, Google and WATI

https://n8nworkflows.xyz/workflows/send-a-personalized-whatsapp-morning-brief-with-gpt-4o--google-and-wati-13731


# Send a personalized WhatsApp morning brief with GPT-4o, Google and WATI

# Reference Document: WhatsApp Daily Briefing Bot

## 1. Workflow Overview
This workflow automates the delivery of a personalized "Morning Briefing" via WhatsApp using WATI. It integrates real-time weather data, top news headlines based on user interests, and personal agenda items from Google Calendar and Google Tasks. An AI-generated greeting from OpenAI (GPT-4o) adds a touch of personalization and motivation to the message.

The workflow is designed with a dual-entry system:
- **1.1 Automated Schedule:** Runs every morning at 7:00 AM for all registered subscribers.
- **1.2 On-Demand Interaction:** Responds to user commands (brief, subscribe, stop) received via WhatsApp.

---

## 2. Block-by-Block Analysis

### 2.1 Entry Points & Intent Routing
Handles how the workflow starts, either via a timer or a user message, and determines the appropriate action.

*   **Nodes Involved:** `Schedule Trigger – 7AM Daily`, `Wati Trigger1`, `Intent Router`.
*   **Node Details:**
    *   **Schedule Trigger:** Fires daily at `0 7 * * *`.
    *   **Wati Trigger1:** Listens for `messageReceived` events from the WATI API.
    *   **Intent Router (Switch):** Filters the incoming text from WATI. 
        *   "brief" -> Triggers an instant briefing.
        *   "subscribe" -> Routes to the subscription logic.
        *   "stop" -> Routes to the unsubscription logic.
*   **Potential Failures:** Webhook URL not configured in WATI; invalid Cron expression.

### 2.2 User Configuration & Subscription Management
Manages the list of users and their specific preferences (City, Interests).

*   **Nodes Involved:** `Load User Config`, `Add Subscriber`, `WATI – Confirm Subscribe`, `Remove Subscriber`, `WATI – Confirm Unsubscribe`.
*   **Node Details:**
    *   **Load User Config (Code):** A central node that either identifies a single on-demand user or iterates through a hardcoded array of subscribers for the scheduled run. It sets variables like `city` and `interests`.
    *   **Add/Remove Subscriber (Code):** Simple logic to format confirmation messages for users opting in or out.
    *   **WATI Confirmations:** Sends immediate feedback to the user regarding their subscription status.
*   **Note:** In this template, the subscriber list is managed within the code node itself.

### 2.3 Data Retrieval (Weather & News)
Fetches live external data to populate the briefing.

*   **Nodes Involved:** `Fetch Weather`, `Fetch News`.
*   **Node Details:**
    *   **Fetch Weather (HTTP Request):** Performs a `GET` request to OpenWeatherMap API using the user's city. Returns temperature, feels-like, and weather descriptions.
    *   **Fetch News (HTTP Request):** Performs a `GET` request to NewsAPI using the `interests` defined in the user config. Returns the top 3 headlines.
*   **Variables:** `{{ $json.interests }}`, `{{ $json.city }}`.
*   **Failures:** API Key expiration; Rate limits; Invalid city name.

### 2.4 Agenda Retrieval (Google Workspace)
Connects to Google services to pull the user's schedule.

*   **Nodes Involved:** `Get Today’s Calendar Events`, `Get Due Tasks`.
*   **Node Details:**
    *   **Google Calendar:** Retrieves all events for the current day from the `primary` calendar.
    *   **Google Tasks:** Retrieves all pending tasks from the user's default task list.
*   **Requirements:** Google OAuth2 credentials with appropriate scopes.

### 2.5 AI Personalization & Message Assembly
Uses AI to create a unique greeting and merges all data into a single WhatsApp-friendly string.

*   **Nodes Involved:** `OpenAI – Generate Daily Opener`, `Assemble Briefing`, `WATI – Send Briefing`.
*   **Node Details:**
    *   **OpenAI – Generate Daily Opener (HTTP Request):** Sends a `POST` request to GPT-4o. It passes the user's name and weather as context to generate a short (~15 word) motivational greeting.
    *   **Assemble Briefing (Code):** The "Brain" of the formatting. It uses `try/catch` blocks for every data source (Weather, News, Calendar, Tasks) to ensure that if one API fails, the rest of the message still sends. It maps icons (emojis) to weather conditions and formats the final string.
    *   **WATI – Send Briefing:** Dispatches the final formatted string to the user's phone number.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Schedule Trigger | Schedule Trigger | Automated Entry | None | Load User Config | Cron: `0 7 * * *` |
| Wati Trigger1 | Wati Trigger | On-demand Entry | None | Intent Router | User sends WhatsApp message |
| Intent Router | Switch | Logic Routing | Wati Trigger1 | Load User Config, Add/Remove Subscriber | Routes: brief, subscribe, stop |
| Load User Config | Code | Data Definition | Intent Router, Schedule Trigger | Fetch Weather | Edit this node to add/remove subscribers permanently |
| Add Subscriber | Code | Sub Logic | Intent Router | WATI – Confirm Subscribe | Logs sender's phone + name |
| WATI – Confirm Subscribe | Wati | Messaging | Add Subscriber | None | Sends: "You're subscribed!" |
| Remove Subscriber | Code | Unsub Logic | Intent Router | WATI – Confirm Unsubscribe | Prepares opt-out confirmation |
| WATI – Confirm Unsubscribe | Wati | Messaging | Remove Subscriber | None | Sends: "You've been unsubscribed." |
| Fetch Weather | HTTP Request | Data Fetch | Load User Config | Fetch News | URL: api.openweathermap.org |
| Fetch News | HTTP Request | Data Fetch | Fetch Weather | Get Today's Cal Events | URL: newsapi.org |
| Get Today’s Calendar Events | Google Calendar | Data Fetch | Fetch News | Get Due Tasks | Resource: Event, Operation: getAll |
| Get Due Tasks | Google Tasks | Data Fetch | Get Today’s Calendar Events | OpenAI – Opener | Resource: Task, Operation: getAll |
| OpenAI – Generate Daily Opener | HTTP Request | AI Processing | Get Due Tasks | Assemble Briefing | gpt-4o crafts unique greeting |
| Assemble Briefing | Code | Data Merging | OpenAI – Opener | WATI – Send Briefing | Merges all data into a formatted card |
| WATI – Send Briefing | Wati | Final Delivery | Assemble Briefing | None | Delivers complete morning briefing |

---

## 4. Reproducing the Workflow from Scratch

### Prerequisites
- **WATI Account:** API Key and Endpoint URL.
- **OpenWeatherMap API Key:** For weather data.
- **NewsAPI Key:** For headlines.
- **OpenAI API Key:** For GPT-4o.
- **Google Cloud Console Project:** Enabled Calendar and Tasks APIs with OAuth2 credentials.

### Step-by-Step Setup
1.  **Triggers:**
    *   Create a **Schedule Trigger** set to `0 7 * * *`.
    *   Create a **WATI Trigger** (Event: `messageReceived`).
2.  **Routing:**
    *   Connect the WATI Trigger to a **Switch Node** (Intent Router). Set rules for string equals: "brief", "subscribe", "stop".
3.  **Config:**
    *   Create a **Code Node** (Load User Config). Define an array of objects containing `phone`, `name`, `city`, and `interests`.
4.  **Integration Chain:**
    *   **HTTP Request (Weather):** Use GET `https://api.openweathermap.org/data/2.5/weather?q={{city}}&appid={{API_KEY}}&units=metric`.
    *   **HTTP Request (News):** Use GET `https://newsapi.org/v2/top-headlines?q={{interests}}&apiKey={{API_KEY}}&pageSize=3`.
    *   **Google Calendar:** Use "Event: Get All". Set "Time Min" to start of today and "Time Max" to end of today.
    *   **Google Tasks:** Use "Task: Get All".
5.  **AI Layer:**
    *   **HTTP Request (OpenAI):** `POST` to `https://api.openai.com/v1/chat/completions`. Use a prompt: *"Write a 15-word morning greeting for {{name}} in {{city}}. The weather is {{weather}}."*
6.  **Formatting:**
    *   Create a **Code Node** (Assemble Briefing). Use Javascript to map the results of all previous nodes into a single string. Ensure you use `try/catch` or optional chaining (`?.`) so the workflow doesn't stop if one API is empty.
7.  **Final Step:**
    *   Create a **WATI Node** (Send Message). Set "Destination" to `{{phone}}` and "Message" to the output of the Assemble node.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **WATI Documentation** | [WATI Developer Portal](https://developer.wati.io/) |
| **OpenWeatherMap API** | [API Reference](https://openweathermap.org/api) |
| **NewsAPI** | [NewsAPI.org](https://newsapi.org/) |
| **On-Demand Refresh** | Users can reply "brief" at any time to get an updated version of their daily summary. |