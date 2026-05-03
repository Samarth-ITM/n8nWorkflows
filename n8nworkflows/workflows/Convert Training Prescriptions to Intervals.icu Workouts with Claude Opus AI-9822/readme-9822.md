Convert Training Prescriptions to Intervals.icu Workouts with Claude Opus AI

https://n8nworkflows.xyz/workflows/convert-training-prescriptions-to-intervals-icu-workouts-with-claude-opus-ai-9822


# Convert Training Prescriptions to Intervals.icu Workouts with Claude Opus AI

### 1. Workflow Overview

This workflow automates the conversion of user-submitted training prescriptions into structured workout data formatted specifically for import into Intervals.icu. It targets athletes and coaches who want to seamlessly translate free-text or loosely structured workout descriptions into a precise intervals.icu JSON format, ready for upload. The workflow integrates athlete-specific physiological data retrieved from Intervals.icu, uses advanced AI (Claude Opus 4.1) for workout formatting, and submits the final structured workout back to Intervals.icu.

Logical blocks:

- **1.1 Input Reception:** Receives workout input from a web form.
- **1.2 Athlete Data Retrieval and Preparation:** Fetches athlete profile and sets variables for AI.
- **1.3 AI Processing and Workout Formatting:** Uses Claude Opus AI and a structured output parser to convert the workout text to intervals.icu JSON.
- **1.4 Output Preparation and API Submission:** Prepares AI output for Intervals.icu API and sends it.
- **1.5 Completion Feedback:** Presents a simple form signaling workflow completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects user input for workout date, title, and description via a web form.
- **Nodes Involved:**  
  - `CreateWorkoutForm`  
  - `GetAthleteInfo` (starts next block)

- **Node Details:**

  - **CreateWorkoutForm**  
    - Type: Form Trigger  
    - Role: Entry point collecting structured data (workout date, title, steps) via a form webhook.  
    - Configuration: Three fields — date (required), title (required), and description (required). No attribution appended.  
    - Inputs: none (trigger node)  
    - Outputs: connects to `GetAthleteInfo`  
    - Edge cases: Form not submitted; invalid date or empty fields (form-level validation).  

  - **GetAthleteInfo**  
    - Type: HTTP Request  
    - Role: Fetch athlete profile from Intervals.icu API using HTTP Basic Auth credentials.  
    - Configuration: GET request to `https://intervals.icu/api/v1/athlete/0/` with authentication.  
    - Inputs: From `CreateWorkoutForm`  
    - Outputs: to `SetAthleteVars`  
    - Edge cases: API failures (auth error, timeout), missing athlete data, network issues.

#### 2.2 Athlete Data Retrieval and Preparation

- **Overview:** Processes raw athlete data into variables for AI prompt usage.
- **Nodes Involved:**  
  - `SetAthleteVars`  
  - `CreateIntervalsWorkoutAgent` (starts next block)

- **Node Details:**

  - **SetAthleteVars**  
    - Type: Set Node  
    - Role: Extracts and formats athlete physiological parameters into formatted strings for AI input, e.g., max HR, threshold pace formatted as mm:ss per km, FTP, sex, date of birth, weight, height, timezone.  
    - Key Expressions: Uses expressions to format pace and append units (e.g., bpm, kg, height units).  
    - Inputs: From `GetAthleteInfo`  
    - Outputs: to `CreateIntervalsWorkoutAgent`  
    - Edge cases: Missing or malformed athlete parameters, JSON path errors.

#### 2.3 AI Processing and Workout Formatting

- **Overview:** Uses Claude Opus 4.1 AI model with a detailed system prompt to convert workout text plus athlete data into a fully formatted intervals.icu JSON workout object.
- **Nodes Involved:**  
  - `CreateIntervalsWorkoutAgent`  
  - `Anthropic Chat Model` (AI model node referenced but unused/legacy)  
  - `Structured Output Parser` (mentioned but not directly connected here; see notes below)  
  - `Google Gemini Chat Model` and `Structured Output Parser` exist but are unused in main flow.

- **Node Details:**

  - **CreateIntervalsWorkoutAgent**  
    - Type: Langchain Agent Node  
    - Role: Core AI processing node; takes combined user workout input and athlete variables, applies a complex system message describing detailed formatting rules, and outputs structured JSON workout data.  
    - Configuration:  
      - Text inputs populated dynamically from form and `SetAthleteVars` outputs via template expressions.  
      - System message contains an extensive workout formatting specification including workout type detection, zones, formatting rules, examples, and output schema.  
    - Inputs: From `SetAthleteVars` (main), also connected from `Anthropic Chat Model` AI output (legacy or alternate?)  
    - Outputs: to `PrepareAgentOutputToWorkoutArray`  
    - Edge cases: AI model errors, malformed AI output, timeouts, inconsistencies in input data, invalid JSON output from AI.

  - **Anthropic Chat Model** and **Google Gemini Chat Model** and **Structured Output Parser** nodes are present but not directly wired in main workflow path. They appear to be legacy or alternative AI configurations. The main AI node is `CreateIntervalsWorkoutAgent` with internal model set to Claude Opus 4.1.

#### 2.4 Output Preparation and API Submission

- **Overview:** Transforms AI JSON output into Intervals.icu API format and uploads the workout.
- **Nodes Involved:**  
  - `PrepareAgentOutputToWorkoutArray`  
  - `CreateWorkoutAPI`  
  - `TheEndForm` (next block)

- **Node Details:**

  - **PrepareAgentOutputToWorkoutArray**  
    - Type: Code Node (JavaScript)  
    - Role: Converts AI output JSON object into an array format as required by Intervals.icu bulk event API.  
    - Key Logic: Extracts `workoutDate`, `workoutName`, `workoutType`, `workoutDescription`, and `externalId` from AI output, packages into array under key `workoutArray`.  
    - Inputs: From `CreateIntervalsWorkoutAgent`  
    - Outputs: to `CreateWorkoutAPI`  
    - Edge cases: Missing or malformed AI output, empty array.

  - **CreateWorkoutAPI**  
    - Type: HTTP Request  
    - Role: Sends POST request to Intervals.icu bulk events API to create or update the workout.  
    - Configuration:  
      - URL: `https://intervals.icu/api/v1/athlete/0/events/bulk?upsert=true`  
      - Method: POST  
      - Body: JSON from `PrepareAgentOutputToWorkoutArray`  
      - Auth: HTTP Basic Auth with Intervals.icu credentials  
    - Inputs: From `PrepareAgentOutputToWorkoutArray`  
    - Outputs: to `TheEndForm`  
    - Edge cases: API errors, authentication failure, invalid request body, network issues.

#### 2.5 Completion Feedback

- **Overview:** Displays a simple form signaling that the workout creation process finished.
- **Nodes Involved:**  
  - `TheEndForm`

- **Node Details:**

  - **TheEndForm**  
    - Type: Form Node  
    - Role: Completion form with a title "The End - Bye" to confirm workflow completion to user.  
    - Inputs: From `CreateWorkoutAPI`  
    - Outputs: None (end of workflow)  
    - Edge cases: Form display failure.

---

### 3. Summary Table

| Node Name                   | Node Type                                | Functional Role                                   | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                         |
|-----------------------------|-----------------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|---------------------------------------------------------------------------------------------------|
| CreateWorkoutForm            | Form Trigger                            | Collect workout date, title, description input  | None                             | GetAthleteInfo                  | ## Start form - workout date - workout title - workout description                                |
| GetAthleteInfo              | HTTP Request                           | Retrieve athlete info from Intervals.icu API    | CreateWorkoutForm                | SetAthleteVars                  | ## Get athlete info from intervals.icu - get through API - convert to usable format               |
| SetAthleteVars              | Set                                   | Format athlete variables for AI prompt           | GetAthleteInfo                  | CreateIntervalsWorkoutAgent     | ## Get athlete info from intervals.icu - get through API - convert to usable format               |
| CreateIntervalsWorkoutAgent | Langchain Agent                        | AI formats workout into intervals.icu JSON       | SetAthleteVars, Anthropic Chat Model (AI input) | PrepareAgentOutputToWorkoutArray | ## 🏋️ Workout Formatter for Intervals.icu - Converts training prescriptions to structured JSON - Uses Claude Opus 4.1 + structured output parser |
| PrepareAgentOutputToWorkoutArray | Code                                | Convert AI output to workout array for API       | CreateIntervalsWorkoutAgent     | CreateWorkoutAPI                | ## Get athlete info from intervals.icu - convert output array to workout array for API - call API to create workout |
| CreateWorkoutAPI            | HTTP Request                          | Upload workout JSON to Intervals.icu API         | PrepareAgentOutputToWorkoutArray | TheEndForm                     | ## Get athlete info from intervals.icu - convert output array to workout array for API - call API to create workout |
| TheEndForm                 | Form                                  | Completion confirmation form                      | CreateWorkoutAPI                | None                           | ## End form - simple form to mention workout is created                                           |
| Anthropic Chat Model         | Langchain LM Chat Anthropic           | (Legacy/alternate) AI model node                   | None                            | CreateIntervalsWorkoutAgent     | ## 🏋️ Workout Formatter for Intervals.icu - Converts training prescriptions → structured intervals.icu JSON format |
| Google Gemini Chat Model     | Langchain LM Chat Google Gemini       | (Unused) Alternative AI model node                 | None                            | Structured Output Parser        |                                                                                                   |
| Structured Output Parser     | Langchain Output Parser Structured    | (Unused) Parse AI output into JSON                 | Google Gemini Chat Model        | CreateIntervalsWorkoutAgent     |                                                                                                   |
| Sticky Note                 | Sticky Note                          | Notes for users                                    | None                            | None                           | ## Start form - workout date - workout title - workout description                                |
| Sticky Note1                | Sticky Note                          | Notes for users                                    | None                            | None                           | ## Get athlete info from intervals.icu - get through API - convert to usable format               |
| Sticky Note2                | Sticky Note                          | Notes for users                                    | None                            | None                           | ## 🏋️ Workout Formatter for Intervals.icu - Converts training prescriptions → structured intervals.icu JSON format |
| Sticky Note3                | Sticky Note                          | Notes for users                                    | None                            | None                           | ## Get athlete info from intervals.icu - convert output array to workout array for API - call API to create workout |
| Sticky Note4                | Sticky Note                          | Notes for users                                    | None                            | None                           | ## End form - simple form to mention workout is created                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create `CreateWorkoutForm` node (Form Trigger):**  
   - Set webhook ID (auto-generated or fixed).  
   - Form title: "Create workout".  
   - Add three required fields:  
     - Date (fieldType: date, label: workoutdate)  
     - Text input (label: workouttitle, placeholder: "Workout title")  
     - Textarea (label: workoutsteps)  
   - Disable append attribution.  

2. **Create `GetAthleteInfo` node (HTTP Request):**  
   - Method: GET  
   - URL: `https://intervals.icu/api/v1/athlete/0/`  
   - Authentication: HTTP Basic Auth with Intervals.icu credentials.  
   - Send headers: true.  
   - Connect input from `CreateWorkoutForm`.  

3. **Create `SetAthleteVars` node (Set):**  
   - Create variables with expressions extracting from `$json` fields of athlete data:  
     - max_hr: `"={{ $json.sportSettings[1].max_hr }} bpm"`  
     - lt_hr: `"={{ $json.sportSettings[1].lthr }} bpm"`  
     - threshold_pace: convert pace seconds to mm:ss format (custom JS expression).  
     - ftp: `"={{ $json.sportSettings[1].ftp }} watt"`  
     - sex, date_of_birth, body_weight (append "kg"), timezone, height (with units).  
   - Connect input from `GetAthleteInfo`.  

4. **Create `CreateIntervalsWorkoutAgent` node (Langchain Agent):**  
   - Configure AI model: Claude Opus 4.1 (Anthropic API credentials).  
   - Set system message with full workout formatting instructions (copy detailed formatting rules).  
   - Set text input with template expressions combining:  
     - Workout date, title, steps from form inputs.  
     - Athlete variables from `SetAthleteVars`.  
   - Connect input from `SetAthleteVars`.  

5. **Create `PrepareAgentOutputToWorkoutArray` node (Code):**  
   - Add JS code to convert AI output JSON to an array with required keys (`category`, `type`, `start_date_local`, `name`, `description`, `external_id`).  
   - Connect input from `CreateIntervalsWorkoutAgent`.  

6. **Create `CreateWorkoutAPI` node (HTTP Request):**  
   - Method: POST  
   - URL: `https://intervals.icu/api/v1/athlete/0/events/bulk?upsert=true`  
   - Authentication: HTTP Basic Auth with Intervals.icu credentials.  
   - Body type: JSON, content from `PrepareAgentOutputToWorkoutArray` output.  
   - Connect input from `PrepareAgentOutputToWorkoutArray`.  

7. **Create `TheEndForm` node (Form):**  
   - Operation: Completion  
   - Completion title: "The End - Bye"  
   - Connect input from `CreateWorkoutAPI`.  

8. **(Optional) Add Sticky Notes:**  
   - Add notes above each logical block with summaries per sections 2.1 to 2.5.  
   - Use colors and size to improve readability.  

9. **Set Credentials:**  
   - Configure Anthropic API credentials for Claude Opus model usage.  
   - Configure HTTP Basic Auth credentials for Intervals.icu API access.  

10. **Deploy and Test:**  
    - Trigger `CreateWorkoutForm` webhook, submit sample workout data.  
    - Verify athlete data fetching.  
    - Confirm AI outputs structured JSON as per formatting specs.  
    - Check workout creation via Intervals.icu API.  
    - Confirm completion form appears after successful submission.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                            |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------|
| Detailed workout formatting instructions, including zone tables and examples, are embedded in the AI system prompt for precise output.   | Embedded in `CreateIntervalsWorkoutAgent` system message. |
| Uses Claude Opus 4.1 model from Anthropic for best formatting results.                                                                      | Anthropic API credentials configured.     |
| Intervals.icu API requires HTTP Basic Authentication with valid athlete credentials.                                                       | API endpoints: `https://intervals.icu/api/v1/athlete/0/` and `/events/bulk` |
| Workout JSON output includes fields: workoutDate (ISO), workoutName, workoutType, workoutDescription (with `\n` line breaks), externalId.  | Required for proper Intervals.icu import. |
| Transition and rest formatting rules are critical to maintain workout integrity in Intervals.icu.                                           | See system message in AI prompt.           |

---

**Disclaimer:** The provided text originates exclusively from an automated n8n workflow. It complies strictly with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.