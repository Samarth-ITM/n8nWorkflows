Run WhatsApp quizzes and track student progress with Wati, GPT-4.1 and Sheets

https://n8nworkflows.xyz/workflows/run-whatsapp-quizzes-and-track-student-progress-with-wati--gpt-4-1-and-sheets-13655


# Run WhatsApp quizzes and track student progress with Wati, GPT-4.1 and Sheets

# Workflow Reference: WhatsApp Learning Assistant

This document provides a technical analysis of the n8n workflow designed to automate educational quizzes via WhatsApp, leveraging **WATI** for messaging, **OpenAI (GPT-4.1)** for content generation, and **Google Sheets** as a lightweight database for session and progress tracking.

---

### 1. Workflow Overview

The workflow acts as an automated tutor. It processes incoming WhatsApp messages to identify user intent, generates dynamic quizzes on any topic, evaluates student responses, and maintains a historical record of performance.

**Logical Blocks:**
*   **1.1 Trigger & Route:** Captures incoming webhooks from WATI and redirects the logic based on keyword commands (`quiz`, `answer`, `progress`).
*   **1.2 Quiz Generation:** Extracts the requested topic, uses AI to generate structured MCQs, saves the "active" session to Google Sheets, and sends the quiz to the student.
*   **1.3 Answer Evaluation:** Parses student replies, retrieves the corresponding correct answers from the database, calculates the score, logs the result, and provides feedback.
*   **1.4 Progress Report:** Aggregates historical data from Google Sheets to generate a visual performance summary for the student.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Route
*   **Overview:** The entry point of the workflow. It listens for WhatsApp messages and determines which functional branch to execute.
*   **Nodes Involved:** `Wati Trigger1`, `Route Message`.
*   **Node Details:**
    *   **Wati Trigger1:** Listens for the `messageReceived` event. It outputs the sender's phone number (`waId`), name, and message text.
    *   **Route Message (Switch):** Uses string operations (e.g., `startsWith`) to branch logic:
        *   `quiz ` -> Quiz Generation branch.
        *   `answer ` -> Answer Evaluation branch.
        *   `progress` -> Progress Report branch.

#### 2.2 Quiz Generation
*   **Overview:** Handles the creation of new learning content.
*   **Nodes Involved:** `Extract Topic`, `AI Agent`, `OpenAI Chat Model`, `Format Quiz & Build Message`, `Google Sheets – Save Active Quiz`, `Send Quiz`.
*   **Node Details:**
    *   **Extract Topic (Code):** Uses Regex to strip the "quiz" prefix and generates a `sessionKey` based on the phone number and current date.
    *   **AI Agent (LangChain):** Instructed via a System Message to return **only** a valid JSON object containing 3 MCQ questions.
    *   **Format Quiz & Build Message (Code):** Parses the AI's JSON output. It constructs a user-friendly WhatsApp message using emojis and serializes the correct answers into a comma-separated string for storage.
    *   **Google Sheets – Save Active Quiz:** Appends the session data (Phone, Topic, Correct Answers, Date) to the "Active Quizzes" sheet.

#### 2.3 Answer Evaluation
*   **Overview:** Processes student submissions and updates their score history.
*   **Nodes Involved:** `Parse Student Answers`, `Google Sheets – Read Active Quiz`, `Evaluate & Calculate Score`, `Google Sheets – Save Score`, `Send Score`.
*   **Node Details:**
    *   **Parse Student Answers (Code):** Extracts answers using the pattern `1a 2b 3c`.
    *   **Google Sheets – Read Active Quiz:** Retrieves all rows from the "Active Quizzes" sheet to find the session matching the student's `sessionKey`.
    *   **Evaluate & Calculate Score (Code):** Compares student input against the stored "Correct Answers". It calculates a percentage and selects a motivational feedback message.
    *   **Google Sheets – Save Score:** Appends the final results (Score, Total, Percentage, Topic) to the "Scores" sheet.

#### 2.4 Progress Report
*   **Overview:** Provides the student with a summary of their learning journey.
*   **Nodes Involved:** `Google Sheets – Read Progress`, `Build Progress Report`, `Send Score1`.
*   **Node Details:**
    *   **Build Progress Report (Code):** Performs data aggregation. It calculates the average score, identifies the "Best Topic," and generates a visual progress bar using ASCII characters (`█` and `░`). It also lists the last 5 quiz results.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Wati Trigger1 | Wati Trigger | Entry point for WhatsApp messages | None | Route Message | 1️⃣ Trigger & Route |
| Route Message | Switch | Intent Classification | Wati Trigger1 | Extract Topic, Parse Student Answers, Google Sheets – Read Progress | 1️⃣ Trigger & Route |
| Extract Topic | Code | String parsing for Quiz Request | Route Message | AI Agent | 2️⃣ Quiz Generation |
| AI Agent | AI Agent | Generates MCQ content | Extract Topic | Format Quiz & Build Message | 2️⃣ Quiz Generation |
| Format Quiz & Build Message | Code | Formats WhatsApp UI & prepares DB string | AI Agent | Google Sheets – Save Active Quiz | 2️⃣ Quiz Generation |
| Google Sheets – Save Active Quiz | Google Sheets | Stores current quiz session | Format Quiz & Build Message | Send Quiz | 2️⃣ Quiz Generation |
| Send Quiz | Wati | Sends quiz text to WhatsApp | Google Sheets – Save Active Quiz | None | 2️⃣ Quiz Generation |
| Parse Student Answers | Code | Extracts student MCQ choices | Route Message | Google Sheets – Read Active Quiz | 3️⃣ Answer Evaluation |
| Google Sheets – Read Active Quiz | Google Sheets | Fetches correct answers | Parse Student Answers | Evaluate & Calculate Score | 3️⃣ Answer Evaluation |
| Evaluate & Calculate Score | Code | Grades the quiz | Google Sheets – Read Active Quiz | Google Sheets – Save Score | 3️⃣ Answer Evaluation |
| Google Sheets – Save Score | Google Sheets | Logs final results | Evaluate & Calculate Score | Send Score | 3️⃣ Answer Evaluation |
| Send Score | Wati | Sends feedback to WhatsApp | Google Sheets – Save Score | None | 3️⃣ Answer Evaluation |
| Google Sheets – Read Progress | Google Sheets | Fetches history | Route Message | Build Progress Report | 4️⃣ Progress Report |
| Build Progress Report | Code | Aggregates user stats | Google Sheets – Read Progress | Send Score1 | 4️⃣ Progress Report |
| Send Score1 | Wati | Sends report to WhatsApp | Build Progress Report | None | 4️⃣ Progress Report |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation (Google Sheets):** Create a spreadsheet with two sheets:
    *   **Active Quizzes:** Columns: `phone`, `today`, `topic`, `sessionKey`, `questionCount`, `correctAnswers`.
    *   **Scores:** Columns: `date`, `phone`, `senderName`, `topic`, `score`, `total`, `percentage`.
2.  **Trigger Setup:** Add a **Wati Trigger** node. Connect it to your WATI account and set the event to `messageReceived`.
3.  **Routing:** Add a **Switch** node. Create three rules:
    *   `String startsWith "quiz "`
    *   `String startsWith "answer "`
    *   `String equals "progress"` (using `.toLowerCase()` in the expression).
4.  **Quiz Branch (Branch 1):**
    *   Add a **Code** node to extract the topic (regex: `/^quiz\s+/i`).
    *   Add an **AI Agent** with an **OpenAI Chat Model**. Set the system prompt to enforce a specific JSON schema for the 3 MCQs.
    *   Add a **Code** node to format the WhatsApp message and serialize the answer key (e.g., `1:a,2:c`).
    *   Add a **Google Sheets (Append)** node to save the session.
    *   Add a **Wati (Send Message)** node.
5.  **Answer Branch (Branch 2):**
    *   Add a **Code** node to parse student responses (regex: `/([1-9][a-dA-D])/g`).
    *   Add a **Google Sheets (Get All)** node to read the "Active Quizzes".
    *   Add a **Code** node to find the specific row via `sessionKey`, compare answers, and generate the result text.
    *   Add a **Google Sheets (Append)** node to the "Scores" sheet.
    *   Add a **Wati (Send Message)** node for feedback.
6.  **Progress Branch (Branch 3):**
    *   Add a **Google Sheets (Get All)** node to read the "Scores" sheet.
    *   Add a **Code** node to filter rows by the student’s phone number and calculate averages/visual progress bars.
    *   Add a **Wati (Send Message)** node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Project Author** | Designed by Deepanshi Singhal. |
| **WATI API** | Requires WATI Header Authentication or OAuth2 depending on your instance. |
| **OpenAI Model** | Configured for `gpt-4.1-mini`. Ensure your API key has access to this specific model version. |
| **Session Logic** | The workflow uses `phone_YYYY-MM-DD` as a session key, limiting users to one quiz per topic/day if not cleared. |