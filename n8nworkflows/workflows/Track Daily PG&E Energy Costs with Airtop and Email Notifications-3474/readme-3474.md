Track Daily PG&E Energy Costs with Airtop and Email Notifications

https://n8nworkflows.xyz/workflows/track-daily-pg-e-energy-costs-with-airtop-and-email-notifications-3474


# Track Daily PG&E Energy Costs with Airtop and Email Notifications

### 1. Workflow Overview

This workflow, titled **"PG&E Daily Cost Tracker"**, automates the process of retrieving daily energy cost data from a Pacific Gas and Electric (PG&E) account, formatting the data into an email-friendly report, and sending it to a specified email address. It is designed for users who want to monitor their daily energy consumption costs automatically without manual checking.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger & Initialization:** Starts the workflow daily at a fixed time and sets up user credentials and email variables.
- **1.2 Session and Browser Setup:** Creates an Airtop session and browser window to interact with the PG&E website.
- **1.3 Login Process:** Automates typing the PG&E username and password and handles potential modals.
- **1.4 Navigation to Energy Data:** Navigates through the PG&E website to reach the energy cost and usage details pages.
- **1.5 Data Extraction:** Extracts daily energy costs (electricity and natural gas) from the webpage content using Airtop’s extraction capabilities.
- **1.6 Email Notification:** Sends a formatted daily summary email with the extracted energy cost data.
- **1.7 Session Termination:** Ends the Airtop session to clean up resources.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Initialization

- **Overview:** This block triggers the workflow every day at 8 AM and initializes variables for PG&E credentials and the recipient email.
- **Nodes Involved:** Schedule Trigger, Variables
- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Initiates the workflow daily at 8:00 AM.
    - Configuration: Interval set to trigger at hour 8 (8 AM).
    - Inputs: None (start node)
    - Outputs: Variables node
    - Edge cases: Misconfigured time zone could cause unexpected trigger times.

  - **Variables**
    - Type: Set
    - Role: Holds user credentials (`PGE_Username`, `PGE_Password`) and recipient `Email`.
    - Configuration: Three string variables initialized as empty strings; user must fill these.
    - Inputs: Schedule Trigger
    - Outputs: Create session node
    - Edge cases: Empty or incorrect credentials will cause login failure downstream.
    - Sticky Note: Reminder to enter PG&E credentials here.

#### 2.2 Session and Browser Setup

- **Overview:** Establishes an Airtop session and opens a browser window for interacting with the PG&E website.
- **Nodes Involved:** Create session, Create browser window
- **Node Details:**

  - **Create session**
    - Type: Airtop
    - Role: Starts a new Airtop session with a named profile (`cesar-prod`) and a 5-minute timeout.
    - Configuration: Profile name set; timeout 5 minutes.
    - Inputs: Variables
    - Outputs: Create browser window
    - Edge cases: Session creation failure due to Airtop service issues or invalid profile.

  - **Create browser window**
    - Type: Airtop
    - Role: Opens a browser window at https://m.pge.com/ with live view enabled and disables resizing.
    - Configuration: URL set to PG&E mobile site; waitUntil load event.
    - Inputs: Create session
    - Outputs: Type username
    - Edge cases: Network issues or page load failures.

#### 2.3 Login Process

- **Overview:** Automates entering the username and password to log into the PG&E account, including handling any modal dialogs.
- **Nodes Involved:** Type username, Type password, Close modal (if any), Wait 5 secs
- **Node Details:**

  - **Type username**
    - Type: Airtop
    - Role: Types the username into the username text box.
    - Configuration: Text set to `PGE_Username` variable; targets USERNAME text box.
    - Inputs: Create browser window
    - Outputs: Type password
    - Edge cases: Incorrect selector or username field changes may cause failure.

  - **Type password**
    - Type: Airtop
    - Role: Types the password into the password text box and submits the form by pressing Enter.
    - Configuration: Text set to `PGE_Password` variable; presses Enter key; waits for navigation network idle.
    - Inputs: Type username
    - Outputs: Close modal (if any)
    - Edge cases: Incorrect password, login failure, or page structure changes.

  - **Close modal (if any)**
    - Type: Airtop
    - Role: Dismisses any modal dialog that might block further navigation.
    - Configuration: Clicks a button to close modals if present.
    - Inputs: Type password
    - Outputs: Wait 5 secs
    - Edge cases: Modal not present (safe), modal selector changes.

  - **Wait 5 secs**
    - Type: Wait
    - Role: Pauses execution for 5 seconds to allow page elements to load.
    - Configuration: Fixed 5-second wait.
    - Inputs: Close modal (if any)
    - Outputs: Go to "Energy Usage Details"
    - Edge cases: Fixed wait may be insufficient or excessive depending on network speed.

#### 2.4 Navigation to Energy Data

- **Overview:** Navigates through the PG&E website to reach the detailed energy usage and cost pages.
- **Nodes Involved:** Go to "Energy Usage Details", Go to "Energy Costs", Go to "Electricity and Gas"
- **Node Details:**

  - **Go to "Energy Usage Details"**
    - Type: Airtop
    - Role: Clicks the box labeled "ENERGY USAGE DETAILS" to view usage and costs over time.
    - Configuration: Waits for page load after click.
    - Inputs: Wait 5 secs
    - Outputs: Go to "Energy Costs"
    - Edge cases: UI changes may break selector.

  - **Go to "Energy Costs"**
    - Type: Airtop
    - Role: Navigates to the "Energy Costs" section.
    - Configuration: Waits for page load with network idle.
    - Inputs: Go to "Energy Usage Details"
    - Outputs: Go to "Electricity and Gas"
    - Edge cases: Page structure changes.

  - **Go to "Electricity and Gas"**
    - Type: Airtop
    - Role: Navigates to the combined view of electricity and gas costs.
    - Configuration: Waits for network idle after navigation.
    - Inputs: Go to "Energy Costs"
    - Outputs: Extract Costs
    - On error: Continues regular output (to handle accounts without combined view)
    - Edge cases: Some accounts may not have combined view; fallback handled.

#### 2.5 Data Extraction

- **Overview:** Extracts daily energy cost data from the webpage content and formats it into an HTML email body.
- **Nodes Involved:** Extract Costs
- **Node Details:**

  - **Extract Costs**
    - Type: Airtop Extraction
    - Role: Parses the webpage content to extract daily energy costs (natural gas and electricity), formats as an HTML email body.
    - Configuration: Uses a detailed prompt instructing to extract dates, total combined costs, natural gas costs, and electricity costs; if natural gas costs are missing, only electricity costs are reported.
    - Inputs: Go to "Electricity and Gas"
    - Outputs: End session, Send email
    - Edge cases: Changes in webpage layout or data format may cause extraction errors; prompt-based extraction relies on consistent page content.
    - Notes: Sticky Note mentions some PG&E accounts have a "Combined" view.

#### 2.6 Email Notification

- **Overview:** Sends the formatted daily energy cost report via email to the configured recipient.
- **Nodes Involved:** Send email
- **Node Details:**

  - **Send email**
    - Type: Gmail
    - Role: Sends an email with the extracted energy cost report.
    - Configuration: Recipient email set from `Email` variable; subject "Daily energy costs report"; sender name "Airtop Monitor"; disables attribution.
    - Inputs: Extract Costs
    - Outputs: None (end node)
    - Edge cases: Gmail authentication errors, invalid recipient email, or email sending limits.

#### 2.7 Session Termination

- **Overview:** Ends the Airtop session to free resources.
- **Nodes Involved:** End session
- **Node Details:**

  - **End session**
    - Type: Airtop
    - Role: Terminates the active Airtop session.
    - Configuration: Uses session ID from Create session node.
    - Inputs: Extract Costs
    - Outputs: None (end node)
    - Edge cases: Session termination failure is non-critical but may cause resource leakage.

---

### 3. Summary Table

| Node Name              | Node Type            | Functional Role                          | Input Node(s)           | Output Node(s)                  | Sticky Note                                                                                  |
|------------------------|----------------------|----------------------------------------|------------------------|-------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger       | Schedule Trigger     | Triggers workflow daily at 8 AM        | None                   | Variables                     |                                                                                              |
| Variables             | Set                  | Holds PG&E credentials and email       | Schedule Trigger       | Create session                | ## Heads up! To get this workflow running correctly, please enter your PG&E credentials below |
| Create session        | Airtop               | Creates Airtop session                  | Variables              | Create browser window         |                                                                                              |
| Create browser window  | Airtop               | Opens browser window at PG&E site      | Create session         | Type username                 |                                                                                              |
| Type username         | Airtop               | Types PG&E username                     | Create browser window  | Type password                 |                                                                                              |
| Type password         | Airtop               | Types PG&E password and submits login  | Type username          | Close modal (if any)          |                                                                                              |
| Close modal (if any)  | Airtop               | Dismisses any modal dialogs             | Type password          | Wait 5 secs                  |                                                                                              |
| Wait 5 secs           | Wait                 | Waits 5 seconds for page load           | Close modal (if any)   | Go to "Energy Usage Details"  |                                                                                              |
| Go to "Energy Usage Details" | Airtop         | Navigates to energy usage details page | Wait 5 secs            | Go to "Energy Costs"          |                                                                                              |
| Go to "Energy Costs"  | Airtop               | Navigates to energy costs section       | Go to "Energy Usage Details" | Go to "Electricity and Gas" |                                                                                              |
| Go to "Electricity and Gas" | Airtop          | Navigates to combined electricity/gas view | Go to "Energy Costs"  | Extract Costs                | Some PG&E accounts have a "Combined" view for gas and electricity                             |
| Extract Costs         | Airtop Extraction    | Extracts and formats daily energy costs | Go to "Electricity and Gas" | End session, Send email     |                                                                                              |
| End session           | Airtop               | Terminates Airtop session                | Extract Costs           | None                         |                                                                                              |
| Send email            | Gmail                | Sends daily energy cost report email    | Extract Costs           | None                         |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Set to trigger daily at 8:00 AM.
   - No credentials needed.

2. **Add a Set node named "Variables"**
   - Create three string variables: `PGE_Username`, `PGE_Password`, and `Email`.
   - Leave values empty for user input.
   - Connect Schedule Trigger → Variables.

3. **Add an Airtop node "Create session"**
   - Operation: Create session.
   - Profile name: `cesar-prod`.
   - Timeout: 5 minutes.
   - Connect Variables → Create session.

4. **Add an Airtop node "Create browser window"**
   - Operation: Open window.
   - URL: `https://m.pge.com/`.
   - Enable live view.
   - Disable window resizing.
   - Wait until page load.
   - Connect Create session → Create browser window.

5. **Add an Airtop node "Type username"**
   - Operation: Type text.
   - Text: Use expression to get `PGE_Username` from Variables node.
   - Target element: USERNAME text box.
   - Connect Create browser window → Type username.

6. **Add an Airtop node "Type password"**
   - Operation: Type text.
   - Text: Use expression to get `PGE_Password` from Variables node.
   - Press Enter key after typing.
   - Wait for navigation network idle.
   - Connect Type username → Type password.

7. **Add an Airtop node "Close modal (if any)"**
   - Operation: Interaction click.
   - Target: Button to dismiss modal if present.
   - Connect Type password → Close modal (if any).

8. **Add a Wait node "Wait 5 secs"**
   - Fixed wait time: 5 seconds.
   - Connect Close modal (if any) → Wait 5 secs.

9. **Add an Airtop node "Go to 'Energy Usage Details'"**
   - Operation: Click interaction.
   - Target: Box labeled "ENERGY USAGE DETAILS".
   - Wait for page load.
   - Connect Wait 5 secs → Go to "Energy Usage Details".

10. **Add an Airtop node "Go to 'Energy Costs'"**
    - Operation: Click interaction.
    - Target: "ENERGY COSTS" section.
    - Wait for page load with network idle.
    - Connect Go to "Energy Usage Details" → Go to "Energy Costs".

11. **Add an Airtop node "Go to 'Electricity and Gas'"**
    - Operation: Click interaction.
    - Target: "Electricity and Gas" combined view.
    - Wait for network idle.
    - On error: Continue regular output.
    - Connect Go to "Energy Costs" → Go to "Electricity and Gas".

12. **Add an Airtop Extraction node "Extract Costs"**
    - Operation: Query extraction.
    - Prompt: Provide detailed instructions to extract daily energy costs including date, total combined, natural gas, and electricity costs.
    - Format output as an HTML email body without subject or greeting.
    - Connect Go to "Electricity and Gas" → Extract Costs.

13. **Add an Airtop node "End session"**
    - Operation: Terminate session.
    - Use session ID from Create session node.
    - Connect Extract Costs → End session.

14. **Add a Gmail node "Send email"**
    - Send To: Use expression to get `Email` from Variables node.
    - Subject: "Daily energy costs report".
    - Message: Use extracted data from Extract Costs node.
    - Sender name: "Airtop Monitor".
    - Disable attribution.
    - Connect Extract Costs → Send email.

15. **Add a Sticky Note**
    - Content: "## Heads up!\nTo get this workflow running correctly, please enter your PG&E credentials below"
    - Position near Variables node for user visibility.

16. **Configure Credentials**
    - Airtop: Ensure valid Airtop API credentials and profile (`cesar-prod`) are set.
    - Gmail: Authenticate Gmail node with OAuth2 credentials for sending emails.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Free Airtop API Key required: https://portal.airtop.ai/?utm_campaign=n8n                         | Required to use Airtop nodes for browser automation                                             |
| Enter PG&E credentials carefully in the Variables node to avoid login failures                   | Sticky Note reminder in workflow                                                                |
| This workflow can be adapted for other energy providers with minor changes                        | Useful for users outside PG&E                                                                    |
| Consider adding data storage or visualization for long-term tracking                             | Suggested customization options in description                                                  |
| Gmail node requires OAuth2 credentials with send email permission                               | Ensure Gmail API is enabled and credentials are set up properly                                 |
| Airtop session timeout is set to 5 minutes; adjust if needed for slower connections             | Prevents session expiration during workflow execution                                           |
| For troubleshooting, enable Airtop live view to watch browser interactions in real time         | Helpful for debugging login and navigation steps                                                |

---

This document provides a complete, detailed reference for understanding, reproducing, and modifying the "PG&E Daily Cost Tracker" workflow in n8n. It covers each node’s purpose, configuration, and potential failure points, enabling users and automation agents to maintain and extend the workflow confidently.