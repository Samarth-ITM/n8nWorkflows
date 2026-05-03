Service Scheduling & Route Planner for Deliveries with Notion, Telegram and Maps

https://n8nworkflows.xyz/workflows/service-scheduling---route-planner-for-deliveries-with-notion--telegram-and-maps-8077


# Service Scheduling & Route Planner for Deliveries with Notion, Telegram and Maps

### 1. Workflow Overview

This workflow, titled **"Graceful Deliveries – Service Scheduling & Route Planner (Notion/Sheets-Ready)"**, is designed to automate the process of managing delivery service bookings. It integrates with Notion or Google Sheets for data storage, Telegram for owner notifications, and email for client communication. The workflow receives booking data via a webhook, processes and enriches it with geolocation data, compiles summaries and actionable links, alerts the delivery owner, and sends a pre-arrival email to the client one hour before the scheduled delivery.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Parsing:** Captures incoming booking requests via webhook and parses the raw data.
- **1.2 Geolocation Processing:** Converts client address data into geographical coordinates using OpenStreetMap’s Nominatim API.
- **1.3 Data Merging and Summary Preparation:** Combines parsed booking information with geocode data and formats a delivery summary including route links.
- **1.4 Notification and Scheduling:** Sends immediate Telegram alerts to the delivery owner, schedules a wait period, and then sends a pre-arrival email to the client.
- **1.5 Response Handling:** Responds back to the webhook source confirming receipt and processing of the booking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Parsing

- **Overview:**  
  This block accepts booking requests through a webhook and extracts relevant booking details from the incoming payload for downstream processing.

- **Nodes Involved:**  
  - Webhook (Bookings)  
  - Parse Booking

- **Node Details:**

  - **Webhook (Bookings)**  
    - Type: Webhook  
    - Role: Entry point receiving HTTP requests with booking data.  
    - Configuration: Default webhook settings (likely POST method).  
    - Inputs: External webhook call.  
    - Outputs: Parsed JSON payload passed to Parse Booking.  
    - Edge Cases: Missing or malformed webhook data, HTTP errors, unauthorized calls if security is not configured.

  - **Parse Booking**  
    - Type: Function  
    - Role: Parses and transforms the raw webhook payload into structured fields.  
    - Configuration: Custom JavaScript code to extract booking details such as customer name, address, delivery time, etc.  
    - Inputs: Data from Webhook (Bookings).  
    - Outputs: Parsed booking data forwarded to Geocode and Merge Data nodes.  
    - Edge Cases: Parsing errors due to unexpected payload format or missing fields.

---

#### 2.2 Geolocation Processing

- **Overview:**  
  This block enriches the booking data by converting the delivery address into geographic coordinates using OpenStreetMap’s Nominatim API, then formats the response for use in routing and notifications.

- **Nodes Involved:**  
  - Geocode (OSM Nominatim)  
  - Format Geocode

- **Node Details:**

  - **Geocode (OSM Nominatim)**  
    - Type: HTTP Request  
    - Role: Sends address data to OSM Nominatim API and retrieves latitude and longitude.  
    - Configuration: HTTP GET request with parameters including the address string, JSON response format, possibly rate limiting considerations.  
    - Inputs: Parsed booking data containing address from Parse Booking node.  
    - Outputs: Raw geocode JSON response passed to Format Geocode.  
    - Edge Cases: API rate limits, network timeouts, invalid or ambiguous addresses leading to no results.

  - **Format Geocode**  
    - Type: Function  
    - Role: Extracts and formats latitude and longitude from the OSM response for consistent downstream use.  
    - Configuration: JavaScript to parse JSON and map relevant fields.  
    - Inputs: Raw geocode data from Geocode node.  
    - Outputs: Formatted geolocation data forwarded to Merge Data.  
    - Edge Cases: Missing or malformed API response, empty results.

---

#### 2.3 Data Merging and Summary Preparation

- **Overview:**  
  This block merges parsed booking data with formatted geolocation data to create a comprehensive summary summary, including routing links (e.g., Google Maps) for the delivery owner.

- **Nodes Involved:**  
  - Merge Data  
  - Build Summary & Links

- **Node Details:**

  - **Merge Data**  
    - Type: Merge  
    - Role: Combines two data streams — the parsed booking data and the geocode information — into a single cohesive dataset.  
    - Configuration: Merge mode likely set to "Wait for all inputs" or "Merge by key" depending on data structure.  
    - Inputs:  
      - Parsed booking data (from Parse Booking fallback output)  
      - Formatted geocode data (from Format Geocode)  
    - Outputs: Combined data to Build Summary & Links.  
    - Edge Cases: Unequal or missing inputs causing incomplete merges.

  - **Build Summary & Links**  
    - Type: Function  
    - Role: Constructs a textual summary of the delivery details and generates actionable links (e.g., directions in maps).  
    - Configuration: JavaScript builds summary including customer info, delivery time, and map URLs.  
    - Inputs: Merged booking and geocode data.  
    - Outputs: Summary and links sent to Telegram notification, wait node, and webhook response.  
    - Edge Cases: Errors in string concatenation or missing fields could cause incomplete messages.

---

#### 2.4 Notification and Scheduling

- **Overview:**  
  This block manages notifications to the delivery owner and client, including immediate alerts and scheduled email reminders prior to delivery time.

- **Nodes Involved:**  
  - Telegram → Owner Alert  
  - Wait until 1h before  
  - Email → Client (Pre-arrival)

- **Node Details:**

  - **Telegram → Owner Alert**  
    - Type: Telegram  
    - Role: Sends an immediate message to the delivery owner’s Telegram account with delivery summary and links.  
    - Configuration: Telegram bot credential setup, chat ID target, message content from Build Summary & Links.  
    - Inputs: Summary data from Build Summary & Links.  
    - Outputs: None (final action for owner alert).  
    - Edge Cases: Telegram API errors, invalid bot token, incorrect chat ID.

  - **Wait until 1h before**  
    - Type: Wait  
    - Role: Pauses workflow execution until one hour before the scheduled delivery time.  
    - Configuration: Wait until a timestamp computed from booking data minus one hour.  
    - Inputs: Delivery time from Build Summary & Links.  
    - Outputs: Triggers Email → Client node after wait.  
    - Edge Cases: Incorrect or missing delivery time causing indefinite wait or immediate trigger.

  - **Email → Client (Pre-arrival)**  
    - Type: Email Send  
    - Role: Sends a pre-arrival email reminder to the client approximately one hour before delivery.  
    - Configuration: SMTP or email service credential, recipient email from booking data, subject and body templated with booking details.  
    - Inputs: Trigger from Wait node, email content constructed from Build Summary & Links or additional formatting.  
    - Outputs: None (final communication step).  
    - Edge Cases: Email delivery failures, invalid email address.

---

#### 2.5 Response Handling

- **Overview:**  
  This block confirms receipt of the booking by responding to the original webhook caller, ensuring synchronous acknowledgment.

- **Nodes Involved:**  
  - Respond to Webhook

- **Node Details:**

  - **Respond to Webhook**  
    - Type: Respond to Webhook  
    - Role: Sends HTTP response back to the webhook caller confirming booking acceptance and processing status.  
    - Configuration: Standard success response, possibly with booking ID or status message.  
    - Inputs: Data from Build Summary & Links.  
    - Outputs: HTTP response to external system.  
    - Edge Cases: Failure to respond in time causing webhook timeouts.

---

### 3. Summary Table

| Node Name                | Node Type              | Functional Role                            | Input Node(s)           | Output Node(s)                      | Sticky Note                |
|--------------------------|------------------------|------------------------------------------|-------------------------|-----------------------------------|----------------------------|
| 📝 Description (Read Me First) | Sticky Note            | Documentation placeholder                 |                         |                                   |                            |
| 🧰 Setup & Credentials     | Sticky Note            | Credentials and environment setup guide   |                         |                                   |                            |
| 📦 Notion / Sheets Schema  | Sticky Note            | Data schema reference for Notion/Sheets  |                         |                                   |                            |
| 🧭 Data Mapper (Sample Payload) | Sticky Note            | Example payload mapping for reference     |                         |                                   |                            |
| Webhook (Bookings)         | Webhook                | Receives incoming booking requests        | External HTTP call      | Parse Booking                    |                            |
| Parse Booking              | Function               | Parses raw webhook data to structured form| Webhook (Bookings)      | Geocode (OSM Nominatim), Merge Data |                            |
| Geocode (OSM Nominatim)   | HTTP Request           | Retrieves latitude/longitude for address  | Parse Booking           | Format Geocode                   |                            |
| Format Geocode             | Function               | Formats geocode API response               | Geocode (OSM Nominatim) | Merge Data                      |                            |
| Merge Data                | Merge                  | Combines parsed booking and geocode data  | Parse Booking, Format Geocode | Build Summary & Links          |                            |
| Build Summary & Links      | Function               | Creates delivery summary and map links    | Merge Data              | Telegram → Owner Alert, Wait until 1h before, Respond to Webhook |                            |
| Telegram → Owner Alert     | Telegram               | Sends delivery alert to owner              | Build Summary & Links   |                                   |                            |
| Wait until 1h before       | Wait                   | Delays workflow until 1 hour before delivery | Build Summary & Links | Email → Client (Pre-arrival)     |                            |
| Email → Client (Pre-arrival) | Email Send             | Sends pre-arrival reminder email to client| Wait until 1h before    |                                   |                            |
| Respond to Webhook         | Respond to Webhook     | Responds to webhook confirming booking     | Build Summary & Links   |                                   |                            |
| ✨ Optional Add-Ons        | Sticky Note            | Placeholder for future feature extensions |                         |                                   |                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Name: `Webhook (Bookings)`  
   - Type: Webhook  
   - HTTP Method: POST (default)  
   - Purpose: Receive incoming booking requests. Save the generated webhook URL.

2. **Create Function Node to Parse Incoming Booking:**  
   - Name: `Parse Booking`  
   - Type: Function  
   - Connect input from `Webhook (Bookings)` node.  
   - Script: Extract booking fields (e.g., customer name, address, delivery time) from webhook JSON payload, normalize data structure.

3. **Create HTTP Request Node for Geocoding:**  
   - Name: `Geocode (OSM Nominatim)`  
   - Type: HTTP Request  
   - Connect input from `Parse Booking` node.  
   - Method: GET  
   - URL: `https://nominatim.openstreetmap.org/search`  
   - Query Parameters:  
     - `q`: Address string from parsed data  
     - `format`: `json`  
     - (Optional) `addressdetails`: `1`  
   - Ensure to respect usage policy and implement rate limiting if needed.

4. **Create Function Node to Format Geocode Response:**  
   - Name: `Format Geocode`  
   - Type: Function  
   - Connect input from `Geocode (OSM Nominatim)`  
   - Script: Extract latitude and longitude from the first search result and create a simplified object.

5. **Create Merge Node:**  
   - Name: `Merge Data`  
   - Type: Merge  
   - Connect two inputs:  
     - Primary: `Parse Booking` (fallback output)  
     - Secondary: `Format Geocode`  
   - Mode: Wait for both inputs to merge the datasets.

6. **Create Function Node to Build Summary and Links:**  
   - Name: `Build Summary & Links`  
   - Type: Function  
   - Connect input from `Merge Data` node.  
   - Script: Compose a textual summary including customer details, delivery time, and generate map route URLs (e.g., Google Maps link with lat/lon).

7. **Create Telegram Node for Owner Alert:**  
   - Name: `Telegram → Owner Alert`  
   - Type: Telegram  
   - Connect input from `Build Summary & Links`.  
   - Configure Telegram credentials (Bot Token).  
   - Set chat ID to delivery owner’s Telegram chat.  
   - Message: Use summary prepared in previous node.

8. **Create Wait Node to Delay Until One Hour Before Delivery:**  
   - Name: `Wait until 1h before`  
   - Type: Wait  
   - Connect input from `Build Summary & Links`.  
   - Configure to wait until timestamp calculated as delivery time minus one hour.

9. **Create Email Send Node for Pre-arrival Notification:**  
   - Name: `Email → Client (Pre-arrival)`  
   - Type: Email Send  
   - Connect input from `Wait until 1h before`.  
   - Configure SMTP or email service credentials.  
   - Recipient: Client email extracted from booking data.  
   - Subject and body: Use templated content with delivery info.

10. **Create Respond to Webhook Node:**  
    - Name: `Respond to Webhook`  
    - Type: Respond to Webhook  
    - Connect input from `Build Summary & Links` (parallel to Telegram and Wait nodes).  
    - Configure to send HTTP 200 response with confirmation message.

11. **Link nodes according to the flow described:**  
    - `Webhook (Bookings)` → `Parse Booking`  
    - `Parse Booking` → `Geocode (OSM Nominatim)` and fallback → `Merge Data` (input 1)  
    - `Geocode (OSM Nominatim)` → `Format Geocode` → `Merge Data` (input 2)  
    - `Merge Data` → `Build Summary & Links`  
    - `Build Summary & Links` → `Telegram → Owner Alert`  
    - `Build Summary & Links` → `Wait until 1h before` → `Email → Client (Pre-arrival)`  
    - `Build Summary & Links` → `Respond to Webhook`

12. **Add Sticky Notes:**  
    - Add notes for description, setup & credentials, schema references, sample payload mapping, and optional add-ons as placeholders for documentation.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                           |
|----------------------------------------------------------------------------------------------------|------------------------------------------|
| The workflow is designed to be compatible with Notion or Google Sheets schemas for booking storage. | See sticky notes: "📦 Notion / Sheets Schema" |
| Telegram notifications require a Telegram Bot and chat ID for owner alerting.                       | Telegram bot setup instructions: https://core.telegram.org/bots/api |
| OpenStreetMap Nominatim usage policy must be respected to avoid service blocking.                    | https://operations.osmfoundation.org/policies/nominatim/ |
| Email notifications require proper SMTP or email service credentials configured in n8n.             | Common providers: Gmail, Outlook, SendGrid |
| Review and test webhook security settings to prevent unauthorized access.                            | Use authentication or IP restrictions if necessary |
| Future enhancements can be added under "✨ Optional Add-Ons" sticky note placeholder.                | Possible extensions: route optimization, SMS alerts |

---

**Disclaimer:**  
The provided text derives exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.