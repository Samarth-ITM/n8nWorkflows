Meetup Registration System with PostgreSQL Database & Interactive Giveaway Tool

https://n8nworkflows.xyz/workflows/meetup-registration-system-with-postgresql-database---interactive-giveaway-tool-5758


# Meetup Registration System with PostgreSQL Database & Interactive Giveaway Tool

---

### 1. Workflow Overview

This n8n workflow, titled **"Meetup Participant Registration & Giveaway App"**, is designed to manage event participant registration and host a live interactive giveaway using randomly selected winners from the registered attendees. It integrates a PostgreSQL database for persistent participant data storage and exposes web frontend interfaces for both registration and giveaway interaction.

The workflow divides logically into two main functional blocks:

- **1.1 Participant Registration Flow:**  
  Handles participant form submissions, maps form data into the database schema, performs database upsert operations to store/update participants, and presents a thank-you confirmation screen.

- **1.2 Giveaway Application:**  
  Provides a public HTTP webhook endpoint serving an interactive HTML app that fetches all participants from the database, formats participant data with privacy safeguards, and enables live random winner selection with visual effects.

---

### 2. Block-by-Block Analysis

#### 2.1 Participant Registration Flow

**Overview:**  
This block manages the participant registration process by receiving form submissions, formatting data for database storage, writing data to a PostgreSQL table, and displaying a thank-you message.

**Nodes Involved:**  
- Participant Form (Form Trigger)  
- Mapping form to database (Set)  
- Save Participant to Database (Postgres)  
- Thank you screen (Form)

**Node Details:**

- **Participant Form**  
  - *Type & Role:* Form Trigger node that exposes a web form for users to submit participant data.  
  - *Configuration:*  
    - Webhook path: `/giveaway`  
    - Custom CSS: Purple-themed, modern styling for usability and branding consistency.  
    - Form Title: "n8n Indonesia Community Meetup #2"  
    - Fields collected: Event (hidden, fixed to "Meetup #2"), Full Name (required), City (required), Bio (required), Whatsapp (required), Discord Username, Threads Username, Instagram Username, Email (email type)  
    - `ignoreBots` enabled to prevent spam submissions.  
  - *Expressions:* Uses default form field values and fixed event name.  
  - *Input/Output:* Trigger node; outputs form data JSON upon submission.  
  - *Potential Failures:* Form submission errors, user input validation issues, webhook connectivity.  
  - *Notes:* Custom styling ensures UX consistency.

- **Mapping form to database**  
  - *Type & Role:* Set node to map and transform incoming form fields into the database schema.  
  - *Configuration:*  
    - Maps user form fields to database column names: e.g., `nama_lengkap` from "Nama Lengkap", `discord_username` from Discord Username, etc.  
    - Constructs a unique `id` by concatenating event name and Whatsapp number (e.g., `Meetup #2--+6281234`).  
    - Captures submission timestamp in `created_at` field.  
  - *Expressions:* Uses expressions like `={{ $json['Nama Lengkap'] }}` to get form data dynamically.  
  - *Input/Output:* Input from Participant Form node; outputs mapped JSON for DB insertion.  
  - *Potential Failures:* Expression errors if expected form fields are missing; malformed data.

- **Save Participant to Database**  
  - *Type & Role:* Postgres node to upsert participant data into `n8n_meetup_participants` table.  
  - *Configuration:*  
    - Operation: Upsert based on `id` column to avoid duplicates.  
    - Columns: Maps all form fields plus event and timestamp.  
    - Schema: `public`  
    - Credentials: PostgreSQL credentials named "Neon N8N Community Indonesia".  
  - *Input/Output:* Takes mapped data from previous node; outputs DB operation results.  
  - *Potential Failures:* DB connection failures, SQL errors, constraint violations.  
  - *Version:* Uses Postgres node version 2.6.  
  - *Edge Cases:* Duplicate Whatsapp numbers may overwrite existing records due to `id` key construction.

- **Thank you screen**  
  - *Type & Role:* Form node used for displaying a completion message after successful registration.  
  - *Configuration:*  
    - Custom CSS consistent with Participant Form styling.  
    - Completion Title: "Glad You're Here!"  
    - Completion Message: "Thank you for being part of our meetup. Don’t miss the giveaway at the end of the session!"  
  - *Input/Output:* Receives input from DB save operation; outputs a static confirmation page to user.  
  - *Potential Failures:* Display issues if form node fails to render or respond.

---

#### 2.2 Giveaway Application

**Overview:**  
This block serves a frontend interactive giveaway app through a public webhook. It retrieves participant data, formats it to protect privacy, and renders a web page where random winners can be picked live with UI animations and confetti effects.

**Nodes Involved:**  
- Giveaway App (Webhook)  
- Get all participants (Postgres)  
- Format participant list (Code)  
- Respond to Giveaway App (Respond to Webhook)

**Node Details:**

- **Giveaway App**  
  - *Type & Role:* Webhook node exposing a GET endpoint `/giveaway` that triggers the giveaway app flow.  
  - *Configuration:*  
    - Path: `/giveaway`  
    - Response mode: Uses Respond to Webhook node downstream.  
  - *Input/Output:* Entry point for giveaway HTML app requests; outputs to Get all participants node.  
  - *Potential Failures:* Webhook connectivity, incorrect method calls.

- **Get all participants**  
  - *Type & Role:* Postgres node to query all participants registered for the event "Meetup #2".  
  - *Configuration:*  
    - Table: `n8n_meetup_participants`  
    - Schema: `public`  
    - Operation: Select all matching `event = 'Meetup #2'`  
    - Return all results.  
    - Credentials: "Neon N8N Community Indonesia".  
  - *Input/Output:* Input trigger from webhook; outputs full participant dataset.  
  - *Potential Failures:* DB connection errors, empty results if no participants.

- **Format participant list**  
  - *Type & Role:* Code node that processes participant data for frontend consumption.  
  - *Configuration:*  
    - Masks Whatsapp numbers by showing only first 4 and last 4 digits, hiding the middle.  
    - Creates display labels combining participant name and redacted Whatsapp number.  
    - Encodes participant IDs in Base64 for privacy and uniqueness (though not output explicitly here).  
    - Outputs array of formatted participant labels under `names`.  
  - *Input/Output:* Input from DB query; outputs formatted participant names for frontend.  
  - *Edge Cases:* Handles short Whatsapp numbers by returning them unredacted.  
  - *Potential Failures:* Runtime errors if fields missing or malformed.

- **Respond to Giveaway App**  
  - *Type & Role:* Respond to Webhook node that returns an HTML Single Page Application.  
  - *Configuration:*  
    - Responds with an HTML document embedding the participant list JSON into JavaScript.  
    - HTML includes a UI for pasting participant names, selecting how many winners to pick, rolling animations, confetti effects using external library, and winner display.  
    - Inline JavaScript handles UI and logic for random selection with animation.  
  - *Input/Output:* Receives formatted participant data; outputs HTTP response to client browser.  
  - *Potential Failures:* Large participant lists may cause performance issues; browser compatibility for JavaScript; network latency.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                        | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                             |
|-------------------------|------------------------|-------------------------------------|------------------------|--------------------------|-------------------------------------------------------------------------------------------------------|
| Participant Form        | Form Trigger           | Entry point for participant registration form | —                      | Mapping form to database  | Describes participant registration flow with detailed steps and form styling.                         |
| Mapping form to database | Set                    | Maps form fields to DB schema       | Participant Form       | Save Participant to Database | Detailed mapping of form fields to DB columns, including unique ID construction.                      |
| Save Participant to Database | Postgres              | Upserts participant data into DB    | Mapping form to database | Thank you screen          | Executes upsert to store participant data; avoids duplicates by ID.                                  |
| Thank you screen         | Form                   | Displays confirmation after submission | Save Participant to Database | —                      | Shows a thank-you message with consistent styling after successful registration.                      |
| Giveaway App             | Webhook                | Public endpoint serving giveaway app | —                      | Get all participants      | Serves as public GET endpoint providing the giveaway interactive app.                                |
| Get all participants     | Postgres               | Retrieves all participants for event | Giveaway App           | Format participant list   | Queries DB for all participants of specific event 'Meetup #2'.                                      |
| Format participant list  | Code                   | Formats participant data for frontend | Get all participants   | Respond to Giveaway App   | Masks Whatsapp numbers and prepares participant names array for frontend display.                     |
| Respond to Giveaway App  | Respond to Webhook     | Returns interactive giveaway HTML page | Format participant list | —                        | Returns SPA HTML with JS for random winner selection and confetti effects.                           |
| Sticky Note              | Sticky Note            | Documentation note                  | —                      | —                        | Explains workflow purpose and use case summary.                                                      |
| Sticky Note1             | Sticky Note            | Documentation note                  | —                      | —                        | Explains participant registration flow in steps.                                                     |
| Sticky Note2             | Sticky Note            | Documentation note                  | —                      | —                        | Explains giveaway app flow with steps and UI details.                                                |
| Sticky Note3             | Sticky Note            | Documentation note                  | —                      | —                        | Shows preview GIF of the giveaway app in action.                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Participant Registration Form Trigger node:**  
   - Type: Form Trigger  
   - Path: `giveaway`  
   - Form Title: "n8n Indonesia Community Meetup #2"  
   - Add form fields:  
     - Hidden Field: Event = "Meetup #2"  
     - Nama Lengkap (required text)  
     - Domisili (Kota) (required text)  
     - Bio (textarea, required)  
     - Whatsapp (required text)  
     - Discord Username (optional text)  
     - Threads Username (optional text)  
     - Instagram Username (optional text)  
     - Email (email type, optional)  
   - Paste custom CSS for styling (purple theme) as per original node.  
   - Enable `ignoreBots` to reduce spam.  
   - Set submit button label to "Submit".

2. **Add a Set node named "Mapping form to database":**  
   - Map form data to appropriate database fields:  
     - `nama_lengkap` = `{{ $json['Nama Lengkap'] }}`  
     - `discord_username` = `{{ $json['Discord Username (yang join di discord channel)'] }}`  
     - `domisili_kota` = `{{ $json['Domisili (Kota)'] }}`  
     - `bio` = `{{ $json['Bio (pekerjaan, specialty, dsb)'] }}`  
     - `email` = `{{ $json.Email }}`  
     - `whatsapp` = `{{ $json.Whatsapp }}`  
     - `threads_username` = `{{ $json['Threads Username'] }}`  
     - `instagram_username` = `{{ $json['Instagram Username'] }}`  
     - `created_at` = `{{ $json.submittedAt }}`  
     - `event` = `{{ $json.Event }}`  
     - `id` = Concatenate event and whatsapp: `={{ $json.Event }}--{{ $json.Whatsapp }}`

3. **Create a Postgres node "Save Participant to Database":**  
   - Credentials: Setup PostgreSQL credentials (e.g., Neon N8N Community Indonesia) with access to `n8n_meetup_participants` table.  
   - Operation: Upsert  
   - Table: `n8n_meetup_participants`  
   - Schema: `public`  
   - Matching column: `id` to avoid duplicates  
   - Map columns as per previous set node.  
   - Connect input from "Mapping form to database".

4. **Add a Form node "Thank you screen":**  
   - Set operation to "completion" mode.  
   - Completion Title: "Glad You're Here!"  
   - Completion Message: "Thank you for being part of our meetup. Don’t miss the giveaway at the end of the session!"  
   - Use the same custom CSS as the participant form for consistent UI.  
   - Connect input from "Save Participant to Database".

5. **Create a Webhook node "Giveaway App":**  
   - Method: GET  
   - Path: `giveaway` (same as registration form path; ensure differentiation in methods or separate deployment if needed)  
   - Response mode: Set to pass output to Respond to Webhook node.  

6. **Create a Postgres node "Get all participants":**  
   - Credentials: Use PostgreSQL credentials as above.  
   - Operation: Select  
   - Table: `n8n_meetup_participants`  
   - Schema: `public`  
   - Add WHERE filter: `event = 'Meetup #2'`  
   - Return all results.  
   - Connect input from "Giveaway App".

7. **Add a Code node "Format participant list":**  
   - JavaScript code:  
     - Reads all participant records.  
     - Masks Whatsapp numbers by showing first 4 and last 4 digits, with middle replaced by asterisks.  
     - Constructs display labels combining participant name and redacted Whatsapp number.  
     - Outputs JSON with key `names` containing array of these labels.  
   - Connect input from "Get all participants".

8. **Add a Respond to Webhook node "Respond to Giveaway App":**  
   - Respond with: Text (HTML)  
   - Paste the full HTML SPA code as in the original node, which includes:  
     - Embedded participant names JSON via `{{ JSON.stringify($json.names) }}`  
     - UI for name input textarea (pre-filled with participant names)  
     - Number input for how many winners to pick  
     - Button triggering rolling name selection animation and confetti effect using canvas-confetti library  
   - Connect input from "Format participant list".

9. **Connect the nodes:**  
   - Participant Form → Mapping form to database → Save Participant to Database → Thank you screen  
   - Giveaway App → Get all participants → Format participant list → Respond to Giveaway App  

10. **Set credentials:**  
    - PostgreSQL credentials with rights to read/write `n8n_meetup_participants`.  
    - No other external credentials required.

11. **Test the workflow:**  
    - Submit participant form to check database insertion and thank-you response.  
    - Access `/giveaway` GET endpoint to verify giveaway app loads with participant list.  
    - Use the giveaway UI to pick random winners and observe animations.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow is designed for community events like meetups to handle registration and live giveaways.  | Use for events needing participant data collection plus random prize draws.                         |
| Custom CSS uses the "Inter" font and purple color palette for branding consistency.                  | CSS embedded in both form and thank-you screens.                                                   |
| Giveaway app uses `canvas-confetti` library from CDN for confetti effects on winner selection.      | https://cdn.jsdelivr.net/npm/canvas-confetti@1.6.0/dist/confetti.browser.min.js                     |
| Preview GIF demonstrating the giveaway app rolling and winner selection is available.               | https://raw.githubusercontent.com/Ficky-Dev/images/refs/heads/main/Meetup%20Giveaway.gif           |
| Unique participant ID constructed from event name plus Whatsapp number to avoid duplicate entries.  | May require adjustment if Whatsapp numbers are inconsistent or privacy is a concern.               |
| The giveaway app frontend is a Single Page Application embedded in the webhook response HTML.        | Facilitates live interaction without separate hosting.                                             |

---

This completes the structured analysis and detailed reconstruction guide for the "Meetup Participant Registration & Giveaway App" n8n workflow.