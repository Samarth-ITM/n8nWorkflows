Classify and route email attachments with easybits, Gmail and Google Drive

https://n8nworkflows.xyz/workflows/classify-and-route-email-attachments-with-easybits--gmail-and-google-drive-14978


# Classify and route email attachments with easybits, Gmail and Google Drive

We need to produce a comprehensive, structured reference document covering this n8n workflow. We must not skip any nodes, not include raw JSON (except minimal examples), avoid words "tutorial", "cheatsheet", "README". Use fluent, professional English. Include sections:

1. Workflow Overview
2. Block-by-Block Analysis (with overview, nodes involved, node details for each node in block)
3. Summary Table
4. Reproducing the Workflow from Scratch (step-by-step)
5. General Notes & Resources

We must analyze entire JSON. There are many nodes: 

- "Gmail Trigger"
- "easybits: Classify Document"
- "Route by Document Type"
- "Merge: Invoice Binary"
- "Merge: Contract Binary"
- "Merge: Purchase Order Binary"
- "easybits: Extract Invoice"
- "easybits: Extract Contract"
- "Extractor: Purchase Orders"
- "Merge: Invoice Data + File"
- "Merge: Contract Data + File"
- "Merge: Purchase Order Data + File"
- "Upload Invoice to Drive"
- "Upload Contract to Drive"
- "Upload Purchase Order to Drive"
- "Update Master Finance Sheet"
- "Slack: Notify Sales – New Contract"
- "Slack: Notify Team – Restock Update"

Also sticky notes: "Sticky Note" through "Sticky Note8". The sticky notes should be included in summary table for nodes they cover.

The sticky notes:

- Sticky Note: "## 📧 Email Intake Polls Gmail every minute for new emails. Downloads attachments automatically. The binary file is branched to all merge nodes so the original document stays available throughout the workflow." This covers Gmail Trigger and maybe the merge nodes? Actually covers Gmail Trigger and connections. The sticky note position maybe covers Gmail Trigger and possibly other nodes, but we consider it covers the Gmail Trigger node.

- Sticky Note1: "## 🏷️ Document Classification Sends the email attachment to **easybits Extractor** configured for classification. Returns a `document_class` field with one of three values: `Invoice`, `Contracts`, or `Purchase Orders`. The Switch node routes the item down the matching path." This covers "easybits: Classify Document" and "Route by Document Type" maybe.

- Sticky Note2: "## 🔀 Route by Document Type Reads the `document_class` returned by the classification step and routes the item to the matching path: **Invoice**, **Contract**, or **Purchase Order**. Each output connects to its own extraction and delivery pipeline." This covers "Route by Document Type" node.

- Sticky Note3: "## 🔗 Merge: Restore Original File The classification step strips the binary from the item. These Merge nodes reunite the original email attachment (from the Gmail Trigger) with the routed item so the Extractor receives both the document class and the actual file." This covers the three "Merge: Invoice Binary", "Merge: Contract Binary", "Merge: Purchase Order Binary" nodes.

- Sticky Note4: "## 🤖 Document-Specific Extraction Each route has its own **easybits Extractor** pipeline configured for that document type. Invoices extract vendor, amount, due date. Contracts extract client, value, notice period. Purchase Orders extract supplier, items, delivery date." This covers "easybits: Extract Invoice", "easybits: Extract Contract", "Extractor: Purchase Orders" nodes.

- Sticky Note5: "## 🔗 Merge: Combine Extracted Data + File The Extractor returns structured JSON but strips the binary again. These Merge nodes recombine the extracted data with the original file so the Google Drive upload node has both the file to store and the JSON to pass downstream." This covers "Merge: Invoice Data + File", "Merge: Contract Data + File", "Merge: Purchase Order Data + File".

- Sticky Note6: "## ☁️ Upload to Google Drive Stores the original document in a dedicated folder per type: **Invoices**, **Contracts**, or **Purchase Orders**. The Drive link is passed to the next node for reference in Sheets or Slack." This covers "Upload Invoice to Drive", "Upload Contract to Drive", "Upload Purchase Order to Drive".

- Sticky Note7: "## 📤 Notify & Log Each route ends with a different action: **Invoices** are appended to the Master Finance Sheet. **Contracts** trigger a Slack message to Mike with key terms and a Drive link. **Purchase Orders** post a restock update to the team channel." This covers "Update Master Finance Sheet", "Slack: Notify Sales – New Contract", "Slack: Notify Team – Restock Update".

- Sticky Note8: "## 📬 Smart Mailroom: Auto-Classify & Route Documents with easybits ... Setup Guide..." This covers entire workflow, but not tied to specific nodes. We'll include its content in "General Notes & Resources".

We must create a table summary of nodes with columns: Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note (content). For nodes covered by sticky note(s), include note content. Possibly multiple sticky notes for a node, we can combine them or list both. For nodes not covered by sticky note, leave blank.

Connections from JSON: Let's parse connections:

"connections": {
  "Gmail Trigger": {
    "main": [[
      {"node":"easybits: Classify Document","type":"main","index":0},
      {"node":"Merge: Invoice Binary","type":"main","index":0},
      {"node":"Merge: Invoice Data + File","type":"main","index":0},
      {"node":"Merge: Contract Binary","type":"main","index":0},
      {"node":"Merge: Purchase Order Binary","type":"main","index":1},
      {"node":"Merge: Contract Data + File","type":"main","index":0},
      {"node":"Merge: Purchase Order Data + File","type":"main","index":1}
    ]]
  },
  "Merge: Invoice Binary": {
    "main": [[ {"node":"easybits: Extract Invoice","type":"main","index":0} ]]
  },
  "Merge: Contract Binary": {
    "main": [[ {"node":"easybits: Extract Contract","type":"main","index":0} ]]
  },
  "Route by Document Type": {
    "main": [
      [ {"node":"Merge: Invoice Binary","type":"main","index":1} ], // output 0 (Invoice)
      [ {"node":"Merge: Contract Binary","type":"main","index":1} ], // output 1 (Contract)
      [ {"node":"Merge: Purchase Order Binary","type":"main","index":0} ] // output 2 (PO)
    ]
  },
  "Upload Invoice to Drive": {
    "main": [[ {"node":"Update Master Finance Sheet","type":"main","index":0} ]]
  },
  "Upload Contract to Drive": {
    "main": [[ {"node":"Slack: Notify Sales – New Contract","type":"main","index":0} ]]
  },
  "easybits: Extract Invoice": {
    "main": [[ {"node":"Merge: Invoice Data + File","type":"main","index":1} ]]
  },
  "Extractor: Purchase Orders": {
    "main": [[ {"node":"Merge: Purchase Order Data + File","type":"main","index":0} ]]
  },
  "Merge: Invoice Data + File": {
    "main": [[ {"node":"Upload Invoice to Drive","type":"main","index":0} ]]
  },
  "easybits: Extract Contract": {
    "main": [[ {"node":"Merge: Contract Data + File","type":"main","index":1} ]]
  },
  "Merge: Contract Data + File": {
    "main": [[ {"node":"Upload Contract to Drive","type":"main","index":0} ]]
  },
  "easybits: Classify Document": {
    "main": [[ {"node":"Route by Document Type","type":"main","index":0} ]]
  },
  "Merge: Purchase Order Binary": {
    "main": [[ {"node":"Extractor: Purchase Orders","type":"main","index":0} ]]
  },
  "Upload Purchase Order to Drive": {
    "main": [[ {"node":"Slack: Notify Team – Restock Update","type":"main","index":0} ]]
  },
  "Merge: Purchase Order Data + File": {
    "main": [[ {"node":"Upload Purchase Order to Drive","type":"main","index":0} ]]
  }
}

Additionally, maybe missing connections for "Slack: Notify Sales – New Contract" and "Slack: Notify Team – Restock Update"? Not listed but probably they are final nodes (no outgoing connections). Similarly "Update Master Finance Sheet" final.

Now, for each node details: We need to describe type, version, configuration choices, key expressions or variables, input and output connections, edge cases.

We'll describe each node in the block analysis.

The workflow comprises logical blocks:

- Email Intake (Gmail Trigger)
- Document Classification (easybits: Classify Document)
- Routing (Route by Document Type)
- Merge for Binary (three Merge nodes)
- Document-specific Extraction (three easybits Extractor nodes)
- Merge for Data + File (three Merge nodes)
- Upload to Drive (three Google Drive Upload nodes)
- Action per route (Google Sheets Update for Invoice, Slack notifications for Contract and PO)

We may combine in block analysis grouping similar nodes together.

Need to provide a step-by-step reproduction instructions, referencing required credentials, pipeline IDs, folder IDs, channel IDs.

General notes: Provide link to easybits (extractor.easybits.tech). Provide the description from sticky note 8.

Alright.

Let's think about all node details.

**Gmail Trigger** (node id: 236c284d-88ac-42d7-bb83-880796b11c8a)

- Type: n8n-nodes-base.gmailTrigger, version 1.3
- Parameters: 
  - simple: false (means detailed output)
  - filters: {} (no filter)
  - options: downloadAttachments: true
  - pollTimes: item: [{ mode: "everyMinute" }] (so polling every minute)
- Input: None (trigger)
- Output: Connects to "easybits: Classify Document" (index 0) and multiple Merge nodes (invoice binary, invoice data+file, contract binary, purchase order binary, contract data+file, purchase order data+file). Actually based on connections, the Gmail Trigger output goes to multiple nodes: easybits: Classify Document, Merge: Invoice Binary, Merge: Invoice Data + File, Merge: Contract Binary, Merge: Purchase Order Binary (index 1), Merge: Contract Data + File, Merge: Purchase Order Data + File. That's important: the Gmail Trigger splits the binary and email data to many nodes.

- Edge Cases: If email has multiple attachments, it may create multiple items; must ensure the binary fields names: attachment_0 etc. Also, if no attachment, the downloadAttachments true yields no binary.

- Version: 1.3.

**easybits: Classify Document** (id: 9456f8f9-d8e9-4f44-9f18-7a9c760c997f)

- Type: @easybits/n8n-nodes-extractor.easybitsExtractor, version 2.
- Parameters: {} (empty) – meaning the node relies on credentials and pipeline ID set up in node config. In n8n easybits node, you need to provide pipeline ID and credentials. The node sends the incoming binary to the extraction pipeline and returns JSON with data fields, specifically a "document_class" field.

- Input: Gmail Trigger.
- Output: Route by Document Type.

- Edge Cases: If the pipeline returns unexpected class, Switch may route to fallback. Should consider fallback path.

**Route by Document Type** (Switch node)

- Type: n8n-nodes-base.switch, version 3.4.
- Parameters:
  - rules: values (list):
    - Rule 0: condition: leftValue "={{ $json.data.document_class }}" equals "Invoice"
    - Rule 1: leftValue equals "Contracts"
    - Rule 2: leftValue equals "Purchase Orders"
  - options: {} default.

- Input: easybits: Classify Document.
- Outputs: three outputs:
  - Output 0 (Invoice) connects to Merge: Invoice Binary (index 1) – Wait: The connection indicates output 0 goes to Merge: Invoice Binary with index 1 (so second input of Merge? Actually Merge node v3.2 has two inputs; input 0 is the left, input 1 is the right). So we need to detail that the output goes to input 1 of Merge Invoice Binary. Similarly output 1 to input 1 of Merge Contract Binary, output 2 to input 0 of Merge Purchase Order Binary (since index 0).
  
- Edge Cases: If classification returns any other string or empty, it goes to fallback path? No fallback defined, so item will be ignored or cause error.

**Merge: Invoice Binary** (n8n-nodes-base.merge, v3.2)

- Input 0: from Gmail Trigger (binary + email data)
- Input 1: from Route by Document Type (Invoice class)
- Mode: default "combine" (I think merge v3.2 default mode is "append"? Actually v3.2 has mode options: "append" "combine" "choose". Default is "append" if unspecified. The node is not specifying mode, so default "append"? In earlier versions default is "append". We'll note default "append" (concatenates items). Actually we need to check: n8n merge v3.2 default mode is "append". So the node will keep the data from both inputs; but typically we want to combine the binary with classification data. However we also have Merge: Invoice Data + File later which uses "combine" with combineByPosition. So this Merge node likely just merges the binary with classification data. The details: no parameters.

- Output: "easybits: Extract Invoice".

**Merge: Contract Binary** similarly, Merge: Purchase Order Binary similarly.

**easybits: Extract Invoice** (Extractor node)

- Input: Merge: Invoice Binary.
- Output: Merge: Invoice Data + File (index 1).

- Note: Node type version 2, no parameters; expects pipeline ID for Invoice fields extraction.

**easybits: Extract Contract** similarly.

**Extractor: Purchase Orders** (note different naming but same node type)

- Input: Merge: Purchase Order Binary.
- Output: Merge: Purchase Order Data + File (index 0). Wait: It connects to index 0 of Merge: Purchase Order Data + File. We need to check: The connection says "Extractor: Purchase Orders" main -> [ {"node":"Merge: Purchase Order Data + File","type":"main","index":0} ] (so goes to input 0 of the merge node). But the merge node expects two inputs: input 0 from Gmail? Actually the Merge: Purchase Order Data + File connections: "Gmail Trigger" goes to index 1 (for Purchase Order Data + File). Wait check connections: Under "Gmail Trigger", "Merge: Purchase Order Data + File" appears as index 1. So we have:

- Merge: Purchase Order Data + File receives input 0 from Extractor: Purchase Orders, and input 1 from Gmail Trigger. Good.

So each Merge: Data + File gets: Input 0 from Extractor, Input 1 from Gmail Trigger.

**Merge: Invoice Data + File** (merge node)

- Mode: "combine" (explicitly set), combineByPosition (default combine mode).
- Input 0: Gmail Trigger? Actually from connections: "Merge: Invoice Data + File" appears as index 0 from Gmail Trigger. So input 0 from Gmail Trigger (original email data), input 1 from easybits: Extract Invoice. So combine mode will merge fields by position: combine fields from input 0 and input 1 into a single item.

- Output: Upload Invoice to Drive.

**Merge: Contract Data + File** similarly (Gmail Trigger input 0, Extract Contract input 1).

**Merge: Purchase Order Data + File** similarly (Gmail Trigger input 1, Extractor: Purchase Orders input 0). Wait careful: In connections, Gmail Trigger sends to Merge: Purchase Order Data + File index 1. So Merge: Purchase Order Data + File input 0 from Extractor: Purchase Orders (extracted data), input 1 from Gmail Trigger (original data). The combine mode uses combineByPosition, merging items by index.

**Upload Invoice to Drive** (Google Drive node)

- Type: n8n-nodes-base.googleDrive, version 3.
- Parameters:
  - driveId: __rl: true, mode: list, value: "My Drive"
  - folderId: __rl: true, mode: list, value: "YOUR_FOLDER_ID" (placeholder)
  - inputDataFieldName: "attachment_0"
  - options: {} (no other options)
- Input: Merge: Invoice Data + File.
- Output: Update Master Finance Sheet.

**Upload Contract to Drive** (Google Drive node)

- Parameters:
  - driveId: "My Drive"
  - folderId: "root" (root folder)
  - inputDataFieldName: "attachment_0"
- Input: Merge: Contract Data + File.
- Output: Slack: Notify Sales – New Contract.

**Upload Purchase Order to Drive** (Google Drive node)

- Parameters similar to Contract: folderId: "root"
- Input: Merge: Purchase Order Data + File.
- Output: Slack: Notify Team – Restock Update.

**Update Master Finance Sheet** (Google Sheets node)

- Type: n8n-nodes-base.googleSheets, version 4.7.
- Parameters:
  - operation: "appendOrUpdate"
  - sheetName: list, value: "gid=0" (default Sheet1)
  - documentId: list, value: "YOUR_SPREADSHEET_ID"
  - columns: mappingMode: "defineBelow"
    - column values:
      - Invoice Number: ={{ $('Merge: Invoice Data + File').item.json.data.invoice_number }}
      - Final Amount (EUR): ={{ $('Merge: Invoice Data + File').item.json.data.total_amount }}
    - MatchingColumns: ["Invoice Number"]
  - Additional schema fields defined but removed: "Original Amount", "Currency", "Exchange Rate"
- Input: Upload Invoice to Drive.
- Output: None (final node for this route).

**Slack: Notify Sales – New Contract** (Slack node)

- Type: n8n-nodes-base.slack, version 2.4.
- Parameters:
  - select: "channel"
  - messageType: "block"
  - blocksUi: formatted message with fields referencing extracted contract data from Merge: Contract Data + File, and a link to uploaded file via Upload Contract to Drive. Contains placeholders: $channel id etc.
  - channelId: placeholder "YOUR_CHANNEL_ID"
- Input: Upload Contract to Drive.
- Output: None.

**Slack: Notify Team – Restock Update** similarly.

Now we need to create Block-by-Block Analysis. We'll group nodes into logical blocks:

- Block 1: Email Intake (Gmail Trigger)
- Block 2: Document Classification (easybits: Classify Document)
- Block 3: Routing (Route by Document Type)
- Block 4: Binary Restoration (three Merge nodes)
- Block 5: Document-Specific Extraction (three Extractors)
- Block 6: Data & File Combination (three Merge nodes)
- Block 7: Upload to Google Drive (three Google Drive nodes)
- Block 8: Action per Document Type (Google Sheets + Slack notifications)

Now detail each block.

**Block 1 – Email Intake**:

- Overview: Polls Gmail every minute for new emails, downloads attachments, and fans out the email payload and binaries to multiple downstream nodes.
- Nodes Involved: Gmail Trigger.
- Node Details: type, version, configuration, connections, edge cases, sticky note (Sticky Note: 📧 Email Intake...).

**Block 2 – Document Classification**:

- Overview: Sends the email attachment to easybits classification pipeline; returns a `document_class` label.
- Nodes: easybits: Classify Document.
- Node details.

**Block 3 – Routing**:

- Overview: Routes the item based on classification to the appropriate path.
- Nodes: Route by Document Type (Switch).
- Node details.

**Block 4 – Binary Restoration**:

- Overview: Merges the original email binary with the routed classification data before extraction.
- Nodes: Merge: Invoice Binary, Merge: Contract Binary, Merge: Purchase Order Binary.
- Node details.

**Block 5 – Document-Specific Extraction**:

- Overview: Each route runs a dedicated easybits pipeline to extract fields.
- Nodes: easybits: Extract Invoice, easybits: Extract Contract, Extractor: Purchase Orders.
- Node details.

**Block 6 – Data & File Combination**:

- Overview: Merges the extraction results with the original file for uploading.
- Nodes: Merge: Invoice Data + File, Merge: Contract Data + File, Merge: Purchase Order Data + File.
- Node details.

**Block 7 – Upload to Google Drive**:

- Overview: Uploads the original document to a dedicated Drive folder per type.
- Nodes: Upload Invoice to Drive, Upload Contract to Drive, Upload Purchase Order to Drive.
- Node details.

**Block 8 – Notify & Log**:

- Overview: For invoices, updates a Google Sheet; for contracts and POs, sends Slack messages.
- Nodes: Update Master Finance Sheet, Slack: Notify Sales – New Contract, Slack: Notify Team – Restock Update.
- Node details.

We also need to note the sticky notes associated with nodes.

Sticky Note assignments:

- Sticky Note covers Gmail Trigger (Node "Gmail Trigger").

- Sticky Note1 covers "easybits: Classify Document" and maybe "Route by Document Type"? Actually sticky note1 title: "🏷️ Document Classification" includes classification step and Switch node. So it covers both classification and routing maybe. We'll assign sticky note content to those nodes.

- Sticky Note2 covers "Route by Document Type".

- Sticky Note3 covers the three Merge Binary nodes: Invoice Binary, Contract Binary, Purchase Order Binary.

- Sticky Note4 covers the three extraction nodes: Extract Invoice, Extract Contract, Extractor: Purchase Orders.

- Sticky Note5 covers three Merge Data+File nodes.

- Sticky Note6 covers the three Upload to Drive nodes.

- Sticky Note7 covers the final action nodes: Update Master Finance Sheet, Slack Notify Sales, Slack Notify Team.

- Sticky Note8 is general high-level overview; we will include its content in General Notes.

We must include the content in the summary table.

Now summarizing all.

**Summary Table**: Node Name, Node Type, Functional Role, Input Node(s), Output Node(s), Sticky Note.

We'll list each node, including sticky note content.

Now Step-by-step reproduction:

1. Create new workflow.
2. Add Gmail Trigger node; configure OAuth2, poll every minute, enable download attachments.
3. Add easybits Classify Document node; add easybits credential and pipeline ID for classification.
4. Connect Gmail Trigger to easybits Classify Document.
5. Add Switch node "Route by Document Type"; configure rules for Invoice, Contracts, Purchase Orders using expression `$json.data.document_class`.
6. Connect easybits Classify Document output to Switch.
7. Add Merge nodes (Invoice Binary, Contract Binary, Purchase Order Binary) - each as Append mode.
8. Connect Gmail Trigger to input 0 of each Merge node; connect Switch output to input 1 of the corresponding Merge nodes.
9. Add easybits extraction nodes for Invoice, Contract, Purchase Orders, with respective pipeline credentials.
10. Connect Merge Binary nodes to respective extraction nodes.
11. Add Merge nodes for Data + File (Invoice, Contract, PO) in Combine mode with Combine By Position.
12. Connect Gmail Trigger to input 0 (Invoice Data+File) and input 0 (Contract Data+File) maybe need to ensure input order. Actually we need to connect Gmail Trigger to Merge Data+File nodes accordingly.
13. Connect Extract nodes to input 1 of Merge Data+File nodes.
14. Add Google Drive Upload nodes for each type, set folder IDs, driveId, inputDataFieldName to "attachment_0".
15. Connect Merge Data+File nodes to Upload nodes.
16. Add Google Sheets node for Invoice: operation appendOrUpdate, set spreadsheet ID, column mapping, etc.
17. Connect Upload Invoice to Drive to Google Sheets node.
18. Add Slack node for Contract: configure block kit message, channel ID, etc.
19. Connect Upload Contract to Drive to Slack node.
20. Add Slack node for PO similarly.
21. Connect Upload Purchase Order to Drive to Slack PO node.
22. Connect Gmail Trigger also directly to Merge Data+File nodes (as per connections) (must be done explicitly).
23. Optionally add any extra connections: Gmail Trigger to Merge: Invoice Data+File, Merge: Contract Data+File, Merge: Purchase Order Data+File.

Need to clarify order of connections: According to JSON, Gmail Trigger connections go to all those merges as described.

Therefore we need to specify that Gmail Trigger must be linked to the following Merge nodes as input 0 for Invoice Data+File and Contract Data+File, and input 1 for Purchase Order Data+File.

Also note for Merge Binary nodes: Gmail Trigger to input 0; Switch to input 1 (or 0 for PO). Actually from Switch connections: Invoice route -> Merge: Invoice Binary index 1, Contract route -> Merge: Contract Binary index 1, PO route -> Merge: Purchase Order Binary index 0. So we must specify accordingly.

Now we need to add notes about credentials:

- easybits: requires API key and pipeline ID per node.
- Gmail: OAuth2.
- Google Drive: OAuth2 for Drive API.
- Google Sheets: OAuth2 for Sheets.
- Slack: OAuth2.

Now let's draft each section.

**Section 1: Workflow Overview**: Provide concise explanation.

**Section 2: Block-by-Block Analysis**: Provide details.

**Section 3: Summary Table**.

**Section 4: Reproducing Workflow from Scratch**: Numbered steps.

**Section 5: General Notes & Resources**: Provide note content from Sticky Note8 and any other resources.

Now we need to format all nicely.

Edge Cases: If classification returns unknown class, workflow may not route. Could add a fallback path. If email has no attachment, downstream merges may have missing binary causing extraction to fail. The workflow depends on attachment name being "attachment_0". Gmail Trigger might generate that. If multiple attachments, need to handle multiple items.

Potential errors: Google Drive folder IDs not set (placeholder). Slack channel ID placeholder. Google Sheets column mapping mismatches.

Now start drafting.

We'll use markdown sections.

We'll need to ensure we don't skip any node, including sticky notes as separate entries? The sticky notes themselves are nodes (type stickyNote). They must be included in the summary table with Functional Role "Annotation" and note content.

Thus we need to list sticky notes too.

We should include sticky note content as per sticky notes:

- Sticky Note: "## 📧 Email Intake\nPolls Gmail every minute for new emails. Downloads attachments automatically. The binary file is branched to all merge nodes so the original document stays available throughout the workflow."

- Sticky Note1: "## 🏷️ Document Classification\nSends the email attachment to **easybits Extractor** configured for classification. Returns a `document_class` field with one of three values: `Invoice`, `Contracts`, or `Purchase Orders`. The Switch node routes the item down the matching path."

- Sticky Note2: "## 🔀 Route by Document Type\nReads the `document_class` returned by the classification step and routes the item to the matching path: **Invoice**, **Contract**, or **Purchase Order**. Each output connects to its own extraction and delivery pipeline."

- Sticky Note3: "## 🔗 Merge: Restore Original File\nThe classification step strips the binary from the item. These Merge nodes reunite the original email attachment (from the Gmail Trigger) with the routed item so the Extractor receives both the document class and the actual file."

- Sticky Note4: "## 🤖 Document-Specific Extraction\nEach route has its own **easybits Extractor** pipeline configured for that document type. Invoices extract vendor, amount, due date. Contracts extract client, value, notice period. Purchase Orders extract supplier, items, delivery date."

- Sticky Note5: "## 🔗 Merge: Combine Extracted Data + File\nThe Extractor returns structured JSON but strips the binary again. These Merge nodes recombine the extracted data with the original file so the Google Drive upload node has both the file to store and the JSON to pass downstream."

- Sticky Note6: "## ☁️ Upload to Google Drive\nStores the original document in a dedicated folder per type: **Invoices**, **Contracts**, or **Purchase Orders**. The Drive link is passed to the next node for reference in Sheets or Slack."

- Sticky Note7: "## 📤 Notify & Log\nEach route ends with a different action: **Invoices** are appended to the Master Finance Sheet. **Contracts** trigger a Slack message to Mike with key terms and a Drive link. **Purchase Orders** post a restock update to the team channel."

- Sticky Note8: Contains extensive description; will be placed in General Notes.

Now include these in the summary table.

Now produce the full answer.

We'll produce each node details carefully.

**Node details**:

For each node, we need to discuss configuration choices, key expressions, input and output connections, version, edge cases.

Let's start writing.

--- 

We'll write in a professional tone.

Let's produce the final answer. 1. Workflow Overview  

**Smart Mailroom Workflow powered by easybits** automatically processes inbound business documents received by email, classifies each attachment, extracts type‑specific data, stores the file in Google Drive, and triggers downstream actions:

1. **Email Intake** – Gmail polls for new messages with attachments every minute and distributes the binary payload to all downstream branches.  
2. **Classification** – The attachment is sent to an easybits classification pipeline that returns a `document_class` label (`Invoice`, `Contracts`, or `Purchase Orders`).  
3. **Routing** – A Switch node directs the item into the appropriate pipeline based on that label.  
4. **Binary Restoration** – Because the classification step strips the binary, three Merge nodes re‑attach the original file to the routed item.  
5. **Document‑Specific Extraction** – Each branch runs a dedicated easybits extraction pipeline to pull structured fields (invoice number, client name, PO number, etc.).  
6. **Data + File Re‑combination** – A second set of Merge nodes fuses the extracted JSON with the original binary so the upload node has both pieces.  
7. **Upload to Google Drive** – The file is stored in a type‑specific Drive folder (Invoices, Contracts, Purchase Orders).  
8. **Notify & Log** – Invoices are appended to a Master Finance Sheet; Contracts and Purchase Orders generate formatted Slack messages that include a Drive link.

The design is linear per branch but uses a fan‑out from the Gmail Trigger to keep the binary available throughout the workflow and a fan‑in after extraction to re‑associate the file with the extracted data.

---

## 2. Block‑by‑Block Analysis  

### 2.1 Email Intake  

| Aspect | Details |
|--------|---------|
| **Overview** | Polls a Gmail inbox for new messages, downloads every attachment, and fans the email payload (metadata + binary) to multiple downstream Merge nodes. |
| **Nodes Involved** | • Gmail Trigger |
| **Node Details – Gmail Trigger** | • **Type:** `n8n-nodes-base.gmailTrigger` (v1.3) <br>• **Configuration:** `simple: false` (returns full message object), `filters: {}` (no label filter), `options.downloadAttachments: true`, polling every minute (`pollTimes.item[0].mode: "everyMinute"`). <br>• **Key Expressions:** None – the node emits items with binary property `attachment_0`. <br>• **Input:** None (trigger). <br>• **Outputs:** <br> - `easybits: Classify Document` (index 0) <br> - `Merge: Invoice Binary` (input 0) <br> - `Merge: Invoice Data + File` (input 0) <br> - `Merge: Contract Binary` (input 0) <br> - `Merge: Purchase Order Binary` (input 1) <br> - `Merge: Contract Data + File` (input 0) <br> - `Merge: Purchase Order Data + File` (input 1) <br>• **Edge Cases:** Emails without attachments produce items with no binary; downstream Merge nodes will receive `null` binary and may cause extraction errors. Multiple attachments create multiple items per email, each processed independently. <br>• **Sticky Note:** *“## 📧 Email Intake – Polls Gmail every minute for new emails. Downloads attachments automatically. The binary file is branched to all merge nodes so the original document stays available throughout the workflow.”* |

---

### 2.2 Document Classification  

| Aspect | Details |
|--------|---------|
| **Overview** | Sends the attachment to an easybits classification pipeline that returns a single‑label `document_class`. |
| **Nodes Involved** | • easybits: Classify Document |
| **Node Details – easybits: Classify Document** | • **Type:** `@easybits/n8n-nodes-extractor.easybitsExtractor` (v2) <br>• **Configuration:** No static parameters; requires an easybits credential and a **Pipeline ID** that defines a single field `document_class` with a prompt instructing the model to output exactly one of `Invoice`, `Contracts`, or `Purchase Orders`. <br>• **Key Expressions:** `{{ $json.data.document_class }}` is consumed by the downstream Switch node. <br>• **Input:** Gmail Trigger. <br>• **Output:** `Route by Document Type`. <br>• **Edge Cases:** If the classification model returns an unexpected label, the Switch node has no fallback branch; the item will be dropped. The node also removes the binary from its output, necessitating the later Merge steps. <br>• **Sticky Note:** *“## 🏷️ Document Classification – Sends the email attachment to **easybits Extractor** configured for classification. Returns a `document_class` field with one of three values: `Invoice`, `Contracts`, or `Purchase Orders`. The Switch node routes the item down the matching path.”* |

---

### 2.3 Routing  

| Aspect | Details |
|--------|---------|
| **Overview** | Reads the `document_class` label and steers the item into the appropriate branch (Invoice, Contract, Purchase Order). |
| **Nodes Involved** | • Route by Document Type |
| **Node Details – Route by Document Type** | • **Type:** `n8n-nodes-base.switch` (v3.4) <br>• **Configuration:** Three rules defined: <br> 1️⃣ `leftValue = {{ $json.data.document_class }}` **equals** `Invoice` → Output 0 <br> 2️⃣ `leftValue = {{ $json.data.document_class }}` **equals** `Contracts` → Output 1 <br> 3️⃣ `leftValue = {{ $json.data.document_class }}` **equals** `Purchase Orders` → Output 2 <br>• **Input:** easybits: Classify Document. <br>• **Outputs:** <br> - Output 0 → `Merge: Invoice Binary` (input 1) <br> - Output 1 → `Merge: Contract Binary` (input 1) <br> - Output 2 → `Merge: Purchase Order Binary` (input 0) <br>• **Edge Cases:** If `document_class` is `null` or a value not listed, the item will not be routed (no fallback). <br>• **Sticky Note:** *“## 🔀 Route by Document Type – Reads the `document_class` returned by the classification step and routes the item to the matching path: **Invoice**, **Contract**, or **Purchase Order**. Each output connects to its own extraction and delivery pipeline.”* |

---

### 2.4 Binary Restoration (Merge Original File)  

| Aspect | Details |
|--------|---------|
| **Overview** | Re‑attaches the original email binary to the routed item, because the classification node discards the binary. |
| **Nodes Involved** | • Merge: Invoice Binary <br>• Merge: Contract Binary <br>• Merge: Purchase Order Binary |
| **Node Details – Merge (all three)** | • **Type:** `n8n-nodes-base.merge` (v3.2) <br>• **Configuration:** Default mode (`append`) – concatenates items from the two inputs while preserving all fields. No explicit `mode` set. <br>• **Input 0:** Gmail Trigger (provides the original `attachment_0` binary). <br>• **Input 1 (Invoice/Contract) or Input 0 (PO):** Output from the Switch node (provides the classification data). <br>• **Output:** Each Merge node forwards its combined item to the corresponding extraction node. <br>• **Edge Cases:** If the Gmail Trigger item has no binary, the merged item will lack the attachment, causing extraction failure. <br>• **Sticky Note:** *“## 🔗 Merge: Restore Original File – The classification step strips the binary from the item. These Merge nodes reunite the original email attachment (from the Gmail Trigger) with the routed item so the Extractor receives both the document class and the actual file.”* |

---

### 2.5 Document‑Specific Extraction  

| Aspect | Details |
|--------|---------|
| **Overview** | Each branch calls a dedicated easybits extraction pipeline that returns structured fields for that document type. |
| **Nodes Involved** | • easybits: Extract Invoice <br>• easybits: Extract Contract <br>• Extractor: Purchase Orders |
| **Node Details – easybits Extraction Nodes** | • **Type:** `@easybits/n8n-nodes-extractor.easybitsExtractor` (v2) for all three. <br>• **Configuration:** No static parameters; each node must be linked to its own **Pipeline ID** and credential that defines the expected fields: <br> - Invoice pipeline → fields `invoice_number`, `total_amount`, `currency`, `due_date`, `vendor_name` <br> - Contract pipeline → fields `client_name`, `contract_type`, `contract_value`, `currency`, `start_date`, `notice_period` <br> - Purchase Order pipeline → fields `supplier_name`, `po_number`, `order_date`, `expected_delivery_date`, `total_amount`, `currency` <br>• **Input:** Respective `Merge: … Binary` node. <br>• **Output:** Corresponding `Merge: … Data + File` node (input 1 for Invoice & Contract, input 0 for PO). <br>• **Edge Cases:** If the pipeline returns missing fields or a different schema, downstream Merge/Sheets/Slack nodes may reference undefined variables. <br>• **Sticky Note:** *“## 🤖 Document‑Specific Extraction – Each route has its own **easybits Extractor** pipeline configured for that document type. Invoices extract vendor, amount, due date. Contracts extract client, value, notice period. Purchase Orders extract supplier, items, delivery date.”* |

---

### 2.6 Data + File Re‑combination  

| Aspect | Details |
|--------|---------|
| **Overview** | Merges the extracted JSON data with the original binary so the upload node receives a single item containing both. |
| **Nodes Involved** | • Merge: Invoice Data + File <br>• Merge: Contract Data + File <br>• Merge: Purchase Order Data + File |
| **Node Details – Merge (Data + File)** | • **Type:** `n8n-nodes-base.merge` (v3.2) <br>• **Configuration:** `mode: "combine"` and `combineBy: "combineByPosition"` (explicit) – joins items from both inputs by position, preserving all fields from each source. <br>• **Input 0:** <br> - Invoice/Contract: Gmail Trigger (original email payload + binary) <br> - Purchase Order: easybits Extractor output (extracted JSON) <br>• **Input 1:** <br> - Invoice/Contract: easybits Extractor output <br> - Purchase Order: Gmail Trigger (original payload) <br>• **Output:** Corresponding Google Drive upload node. <br>• **Edge Cases:** If the two inputs have a different number of items, only the matching pairs are combined; unmatched items are dropped. <br>• **Sticky Note:** *“## 🔗 Merge: Combine Extracted Data + File – The Extractor returns structured JSON but strips the binary again. These Merge nodes recombine the extracted data with the original file so the Google Drive upload node has both the file to store and the JSON to pass downstream.”* |

---

### 2.7 Upload to Google Drive  

| Aspect | Details |
|--------|---------|
| **Overview** | Stores the original document in a type‑specific Drive folder and makes the `webViewLink` available for later nodes. |
| **Nodes Involved** | • Upload Invoice to Drive <br>• Upload Contract to Drive <br>• Upload Purchase Order to Drive |
| **Node Details – Google Drive Upload Nodes** | • **Type:** `n8n-nodes-base.googleDrive` (v3) <br>• **Configuration (common):** `driveId: "My Drive"`, `inputDataFieldName: "attachment_0"`, `options: {}`. <br>• **Folder IDs (placeholder):** <br> - Invoice: `YOUR_FOLDER_ID` (must be replaced with the actual Drive folder for Invoices) <br> - Contract & PO: `"root"` (root folder; should be changed to dedicated folders). <br>• **Input:** Respective `Merge: … Data + File` node. <br>• **Output:** <br> - Invoice → `Update Master Finance Sheet` <br> - Contract → `Slack: Notify Sales – New Contract` <br> - Purchase Order → `Slack: Notify Team – Restock Update` <br>• **Edge Cases:** If the folder ID is invalid, the node will throw a permission error. Large attachments may hit Google Drive API size limits. <br>• **Sticky Note:** *“## ☁️ Upload to Google Drive – Stores the original document in a dedicated folder per type: **Invoices**, **Contracts**, or **Purchase Orders**. The Drive link is passed to the next node for reference in Sheets or Slack.”* |

---

### 2.8 Notify & Log  

| Aspect | Details |
|--------|---------|
| **Overview** | Performs the final action for each route: invoices update a spreadsheet; contracts and POs post Slack notifications with extracted data and a Drive link. |
| **Nodes Involved** | • Update Master Finance Sheet (Google Sheets) <br>• Slack: Notify Sales – New Contract <br>• Slack: Notify Team – Restock Update |
| **Node Details – Update Master Finance Sheet** | • **Type:** `n8n-nodes-base.googleSheets` (v4.7) <br>• **Configuration:** <br> - Operation: `appendOrUpdate` (upsert) <br> - Document ID: `YOUR_SPREADSHEET_ID` (placeholder) <br> - Sheet name: `Sheet1` (gid 0) <br> - Matching column: `Invoice Number` (used for upsert) <br> - Columns defined: <br>  • `Invoice Number` = `{{ $('Merge: Invoice Data + File').item.json.data.invoice_number }}` <br>  • `Final Amount (EUR)` = `{{ $('Merge: Invoice Data + File').item.json.data.total_amount }}` <br> - Additional columns (`Original Amount`, `Currency`, `Exchange Rate`) are present in the schema but marked `removed:true`. <br>• **Input:** `Upload Invoice to Drive`. <br>• **Output:** None (terminal). <br>• **Edge Cases:** If the spreadsheet lacks the `Invoice Number` column, upsert will fail. Incorrect mapping of fields will produce empty cells. |
| **Node Details – Slack: Notify Sales – New Contract** | • **Type:** `n8n-nodes-base.slack` (v2.4) <br>• **Configuration:** <br> - Message type: `block` (Block Kit) <br> - Channel: `YOUR_CHANNEL_ID` (placeholder) <br> - Block UI template containing extracted contract fields (`client_name`, `contract_type`, `contract_value`, `currency`, `start_date`, `notice_period`) and a Drive link (`<{{ $('Upload Contract to Drive').item.json.webViewLink }}|View in Drive>`). <br>• **Input:** `Upload Contract to Drive`. <br>• **Output:** None (terminal). <br>• **Edge Cases:** If the channel ID is incorrect or the Slack app lacks posting permission, the node will throw an auth error. |
| **Node Details – Slack: Notify Team – Restock Update** | • **Type:** `n8n-nodes-base.slack` (v2.4) <br>• **Configuration:** Similar to the contract node but for Purchase Order fields (`supplier_name`, `po_number`, `order_date`, `expected_delivery_date`, `total_amount`, `currency`) and a Drive link (`<{{ $('Upload Purchase Order to Drive').item.json.webViewLink }}|View in Drive>`). <br>• **Input:** `Upload Purchase Order to Drive`. <br>• **Output:** None (terminal). <br>• **Edge Cases:** Same as above; placeholder channel ID must be replaced. |
| **Sticky Note:** *“## 📤 Notify & Log – Each route ends with a different action: **Invoices** are appended to the Master Finance Sheet. **Contracts** trigger a Slack message to Mike with key terms and a Drive link. **Purchase Orders** post a restock update to the team channel.”* |

---

## 3. Summary Table  

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|----------------|---------------|----------------|-------------|
| Gmail Trigger | gmailTrigger (v1.3) | Entry point – polls Gmail for new emails and downloads attachments | – | easybits: Classify Document; Merge: Invoice Binary (0); Merge: Invoice Data + File (0); Merge: Contract Binary (0); Merge: Purchase Order Binary (1); Merge: Contract Data + File (0); Merge: Purchase Order Data + File (1) | ## 📧 Email Intake – Polls Gmail every minute for new emails. Downloads attachments automatically. The binary file is branched to all merge nodes so the original document stays available throughout the workflow. |
| easybits: Classify Document | @easybits/n8n-nodes-extractor.easybitsExtractor (v2) | Sends attachment to classification pipeline and returns `document_class` | Gmail Trigger | Route by Document Type | ## 🏷️ Document Classification – Sends the email attachment to **easybits Extractor** configured for classification. Returns a `document_class` field with one of three values: `Invoice`, `Contracts`, or `Purchase Orders`. The Switch node routes the item down the matching path. |
| Route by Document Type | switch (v3.4) | Routes item based on `document_class` value | easybits: Classify Document | Merge: Invoice Binary (1); Merge: Contract Binary (1); Merge: Purchase Order Binary (0) | ## 🔀 Route by Document Type – Reads the `document_class` returned by the classification step and routes the item to the matching path: **Invoice**, **Contract**, or **Purchase Order**. Each output connects to its own extraction and delivery pipeline. |
| Merge: Invoice Binary | merge (v3.2) | Re‑attaches original binary to invoice‑routed item | Gmail Trigger (0); Route by Document Type (output 0 → input 1) | easybits: Extract Invoice | ## 🔗 Merge: Restore Original File – The classification step strips the binary from the item. These Merge nodes reunite the original email attachment (from the Gmail Trigger) with the routed item so the Extractor receives both the document class and the actual file. |
| Merge: Contract Binary | merge (v3.2) | Re‑attaches original binary to contract‑routed item | Gmail Trigger (0); Route by Document Type (output 1 → input 1) | easybits: Extract Contract | Same as above |
| Merge: Purchase Order Binary | merge (v3.2) | Re‑attaches original binary to PO‑routed item | Gmail Trigger (1); Route by Document Type (output 2 → input 0) | Extractor: Purchase Orders | Same as above |
| easybits: Extract Invoice | @easybits/n8n-nodes-extractor.easybitsExtractor (v2) | Extracts invoice‑specific fields (invoice_number, total_amount, etc.) | Merge: Invoice Binary | Merge: Invoice Data + File (input 1) | ## 🤖 Document‑Specific Extraction – Each route has its own **easybits Extractor** pipeline configured for that document type. Invoices extract vendor, amount, due date. Contracts extract client, value, notice period. Purchase Orders extract supplier, items, delivery date. |
| easybits: Extract Contract | @easybits/n8n-nodes-extractor.easybitsExtractor (v2) | Extracts contract‑specific fields | Merge: Contract Binary | Merge: Contract Data + File (input 1) | Same as above |
| Extractor: Purchase Orders | @easybits/n8n-nodes-extractor.easybitsExtractor (v2) | Extracts PO‑specific fields | Merge: Purchase Order Binary | Merge: Purchase Order Data + File (input 0) | Same as above |
| Merge: Invoice Data + File | merge (v3.2) | Combines extracted invoice data with original binary (Combine mode) | Gmail Trigger (0); easybits: Extract Invoice (1) | Upload Invoice to Drive | ## 🔗 Merge: Combine Extracted Data + File – The Extractor returns structured JSON but strips the binary again. These Merge nodes recombine the extracted data with the original file so the Google Drive upload node has both the file to store and the JSON to pass downstream. |
| Merge: Contract Data + File | merge (v3.2) | Combines extracted contract data with original binary (Combine mode) | Gmail Trigger (0); easybits: Extract Contract (1) | Upload Contract to Drive | Same as above |
| Merge: Purchase Order Data + File | merge (v3.2) | Combines extracted PO data with original binary (Combine mode) | Gmail Trigger (1); Extractor: Purchase Orders (0) | Upload Purchase Order to Drive | Same as above |
| Upload Invoice to Drive | googleDrive (v3) | Uploads invoice PDF to a Drive folder (placeholder `YOUR_FOLDER_ID`) | Merge: Invoice Data + File | Update Master Finance Sheet | ## ☁️ Upload to Google Drive – Stores the original document in a dedicated folder per type: **Invoices**, **Contracts**, or **Purchase Orders**. The Drive link is passed to the next node for reference in Sheets or Slack. |
| Upload Contract to Drive | googleDrive (v3) | Uploads contract PDF to Drive (root folder) | Merge: Contract Data + File | Slack: Notify Sales – New Contract | Same as above |
| Upload Purchase Order to Drive | googleDrive (v3) | Uploads PO PDF to Drive (root folder) | Merge: Purchase Order Data + File | Slack: Notify Team – Restock Update | Same as above |
| Update Master Finance Sheet | googleSheets (v4.7) | Appends/updates invoice data in a Master Finance spreadsheet | Upload Invoice to Drive | – | ## 📤 Notify & Log – Each route ends with a different action: **Invoices** are appended to the Master Finance Sheet. **Contracts** trigger a Slack message to Mike with key terms and a Drive link. **Purchase Orders** post a restock update to the team channel. |
| Slack: Notify Sales – New Contract | slack (v2.4) | Sends a Block Kit message with contract details and Drive link | Upload Contract to Drive | – | Same as above |
| Slack: Notify Team – Restock Update | slack (v2.4) | Sends a Block Kit message with PO details and Drive link | Upload Purchase Order to Drive | – | Same as above |
| Sticky Note | stickyNote (v1) | Annotation covering Email Intake block | – | – | ## 📧 Email Intake – Polls Gmail every minute for new emails. Downloads attachments automatically. The binary file is branched to all merge nodes so the original document stays available throughout the workflow. |
| Sticky Note1 | stickyNote (v1) | Annotation covering Classification block | – | – | ## 🏷️ Document Classification – Sends the email attachment to **easybits Extractor** configured for classification. Returns a `document_class` field with one of three values: `Invoice`, `Contracts`, or `Purchase Orders`. The Switch node routes the item down the matching path. |
| Sticky Note2 | stickyNote (v1) | Annotation covering Routing block | – | – | ## 🔀 Route by Document Type – Reads the `document_class` returned by the classification step and routes the item to the matching path: **Invoice**, **Contract**, or **Purchase Order**. Each output connects to its own extraction and delivery pipeline. |
| Sticky Note3 | stickyNote (v1) | Annotation covering Binary Restoration block | – | – | ## 🔗 Merge: Restore Original File – The classification step strips the binary from the item. These Merge nodes reunite the original email attachment (from the Gmail Trigger) with the routed item so the Extractor receives both the document class and the actual file. |
| Sticky Note4 | stickyNote (v1) | Annotation covering Extraction block | – | – | ## 🤖 Document‑Specific Extraction – Each route has its own **easybits Extractor** pipeline configured for that document type. Invoices extract vendor, amount, due date. Contracts extract client, value, notice period. Purchase Orders extract supplier, items, delivery date. |
| Sticky Note5 | stickyNote (v1) | Annotation covering Data + File Merge block | – | – | ## 🔗 Merge: Combine Extracted Data + File – The Extractor returns structured JSON but strips the binary again. These Merge nodes recombine the extracted data with the original file so the Google Drive upload node has both the file to store and the JSON to pass downstream. |
| Sticky Note6 | stickyNote (v1) | Annotation covering Upload to Drive block | – | – | ## ☁️ Upload to Google Drive – Stores the original document in a dedicated folder per type: **Invoices**, **Contracts**, or **Purchase Orders**. The Drive link is passed to the next node for reference in Sheets or Slack. |
| Sticky Note7 | stickyNote (v1) | Annotation covering Notify & Log block | – | – | ## 📤 Notify & Log – Each route ends with a different action: **Invoices** are appended to the Master Finance Sheet. **Contracts** trigger a Slack message to Mike with key terms and a Drive link. **Purchase Orders** post a restock update to the team channel. |
| Sticky Note8 | stickyNote (v1) | General workflow description & setup guide | – | – | (Content is reproduced in Section 5 – General Notes & Resources) |

---

## 4. Reproducing the Workflow from Scratch  

Follow these steps in a new n8n instance. All placeholder IDs (`YOUR_FOLDER_ID`, `YOUR_SPREADSHEET_ID`, `YOUR_CHANNEL_ID`) must be replaced with actual values before activation.

1. **Create a new workflow** and name it `Smart Mailroom Workflow powered by easybits`.

2. **Add the Gmail Trigger node**  
   - Type: `Gmail Trigger` (v1.3)  
   - Authenticate with a Gmail OAuth2 credential.  
   - Set **Poll Times** → `Every Minute`.  
   - Enable **Download Attachments** under *Options*.  
   - Leave *Simple* unchecked to keep the full email payload.  

3. **Add the easybits Classification node**  
   - Type: `easybits Extractor` (v2)  
   - Create a credential pointing to your easybits account.  
   - In the node, select or paste the **Pipeline ID** for the classification pipeline (the pipeline must define a single field `document_class`).  
   - No other parameters are required.  

4. **Connect** Gmail Trigger → easybits: Classify Document (main output 0).

5. **Add the Switch node** – `Route by Document Type`  
   - Type: `Switch` (v3.4)  
   - Add three rules:  
     - Rule 0: `{{ $json.data.document_class }}` **equals** `Invoice` → Output 0  
     - Rule 1: `{{ $json.data.document_class }}` **equals** `Contracts` → Output 1  
     - Rule 2: `{{ $json.data.document_class }}` **equals** `Purchase Orders` → Output 2  

6. **Connect** easybits: Classify Document → Route by Document Type (output 0).

7. **Add three Merge nodes** for binary restoration:  
   - `Merge: Invoice Binary` (v3.2)  
   - `Merge: Contract Binary` (v3.2)  
   - `Merge: Purchase Order Binary` (v3.2)  

   Leave the default mode (`Append`).  

8. **Wire the binary restoration nodes**:  
   - Gmail Trigger → **Input 0** of each Merge node.  
   - Route by Document Type → **Input 1** of `Merge: Invoice Binary` and `Merge: Contract Binary`.  
   - Route by Document Type → **Input 0** of `Merge: Purchase Order Binary`.  

9. **Add three easybits Extraction nodes** (one per document type):  
   - `easybits: Extract Invoice` (v2) – link to the Invoice pipeline.  
   - `easybits: Extract Contract` (v2) – link to the Contract pipeline.  
   - `Extractor: Purchase Orders` (v2) – link to the PO pipeline.  

   Each node requires its own **Pipeline ID** credential.  

10. **Connect** the binary restoration outputs to the extraction nodes:  
    - `Merge: Invoice Binary` → easybits: Extract Invoice  
    - `Merge: Contract Binary` → easybits: Extract Contract  
    - `Merge: Purchase Order Binary` → Extractor: Purchase Orders  

11. **Add three Merge nodes** for data + file re‑combination:  
    - `Merge: Invoice Data + File` (v3.2) – set mode **Combine** and **Combine By Position**.  
    - `Merge: Contract Data + File` (v3.2) – same mode.  
    - `Merge: Purchase Order Data + File` (v3.2) – same mode.  

12. **Wire the data‑file merges**:  
    - For Invoice & Contract: Gmail Trigger → **Input 0**; Extraction node → **Input 1**.  
    - For Purchase Order: Gmail Trigger → **Input 1**; Extraction node → **Input 0**.  

13. **Add three Google Drive upload nodes**:  
    - `Upload Invoice to Drive` (v3) – set `folderId` to your **Invoices** folder, `inputDataFieldName` = `attachment_0`.  
    - `Upload Contract to Drive` (v3) – set `folderId` to your **Contracts** folder (or replace `"root"` with the actual folder).  
    - `Upload Purchase Order to Drive` (v3) – set `folderId` to your **Purchase Orders** folder (or replace `"root"`).  

14. **Connect** the data‑file merges to the upload nodes:  
    - `Merge: Invoice Data + File` → Upload Invoice to Drive  
    - `Merge: Contract Data + File` → Upload Contract to Drive  
    - `Merge: Purchase Order Data + File` → Upload Purchase Order to Drive  

15. **Add the Google Sheets node**  
    - `Update Master Finance Sheet` (v4.7)  
    - Authenticate with a Google Sheets credential.  
    - Set **Operation** → `Append or Update`.  
    - **Document ID** → your Master Finance spreadsheet (`YOUR_SPREADSHEET_ID`).  
    - **Sheet Name** → `Sheet1` (or the desired tab).  
    - **Matching Column** → `Invoice Number`.  
    - **Columns** (Define Below): <br> `Invoice Number` = `{{ $('Merge: Invoice Data + File').item.json.data.invoice_number }}` <br> `Final Amount (EUR)` = `{{ $('Merge: Invoice Data + File').item.json.data.total_amount }}`  
    - (Optionally add additional columns; they are currently marked removed in the template.)  

16. **Add two Slack nodes** for notifications  
    - `Slack: Notify Sales – New Contract` (v2.4) – select channel `YOUR_CHANNEL_ID` (e.g., `#contracts`). Use the **Block Kit** message provided in the node, referencing fields from `Merge: Contract Data + File` and the Drive link from `Upload Contract to Drive`.  
    - `Slack: Notify Team – Restock Update` (v2.4) – select channel `YOUR_CHANNEL_ID` (e.g., `#operations`). Use the Block Kit message referencing `Merge: Purchase Order Data + File` and the Drive link from `Upload Purchase Order to Drive`.  

17. **Connect the final actions**:  
    - `Upload Invoice to Drive` → `Update Master Finance Sheet`  
    - `Upload Contract to Drive` → `Slack: Notify Sales – New Contract`  
    - `Upload Purchase Order to Drive` → `Slack: Notify Team – Restock Update`  

18. **Optionally add the Sticky Note annotations** (copy the content from the original workflow for reference).  

19. **Test** the workflow:  
    - Send three separate emails (one with an invoice, one with a contract, one with a purchase order) to the monitored Gmail address.  
    - Verify that each route processes correctly, the file appears in the correct Drive folder, the Sheet row is created for the invoice, and the Slack messages appear in the designated channels.  

20. **Activate** the workflow and monitor the execution log for any credential or permission errors.

---

## 5. General Notes & Resources  

| Note Content | Context or Link |
|--------------|-----------------|
| **Smart Mailroom Workflow Overview** – Full description of the workflow’s purpose, how it works, and step‑by‑step setup guide (including creating easybits pipelines, Gmail, Drive, Sheets, Slack configuration). | (Extracted from Sticky Note 8) |
| **easybits Extractor** – Create classification and extraction pipelines at `https://extractor.easybits.tech`. Each pipeline must define the exact field names listed in the node configuration. | https://extractor.easybits.tech |
| **Gmail Trigger** – Uses OAuth2; enable *Download Attachments* to ensure binary data (`attachment_0`) is available. | Gmail OAuth2 documentation |
| **Google Drive** – Create three folders (`Invoices`, `Contracts`, `Purchase Orders`) and replace the placeholder `YOUR_FOLDER_ID` (and the `"root"` entries) with the actual folder IDs. | Google Drive API |
| **Google Sheets** – The `Update Master Finance Sheet` node uses `Append or Update` with `Invoice Number` as the matching key. Ensure the spreadsheet contains that column header. | Google Sheets API |
| **Slack** – The workflow uses Block Kit messages; channel IDs must be supplied (`YOUR_CHANNEL_ID`). Ensure the Slack app has `chat:write` and `channels:read` scopes. | Slack Block Kit Builder |
| **Edge Cases** – If an email contains no attachment, the downstream nodes will receive no binary; consider adding a condition to filter such messages. If classification returns an unexpected label, the Switch has no fallback; add a default path if needed. | — |
| **Data Flow** – The binary (`attachment_0`) is preserved by repeatedly merging it back after each transformation that discards it. This pattern ensures the final upload node always has the original file. | — |
| **Credential Management** – Store all OAuth2 credentials (Gmail, Google Drive, Sheets, Slack) and API keys (easybits) in n8n’s credential store; avoid hard‑coding them in the workflow JSON. | — |
| **Version Compatibility** – The workflow uses node versions listed in the Summary Table. If you import into a newer n8n version, verify that each node type still supports the same version or upgrade accordingly. | — |
| **Performance** – Polling every minute may generate many executions for high‑volume inboxes; consider switching to a Gmail Push (watch) trigger for near‑real‑time processing with lower API load. | — |

---