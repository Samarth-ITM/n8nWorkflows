Promote calendar events with OpenAI graphics to LinkedIn, X, and Facebook

https://n8nworkflows.xyz/workflows/promote-calendar-events-with-openai-graphics-to-linkedin--x--and-facebook-14971


# Promote calendar events with OpenAI graphics to LinkedIn, X, and Facebook

# Technical Analysis: Event Promotion Automation Workflow

This document provides a comprehensive technical breakdown of the n8n workflow designed to automatically promote calendar events across LinkedIn, X (Twitter), and Facebook using AI-generated captions and programmatically created graphics.

---

### 1. Workflow Overview

The primary purpose of this workflow is to convert a Google Calendar event into a multi-stage social media marketing campaign. Instead of a single post, it calculates three distinct promotion windows (48h, 24h, and 1h before the event) and schedules posts for each.

**Logical Blocks:**
- **1.1 Input Reception & Validation:** Triggers on new calendar events and filters out those without sufficient lead time.
- **1.2 Graphic Generation & Hosting:** Fetches a brand template, overlays event-specific text, and hosts the resulting image on a public CDN.
- **1.3 Scheduling Logic:** Calculates three specific timestamps relative to the event start date.
- **1.4 Execution Loop & AI Content:** Iterates through the schedule, waits for the target time, and generates platform-specific captions using OpenAI.
- **1.5 Multi-Channel Dispatch & Logging:** Posts the content to three social networks and logs the activity in Google Sheets.
- **1.6 Admin Notification:** Sends a final confirmation via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation
**Overview:** Captures new event data and ensures the event is far enough in the future to justify a promotional campaign.
- **Nodes Involved:** `Google Calendar — New Event Trigger`, `Code — Parse & Enrich Event Data`, `IF — Event Has Enough Lead Time (2h+)`.
- **Node Details:**
    - **Google Calendar Trigger:** Watches for `eventCreated`. 
    - **Code (Parse & Enrich):** JavaScript logic that cleans HTML from descriptions, calculates `hoursUntil` the event, and formats readable date/time strings.
    - **IF Node:** Checks if `hoursUntil >= 2`. If false, the workflow terminates to prevent posting for events that are starting immediately.
    - **Potential Failures:** Calendar API authentication expiry; missing event summary/title.

#### 2.2 Graphic Generation & Hosting
**Overview:** Creates a visual asset by combining a static template with dynamic event text and uploading it for social media access.
- **Nodes Involved:** `HTTP — Fetch Base Banner Template`, `Edit Image — Compose Event Promo Graphic`, `Upload a File (UploadToURL)`.
- **Node Details:**
    - **HTTP Request:** Downloads a binary `.jpg` banner from a fixed URL.
    - **Edit Image:** Performs a "composite" operation to overlay the event title, date, and location onto the banner.
    - **UploadToURL:** A specialized node that pushes the binary image to a CDN to provide a public URL (required by social media APIs).
    - **Potential Failures:** HTTP timeout when fetching template; UploadToURL API limit or credential error.

#### 2.3 Scheduling Logic
**Overview:** Converts the event start time into three separate "post slots."
- **Nodes Involved:** `DateTime — Parse Event Start Time`, `Code — Build 3 Schedule Slots`.
- **Node Details:**
    - **DateTime Node:** Standardizes the event start time.
    - **Code (Build Slots):** Generates an array of 3 items. Each item contains the `scheduledAt` ISO timestamp (Event Start minus 48h, 24h, and 1h) and an `urgency` tag (early, reminder, last_call).
    - **Input/Output:** Takes one event $\rightarrow$ Outputs three distinct schedule objects.

#### 2.4 Execution Loop & AI Content
**Overview:** Manages the timing and content creation for each of the three scheduled slots.
- **Nodes Involved:** `Loop Over Items — Each Schedule Slot`, `Wait — Until Scheduled Post Time`, `OpenAI — Generate Time-Aware Captions`, `Code — Parse Captions with Fallback`.
- **Node Details:**
    - **Loop (Split in Batches):** Processes the 3 slots sequentially.
    - **Wait Node:** Pauses the execution until the `scheduledAt` timestamp is reached.
    - **OpenAI Node:** Uses a prompt to create three different captions (LinkedIn, X, FB) based on the `urgency` level (e.g., "last call" for the 1h slot).
    - **Code (Parse/Fallback):** Parses the AI JSON response. If the AI fails or returns invalid JSON, it applies a hard-coded template fallback to ensure the post still goes live.

#### 2.5 Multi-Channel Dispatch & Logging
**Overview:** Executes the actual posting and maintains a record of all actions.
- **Nodes Involved:** `LinkedIn — Publish Event Post`, `Twitter/X — Publish Event Tweet`, `Facebook — Publish Event Photo Post`, `Google Sheets — Append Post Log Row`.
- **Node Details:**
    - **LinkedIn/Twitter Nodes:** Standard API integrations using generated captions.
    - **Facebook (HTTP Request):** Calls the Graph API `/photos` endpoint, sending the `hostedImageUrl` and caption.
    - **Google Sheets:** Updates a sheet named `EventPostLog` with metadata (EventID, Platform, Status, ImageURL).
    - **Potential Failures:** API rate limits on X (Twitter); Facebook Page token expiration.

#### 2.6 Admin Notification
**Overview:** Final confirmation that the loop has completed for a specific slot.
- **Nodes Involved:** `Merge — Collect All Platform Results`, `Telegram — Notify Admin of Post Dispatch`.
- **Node Details:**
    - **Merge:** Acts as a synchronization point to ensure all posts are attempted before notifying the admin.
    - **Telegram:** Sends a Markdown-formatted summary including the image URL and the event name.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Google Calendar — New Event Trigger | Google Calendar | Event Intake | - | Code — Parse & Enrich... | Step 1 — Google Calendar Trigger & Event Data Extraction |
| Code — Parse & Enrich Event Data | Code | Data Cleaning | Google Calendar... | IF — Event Has Enough... | Step 1 — Google Calendar Trigger & Event Data Extraction |
| IF — Event Has Enough Lead Time (2h+) | IF | Validation | Code — Parse... | HTTP — Fetch Base... | Step 1 — Google Calendar Trigger & Event Data Extraction |
| HTTP — Fetch Base Banner Template | HTTP Request | Asset Retrieval | IF — Event Has... | Edit Image — Compose... | Step 2 — Graphic Generation & UploadToURL |
| Edit Image — Compose Event Promo Graphic | Edit Image | Image Synthesis | HTTP — Fetch Base... | Upload a File | Step 2 — Graphic Generation & UploadToURL |
| Upload a File | UploadToURL | CDN Hosting | Edit Image... | DateTime — Parse... | Step 2 — Graphic Generation & UploadToURL |
| DateTime — Parse Event Start Time | DateTime | Time Parsing | Upload a File | Code — Build 3... | Step 2 — Graphic Generation & UploadToURL |
| Code — Build 3 Schedule Slots | Code | Schedule Logic | DateTime — Parse... | Loop Over Items... | Step 2 — Graphic Generation & UploadToURL |
| Loop Over Items — Each Schedule Slot | Split In Batches | Sequence Mgmt | Code — Build 3... | Wait — Until Scheduled... | Step 3 — Loop, Wait & Scheduled Multi-Platform Posting |
| Wait — Until Scheduled Post Time | Wait | Temporal Pause | Loop Over Items... | OpenAI — Generate... | Step 3 — Loop, Wait & Scheduled Multi-Platform Posting |
| OpenAI — Generate Time-Aware Captions | OpenAI | Content Creation | Wait — Until... | Code — Parse Captions... | Step 3 — Loop, Wait & Scheduled Multi-Platform Posting |
| Code — Parse Captions with Fallback | Code | JSON Sanitization | OpenAI — Generate... | Google Sheets, LinkedIn, Twitter, Facebook | Step 3 — Loop, Wait & Scheduled Multi-Platform Posting |
| Google Sheets — Append Post Log Row | Google Sheets | Audit Trail | Code — Parse... | Merge — Collect... | Step 4 — Google Sheets Log & Telegram Notification |
| LinkedIn — Publish Event Post | LinkedIn | Social Posting | Code — Parse... | Merge — Collect... | Step 4 — Google Sheets Log & Telegram Notification |
| Twitter/X — Publish Event Tweet | Twitter | Social Posting | Code — Parse... | Merge — Collect... | Step 4 — Google Sheets Log & Telegram Notification |
| Facebook — Publish Event Photo Post | HTTP Request | Social Posting | Code — Parse... | Merge — Collect... | Step 4 — Google Sheets Log & Telegram Notification |
| Merge — Collect All Platform Results | Merge | Sync Point | All Posting Nodes | Telegram — Notify... | Step 4 — Google Sheets Log & Telegram Notification |
| Telegram — Notify Admin of Post Dispatch | Telegram | Admin Alert | Merge — Collect... | - | Step 4 — Google Sheets Log & Telegram Notification |

---

### 4. Reproducing the Workflow from Scratch

#### Step 1: Trigger and Data Prep
1. Create a **Google Calendar Trigger** node. Set it to "Event Created" and select your target calendar.
2. Add a **Code** node to extract `summary`, `description`, `location`, and calculate the time difference between now and the event start.
3. Add an **IF** node to check if the event is at least 2 hours in the future.

#### Step 2: Visual Asset Pipeline
4. Use an **HTTP Request** node to GET your banner image (Response format: File).
5. Use the **Edit Image** node. Set operation to "Composite" and use expressions to place the event title and date over the banner.
6. Use the **UploadToURL** node to upload the resulting binary and capture the returned public URL.
7. Use a **DateTime** node to ensure the start time is correctly parsed.
8. Add a **Code** node to create an array of 3 items. For each item, subtract 48h, 24h, and 1h respectively from the event start time.

#### Step 3: Scheduling and AI Generation
9. Add a **Split In Batches (Loop)** node to process the 3 schedule slots.
10. Add a **Wait** node. Configure it to resume at the `scheduledAt` timestamp from the previous node.
11. Add an **OpenAI** node. Prompt it to generate three social media captions based on the `urgency` level of the current slot.
12. Add a **Code** node to parse the OpenAI JSON response, providing string fallbacks in case the AI output is malformed.

#### Step 4: Distribution and Logging
13. Create four parallel paths from the Parse Code node:
    - **LinkedIn Node:** Map the `linkedinCaption` and the hosted image URL.
    - **Twitter Node:** Map the `twitterCaption`.
    - **HTTP Request (Facebook):** POST to `graph.facebook.com` with `url`, `message`, and `access_token`.
    - **Google Sheets Node:** Append a row to `EventPostLog` with all event and posting metadata.
14. Connect all four nodes to a **Merge** node (Mode: Pass-through).
15. Finally, connect the Merge node to a **Telegram** node to send the admin confirmation.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| **Sheet Structure:** Create a sheet named `EventPostLog` with columns: `EventID`, `EventName`, `Platform`, `ScheduledTime`, `PostedAt`, `Status`, `ImageURL`. | Google Sheets Setup |
| **Facebook API:** Ensure your Page Access Token has `pages_manage_posts` and `pages_read_engagement` scopes. | Facebook Graph API |
| **Image Template:** The banner template must be a public URL accessible by n8n. | Branding Asset |
| **Wait Node:** Ensure n8n is configured with a persistent database (not SQLite in-memory) if wait times exceed several hours to prevent data loss on restart. | n8n Infrastructure |