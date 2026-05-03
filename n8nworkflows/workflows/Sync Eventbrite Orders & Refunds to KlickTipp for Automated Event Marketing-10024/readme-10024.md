Sync Eventbrite Orders & Refunds to KlickTipp for Automated Event Marketing

https://n8nworkflows.xyz/workflows/sync-eventbrite-orders---refunds-to-klicktipp-for-automated-event-marketing-10024


# Sync Eventbrite Orders & Refunds to KlickTipp for Automated Event Marketing

### 1. Workflow Overview

This workflow automates the synchronization of Eventbrite order and refund events with KlickTipp to enable automated event marketing and accurate contact segmentation. It listens for new orders and refunds on Eventbrite, processes each event accordingly, updates contact tags and custom fields in KlickTipp, and applies segmentation based on payment status.

**Target Use Cases:**  
- Event organizers who want to automate attendee management and marketing segmentation based on Eventbrite data.  
- Digital marketers using KlickTipp for email marketing and segmentation, needing real-time synchronization with Eventbrite registrations and refunds.  
- Automation specialists integrating Eventbrite with KlickTipp to eliminate manual data imports.

**Logical Blocks:**  
- **1.1 Data Reception & Routing:** Receive Eventbrite webhook events for orders and refunds; route data based on event type.  
- **2.1 Saving Primary Data (Processing Orders):** Fetch detailed event information, subscribe attendees in KlickTipp, and enrich contact data with event info.  
- **2.2 Saving Secondary Data for Segmentation:** Apply tags and segmentation based on purchase status (paid vs free).  
- **3.1 Refund & Segmentation:** Tag contacts in KlickTipp for refunds, removing or adjusting segmentation accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Data Reception & Routing

**Overview:**  
This block listens for Eventbrite order and refund events via webhook and routes the incoming data to appropriate processing branches (order or refund).

**Nodes Involved:**  
- Check for new orders and refunds (Eventbrite Trigger)  
- Order or refund? (Switch Node)  
- Sticky Note (general block note)

**Node Details:**  

- **Check for new orders and refunds**  
  - *Type:* Eventbrite Trigger  
  - *Role:* Listens for Eventbrite events of type `order.placed` and `order.refunded` for a specific event and organization, triggering the workflow on new data.  
  - *Configuration:* OAuth2 authentication to Eventbrite; filters on specific event ID and organization ID; listens for two action types.  
  - *Inputs:* External webhook call from Eventbrite.  
  - *Outputs:* Emits JSON payloads with order/refund data.  
  - *Edge cases:* Webhook failures, OAuth token expiration, event or organization ID mismatches.

- **Order or refund?**  
  - *Type:* Switch  
  - *Role:* Routes data based on the status field (`placed` for new orders, `refunded` for refunds).  
  - *Configuration:* Two outputs: "New order" if status equals `placed`, "Refund" if status equals `refunded`.  
  - *Inputs:* JSON from Eventbrite Trigger.  
  - *Outputs:* Two branches for further processing.  
  - *Edge cases:* Unexpected status values cause the workflow to halt or skip processing; expression evaluation failure.

---

#### 2.1 Saving Primary Data (Processing Orders)

**Overview:**  
Fetches detailed event data from Eventbrite for new orders and subscribes the attendee to KlickTipp, enriching their profile with event information.

**Nodes Involved:**  
- Get event data (HTTP Request)  
- Subscribe event attendee to KlickTipp (KlickTipp node)  
- Purchase? (If node for fee check)  
- Tag attendee in case of purchase (KlickTipp node)  
- Sticky Notes (describing process)

**Node Details:**  

- **Get event data**  
  - *Type:* HTTP Request  
  - *Role:* Calls Eventbrite API to retrieve detailed event information for the relevant event ID.  
  - *Configuration:* OAuth2 credentials for Eventbrite; URL dynamically built using event ID from incoming data.  
  - *Inputs:* New order data containing event ID.  
  - *Outputs:* Event details JSON including event name, URL, and end date.  
  - *Edge cases:* API request failures, invalid or missing event IDs, OAuth token issues.

- **Subscribe event attendee to KlickTipp**  
  - *Type:* KlickTipp subscriber node  
  - *Role:* Subscribes the attendee’s email to a KlickTipp list and populates custom fields with attendee and event data.  
  - *Configuration:* Uses email from Eventbrite order; tags the subscriber with “Eventbrite | Buyer” tag; sets custom fields: first name, last name, event name, event URL, event end timestamp (converted to Unix seconds).  
  - *Inputs:* Event data from previous node and order data.  
  - *Outputs:* KlickTipp API response confirming subscription.  
  - *Edge cases:* KlickTipp API errors, invalid tag or field IDs, missing email or data fields.

- **Purchase?**  
  - *Type:* If node  
  - *Role:* Checks if the event order was paid (base price > 0) or free.  
  - *Configuration:* Parses and cleans the price string from the order JSON, converting it to float for comparison > 0.  
  - *Inputs:* Order data.  
  - *Outputs:* True branch for paid events, false branch ignored (no node connected).  
  - *Edge cases:* Price parsing errors due to unexpected formatting, missing cost data.

- **Tag attendee in case of purchase**  
  - *Type:* KlickTipp contact-tagging node  
  - *Role:* Applies a “paid attendee” tag in KlickTipp for segmentation.  
  - *Configuration:* Uses subscriber email; applies tag with ID corresponding to paid event attendees.  
  - *Inputs:* Email from subscription node.  
  - *Outputs:* Confirmation of tagging.  
  - *Edge cases:* API failure, invalid tag ID, missing email.

---

#### 2.2 Saving Secondary Data for Segmentation (Implicit in 2.1 & 3.1)

This block is described in sticky notes but functionally covered by tagging nodes applying segmentation tags based on purchase or refund status.

---

#### 3.1 Refund & Segmentation

**Overview:**  
For refunded orders, this block applies refund tags in KlickTipp to update segmentation and remove buyer tags if configured in KlickTipp.

**Nodes Involved:**  
- Tag contact for refund (KlickTipp node)  
- Sticky Note (description)

**Node Details:**  

- **Tag contact for refund**  
  - *Type:* KlickTipp contact-tagging node  
  - *Role:* Tags contacts in KlickTipp as refundees to adjust marketing segmentation.  
  - *Configuration:* Uses email from Eventbrite refund event; applies refund tag ID.  
  - *Inputs:* Refund event data from the switch node.  
  - *Outputs:* Confirmation of tagging.  
  - *Edge cases:* API errors, invalid or missing email, incorrect tag ID.

---

### 3. Summary Table

| Node Name                      | Node Type                     | Functional Role                                  | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                        |
|-------------------------------|-------------------------------|-------------------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Check for new orders and refunds | Eventbrite Trigger            | Listens for Eventbrite order/refund events       | -                                | Order or refund?                 | ## 1. Data Reception: Listens for new Eventbrite orders or refunds via webhook trigger. Routes each record accordingly. |
| Order or refund?               | Switch                        | Routes data to order or refund processing        | Check for new orders and refunds | Get event data (order), Tag contact for refund (refund) | ## 1. Data reception & routing                                                                                     |
| Get event data                | HTTP Request                  | Fetches detailed event info from Eventbrite      | Order or refund? (New order)      | Subscribe event attendee to KlickTipp | ## 2. Process Orders: Fetches event details and subscribes contacts to KlickTipp, adds event info automatically.   |
| Subscribe event attendee to KlickTipp | KlickTipp subscriber         | Subscribes attendee and enriches profile         | Get event data                   | Purchase?                       | ## 2. Process Orders                                                                                                |
| Purchase?                     | If                           | Checks if event was paid or free                   | Subscribe event attendee to KlickTipp | Tag attendee in case of purchase | ## 2. Process Orders                                                                                                |
| Tag attendee in case of purchase | KlickTipp contact-tagging     | Tags paid attendees for segmentation              | Purchase? (true branch)           | -                               |                                                                                                                    |
| Tag contact for refund        | KlickTipp contact-tagging     | Tags contacts for refunds to update segmentation | Order or refund? (Refund branch) | -                               | ## 3. Refund & Segmentation: Applies refund tags in KlickTipp when Eventbrite reports a refund.                     |
| Sticky Note                  | Sticky Note                  | Block description notes                            | -                                | -                               | Community Node Disclaimer and detailed workflow introduction (general info).                                       |
| Sticky Note1                 | Sticky Note                  | Block description notes                            | -                                | -                               | ## 2. Saving primary data                                                                                           |
| Sticky Note2                 | Sticky Note                  | Block description notes                            | -                                | -                               | ## 2. Saving secondary data for segmentation                                                                        |
| Sticky Note3                 | Sticky Note                  | General introduction and instructions             | -                                | -                               | Community Node Disclaimer and detailed workflow description                                                         |
| Sticky Note4                 | Sticky Note                  | Block description notes                            | -                                | -                               | ## 2. Process Orders                                                                                                |
| Sticky Note5                 | Sticky Note                  | Block description notes                            | -                                | -                               | ## 1. Data Reception                                                                                                |
| Sticky Note6                 | Sticky Note                  | Block description notes                            | -                                | -                               | ## 3. Refund & Segmentation                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Eventbrite Trigger node:**  
   - Type: Eventbrite Trigger  
   - Configure OAuth2 credentials for Eventbrite API access.  
   - Set event ID to the specific Eventbrite event (e.g., `1764245398479`).  
   - Set organization ID accordingly.  
   - Select actions: `order.placed`, `order.refunded`.  
   - Position this as the starting node.

2. **Add Switch node named "Order or refund?":**  
   - Connect input from Eventbrite Trigger node.  
   - Configure two outputs:  
     - Output 1 (label: "New order") with condition: `status == "placed"`  
     - Output 2 (label: "Refund") with condition: `status == "refunded"`  

3. **Create HTTP Request node "Get event data":**  
   - Connect from "New order" output of Switch.  
   - Method: GET  
   - URL: `https://www.eventbriteapi.com/v3/events/{{ $json.event_id }}`  
   - Use Eventbrite OAuth2 credentials.  
   - This fetches detailed event info.

4. **Create KlickTipp node "Subscribe event attendee to KlickTipp":**  
   - Connect output from "Get event data".  
   - Operation: Subscribe subscriber to list.  
   - Use KlickTipp API credentials.  
   - Email: `={{ $('Check for new orders and refunds').item.json.email }}`  
   - List ID: your target KlickTipp list ID (e.g., `358895`).  
   - Tag ID: ID for "Eventbrite | Buyer" tag (e.g., `13634764`).  
   - Map custom fields:  
     - First name: from Eventbrite order JSON (`first_name`)  
     - Last name: from Eventbrite order JSON (`last_name`)  
     - Event name: `$json.name.text` from event data  
     - Event page URL: `$json.url` from event data  
     - Event end timestamp: convert event end date to Unix timestamp in seconds.

5. **Create If node "Purchase?":**  
   - Connect from "Subscribe event attendee to KlickTipp".  
   - Condition: Check if base price from the order is greater than zero.  
   - Expression: Parse and clean `costs.base_price.display` string to float and check `> 0`.

6. **Create KlickTipp node "Tag attendee in case of purchase":**  
   - Connect from "true" output of "Purchase?" node.  
   - Operation: Contact tagging.  
   - Use KlickTipp API credentials.  
   - Email: from subscription node output.  
   - Tag ID: ID for paid attendee tag (e.g., `13637318`).

7. **Create KlickTipp node "Tag contact for refund":**  
   - Connect from "Refund" output of "Order or refund?" Switch.  
   - Operation: Contact tagging.  
   - Use KlickTipp API credentials.  
   - Email: `={{ $('Check for new orders and refunds').item.json.email }}`  
   - Tag ID: ID for refund tag (e.g., `13637325`).

8. **Add sticky notes** as needed for clarity:  
   - For each logical block, add explanatory sticky notes describing purpose and flow.

9. **Credentials setup:**  
   - Configure Eventbrite OAuth2 credentials with required scopes for event and order data.  
   - Configure KlickTipp API credentials with appropriate API key and permissions.

10. **Testing:**  
    - Test with a placed order event to verify subscription and tagging.  
    - Test with a refund event to verify refund tagging.  
    - Ensure tag IDs and field IDs match those in your KlickTipp account.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| **Community Node Disclaimer:** This workflow uses KlickTipp community nodes and works only on self-hosted n8n instances.                                                                                                                                                                                                                                                                                                                                                                         | Workflow description and prerequisites.                                                         |
| **Workflow Introduction:** Automates Eventbrite order and refund processing by syncing data directly to KlickTipp, keeping segmentation accurate and automated.                                                                                                                                                                                                                                                                                                                                 | Workflow description.                                                                            |
| **Requirements:** Self-hosted n8n, Eventbrite OAuth2 account, KlickTipp API access, KlickTipp custom fields and tags for event data and segmentation (Eventbrite event name, start timestamp, event page URL, Buyer tag, Refundee tag, Registrant tag).                                                                                                                                                                                                                                                | Workflow description.                                                                            |
| **Setup Tips:** Replace tag and field IDs with your own KlickTipp IDs. Enable auto tag removal in KlickTipp so Buyer tags are removed when Refundee tags are added.                                                                                                                                                                                                                                                                                                                             | Workflow description.                                                                            |
| **Campaign Expansion Ideas:** Track attendance vs. no-shows, add VIP or ticket-type segmentation, trigger follow-up automations, connect to other tools for reminders, surveys, or upsells.                                                                                                                                                                                                                                                                                                      | Workflow description.                                                                            |
| **Documentation & Workflow Sharing:** The detailed community node info and workflow logic is documented within sticky notes inside the workflow for easy reference and customization.                                                                                                                                                                                                                                                                                                            | Internal sticky notes within the workflow.                                                      |

---

**Disclaimer:**  
The provided text is generated from an automated n8n workflow respecting current content policies. It contains no illegal, offensive, or protected material. All data handled is legal and public.