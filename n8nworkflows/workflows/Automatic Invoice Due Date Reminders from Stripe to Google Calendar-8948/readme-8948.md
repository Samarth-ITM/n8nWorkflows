Automatic Invoice Due Date Reminders from Stripe to Google Calendar

https://n8nworkflows.xyz/workflows/automatic-invoice-due-date-reminders-from-stripe-to-google-calendar-8948


# Automatic Invoice Due Date Reminders from Stripe to Google Calendar

### 1. Workflow Overview

This workflow automates the creation of Google Calendar reminders for Stripe draft invoices that are due within the next 7 days. It is designed to run daily, fetching draft invoices from Stripe, filtering them based on due dates, and creating calendar events only for new invoices that do not already have reminders scheduled. The workflow is ideal for freelancers, agencies, or businesses seeking automated invoice due date tracking and timely reminders.

Logical blocks in the workflow:

- **1.1 Scheduled Trigger:** Initiates the workflow daily at 8:00 AM.
- **1.2 Stripe Invoice Fetching and Splitting:** Retrieves draft invoices from Stripe and splits the response array for individual processing.
- **1.3 Due Date Filtering:** Filters invoices to keep only those due within the next 7 days.
- **1.4 Google Calendar Event Retrieval:** Fetches existing calendar events to detect duplicates.
- **1.5 Duplicate Detection and Filtering:** Compares current invoices to existing calendar events to prevent duplicate reminders.
- **1.6 Calendar Event Creation:** Creates new calendar events for invoices that do not yet have reminders scheduled.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:** This block initiates the workflow on a fixed daily schedule at 8:00 AM.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Schedule Setup (Sticky Note)  

- **Node Details:**

  - **Schedule Trigger**
    - Type: Schedule Trigger
    - Role: Starts workflow execution daily at a specified time.
    - Configuration: Cron expression set to `0 8 * * *` (8:00 AM daily).
    - Input: None (trigger node).
    - Output: Triggers downstream nodes.
    - Edge Cases: Misconfigured cron expressions or timezone mismatches could cause incorrect trigger timing.
    - Notes: Refer to "Schedule Setup" sticky note for cron format details.

  - **Schedule Setup** (Sticky Note)
    - Type: Sticky Note
    - Role: Documents the cron schedule setup and instructions.
    - Content: Explains the cron expression format and setup tips.
    - Input/Output: Informational only.

#### 1.2 Stripe Invoice Fetching and Splitting

- **Overview:** Fetches draft invoices from Stripe and splits the returned array for individual invoice processing.
- **Nodes Involved:**  
  - Get Stripe Invoices  
  - Split Invoices  
  - Array Processing (Sticky Note)  
  - Stripe Setup Guide (Sticky Note)  

- **Node Details:**

  - **Get Stripe Invoices**
    - Type: HTTP Request
    - Role: Queries Stripe API for draft invoices.
    - Configuration:
      - URL: `https://api.stripe.com/v1/invoices`
      - Query parameters: `status=draft`, `limit=100`
      - Authentication: Stripe API credentials with Secret Key.
      - Headers: `Content-Type: application/x-www-form-urlencoded`
    - Input: Trigger from Schedule Trigger.
    - Output: JSON response containing invoices in a `data` array.
    - Edge Cases:
      - API authentication errors (invalid or missing Secret Key).
      - Rate limiting by Stripe API.
      - Empty or paginated results not handled (limited to 100 invoices).
    - Sticky Note Reference: "Stripe Setup Guide" explains credential setup and query params.

  - **Split Invoices**
    - Type: Item Lists
    - Role: Splits the `data` array of invoices into individual items for downstream processing.
    - Configuration:
      - Field to split out: `data`
    - Input: Output from Get Stripe Invoices.
    - Output: Individual invoice items.
    - Edge Cases: If `data` field is missing or empty, no items will be processed.

  - **Array Processing** (Sticky Note)
    - Type: Sticky Note
    - Role: Explains the purpose of splitting the invoice array.
    - Content: Describes processing each invoice individually.

  - **Stripe Setup Guide** (Sticky Note)
    - Type: Sticky Note
    - Role: Provides instructions for Stripe API credential setup and query parameters.
    - Content: Steps for credential creation and query explanation.

#### 1.3 Due Date Filtering

- **Overview:** Filters invoices to pass only those with a valid due date that falls within the next 7 days, including today.
- **Nodes Involved:**  
  - Filter Due Invoices (If node)  
  - Date Filter Logic (Sticky Note)  

- **Node Details:**

  - **Filter Due Invoices**
    - Type: If
    - Role: Filters invoices based on due date criteria.
    - Configuration:
      - Conditions (AND logic):
        1. Invoice has `due_date` field (exists).
        2. `due_date` ≤ current date + 7 days (in Unix timestamp).
        3. `due_date` ≥ current date (in Unix timestamp).
      - Uses JavaScript expressions to calculate timestamps.
    - Input: Individual invoices from Split Invoices.
    - Output: Only invoices passing all conditions.
    - Edge Cases:
      - Missing or malformed `due_date` field causes invoice to be filtered out.
      - Timezone assumptions: All dates are Unix timestamps in seconds.
      - Invoices due in the past or beyond 7 days are excluded.

  - **Date Filter Logic** (Sticky Note)
    - Type: Sticky Note
    - Role: Documents the filtering logic for due dates.
    - Content: Explanation of date comparisons and Unix timestamp usage.

#### 1.4 Google Calendar Event Retrieval

- **Overview:** Retrieves existing calendar events within the next 30 days to check for existing invoice reminders and avoid duplicates.
- **Nodes Involved:**  
  - Google Calendar Get Events  
  - Calendar API Setup (Sticky Note)  

- **Node Details:**

  - **Google Calendar Get Events**
    - Type: Google Calendar
    - Role: Fetches calendar events from specified calendar.
    - Configuration:
      - Calendar ID: `techdomesolutionspvtltd@gmail.com` (replace with user's calendar)
      - Time range: Start = now, End = now + 30 days (Unix timestamp converted)
      - OAuth2 credentials: Google Calendar OAuth2 API
    - Input: Output of Filter Due Invoices.
    - Output: List of calendar events.
    - Edge Cases:
      - Credential expiration or invalid OAuth tokens.
      - Calendar ID must be valid and accessible.
      - Large event sets may cause performance issues.

  - **Calendar API Setup** (Sticky Note)
    - Type: Sticky Note
    - Role: Provides configuration instructions for Google Calendar access.
    - Content: Steps for OAuth2 credential creation, calendar selection, and time range setup.

#### 1.5 Duplicate Detection and Filtering

- **Overview:** Checks if an invoice already has a corresponding calendar event by extracting invoice IDs from event descriptions and comparing them with the current invoice ID.
- **Nodes Involved:**  
  - Check Invoice Exists (Set)  
  - IF Not Exists (If)  
  - Duplicate Prevention (Sticky Note)  
  - New Invoice Check (Sticky Note)  

- **Node Details:**

  - **Check Invoice Exists**
    - Type: Set
    - Role: Extracts existing invoice IDs from calendar events and sets current invoice ID for comparison.
    - Configuration:
      - Creates two fields:
        - `existing_invoice_ids`: Array of invoice IDs found by regex matching `invoice_id:XXXXXX` in event descriptions.
        - `current_invoice_id`: ID of the current invoice from Filter Due Invoices node.
    - Input: List of calendar events from Google Calendar Get Events.
    - Output: Enriched data with invoice ID arrays.
    - Edge Cases:
      - Events without descriptions or malformed descriptions won't contribute IDs.
      - Regex relies on exact pattern `invoice_id:...`; any deviation causes misses.

  - **IF Not Exists**
    - Type: If
    - Role: Filters invoices to pass only if they are not already in the calendar events or if the current invoice ID exists.
    - Configuration:
      - Conditions (OR logic):
        1. Current invoice ID not included in `existing_invoice_ids`.
        2. `current_invoice_id` is non-empty.
    - Input: Output from Check Invoice Exists.
    - Output: Only new invoices proceed.
    - Edge Cases:
      - If `existing_invoice_ids` is empty, all invoices pass.
      - Empty or null current invoice IDs still pass if condition 2 is true.

  - **Duplicate Prevention** (Sticky Note)
    - Type: Sticky Note
    - Role: Documents the duplicate detection logic and fields created.
    - Content: Explains extraction of invoice IDs from event descriptions and comparison.

  - **New Invoice Check** (Sticky Note)
    - Type: Sticky Note
    - Role: Explains the logic to allow only new invoices to proceed.
    - Content: Conditions and expected behavior.

#### 1.6 Calendar Event Creation

- **Overview:** Creates Google Calendar events as reminders for invoices that passed all filters and do not already have reminders.
- **Nodes Involved:**  
  - Google Calendar Create Event  
  - Calendar Event Setup (Sticky Note)  

- **Node Details:**

  - **Google Calendar Create Event**
    - Type: Google Calendar
    - Role: Creates a calendar event for the invoice due date.
    - Configuration:
      - Calendar ID: `techdomesolutionspvtltd@gmail.com` (replace as needed)
      - Event summary: "Pending Invoice Due"
      - Start time: Invoice `due_date` (converted from Unix timestamp to ISO string)
      - End time: One hour after start time
      - Additional fields: Empty attendees list
      - OAuth2 credentials: Google Calendar OAuth2 API
    - Input: Output from IF Not Exists (new invoices only).
    - Output: Confirmation of event creation.
    - Edge Cases:
      - Credential errors or permission issues.
      - Invalid date formats cause failures.
      - Event title and description are minimal; customization possible.

  - **Calendar Event Setup** (Sticky Note)
    - Type: Sticky Note
    - Role: Describes event details and suggests customization options.
    - Content: Title, duration, description format, and ideas for enhancements like including customer name or invoice URL.

---

### 3. Summary Table

| Node Name                 | Node Type           | Functional Role                            | Input Node(s)          | Output Node(s)              | Sticky Note                                                                                                    |
|---------------------------|---------------------|-------------------------------------------|-----------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------|
| Workflow Description      | Sticky Note         | Documents workflow purpose and use case   | None                  | None                        | Provides overview and prerequisites for the Stripe to Google Calendar invoice reminder automation.            |
| Schedule Setup            | Sticky Note         | Documents schedule setup instructions      | None                  | None                        | Explains daily 8:00 AM cron expression and setup tips.                                                        |
| Schedule Trigger          | Schedule Trigger    | Triggers workflow daily at 8:00 AM         | None                  | Get Stripe Invoices         |                                                                                                               |
| Stripe Setup Guide        | Sticky Note         | Documents Stripe API credential setup      | None                  | None                        | Explains Stripe API key setup and query parameters for draft invoices.                                        |
| Get Stripe Invoices       | HTTP Request        | Fetches draft invoices from Stripe API     | Schedule Trigger      | Split Invoices              |                                                                                                               |
| Array Processing          | Sticky Note         | Explains splitting invoice data array      | None                  | None                        | Describes processing each invoice individually by splitting the array.                                       |
| Split Invoices            | Item Lists          | Splits Stripe invoice array into items     | Get Stripe Invoices    | Filter Due Invoices         |                                                                                                               |
| Date Filter Logic         | Sticky Note         | Documents due date filter logic             | None                  | None                        | Explains filtering invoices due within 7 days using Unix timestamps.                                         |
| Filter Due Invoices       | If                  | Filters invoices due within next 7 days    | Split Invoices         | Google Calendar Get Events  |                                                                                                               |
| Calendar API Setup        | Sticky Note         | Documents Google Calendar OAuth2 setup     | None                  | None                        | Provides setup instructions and time range configuration for fetching calendar events.                        |
| Google Calendar Get Events| Google Calendar     | Retrieves calendar events in next 30 days  | Filter Due Invoices    | Check Invoice Exists        |                                                                                                               |
| Duplicate Prevention      | Sticky Note         | Explains duplicate invoice detection logic | None                  | None                        | Describes regex matching of invoice IDs in calendar event descriptions to identify duplicates.               |
| Check Invoice Exists      | Set                 | Extracts existing invoice IDs from events  | Google Calendar Get Events | IF Not Exists           |                                                                                                               |
| New Invoice Check         | Sticky Note         | Documents logic to allow only new invoices | None                  | None                        | Explains conditions to proceed only with invoices not already scheduled in calendar.                         |
| IF Not Exists             | If                  | Filters out invoices already scheduled      | Check Invoice Exists   | Google Calendar Create Event|                                                                                                               |
| Calendar Event Setup      | Sticky Note         | Documents calendar event creation details  | None                  | None                        | Describes event title, duration, description and customization suggestions.                                  |
| Google Calendar Create Event | Google Calendar  | Creates new calendar event for invoice due | IF Not Exists          | None                        |                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Sticky Note: "Workflow Description"**
   - Content: Overview of the workflow purpose, key features, prerequisites, and use case.

2. **Create Sticky Note: "Schedule Setup"**
   - Content: Details of the daily cron schedule `0 8 * * *` and setup instructions.

3. **Add Node: Schedule Trigger**
   - Type: Schedule Trigger
   - Parameters: Set cron expression to `0 8 * * *` for daily 8:00 AM execution.
   - Connect to next node: Get Stripe Invoices.

4. **Create Sticky Note: "Stripe Setup Guide"**
   - Content: Steps to configure Stripe API credentials in n8n and query parameters `status=draft`, `limit=100`.

5. **Add Node: Get Stripe Invoices**
   - Type: HTTP Request
   - Parameters:
     - URL: `https://api.stripe.com/v1/invoices`
     - Query Parameters: `status=draft`, `limit=100`
     - Headers: `Content-Type: application/x-www-form-urlencoded`
     - Authentication: Use Stripe API credentials (Secret Key)
   - Connect input from Schedule Trigger.
   - Connect output to Split Invoices.

6. **Create Sticky Note: "Array Processing"**
   - Content: Explanation about splitting the invoices array to process each invoice individually.

7. **Add Node: Split Invoices**
   - Type: Item Lists
   - Parameters: Field to split out: `data`
   - Connect input from Get Stripe Invoices.
   - Connect output to Filter Due Invoices.

8. **Create Sticky Note: "Date Filter Logic"**
   - Content: Explains filtering invoices due within next 7 days using Unix timestamps and JavaScript expressions.

9. **Add Node: Filter Due Invoices**
   - Type: If
   - Parameters:
     - Conditions (AND):
       1. Check if `due_date` exists.
       2. `due_date` ≤ current time + 7 days (in seconds).
       3. `due_date` ≥ current time (in seconds).
     - Use JavaScript expressions for timestamps.
   - Connect input from Split Invoices.
   - Connect output to Google Calendar Get Events.

10. **Create Sticky Note: "Calendar API Setup"**
    - Content: Instructions to create Google Calendar OAuth2 credentials, select target calendar, and set time range for event retrieval.

11. **Add Node: Google Calendar Get Events**
    - Type: Google Calendar
    - Parameters:
      - Calendar: Enter your calendar ID/email (e.g., `techdomesolutionspvtltd@gmail.com`).
      - Start: Current timestamp in seconds.
      - End: Current timestamp + 30 days in seconds.
      - OAuth2 Credentials: Set up and select Google Calendar OAuth2 credentials.
    - Connect input from Filter Due Invoices.
    - Connect output to Check Invoice Exists.

12. **Create Sticky Note: "Duplicate Prevention"**
    - Content: Explains logic for extracting invoice IDs from calendar event descriptions using regex and comparing them with current invoice IDs.

13. **Add Node: Check Invoice Exists**
    - Type: Set
    - Parameters:
      - Field `existing_invoice_ids`: Extract with expression that maps over all calendar events, matches regex `invoice_id:([^\\s\\n]+)` in descriptions, and filters nulls.
      - Field `current_invoice_id`: Set to the `id` field of the current invoice from Filter Due Invoices.
    - Connect input from Google Calendar Get Events.
    - Connect output to IF Not Exists.

14. **Create Sticky Note: "New Invoice Check"**
    - Content: Explains the OR condition to pass only invoices that are new or have a valid current invoice ID.

15. **Add Node: IF Not Exists**
    - Type: If
    - Parameters:
      - Conditions (OR):
        1. Check if `existing_invoice_ids` does NOT include `current_invoice_id`.
        2. Check if `current_invoice_id` exists (non-empty).
    - Connect input from Check Invoice Exists.
    - Connect output to Google Calendar Create Event.

16. **Create Sticky Note: "Calendar Event Setup"**
    - Content: Details about event title ("Pending Invoice"), start time (invoice due date), duration (1 hour), description format (includes invoice ID), and customization suggestions.

17. **Add Node: Google Calendar Create Event**
    - Type: Google Calendar
    - Parameters:
      - Calendar: Use same calendar ID as Get Events.
      - Start: Convert invoice `due_date` from seconds to ISO string.
      - End: Start time + 1 hour.
      - Event summary: "Pending Invoice Due"
      - OAuth2 Credentials: Google Calendar OAuth2 credentials.
    - Connect input from IF Not Exists.
    - No further output nodes.

18. **Set workflow active and test with live Stripe and Google Calendar credentials.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                         | Context or Link                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| This workflow is ideal for automating invoice due date tracking and reminders for freelancers, agencies, or businesses using Stripe and Google Calendar.            | Workflow description sticky note                                          |
| Cron expression `0 8 * * *` runs the workflow every day at 8:00 AM; adjust according to your timezone and requirements.                                             | Schedule Setup sticky note                                                |
| Stripe API credentials require the Secret Key; ensure permissions allow reading invoices with status `draft`.                                                       | Stripe Setup Guide sticky note                                            |
| Google Calendar OAuth2 credentials must have access to the target calendar; replace calendar ID with your calendar's email address.                                | Calendar API Setup sticky note                                            |
| Invoice ID extraction from calendar events depends on the description containing `invoice_id:XXXXXX`. Maintain this format when customizing event descriptions.     | Duplicate Prevention sticky note                                          |
| Event creation is minimal by default; consider adding customer name, invoice amount, or invoice URL for richer reminders.                                          | Calendar Event Setup sticky note                                          |
| Test the workflow with a shorter interval and dummy data before full deployment to avoid spamming your calendar or hitting API rate limits.                        | Schedule Setup sticky note                                                |

---

**Disclaimer:** The provided content is exclusively derived from an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements; all processed data is legal and public.