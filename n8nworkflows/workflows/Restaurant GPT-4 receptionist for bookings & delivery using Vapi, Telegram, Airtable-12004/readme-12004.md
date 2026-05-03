Restaurant GPT-4 receptionist for bookings & delivery using Vapi, Telegram, Airtable

https://n8nworkflows.xyz/workflows/restaurant-gpt-4-receptionist-for-bookings---delivery-using-vapi--telegram--airtable-12004


# Restaurant GPT-4 receptionist for bookings & delivery using Vapi, Telegram, Airtable

## 1. Workflow Overview

**Purpose:**  
This n8n system acts as a **restaurant receptionist** primarily via **Telegram** (text + voice). It uses an **AI Agent (GPT-4.1)** to conduct a structured conversation for:
- **Table bookings** (check availability, book, update, cancel, look up)
- **Food orders** (menu lookup, order validation + pricing, takeaway/delivery creation, delivery address validation, order lookup/cancel)

**Core integrations:**
- **Telegram** (trigger + reply, file download for voice)
- **OpenAI** (chat model + audio transcription)
- **Airtable** (tables: Tables List, Bookings, Menu, Orders)
- **Google Maps Geocoding API** (delivery address validation)
- **MCP server + MCP client tool** (exposes “restaurant tools” as callable sub-workflows)

### 1.1 Telegram Intake & Voice Transcription
Receives Telegram messages; if voice, downloads and transcribes; merges into a single text stream.

### 1.2 AI Agent Conversation + Tool Calling
The AI Agent consumes user text + date context + memory and calls backend tools via MCP to check tables, create/update bookings, fetch menu, validate address, calculate price, and create/cancel orders.

### 1.3 MCP Server Tool Hub (Entry Point for Tools)
An MCP Server Trigger exposes the set of “tool workflows” to AI callers (the agent calls tools through the MCP client tool; MCP routes to these tool workflows).

### 1.4 Booking Tools (Airtable-backed)
Implements:
- Check availability (capacity + overlap checks)
- Book table (create booking record)
- Get booking (search by phone)
- Update booking (update by id)
- Delete booking (delete by id)

### 1.5 Ordering Tools (Airtable-backed)
Implements:
- Get menu
- Calculate price (validate items against menu)
- Validate delivery address (geocode + distance check)
- Create takeaway order (generate orderId + create record)
- Create delivery order (generate orderId + create record)
- Get order (search by phone)
- Cancel order (delete by Airtable record id)

---

## 2. Block-by-Block Analysis

### Block 2.1 — Telegram Message Reception (Text vs Voice)
**Overview:** Receives Telegram messages and routes them either directly (text) or through voice transcription, then merges both paths into one item for the AI agent.  
**Nodes involved:** `Wait for New Message`, `Check if Text`, `Get Voice`, `Transcribe The Voice`, `Send Transcribe Context To AI Agent`, `Merge`

#### Node: Wait for New Message
- **Type / Role:** Telegram Trigger; entry point for Telegram “message” updates.
- **Config:** Listens to `updates: ["message"]`.
- **Outputs:** One item containing `message` (text or voice payload).
- **Failure modes:** Telegram webhook misconfiguration, credential revoked, bot not allowed in chat.

#### Node: Check if Text
- **Type / Role:** IF; branch based on whether `message.text` exists.
- **Condition:** `{{$json.message.text}} exists`
- **True path:** goes to `Merge` (text directly).
- **False path:** goes to `Get Voice`.
- **Failure modes:** Telegram payload differences (e.g., captions), message formats missing `message.text`.

#### Node: Get Voice
- **Type / Role:** Telegram node (File download); fetches voice file by `file_id`.
- **Config:** `resource: file`, `fileId: {{$json.message.voice.file_id}}`
- **Output:** File/binary info for transcription.
- **Failure modes:** `message.voice` missing, file retrieval error, Telegram rate limits.

#### Node: Transcribe The Voice
- **Type / Role:** OpenAI Audio Transcription; converts voice to text.
- **Config:** `resource: audio`, `operation: transcribe`
- **Output:** Typically contains `text` (transcription).
- **Failure modes:** OpenAI auth errors, unsupported audio format, request size limits, timeouts.

#### Node: Send Transcribe Context To AI Agent
- **Type / Role:** Set; maps transcription to the same structure as text messages.
- **Key assignments:**
  - `message.text = {{$json.text}}`
  - `sessionId = {{$('Telegram Trigger1').item.json.message.chat.id}}` *(potential issue: node name mismatch; see edge cases)*
- **Output:** Item with `message.text` to merge.
- **Failure modes / Edge cases:**
  - The expression references `Telegram Trigger1`, but the actual trigger node is named **“Wait for New Message”**. This likely breaks unless another node in the environment is actually named Telegram Trigger1. Correct expression should typically use `{{$('Wait for New Message').item.json.message.chat.id}}` or `{{$json.message.chat.id}}` depending on data shape.

#### Node: Merge
- **Type / Role:** Merge; consolidates text and transcribed voice into one stream.
- **Connections:** Receives from:
  - `Check if Text` (text path)
  - `Send Transcribe Context To AI Agent` (voice path)
- **Output:** Feeds `AI Agent`.
- **Failure modes:** If one branch yields no items in certain merge modes; here default Merge behavior is used (ensure mode aligns with desired behavior).

**Sticky notes affecting this block:**
- “# Telegram Booking Bot … Update connection in 3 nodes”
- “### Check — Check the message type.”
- “### Transcribe — If message is voice…”

---

### Block 2.2 — AI Brain: LLM + Memory + MCP Tooling + Telegram Reply
**Overview:** The agent applies the restaurant conversation policy, uses memory keyed by chat id, and calls backend tools via MCP; then replies to Telegram.  
**Nodes involved:** `AI Agent`, `OpenAI Chat Model`, `Simple Memory`, `Restaurant Tool`, `Reply`

#### Node: AI Agent
- **Type / Role:** LangChain Agent; orchestrates conversation and tool calls.
- **Prompt construction:**  
  `USER MESSAGE: {{$json.message.text}}` plus `TODAY IS: {{$now.format('yyyy-MM-dd hh:mm a')}}`
- **System message:** Very extensive; defines booking flow, ordering flow, delivery validation, guardrails, and tool usage rules.
- **Tool usage expectation:** Calls “restaurant tool” for all backend operations.
- **Retry:** `retryOnFail: true`, `waitBetweenTries: 1000ms`
- **Failure modes:**
  - Overly strict/long system prompt may cause slower responses or token pressure.
  - If tool endpoints fail, agent must handle errors; ensure tool error payloads are consistent.

#### Node: OpenAI Chat Model
- **Type / Role:** LLM provider for the agent.
- **Config:** Model `gpt-4.1`.
- **Failure modes:** API key invalid, model not available, rate limit, latency/timeouts.

#### Node: Simple Memory
- **Type / Role:** Buffer memory; keeps last 10 turns per session.
- **Session key:** `{{$('Merge').item.json.message.chat.id}}`
- **Edge cases:** If incoming data does not include `message.chat.id` in merged item, memory may not persist correctly.

#### Node: Restaurant Tool
- **Type / Role:** MCP Client Tool; provides the agent with a single “tool gateway”.
- **Config:** `endpointUrl: https://maksik.app.n8n.cloud/mcp/451141d4-f7c8-4a46-a1c0-b4fec65aa9d0`
- **Connectivity:** Connected to AI Agent as `ai_tool`.
- **Failure modes:** Endpoint unreachable, auth/access restrictions, tool schema mismatch.

#### Node: Reply
- **Type / Role:** Telegram send message.
- **Config:**
  - `text: {{$json.output}}` (agent’s generated response)
  - `chatId: {{$('Merge').item.json.message.chat.id}}`
  - `appendAttribution: false`
- **Failure modes:** Telegram bot blocked, invalid chat id, message formatting issues.

**Sticky notes affecting this block:**
- “### AI Agent — Brain of the system…”
- “### Reply — sends agent’s response…”
- “# MCP Server — Gives access to all tools”

---

### Block 2.3 — MCP Server Entry Point (Tool Exposure)
**Overview:** Provides an MCP server trigger endpoint used to expose multiple tool workflows to the MCP client tool.  
**Nodes involved:** `MCP Server Trigger`

#### Node: MCP Server Trigger
- **Type / Role:** MCP server trigger; provides MCP path for tool invocations.
- **Config:** `path: 451141d4-f7c8-4a46-a1c0-b4fec65aa9d0`
- **Connections:** Many Tool Workflow nodes connect their `ai_tool` output to this trigger.
- **Failure modes:** MCP path mismatch vs client endpoint, cloud URL changes, permissions.

**Sticky note:** “# MCP Server — Gives access to all tools”

---

### Block 2.4 — Tool Workflow Wrappers (Agent-callable “Tools”)
**Overview:** These nodes register sub-workflows as callable tools with defined input schemas. They don’t implement logic themselves; they delegate to other workflows by ID.  
**Nodes involved (wrappers):**
- `Call "Check Availability"` → workflow “Check Table Availability”
- `Call "Book Table"` → workflow “Book a Table”
- `Call "Get a Booking"` → workflow “Get a Booking”
- `Call "Update appointment"` → workflow “Update a Booking”
- `Call "Delete Appointment"` → workflow “Delete Booking”
- `Call 'Get Menu'` → workflow “Get Menu”
- `Call 'Calculate the Price'` → workflow “Calculate the Price”
- `Call 'Validate the Address'` → workflow “Validate the Address”
- `Call 'Create a Takeaway Order'` → workflow “Create a Takeaway Order”
- `Call 'Create a Delivery Order'` → workflow “Create a Delivery Order”
- `Call 'Get an Order'` → workflow “Get an Order”
- `Call 'Cancel Order'` → workflow “Cancel Order”

#### Common configuration pattern
- **Type / Role:** `@n8n/n8n-nodes-langchain.toolWorkflow`; makes a workflow callable as a structured tool.
- **Inputs:** Uses `$fromAI('fieldName', default, type)` bindings, so the agent fills these fields.
- **Schema:** Declares expected fields (string/number/array).
- **Output:** Returns sub-workflow results back to the agent.
- **Failure modes / Edge cases:**
  - Sub-workflow ID not present in target instance.
  - Sub-workflow expects different input names/types.
  - `$fromAI` returns empty strings if extraction fails; downstream logic must validate.

**Sub-workflow references:** Each wrapper points to a separate workflow by ID. Reproducing requires creating these sub-workflows and updating IDs.

---

### Block 2.5 — “Check Table Availability” Tool Implementation (in this JSON)
**Overview:** Computes if the restaurant can handle a party size, converts request window to UTC, finds overlapping bookings, and returns candidate tables or an unavailability message.  
**Nodes involved:** `When Executed by Another Workflow`, `Set Timezone`, `Convert Time to UTC`, `Get All Tables`, `Check Max Capacity`, `If1`, `Can't Handle Party`, `Get Availability`, `Table-Availability Selector`, `If`, `Normalize Output`, `Table is not available`

#### Node: When Executed by Another Workflow
- **Type / Role:** Execute Workflow Trigger; entry point when called as a sub-workflow/tool.
- **Inputs:** `partyCount (number)`, `startDate (string)`, `endDate (string)`
- **Edge cases:** If `partyCount` missing or string-like, strict code may break.

#### Node: Set Timezone
- **Type / Role:** Set; establishes `timezone = "Europe/Kyiv"`.
- **Note:** Hardcoded; change per restaurant locale.

#### Node: Convert Time to UTC
- **Type / Role:** Code; converts `startDate/endDate` from local timezone to UTC ISO.
- **Key logic:** Calculates offset using `toLocaleString` comparison; then appends `+HH:00` to input strings.
- **Outputs:** `partyCount`, `startDate` (UTC ISO), `endDate` (UTC ISO)
- **Edge cases / failures:**
  - Offset sign handling: code always builds `+${abs(offsetHours)}`; Kyiv can be UTC+2/+3 so OK here, but would be wrong for negative offsets.
  - Input expects ISO-like strings without timezone; malformed dates cause `Invalid Date`.
  - OffsetHours may be fractional in some timezones; code assumes full hours.

#### Node: Get All Tables
- **Type / Role:** Airtable search; retrieves records from `Tables List`.
- **Base/Table:** “Restaurant Booking” → “Tables List”
- **Failure modes:** Airtable auth, base/table IDs mismatch, API limits.

#### Node: Check Max Capacity
- **Type / Role:** Code; checks if requested party <= max table capacity.
- **Inputs referenced:** From `When Executed…` and `Get All Tables`.
- **Outputs:** `{canHandle, maxCapacity, requestedParty}`

#### Node: If1
- **Type / Role:** IF; branches on `{{$json.canHandle}} == true`.
- **False path:** `Can't Handle Party`

#### Node: Can't Handle Party
- **Type / Role:** Set; returns `Message = "We can't handle that party count."`

#### Node: Get Availability
- **Type / Role:** Airtable search on `Bookings` for overlapping bookings.
- **Filter formula:**  
  Overlap check: `{Start Date} < end AND {End Date} > start`  
  Uses values from `Convert Time to UTC`.
- **alwaysOutputData:** true (important: even with zero records, downstream nodes still run).
- **Edge cases:** Airtable formula quoting/escaping; date parsing expects consistent ISO.

#### Node: Table-Availability Selector
- **Type / Role:** Code; removes blocked tables and selects candidate tables meeting capacity.
- **Blocked set:** from booking field `Table ID (from Tables List)` (array)
- **Candidates:** not blocked AND `Capacity >= req.partyCount`, sorted smallest capacity first.
- **Output:** `{available, reason, tables: candidates}`

#### Node: If
- **Type / Role:** IF; checks `{{$json.available}} == true`
- **True:** `Normalize Output`
- **False:** `Table is not available`

#### Node: Normalize Output
- **Type / Role:** Code; outputs normalized structure for the tool caller.
- **Output:** `available: true`, `tableIds` (Airtable record ids), `tableDetails` (full candidate list)
- **Edge case:** It always sets `available: true` (even though it only runs on available branch). That’s consistent, but if reused elsewhere it can be misleading.

#### Node: Table is not available
- **Type / Role:** Set; `Message = "Table is not available at requested time."`

**Sticky notes affecting this block:**
- “## Check Availability Tool …”
- “### Set Timezone & Calculate UTC …”
- “### Get All Tables & Check Capacity …”
- “### Get Availability & Sort …”
- “### Branching …”

---

### Block 2.6 — “Book a Table” Tool Implementation (in this JSON)
**Overview:** Converts start/end times to UTC and creates a booking record in Airtable; includes placeholder for SMS.  
**Nodes involved:** `Trigger Node`, `Set Timezone1`, `Calculate Timezone`, `Create a Booking`, `Replace with SMS node`

#### Node: Trigger Node
- **Type / Role:** Set; placeholder sample structure for inputs (startTime/endTime/name/phone/tableID/notes).
- **Note:** In real tool execution, inputs should come from `When Executed by Another Workflow` in the actual “Book a Table” sub-workflow. Here it’s more like a scaffold/test harness.

#### Node: Set Timezone1
- **Type / Role:** Set; `timezone = Europe/Kyiv`

#### Node: Calculate Timezone
- **Type / Role:** Code; converts `startTime/endTime` to UTC ISO; preserves name/phone/notes/tableID.
- **Edge cases:** Same offset sign limitation as earlier.

#### Node: Create a Booking
- **Type / Role:** Airtable create in `Bookings`.
- **Mapped fields:**
  - `Phone = {{$json.phone}}`
  - `Start Date = {{$json.startTime}}` (UTC ISO)
  - `End Date = {{$json.endTime}}` (UTC ISO)
  - `Tables List = [{{$json.tableID}}]` (linked record array)
  - `Booking Name = "Booking for {{name}}"`
  - `Booking Notes = {{$json.notes}}`
- **Failure modes:** linked record id invalid; Airtable schema mismatch.

#### Node: Replace with SMS node
- **Type / Role:** NoOp placeholder for SMS notification.

**Sticky notes:**  
- “## Book Table Tool …”  
- “### Set Timezone & Calculate UTC …”  
- “### Create a Booking & Send SMS …”

---

### Block 2.7 — “Get a Booking” Tool Implementation (in this JSON)
**Overview:** Searches bookings by phone for upcoming bookings; returns booking info or a “no records” message.  
**Nodes involved:** `Trigger Node1`, `Find a Booking`, `If2`, `Message`

#### Node: Trigger Node1
- **Type / Role:** Set; placeholder with `phone`.

#### Node: Find a Booking
- **Type / Role:** Airtable search in `Bookings`.
- **Filter formula:** `AND({Phone} = "<phone>", {Start Date} >= NOW())`
- **Failure modes:** Phone formatting mismatches; NOW() timezone behavior vs stored UTC.

#### Node: If2
- **Type / Role:** IF; checks existence of `{{$json['Booking Name']}}`.
- **Connections:** True branch is empty (no node connected), False branch → `Message`.
- **Important edge case:** As wired, if a booking *is found*, nothing is returned/forwarded from this block (unless n8n returns the found record implicitly elsewhere). Practically, you should connect the **true** branch to a “return booking info” Set/Code node.

#### Node: Message
- **Type / Role:** Set; `Message = "No records for this phone number."`

**Sticky notes:**  
- “## Get a Booking Tool …”  
- “### Find the Booking …”  
- “### Branching …”

---

### Block 2.8 — “Update a Booking” Tool Implementation (in this JSON)
**Overview:** Converts updated start/end window to UTC and updates an Airtable booking record by id; placeholder for SMS.  
**Nodes involved:** `Trigger Node2`, `Set Timezone2`, `Convert Time to UTC1`, `Update Booking`, `Replace with SMS Node`

#### Node: Trigger Node2
- **Type / Role:** Set; placeholder includes `id, startTime/endTime (but code expects startDate/endDate), name, phone, notes`.
- **Edge case:** Naming inconsistency: node sets `startTime/endTime`, but `Convert Time to UTC1` expects `req.startDate/req.endDate`. In the real sub-workflow, align field names.

#### Node: Set Timezone2
- **Type / Role:** Set timezone.

#### Node: Convert Time to UTC1
- **Type / Role:** Code; reads from `When Executed by Another Workflow` (expects `id,startDate,endDate,...`) and outputs normalized update payload.

#### Node: Update Booking
- **Type / Role:** Airtable update in `Bookings`.
- **Key fields:** `id` for record, plus updated Phone, Start/End Date, Booking Name, Booking Notes.
- **Failure modes:** Missing/invalid `id`, Airtable record not found.

#### Node: Replace with SMS Node
- **Type / Role:** NoOp placeholder.

**Sticky notes:**  
- “## Update Booking Tool …”  
- “### Set Timezone & Calculate UTC …”  
- “### Update a Booking & Send SMS …”

---

### Block 2.9 — “Delete Booking” Tool Implementation (in this JSON)
**Overview:** Deletes a booking record by Airtable record id.  
**Nodes involved:** `Trigger Node3`, `Delete Booking`

#### Node: Trigger Node3
- **Type / Role:** Set; placeholder `id`.

#### Node: Delete Booking
- **Type / Role:** Airtable deleteRecord on `Bookings`.
- **Config:** `id = {{$json.id}}`
- **Failure modes:** Wrong id, record already deleted.

**Sticky notes:**  
- “## Cancel Booking Tool …”  
- “### Cancel the Booking …”

---

### Block 2.10 — “Get Menu” Tool Implementation (in this JSON)
**Overview:** Returns menu records from Airtable to the agent/tool caller.  
**Nodes involved:** `Trigger Node4`, `Get Menu`

#### Node: Trigger Node4
- **Type / Role:** Set; empty placeholder.

#### Node: Get Menu
- **Type / Role:** Airtable search on `Menu`.
- **Output:** full menu dataset.
- **Edge cases:** The agent prompt warns not to read entire menu back—this is enforced by prompt, not by workflow.

**Sticky notes:**  
- “## Get a Menu Tool …”  
- “### Get Menu …”

---

### Block 2.11 — “Calculate the Price” Tool Implementation (in this JSON)
**Overview:** Fetches currently available menu items and validates an order list; returns error if any item invalid/unavailable, otherwise returns total and per-line breakdown.  
**Nodes involved:** `Trigger Node6`, `Fetch Menu`, `Order Validation`, `If3`, `Total Price`

#### Node: Trigger Node6
- **Type / Role:** Set; placeholder `items`.

#### Node: Fetch Menu
- **Type / Role:** Airtable search on `Menu`.
- **Filter:** `{Available}` (truthy filter)
- **Failure modes:** Airtable schema: field must exist and be boolean.

#### Node: Order Validation
- **Type / Role:** Code; validates order array and computes totals.
- **Inputs:**
  - `order = $('When Executed by Another Workflow').first().json.items`
  - `menu = $('Fetch Menu').all().map(i => i.json)`
- **Validation logic:**
  - `order` must be array
  - each item has `name` and `quantity`
  - item must exist in menu by case-insensitive name match on `"Item Name"`
  - item must be `Available === true`
  - menu price must be number
- **Output (success):** `{status: true, total, items: breakdown}`
- **Output (failure):** `{status: false, error: "..."}`
- **Failure modes:** Field naming mismatch between menu and code; quantities not numeric.

#### Node: If3
- **Type / Role:** IF
- **Condition:** `{{$json.status}} == false`
- **Connections:** Only the **false** branch is connected to `Total Price` (since true means status==false in this IF configuration).
- **Edge case:** As configured, if `status` is true (order valid), this node outputs nothing further. The logic seems inverted for intended flow. Usually you want: if `status == true` → Total Price, else → error message.

#### Node: Total Price
- **Type / Role:** Set; `Message = "Total price is {{ $json.total }}$."`
- **Edge case:** Requires `total` to exist; if called with invalid order path, it will be missing.

**Sticky notes:**  
- “## Calculate the Price Tool …”  
- “### Get the Menu & Calculate the Price …”  
- “### Branching …”

---

### Block 2.12 — “Validate the Address” Tool Implementation (in this JSON)
**Overview:** Geocodes an address with Google Maps and checks if it’s within a radius of the restaurant using Haversine distance; returns a human-friendly message.  
**Nodes involved:** `Trigger Node7`, `Geocode Address`, `Set Location and Area`, `Check Eligibility`, `Message1`

#### Node: Trigger Node7
- **Type / Role:** Set; placeholder `address`.
- **Edge case:** It defines `address` as an array type in placeholder; downstream expects `address` string.

#### Node: Geocode Address
- **Type / Role:** HTTP Request to Google Geocoding API.
- **Endpoint:** `https://maps.googleapis.com/maps/api/geocode/json`
- **Query parameters:** `key`, `address={{$json.address}}`, `region`
- **Required setup:** Replace `PASTE YOUR KEY HERE` and `PASTE YOUR REGION HERE`.
- **Failure modes:** Invalid key, quota limits, address encoding issues.

#### Node: Set Location and Area
- **Type / Role:** Set; defines restaurant coordinates and max distance.
- **Values:** `restaurantLat=50.4504`, `restaurantLng=30.5245`, `maxServingDistance=100`
- **Note:** Prompt claims 5km radius; node uses 100km. Align these for consistency.

#### Node: Check Eligibility
- **Type / Role:** Code; parses geocode response and calculates distance.
- **Outputs:**
  - If not found: `{valid:false, error:"Address not found", message:"..." }`
  - If found: `{valid:true, withinRange, formattedAddress, distance, coordinates, message}`
- **Failure modes:** Google response structure changes; missing fields.

#### Node: Message1
- **Type / Role:** Set; `Message = {{$json.message}}`

**Sticky notes:**  
- “## Validate Address Tool …”  
- “### Geocode Address …”  
- “### Set Data …”  
- “### Check Eligibility …”

---

### Block 2.13 — “Create Takeaway Order” Tool Implementation (in this JSON)
**Overview:** Converts pickup time to UTC, generates an 8-digit order id, creates an Airtable order record (Takeaway), and provides placeholder SMS step.  
**Nodes involved:** `Trigger Node5`, `Set Timezone3`, `Convert Time to UTC & Create Order ID`, `Create Order`, `Replace with SMS node1`

#### Node: Trigger Node5
- **Type / Role:** Set placeholder structure: `name, phone, items, pickUpTime, priceTotal, notes`.
- **Edge case:** Placeholder defines `items` as string.

#### Node: Set Timezone3
- **Type / Role:** Set timezone.

#### Node: Convert Time to UTC & Create Order ID
- **Type / Role:** Code
- **Logic:**
  - converts `pickUpTime` to UTC ISO
  - `orderId = random 8-digit number`
- **Edge cases:** Random collisions possible; consider checking uniqueness in Airtable.

#### Node: Create Order
- **Type / Role:** Airtable create in `Orders`.
- **Mapped fields:**
  - `Order Type = "Takeaway"`
  - `Status = "🟡 Confirming"`
  - `Items = JSON.stringify($json.items)`
  - `Total = {{$json.priceTotal}}` (Airtable expects number; tool schema uses string)
  - `Pickup/Delivery Time = {{$json.pickUpTime}}`
- **Failure modes:** Type mismatch for Total, invalid date.

#### Node: Replace with SMS node1
- **Type / Role:** NoOp placeholder.

**Sticky notes:**  
- “## Create a Takeaway Order …”  
- “### Set Timezone & Calculate UTC …”  
- “### Create an Order & Send SMS …”

---

### Block 2.14 — “Create Delivery Order” Tool Implementation (in this JSON)
**Overview:** Converts delivery time to UTC, generates an order id, creates an Airtable order record (Delivery), and provides placeholder SMS step.  
**Nodes involved:** `Trigger Node8`, `Set Timezone4`, `Convert Time to UTC & Create Order ID1`, `Create a Delivery Order`, `Replace with SMS node2`

#### Node: Trigger Node8
- **Type / Role:** Set placeholder: `name, phone, items, deliveryTime, priceTotal, notes, deliveryAddress`.

#### Node: Set Timezone4
- **Type / Role:** Set timezone.

#### Node: Convert Time to UTC & Create Order ID1
- **Type / Role:** Code; converts `deliveryTime` and generates `orderId`.

#### Node: Create a Delivery Order
- **Type / Role:** Airtable create in `Orders`
- **Fields:** `Order Type=Delivery`, includes `Delivery Address`, `Pickup/Delivery Time`
- **Failure modes:** Same type mismatch risks as takeaway.

#### Node: Replace with SMS node2
- **Type / Role:** NoOp placeholder.

**Sticky notes:**  
- “## Create a Delivery Order …”  
- “### Set Timezone & Calculate UTC …”  
- “### Create an Order & Send SMS …”

---

### Block 2.15 — “Get an Order” Tool Implementation (in this JSON)
**Overview:** Searches the Orders table by phone number.  
**Nodes involved:** `Trigger Node9`, `Search Orders by Phone`

#### Node: Trigger Node9
- **Type / Role:** Set placeholder `phone` (but typed as array in placeholder).

#### Node: Search Orders by Phone
- **Type / Role:** Airtable search in `Orders`
- **Filter:** `={Phone} = "<phone>"`
- **Failure modes:** Phone format mismatch.

**Sticky notes:**  
- “## Get an Order Tool …”  
- “### Get Orders …”

---

### Block 2.16 — “Cancel Order” Tool Implementation (in this JSON)
**Overview:** Deletes an Airtable order record by id; placeholder for SMS.  
**Nodes involved:** `Trigger Node10`, `Delete Order`, `Replace with SMS node3`

#### Node: Trigger Node10
- **Type / Role:** Set placeholder `orderID` (typed array in placeholder).

#### Node: Delete Order
- **Type / Role:** Airtable deleteRecord in `Orders`
- **Config:** `id = {{$json.orderID}}`
- **Important:** Airtable deleteRecord needs the **Airtable record id**, not a custom “Order ID” field. The tool wrapper `Call 'Cancel Order'` also passes `orderID` as string; ensure you’re passing the Airtable record id, or implement lookup by “Order ID” first.

#### Node: Replace with SMS node3
- **Type / Role:** NoOp placeholder.

**Sticky notes:**  
- “## Cancel Order Tool …”  
- “### Cancel the Order …”

---

## 3. Summary Table (All Nodes)

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Wait for New Message | telegramTrigger | Telegram entry point | — | Check if Text | # Telegram Booking Bot / ### Check / ### Transcribe |
| Check if Text | if | Route text vs voice | Wait for New Message | Merge, Get Voice | ### Check — Check the message type. |
| Get Voice | telegram | Download voice file | Check if Text | Transcribe The Voice | ### Transcribe — If message is voice… |
| Transcribe The Voice | openAi (audio transcribe) | Speech-to-text | Get Voice | Send Transcribe Context To AI Agent | ### Transcribe — If message is voice… |
| Send Transcribe Context To AI Agent | set | Map transcription to message.text | Transcribe The Voice | Merge | ### Transcribe — If message is voice… |
| Merge | merge | Unify text + transcribed voice | Check if Text, Send Transcribe Context To AI Agent | AI Agent |  |
| AI Agent | langchain.agent | Conversational orchestration + tool calling | Merge | Reply | ### AI Agent — Brain of the system… |
| OpenAI Chat Model | lmChatOpenAi | LLM backend | — | AI Agent | ### AI Agent — Brain of the system… |
| Simple Memory | memoryBufferWindow | Per-chat memory | — | AI Agent (ai_memory) | ### AI Agent — Brain of the system… |
| Restaurant Tool | mcpClientTool | MCP tool gateway for agent | — | AI Agent (ai_tool) | # MCP Server / Gives access to all tools |
| Reply | telegram | Send response to Telegram | AI Agent | — | ### Reply — This node sends agent’s response… |
| MCP Server Trigger | mcpTrigger | MCP tool server entry point | Tool workflow wrappers | — | # MCP Server / Gives access to all tools |
| Call "Check Availability" | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Check Availabilty Tool |
| Call "Book Table" | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Book Table Tool |
| Call "Get a Booking" | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Get a Booking Tool |
| Call "Update appointment" | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Update Booking Tool |
| Call "Delete Appointment" | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Cancel Booking Tool |
| Call 'Get Menu' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Get a Menu Tool |
| Call 'Calculate the Price' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Calculate the Price Tool |
| Call 'Validate the Address' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Validate Address Tool |
| Call 'Create a Takeaway Order' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Create a Takeaway Order |
| Call 'Create a Delivery Order' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Create a Delivery Order |
| Call 'Get an Order' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Get an Order Tool |
| Call 'Cancel Order' | toolWorkflow | Tool wrapper → sub-workflow | AI Agent via MCP | MCP Server Trigger | ## Cancel Order Tool |
| When Executed by Another Workflow | executeWorkflowTrigger | Tool sub-workflow trigger (availability) | — | Set Timezone | ## Check Availabilty Tool |
| Set Timezone | set | Define timezone | When Executed by Another Workflow | Convert Time to UTC | ### Set Timezone & Calculate UTC |
| Convert Time to UTC | code | Convert booking window to UTC | Set Timezone | Get All Tables | ### Set Timezone & Calculate UTC |
| Get All Tables | airtable (search) | Load table inventory | Convert Time to UTC | Check Max Capacity | ### Get All Tables & Check Capacity |
| Check Max Capacity | code | Capacity feasibility | Get All Tables | If1 | ### Get All Tables & Check Capacity |
| If1 | if | Branch: canHandle | Check Max Capacity | Get Availability / Can't Handle Party | ### Branching — If we can't fit the party… |
| Can't Handle Party | set | Return capacity message | If1 | — | ### Branching — If we can't fit the party… |
| Get Availability | airtable (search) | Find overlapping bookings | If1 | Table-Availability Selector | ### Get Availability & Sort |
| Table-Availability Selector | code | Compute candidate tables | Get Availability | If | ### Get Availability & Sort |
| If | if | Branch: availability | Table-Availability Selector | Normalize Output / Table is not available | ### Branching — If no tables available… |
| Normalize Output | code | Output normalized availability | If | — | ### Branching — If no tables available… |
| Table is not available | set | Return unavailability message | If | — | ### Branching — If no tables available… |
| Trigger Node | set | Placeholder inputs (book) | — | Set Timezone1 | ## Book Table Tool |
| Set Timezone1 | set | Define timezone | Trigger Node | Calculate Timezone | ### Set Timezone & Calculate UTC |
| Calculate Timezone | code | Convert booking times to UTC | Set Timezone1 | Create a Booking | ### Set Timezone & Calculate UTC |
| Create a Booking | airtable (create) | Create booking record | Calculate Timezone | Replace with SMS node | ### Create a Booking & Send SMS |
| Replace with SMS node | noOp | Placeholder notification | Create a Booking | — | ### Create a Booking & Send SMS |
| Trigger Node1 | set | Placeholder phone (get booking) | — | Find a Booking | ## Get a Booking Tool |
| Find a Booking | airtable (search) | Lookup booking by phone | Trigger Node1 | If2 | ### Find the Booking |
| If2 | if | Branch found vs not | Find a Booking | (none) / Message | ### Branching — If booking is found… |
| Message | set | No bookings message | If2 | — | ### Branching — If booking is found… |
| Trigger Node2 | set | Placeholder update payload | — | Set Timezone2 | ## Update Booking Tool |
| Set Timezone2 | set | Define timezone | Trigger Node2 | Convert Time to UTC1 | ### Set Timezone & Calculate UTC |
| Convert Time to UTC1 | code | Convert updated dates to UTC | Set Timezone2 | Update Booking | ### Set Timezone & Calculate UTC |
| Update Booking | airtable (update) | Update booking by id | Convert Time to UTC1 | Replace with SMS Node | ### Update a Booking & Send SMS |
| Replace with SMS Node | noOp | Placeholder notification | Update Booking | — | ### Update a Booking & Send SMS |
| Trigger Node3 | set | Placeholder booking id | — | Delete Booking | ## Cancel Booking Tool |
| Delete Booking | airtable (deleteRecord) | Delete booking by id | Trigger Node3 | — | ### Cancel the Booking |
| Trigger Node4 | set | Placeholder (get menu) | — | Get Menu | ## Get a Menu Tool |
| Get Menu | airtable (search) | Fetch menu | Trigger Node4 | — | ### Get Menu |
| Trigger Node6 | set | Placeholder items (price) | — | Fetch Menu | ## Calculate the Price Tool |
| Fetch Menu | airtable (search) | Fetch available menu items | Trigger Node6 | Order Validation | ### Get the Menu & Calculate the Price |
| Order Validation | code | Validate items, compute total | Fetch Menu | If3 | ### Get the Menu & Calculate the Price |
| If3 | if | Branch on status==false | Order Validation | Total Price | ### Branching — If some items… |
| Total Price | set | Format total price message | If3 | — | ### Branching — If some items… |
| Trigger Node7 | set | Placeholder address | — | Geocode Address | ## Validate Address Tool |
| Geocode Address | httpRequest | Google geocoding | Trigger Node7 | Set Location and Area | ### Geocode Address |
| Set Location and Area | set | Restaurant coords + radius | Geocode Address | Check Eligibility | ### Set Data |
| Check Eligibility | code | Distance check | Set Location and Area | Message1 | ### Check Eligibility |
| Message1 | set | Output message | Check Eligibility | — | ### Check Eligibility |
| Trigger Node5 | set | Placeholder takeaway order | — | Set Timezone3 | ## Create a Takeaway Order |
| Set Timezone3 | set | Define timezone | Trigger Node5 | Convert Time to UTC & Create Order ID | ### Set Timezone & Calculate UTC |
| Convert Time to UTC & Create Order ID | code | Convert pickup time + orderId | Set Timezone3 | Create Order | ### Set Timezone & Calculate UTC |
| Create Order | airtable (create) | Create takeaway order record | Convert Time to UTC & Create Order ID | Replace with SMS node1 | ### Create an Order & Send SMS |
| Replace with SMS node1 | noOp | Placeholder notification | Create Order | — | ### Create an Order & Send SMS |
| Trigger Node8 | set | Placeholder delivery order | — | Set Timezone4 | ## Create a Delivery Order |
| Set Timezone4 | set | Define timezone | Trigger Node8 | Convert Time to UTC & Create Order ID1 | ### Set Timezone & Calculate UTC |
| Convert Time to UTC & Create Order ID1 | code | Convert delivery time + orderId | Set Timezone4 | Create a Delivery Order | ### Set Timezone & Calculate UTC |
| Create a Delivery Order | airtable (create) | Create delivery order record | Convert Time to UTC & Create Order ID1 | Replace with SMS node2 | ### Create an Order & Send SMS |
| Replace with SMS node2 | noOp | Placeholder notification | Create a Delivery Order | — | ### Create an Order & Send SMS |
| Trigger Node9 | set | Placeholder phone (get order) | — | Search Orders by Phone | ## Get an Order Tool |
| Search Orders by Phone | airtable (search) | Lookup orders by phone | Trigger Node9 | — | ### Get Orders |
| Trigger Node10 | set | Placeholder orderID | — | Delete Order | ## Cancel Order Tool |
| Delete Order | airtable (deleteRecord) | Delete order by record id | Trigger Node10 | Replace with SMS node3 | ### Cancel the Order |
| Replace with SMS node3 | noOp | Placeholder notification | Delete Order | — | ### Cancel the Order |
| Replace with SMS node1/2/3/Node | noOp | Notification placeholders | Various | — | As above |
| Additional Trigger Nodes (52d601f6, 0499bfe5, 441c5746, 56534c8b, 558267aa) | set | Scaffolding/test harness | — | Various Airtable/code nodes | Covered by respective tool sticky notes |

---

## 4. Reproducing the Workflow from Scratch (Step-by-Step)

1. **Create Telegram bot + credentials**
   1) Create a Telegram bot via BotFather, obtain token.  
   2) In n8n, create **Telegram API** credentials with that token.

2. **Create OpenAI credentials**
   - Add **OpenAI API** credential (API key) for both:
     - Chat model (gpt-4.1)
     - Audio transcription

3. **Prepare Airtable**
   1) Copy the Airtable base (see link in Notes section).  
   2) Create Airtable personal access token and n8n **Airtable Token API** credentials.  
   3) Confirm tables and fields exist:
      - **Tables List** with fields including `Capacity`, `Table ID`
      - **Bookings** with `Phone`, `Start Date`, `End Date`, link to `Tables List`, and a rollup/lookup like `Table ID (from Tables List)`
      - **Menu** with `Item Name`, `Price`, `Available`
      - **Orders** with `Order ID`, `Name`, `Phone`, `Order Type`, `Items`, `Total`, `Pickup/Delivery Time`, `Delivery Address`, `Status`, `Notes`

4. **Create the Telegram intake flow**
   1) Add **Telegram Trigger** node named `Wait for New Message` (updates: message).  
   2) Add **IF** node `Check if Text` with condition `message.text exists`.  
   3) True output → `Merge` node (create later).  
   4) False output → **Telegram** node `Get Voice` (resource: file, fileId from `message.voice.file_id`).  
   5) Add **OpenAI Audio** node `Transcribe The Voice` (operation: transcribe), connect from `Get Voice`.  
   6) Add **Set** node `Send Transcribe Context To AI Agent`:
      - set `message.text = {{$json.text}}`
      - set `sessionId = {{$json.message.chat.id}}` (recommended fix; do not reference a non-existent node name)
   7) Add **Merge** node `Merge` and connect:
      - input 1 from `Check if Text` (true)
      - input 2 from `Send Transcribe Context To AI Agent`

5. **Create the AI agent block**
   1) Add **OpenAI Chat Model** node `OpenAI Chat Model` selecting `gpt-4.1`.  
   2) Add **Memory Buffer Window** node `Simple Memory`:
      - `sessionIdType = customKey`
      - `sessionKey = {{$('Merge').item.json.message.chat.id}}`
      - `contextWindowLength = 10`
   3) Add **MCP Client Tool** node `Restaurant Tool`:
      - `endpointUrl = https://<your-n8n-host>/mcp/<path>`
   4) Add **AI Agent** node `AI Agent`:
      - Prompt text uses `{{$json.message.text}}` and `$now`
      - Paste the provided system message (conversation flow + guardrails)
      - Connect `OpenAI Chat Model` as `ai_languageModel`
      - Connect `Simple Memory` as `ai_memory`
      - Connect `Restaurant Tool` as `ai_tool`
   5) Add **Telegram** node `Reply`:
      - `chatId = {{$('Merge').item.json.message.chat.id}}`
      - `text = {{$json.output}}`
   6) Connect `Merge → AI Agent → Reply`.

6. **Create MCP server tool hub**
   1) Add **MCP Server Trigger** node `MCP Server Trigger` with a chosen `path`.  
   2) Ensure your MCP Client Tool points to the correct MCP URL (matching this path).

7. **Create sub-workflows for each tool (recommended approach)**
   Create separate workflows (to match wrapper expectations), each starting with **Execute Workflow Trigger** defining inputs. Implement logic as in blocks above:
   - Check Table Availability
   - Book a Table
   - Get a Booking
   - Update a Booking
   - Delete Booking
   - Get Menu
   - Calculate the Price
   - Validate the Address
   - Create a Takeaway Order
   - Create a Delivery Order
   - Get an Order
   - Cancel Order

8. **Register tool wrappers in the main workflow**
   For each sub-workflow:
   1) Add a **Tool Workflow** node (LangChain) like `Call "Check Availability"`.  
   2) Select the target workflow by name/id.  
   3) Define its input schema and map fields using `$fromAI(...)`.  
   4) Connect each tool wrapper’s `ai_tool` output to **MCP Server Trigger**.

9. **Configure Google Maps geocoding**
   - In the Validate Address sub-workflow:
     - Set your API key and region
     - Set restaurant coordinates and max distance consistent with your agent prompt (e.g., 5km)

10. **Replace NoOp “SMS” placeholders**
   - Swap `Replace with SMS node*` nodes with real integrations (Twilio, WhatsApp, email, etc.) using booking/order details from upstream nodes.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Download JSON files | https://www.dropbox.com/scl/fo/r6edgytt26f51vmibntqt/AM49duT1dDbdD2deCu2-J5c?rlkey=l495lzd5926yb5mbxxuj73a39&st=wgjc60dp&dl=0 |
| Copy Airtable base | https://airtable.com/appO25aREhJMGVavR/shreEtStEqOedoYTT |
| Voice agent option via Vapi | Copy AI Agent prompt into Vapi and add MCP tool in Vapi tools tab |
| Replace “Do Nothing” nodes | Those are placeholders for SMS/notifications after booking/order creation/update/cancel |
| Timezone | Many code nodes assume `Europe/Kyiv` and compute UTC offset; adjust as needed |
| Consistency warnings | Delivery radius in prompt says 5km but node uses 100km; align them |

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.