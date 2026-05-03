Send instant replies to new Yelp leads with LeadResponder.ai

https://n8nworkflows.xyz/workflows/send-instant-replies-to-new-yelp-leads-with-leadresponder-ai-12883


# Send instant replies to new Yelp leads with LeadResponder.ai

## 1. Workflow Overview

**Purpose:** This workflow automatically sends an instant reply to **new Yelp leads** using the **LeadResponder.ai** API. It is designed for small businesses that want immediate engagement when a prospect reaches out.

**Primary use case:** Poll Yelp leads every minute; when new leads exist, send a predefined reply message to the associated conversation.

### 1.1 Scheduled Lead Polling (Entry Block)
Runs every minute and calls LeadResponder.ai to retrieve newly detected Yelp leads.

### 1.2 Message Preparation
Builds the reply payload (conversation ID + message text) for each lead returned.

### 1.3 Reply Dispatch
Posts the prepared message back to LeadResponder.ai to reply to the Yelp lead conversation.

---

## 2. Block-by-Block Analysis

### Block 1.1 — Scheduled Lead Polling (Entry Block)

**Overview:** Triggers every minute and requests new Yelp leads from LeadResponder.ai. Output items represent leads to respond to.

**Nodes Involved:**
- Schedule Trigger for 1 minute check
- Call to check new leads

#### Node: “Schedule Trigger for 1 minute check”
- **Type / Role:** `Schedule Trigger` — workflow entry point; executes on a time interval.
- **Configuration (interpreted):**
  - Runs every **1 minute**.
- **Inputs:** None (trigger node).
- **Outputs / Connections:**
  - Sends execution to **Call to check new leads**.
- **Version notes:** typeVersion **1.3** (no special requirements).
- **Potential failures / edge cases:**
  - If the n8n instance is stopped/sleeping, scheduled executions won’t run.
  - Time drift or delayed executions under heavy load.

#### Node: “Call to check new leads”
- **Type / Role:** `HTTP Request` — polls LeadResponder.ai for new Yelp leads.
- **Configuration (interpreted):**
  - **GET** request to: `https://api.leadresponder.ai/yelp/leads/new`
  - Sends a header `X-API-KEY` (value is expected to be provided in node params/credentials/environment—see notes below).
  - No explicit query/body parameters configured.
- **Key expressions / variables:** none shown in the node (header value not included in JSON).
- **Inputs:** Trigger event from schedule node.
- **Outputs / Connections:**
  - Output goes to **Prepare message for reply**.
- **Version notes:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - **401/403** if `X-API-KEY` is missing/invalid.
  - API may return **no items** (no new leads), depending on how LeadResponder.ai responds (empty array vs. 200 with message vs. non-200).
  - Network errors/timeouts.
  - Response schema changes (missing `conversation_id`).
- **Other notes:**
  - Workflow includes **pinned test data** for this node showing expected fields like `conversation_id`, `customer_name`, `project_description`, etc.

---

### Block 1.2 — Message Preparation

**Overview:** Creates a static reply message and maps it to each lead’s `conversation_id` to form the payload needed by the reply endpoint.

**Nodes Involved:**
- Prepare message for reply

#### Node: “Prepare message for reply”
- **Type / Role:** `Code` — transforms each lead item into a reply payload.
- **Configuration (interpreted):**
  - Runs **once per item** (“runOnceForEachItem”).
  - JavaScript constructs:
    - `message_text` (static string)
    - returns an object with:
      - `conversation_id: $json?.conversation_id`
      - `message_text`
- **Key expressions / variables used:**
  - Uses optional chaining: `$json?.conversation_id`
  - Static text:
    - `"Hello! Thanks for reaching out. We are reviewing your request and will get back to you shortly."`
- **Inputs:** Items produced by “Call to check new leads”.
- **Outputs / Connections:**
  - Sends payload to **Push message as a reply to a new lead.**
- **Version notes:** typeVersion **2**.
- **Potential failures / edge cases:**
  - If `conversation_id` is missing/null, the next HTTP request will likely fail (validation error or API rejection).
  - If upstream returns a single object instead of an array, behavior depends on n8n’s HTTP node parsing (usually still an item, but schema may differ).

---

### Block 1.3 — Reply Dispatch

**Overview:** Posts the reply to LeadResponder.ai using the conversation ID and message text created in the previous block.

**Nodes Involved:**
- Push message as a reply to a new lead.

#### Node: “Push message as a reply to a new lead.”
- **Type / Role:** `HTTP Request` — sends the reply message to the Yelp conversation via LeadResponder.ai.
- **Configuration (interpreted):**
  - **POST** request to: `https://api.leadresponder.ai/yelp/message/reply`
  - Sends header `X-API-KEY`.
  - Sends body parameters:
    - `conversation_id` = `{{ $json?.conversation_id }}`
    - `message_text` = `{{ $json?.message_text }}`
- **Key expressions / variables used:**
  - `={{ $json?.conversation_id }}`
  - `={{ $json?.message_text }}`
- **Inputs:** Output of “Prepare message for reply”.
- **Outputs / Connections:** None (terminal node).
- **Version notes:** typeVersion **4.3**.
- **Potential failures / edge cases:**
  - **401/403** if API key invalid/missing.
  - **400/422** if `conversation_id` is empty/invalid or message content rejected.
  - Rate limiting (429) if many leads arrive at once.
  - If API expects JSON body vs form-encoded parameters, misconfiguration could cause rejection (depends on LeadResponder.ai expectations; current node uses “bodyParameters” style).

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger for 1 minute check | Schedule Trigger | Run workflow every minute | — | Call to check new leads | ## Yelp Lead Responder; Automate replies for new Yelp leads...; Setup: sign up https://leadresponder.ai/ to obtain an API key...; Video: @[youtube](LWCUU32ZK0A) |
| Call to check new leads | HTTP Request | Fetch new Yelp leads from LeadResponder.ai | Schedule Trigger for 1 minute check | Prepare message for reply | ## Yelp Lead Responder; Automate replies for new Yelp leads...; Setup: sign up https://leadresponder.ai/ to obtain an API key...; Video: @[youtube](LWCUU32ZK0A) |
| Prepare message for reply | Code | Build reply payload per lead | Call to check new leads | Push message as a reply to a new lead. | ## Yelp Lead Responder; Customization: upgrade from static messages to AI-generated personalization...; Video: @[youtube](LWCUU32ZK0A) |
| Push message as a reply to a new lead. | HTTP Request | Send reply message to the lead conversation | Prepare message for reply | — | ## Yelp Lead Responder; Customization: automate scheduling by sending availability...; Video: @[youtube](LWCUU32ZK0A) |
| Sticky Note | Sticky Note | Documentation / setup guidance | — | — | ## Yelp Lead Responder; Automate replies for new Yelp leads...; Setup: sign up https://leadresponder.ai/ to obtain an API key...; Video: @[youtube](LWCUU32ZK0A) |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name: **Yelp-Lead-Responder**
   - Keep it inactive until credentials are set.

2. **Add Trigger node**
   - Add node: **Schedule Trigger**
   - Name: “Schedule Trigger for 1 minute check”
   - Set **Interval**: every **1 minute**

3. **Add lead polling node**
   - Add node: **HTTP Request**
   - Name: “Call to check new leads”
   - Method: **GET**
   - URL: `https://api.leadresponder.ai/yelp/leads/new`
   - Enable **Send Headers**
   - Add header:
     - Name: `X-API-KEY`
     - Value: *(your LeadResponder.ai API key)*
   - Connect: **Schedule Trigger → Call to check new leads**

4. **Add message preparation node**
   - Add node: **Code**
   - Name: “Prepare message for reply”
   - Mode: **Run Once for Each Item**
   - Paste code (conceptually equivalent):
     - Define a `message_text` string (your default reply)
     - Return `{ conversation_id: $json.conversation_id, message_text }`
   - Connect: **Call to check new leads → Prepare message for reply**

5. **Add reply dispatch node**
   - Add node: **HTTP Request**
   - Name: “Push message as a reply to a new lead.”
   - Method: **POST**
   - URL: `https://api.leadresponder.ai/yelp/message/reply`
   - Enable **Send Headers**
     - Header `X-API-KEY`: *(your LeadResponder.ai API key)*
   - Enable **Send Body**
     - Add body parameters:
       - `conversation_id` = expression `{{ $json.conversation_id }}`
       - `message_text` = expression `{{ $json.message_text }}`
   - Connect: **Prepare message for reply → Push message...**

6. **Credentials / secrets handling (recommended)**
   - Store the API key outside node text:
     - Use an n8n **environment variable** (e.g., `LEADRESPONDER_API_KEY`) and reference it in the header value.
     - Or use n8n’s credential system if you wrap it in an HTTP credential pattern.
   - Ensure both HTTP nodes use the same key source.

7. **Add documentation note (optional but matches original)**
   - Add a **Sticky Note** describing setup and linking to:
     - `https://leadresponder.ai/`
     - YouTube reference: `@[youtube](LWCUU32ZK0A)`

8. **Test**
   - Execute workflow manually.
   - Confirm “Call to check new leads” returns items containing `conversation_id`.
   - Confirm the POST node returns success (2xx) and the Yelp conversation receives the reply.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Sign up to obtain an API key, then enter it into the workflow settings (used as `X-API-KEY`). | https://leadresponder.ai/ |
| Workflow idea: monitor Yelp every minute; when a lead is detected, send an automated reply immediately; aim: faster engagement and potentially higher conversion. | Sticky note content |
| Possible enhancements: replace static reply with AI-personalized text; include scheduling/availability in the response. | Sticky note content |
| Video reference | @[youtube](LWCUU32ZK0A) |