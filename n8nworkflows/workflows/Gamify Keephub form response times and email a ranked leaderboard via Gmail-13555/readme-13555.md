Gamify Keephub form response times and email a ranked leaderboard via Gmail

https://n8nworkflows.xyz/workflows/gamify-keephub-form-response-times-and-email-a-ranked-leaderboard-via-gmail-13555


# Gamify Keephub form response times and email a ranked leaderboard via Gmail

### 1. Workflow Overview

This workflow automates the gamification of Keephub form responses. It allows administrators to trigger a report that calculates the speed of form submissions, identifies top performers, and aggregates performance data by organizational unit. The workflow generates a professional, styled HTML leaderboard and emails it directly to the requester.

The logic is organized into four main functional blocks:
*   **1.1 Input Reception:** Captures the target Form ID, user email, and optional competition start parameters via an n8n Form.
*   **1.2 Data Extraction & Itemization:** Fetches all submissions for the specified form and prepares them for individual processing.
*   **1.3 Multi-Source Enrichment:** For every submission, the workflow calculates response duration, retrieves participant identity details, and resolves organizational unit IDs into human-readable names.
*   **1.4 Analytics & Reporting:** Aggregates individual data points into global statistics (average, median, rankings) and constructs a responsive HTML email with embedded bar charts.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initial Fetch
This block handles the user interface and the initial connection to the Keephub API to retrieve the raw dataset.

*   **Nodes Involved:** `🚀 Start here!`, `Find form submissions by form`.
*   **Node Details:**
    *   **🚀 Start here! (Form Trigger):** Acts as the entry point. It collects the user's email, the Keephub Form ID, and optional start date/time for time-measurement offsets.
    *   **Find form submissions by form (Keephub):** Uses the Form ID from the trigger to fetch all submissions. It uses "Login Credentials" for authentication.
    *   **Edge Cases:** If an invalid Form ID is provided, the workflow may fail at this step. If no submissions exist, subsequent code nodes are programmed to return a "No submissions found" message.

#### 2.2 Per-Submission Enrichment
This block processes each submission to add context required for the leaderboard.

*   **Nodes Involved:** `✂️ Split into Items`, `Calculate response duration for a form submission`, `Get submitter details for a form submission`, `Get Orgchart Info`.
*   **Node Details:**
    *   **✂️ Split into Items (Code):** Ensures that the array of submissions is treated as individual items so that n8n executes the following enrichment nodes for every single response.
    *   **Calculate response duration... (Keephub):** Determines the time elapsed for the response. 
    *   **Get submitter details... (Keephub):** Identifies who submitted the form (name, email, position).
    *   **Get Orgchart Info (Keephub):** Resolves the specific organizational unit (e.g., "Store North") associated with the submitter.
    *   **Configuration Note:** The enrichment nodes are set to `continueRegularOutput` on error. This prevents a single corrupted user profile or deleted submission from crashing the entire report generation.

#### 2.3 Analytics & Data Normalization
This block consolidates the enriched data and performs the mathematical calculations needed for the competition.

*   **Nodes Involved:** `🔗 Enrich with User Data`, `📊 Aggregate Stats & Build Report`.
*   **Node Details:**
    *   **🔗 Enrich with User Data (Code):** Merges data from the three enrichment nodes. It specifically handles the logic for "Custom Start Time": if a user provided a start time in the trigger, duration is recalculated from that point; otherwise, it defaults to the Keephub "Time since form created" metric.
    *   **📊 Aggregate Stats & Build Report (Code):** The "Engine" of the workflow. It calculates:
        *   Participant rankings (Best time).
        *   Global Median and Average.
        *   Organizational Unit averages.
        *   HTML/CSS generation for the email, including dynamic bar charts and medal icons for the top 3 participants.

#### 2.4 Delivery
The final stage delivers the insights to the user.

*   **Nodes Involved:** `📧 Send Competition Report`.
*   **Node Details:**
    *   **📧 Send Competition Report (Gmail):** Sends the generated HTML to the email address captured in the initial form. It dynamically sets the subject line to include the name of the winner and the total participant count.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 🚀 Start here! | Form Trigger | User Interface / Input | None | Find form submissions... | See Overview note below. |
| Find form submissions by form | Keephub | Data Extraction | 🚀 Start here! | ✂️ Split into Items | ## 📥 Fetch submissions. Collects all form submissions from Keephub for the given Form ID and splits them into individual items for per-user processing. |
| ✂️ Split into Items | Code | Itemization | Find form submissions... | Calculate response... | ## 📥 Fetch submissions. Collects all form submissions from Keephub for the given Form ID and splits them into individual items for per-user processing. |
| Calculate response duration... | Keephub | Enrichment (Time) | ✂️ Split into Items | Get submitter details... | ## ⚙️ Process & enrich. Calculates response duration per submission — using the custom start time when provided, or Keephub’s default. |
| Get submitter details... | Keephub | Enrichment (User) | Calculate response... | Get Orgchart Info | ## ⚙️ Process & enrich. Fetches submitter details and resolves org-unit names from the orgchart. |
| Get Orgchart Info | Keephub | Enrichment (Org) | Get submitter details... | 🔗 Enrich with User Data | ## ⚙️ Process & enrich. Nodes continue on error so one bad record won’t stop the report. |
| 🔗 Enrich with User Data | Code | Data Normalization | Get Orgchart Info | 📊 Aggregate Stats... | ## ⚙️ Process & enrich. Resolves org-unit names from the orgchart. |
| 📊 Aggregate Stats & Build Report | Code | Analytics / HTML Gen | 🔗 Enrich with User Data | 📧 Send Competition... | ## 📊 Build & send. Aggregates stats, builds a ranked HTML leaderboard with org-unit breakdown, and emails the competition report to the requester. |
| 📧 Send Competition Report | Gmail | Data Delivery | 📊 Aggregate Stats... | None | ## 📊 Build & send. Aggregates stats, builds a ranked HTML leaderboard with org-unit breakdown, and emails the competition report to the requester. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Preparation:** Ensure the **n8n-nodes-keephub** community node (v1.5+) is installed in your n8n instance.
2.  **Trigger Setup:** Create a **Form Trigger** node named `🚀 Start here!`.
    *   Add fields: Email (Required), Form ID (Required), Competition Start Date (Date), Competition Start Time (String).
    *   Configure `formSubmittedText` to "Processing... Please check your email shortly!".
3.  **Initial Fetch:** Connect a **Keephub** node using the `formSubmission:findByForm` operation. Use an expression to map the `contentRef` to the Form ID from the trigger.
4.  **Itemization:** Add a **Code** node named `✂️ Split into Items`. Use the following snippet to ensure individual processing: `return $input.all().map(item => ({ json: item.json }));`.
5.  **Enrichment Chain:**
    *   Add **Keephub** node: `formSubmission:calculateResponseDuration`. Set `onError` to `Continue Regular Output`.
    *   Add **Keephub** node: `formSubmission:getSubmitterDetails`. Set `onError` to `Continue Regular Output`.
    *   Add **Keephub** node: `orgchart:get` (Get Orgchart Info). Use the first ID from the user's `orgunits` array. Set `onError` to `Continue Regular Output`.
6.  **Normalization Logic:** Add a **Code** node (`🔗 Enrich with User Data`). This node must reference the outputs of all enrichment nodes using `$(node_name).all()` to reconstruct the full user profile and handle the date math for custom start times.
7.  **Report Logic:** Add a **Code** node (`📊 Aggregate Stats & Build Report`). This node should iterate through the normalized data to calculate the top 10 rankings and average times. It must output a `json.html` string containing the email template.
8.  **Email Setup:** Add a **Gmail** node. Map the `To` field to the email address from Step 2 and the `Body` to the HTML string from Step 7.
9.  **Credentials:**
    *   **Keephub:** Requires either a Bearer Token or Login Credentials (Username/Password).
    *   **Gmail:** Requires OAuth2 Google Service or Personal account connection.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **🏆 Gamify form response times** | Turn any Keephub form into a timed competition. |
| **Requirements** | n8n-nodes-keephub verified node (v1.5+) |
| **Processing Strategy** | Enrichment nodes use "Continue on Error" to ensure report delivery even with partial data. |
| **Custom Start Times** | Supports measuring duration from a fixed event time rather than just the form creation time. |