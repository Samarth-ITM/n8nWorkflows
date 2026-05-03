Handle Spanish-speaking vet clinic bookings with GPT-4o, Sheets, Calendar and Gmail

https://n8nworkflows.xyz/workflows/handle-spanish-speaking-vet-clinic-bookings-with-gpt-4o--sheets--calendar-and-gmail-13864


# Handle Spanish-speaking vet clinic bookings with GPT-4o, Sheets, Calendar and Gmail

# Reference Documentation: Spanish-speaking Vet Clinic Booking Agent (María)

## 1. Workflow Overview
This workflow implements an autonomous AI Agent named **María** for "Patitas Felices," a veterinary clinic. The system is designed to handle customer interactions in Spanish via a chat interface. It automates the end-to-end process of identifying clients, booking new appointments, rescheduling existing ones, canceling visits, and escalating non-booking inquiries to human staff via email.

The logic is divided into five functional areas:
1.  **Input Reception & Normalization:** Captures chat input and prepares metadata (session IDs, dates).
2.  **AI Orchestration:** The "brain" (GPT-4o) that processes natural language and decides which tool to trigger.
3.  **Client Database Management:** Checks for existing users or registers new ones in Google Sheets.
4.  **Appointment Management:** Real-time integration with Google Calendar for CRUD operations on events.
5.  **Escalation & Error Handling:** Notifies the team of complex requests and handles system-wide failures.

---

## 2. Block-by-Block Analysis

### 2.1 Data Ingestion
**Overview:** Receives the user message and prepares the data environment for the AI.
*   **Nodes Involved:** `When chat message received`, `Normalize`.
*   **Node Details:**
    *   **When chat message received:** Trigger node using n8n LangChain Chat. It captures the user's string and session ID.
    *   **Normalize (Set Node):** Maps the chat input to a structured variable (`message`), sets a static phone number placeholder (should be replaced by dynamic data in production), and calculates a `10_days_from_now` variable used for calendar search limits.

### 2.2 AI Orchestration
**Overview:** The central processing unit that interprets Spanish intent and maintains conversation context.
*   **Nodes Involved:** `María`, `OpenRouter Chat Model`, `OpenAI Chat Model`, `Simple Memory`.
*   **Node Details:**
    *   **María (Agent):** An AI Agent node configured with extensive System Instructions. It defines the persona, mandatory user identification steps, and specific logic for booking (e.g., "Always offer 2-3 slots").
    *   **OpenRouter Chat Model:** Primary LLM provider using `gpt-4o`.
    *   **OpenAI Chat Model:** Secondary fallback model using `gpt-5-mini` (or gpt-4o-mini depending on version availability).
    *   **Simple Memory (Window Buffer):** Keeps track of the last 10 interactions using the `session_id` as the key to ensure the AI remembers the pet's name or previous questions.

### 2.3 Client Database
**Overview:** Interfaces with Google Sheets to maintain a "Single Source of Truth" for clients.
*   **Nodes Involved:** `New client?`, `Add client`.
*   **Node Details:**
    *   **New client? (Google Sheets Tool):** Searches for a row matching the user's phone number. Returns existing details (Name, Pet Name).
    *   **Add client (Google Sheets Tool):** Appends or updates a row with the user's full name, phone, and pet type/name if they are not found in the initial check.

### 2.4 Appointment Management
**Overview:** Provides the AI with direct read/write access to the clinic's schedule.
*   **Nodes Involved:** `Get events`, `Create event`, `Update event`, `Delete event`.
*   **Node Details:**
    *   **Get events:** Searches the calendar within the next 10 days. The AI uses this to find "busy" slots and calculate "free" slots.
    *   **Create event:** Inserts a new booking. The title is formatted as "User Name - Pet Name".
    *   **Update/Delete event:** Modifies or removes existing appointments based on the unique `Event_ID` found by the "Get events" tool.

### 2.5 Escalation & Feedback
**Overview:** Handles non-standard requests and system errors.
*   **Nodes Involved:** `Send a message in Gmail`, `Workflow Error Handler`, `Notify: Workflow Error`.
*   **Node Details:**
    *   **Send a message in Gmail:** A specialized tool for the AI. If a user asks a question Maria cannot answer (e.g., medical advice), she triggers this to notify the clinic staff.
    *   **Error Handling:** An `Error Trigger` node monitors the entire workflow. If a connection fails, it sends an email to the administrator with the error details.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| When chat message received | Chat Trigger | Entry Point | None | Normalize | DATA INGESTION |
| Normalize | Set | Data Formatting | When chat... | María | DATA INGESTION |
| María | Agent | AI Brain | Normalize, AI Tools | None | AI ORCHESTRATION |
| OpenRouter Chat Model | Chat Model | Primary LLM | None | María | AI ORCHESTRATION |
| OpenAI Chat Model | Chat Model | Fallback LLM | None | María | AI ORCHESTRATION |
| Simple Memory | Memory | Context Retention | None | María | AI ORCHESTRATION |
| New client? | Google Sheets Tool | Client Lookup | María (Tool) | María | CLIENT DATABASE |
| Add client | Google Sheets Tool | Client Registration | María (Tool) | María | CLIENT DATABASE |
| Get events | Google Calendar Tool| Availability Check | María (Tool) | María | APPOINTMENT MANAGEMENT |
| Create event | Google Calendar Tool| Booking | María (Tool) | María | APPOINTMENT MANAGEMENT |
| Update event | Google Calendar Tool| Rescheduling | María (Tool) | María | APPOINTMENT MANAGEMENT |
| Delete event | Google Calendar Tool| Cancellation | María (Tool) | María | APPOINTMENT MANAGEMENT |
| Send a message in Gmail | Gmail Tool | Escalation | María (Tool) | María | ESCALATION & FEEDBACK |
| Workflow Error Handler | Error Trigger | Error Monitoring | None | Notify: Error | ERROR HANDLER |
| Notify: Workflow Error | Gmail | Admin Notification | Error Handler | None | ERROR HANDLER |

---

## 4. Reproducing the Workflow from Scratch

1.  **Preparation:**
    *   Create a Google Sheet named `Clientes_Veterinaria` with columns: `Cliente`, `Nombre de la mascota`, `Tipo de animal`, `Teléfono`.
    *   Create/identify a Google Calendar for the clinic.
2.  **Triggers and Inputs:**
    *   Add a **Chat Trigger** node.
    *   Connect it to a **Set** node ("Normalize"). Assign `message` to `{{ $json.chatInput }}` and calculate `10_days_from_now` using `$now.plus({ days: 10 })`.
3.  **AI Setup:**
    *   Add an **AI Agent** node. Set the prompt type to "Define" and paste the Maria persona instructions (included in the JSON `systemMessage`).
    *   Attach an **OpenRouter Chat Model** (Model: `openai/gpt-4o`).
    *   Attach a **Window Buffer Memory** node. Set the session key to `{{ $json.session_id }}`.
4.  **Tool Integration:**
    *   **Sheets:** Create two Google Sheets tool nodes. One for "Get row" (Search by phone) and one for "Append or Update" (Add client).
    *   **Calendar:** Create four Google Calendar tool nodes. Configure them for `GetAll` (Get events), `Create`, `Update`, and `Delete`. Ensure the `Get events` tool description explicitly tells the AI to use it to find free slots.
    *   **Gmail:** Create a Gmail tool node for notifications.
5.  **Logic Connections:**
    *   Connect all Tool nodes (Sheets, Calendar, Gmail) to the **AI Agent**'s tool input.
6.  **Error Handling:**
    *   Add an **Error Trigger** node connected to a standard **Gmail** node to alert the dev team of failures.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Prerequisites** | Requires Google Cloud Console (OAuth2) for Sheets, Calendar, and Gmail. |
| **Timezone Warning** | Ensure n8n Workflow Settings timezone matches Google Calendar's timezone. |
| **Testing** | Use the n8n Chat interface: "Hola, quiero un turno para mi gato Pepito el martes." |
| **AI Model** | Primary: GPT-4o via OpenRouter. Fallback: GPT-5/4o-mini. |