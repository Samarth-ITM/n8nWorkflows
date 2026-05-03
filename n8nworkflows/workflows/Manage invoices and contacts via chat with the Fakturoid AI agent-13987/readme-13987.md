Manage invoices and contacts via chat with the Fakturoid AI agent

https://n8nworkflows.xyz/workflows/manage-invoices-and-contacts-via-chat-with-the-fakturoid-ai-agent-13987


# Manage invoices and contacts via chat with the Fakturoid AI agent

# 1. Workflow Overview

This workflow implements a chat-based AI assistant for managing **contacts and invoices in Fakturoid**, with optional company data enrichment from the **Czech ARES registry**. Users interact in natural Czech through an n8n chat trigger, while an AI Agent decides when to search, create, update, delete, or mark invoices as paid using connected sub-workflows.

The workflow is a **tool-using agent architecture**: the main workflow does not directly call Fakturoid APIs. Instead, it exposes **10 sub-workflows as tools** to the AI Agent, which orchestrates them based on the conversation and its system instructions.

## 1.1 Chat Entry and Conversation Handling

The workflow starts when a chat message is received. The incoming message is passed to an AI Agent, which uses short-term conversation memory and a chat language model to interpret the request and maintain context across turns.

## 1.2 AI Reasoning and Policy Layer

The AI Agent contains a large system prompt defining:
- language behavior (Czech)
- supported actions
- strict separation between `subject_id` and `invoice_id`
- confirmation rules before write/destructive actions
- guidance on reusing context instead of repeating lookups

This block is the decision-making core of the workflow.

## 1.3 Contact Management Tools

A group of toolWorkflow nodes lets the agent:
- search Fakturoid contacts
- create new contacts
- update existing contacts
- delete contacts
- enrich company details from the ARES registry

These tools are intended to support contact lookup and preparation before invoice creation.

## 1.4 Invoice Management Tools

A second group of toolWorkflow nodes lets the agent:
- search invoices
- create invoices
- update invoices
- delete invoices
- record payments / mark invoices as paid

The agent is explicitly instructed to ask for confirmation before creating, deleting, or marking invoices as paid.

---

# 2. Block-by-Block Analysis

## 2.1 Chat Entry and Context Memory

### Overview
This block receives the user's chat message and provides conversational memory to the AI Agent. It is responsible for starting each interaction and preserving recent context so the agent can resolve references like “that invoice” or “that client”.

### Nodes Involved
- When chat message received
- Simple Memory

### Node Details

#### When chat message received
- **Type and technical role:** `@n8n/n8n-nodes-langchain.chatTrigger`; entry-point trigger for chat-based interactions.
- **Configuration choices:** Uses default options; no custom input filtering or advanced trigger configuration is set.
- **Key expressions or variables used:** None visible in configuration.
- **Input and output connections:**
  - No incoming connection; this is the workflow entry point.
  - Outputs to **AI Agent1** via the main connection.
- **Version-specific requirements:** Type version `1.4`. Requires n8n environment with chat trigger support enabled.
- **Edge cases or potential failure types:**
  - Chat UI/webhook access issues
  - Trigger not usable if workflow is inactive
  - Problems if deployed environment does not expose chat functionality properly
- **Sub-workflow reference:** None.

#### Simple Memory
- **Type and technical role:** `@n8n/n8n-nodes-langchain.memoryBufferWindow`; short-term conversational memory for the agent.
- **Configuration choices:** Context window length is set to `10`, meaning the agent retains the latest 10 turns/interactions in memory.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - No main input stream; connected as **ai_memory** into **AI Agent1**.
  - Supplies memory context to the agent.
- **Version-specific requirements:** Type version `1.3`.
- **Edge cases or potential failure types:**
  - Older references may fall out of context after enough conversation turns
  - Ambiguous pronouns may fail if the relevant entity was mentioned more than 10 turns earlier
- **Sub-workflow reference:** None.

---

## 2.2 AI Agent Core and Language Model

### Overview
This block contains the reasoning engine of the workflow. The AI Agent receives the user message, uses the connected language model and memory, applies the system rules, and invokes the correct tools when needed.

### Nodes Involved
- GPT-5-mini
- AI Agent1

### Node Details

#### GPT-5-mini
- **Type and technical role:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`; chat model backend used by the AI Agent.
- **Configuration choices:** No explicit model options are shown beyond the node identity; it uses connected OpenAI credentials named `TEMPLATE`.
- **Key expressions or variables used:** None shown.
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_languageModel`.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:**
  - Invalid or missing OpenAI credentials
  - Model/provider quota exhaustion
  - Timeout or rate limiting
  - Model behavior variance if the credential’s configured model changes
- **Sub-workflow reference:** None.

#### AI Agent1
- **Type and technical role:** `@n8n/n8n-nodes-langchain.agent`; central orchestration node that interprets the conversation and chooses tools.
- **Configuration choices:**
  - Contains a detailed system prompt titled **Invoice Botík – System Prompt**
  - Communicates in **Czech**
  - Enforces strict rules about entity IDs:
    - `subject_id` for contacts
    - `invoice_id` for invoices
  - Requires confirmation before:
    - creating an invoice
    - deleting
    - marking an invoice as paid
  - Encourages reuse of conversation context instead of repeated lookups
  - Defines expected operation flows for contact and invoice handling
  - Defines formatting conventions for currency, dates, and invoice numbers
- **Key expressions or variables used:** None directly; behavior depends on tool outputs and memory context.
- **Input and output connections:**
  - Main input from **When chat message received**
  - AI language model input from **GPT-5-mini**
  - AI memory input from **Simple Memory**
  - AI tool inputs from all toolWorkflow nodes
- **Version-specific requirements:** Type version `3.1`. Requires compatible LangChain-capable n8n version.
- **Edge cases or potential failure types:**
  - Hallucinated reasoning if the LLM ignores instructions
  - Incorrect tool choice if user request is ambiguous
  - Failure to resolve prior context if memory window is exceeded
  - Confirmation loop issues if model behavior drifts from prompt intent
  - Mismatch between expected tool semantics and actual sub-workflow implementation
- **Sub-workflow reference:** Indirectly orchestrates all referenced tool sub-workflows.

---

## 2.3 Contact Management and ARES Enrichment

### Overview
This block supports the full contact lifecycle in Fakturoid and optionally enriches contact data from the Czech ARES business registry. It is used when the user wants to find, create, update, or delete a client/contact, or when invoice creation requires a subject first.

### Nodes Involved
- ARES LOOKUP
- GET_SUBJECT
- CREATE_SUBJECT
- UPDATE_SUBJECT
- DELETE_SUBJECT

### Node Details

#### ARES LOOKUP
- **Type and technical role:** `@n8n/n8n-nodes-langchain.toolWorkflow`; exposes a separate workflow as an AI tool for ARES business registry search.
- **Configuration choices:**
  - Invokes sub-workflow `ARESCaller`
  - Description tells the agent to search Czech business registry by `ICO` or `CompanyName`
  - Input schema includes:
    - `ICO` as number
    - `CompanyName` as string
- **Key expressions or variables used:**
  - `{{$fromAI('ICO', ...)}}`
  - `{{$fromAI('CompanyName', ...)}}`
  These allow the agent to populate tool arguments from natural language.
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - No result for invalid or unknown IČO/company name
  - Type mismatch if the LLM provides non-numeric IČO
  - Upstream ARES availability issues in the called workflow
- **Sub-workflow reference:** `ARESCaller` (`Y8AeWLK8FzC3kZYmAVhVf`)

#### GET_SUBJECT
- **Type and technical role:** `toolWorkflow`; searches existing contacts in Fakturoid.
- **Configuration choices:**
  - Invokes `Fakturoid_get_subject_TEMPLATE`
  - Description explicitly says to use this before creating a new contact to avoid duplicates
  - Accepts one input:
    - `query`
- **Key expressions or variables used:**
  - `{{$fromAI('query', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - No results found
  - Multiple fuzzy matches requiring agent disambiguation
  - Search inconsistency if user provides partial data
- **Sub-workflow reference:** `Fakturoid_get_subject_TEMPLATE` (`x2F0ZxrGsSSVMsHp`)

#### CREATE_SUBJECT
- **Type and technical role:** `toolWorkflow`; creates a new Fakturoid contact/subject.
- **Configuration choices:**
  - Invokes `Fakturoid_create_subject_TEMPLATE`
  - Accepts:
    - `name`
    - `ico`
    - `email`
    - `phone`
    - `street`
    - `city`
    - `zip`
    - `dic`
  - Description recommends using ARES data when available
- **Key expressions or variables used:**
  - `{{$fromAI('name', ...)}}`
  - `{{$fromAI('ico', ...)}}`
  - `{{$fromAI('email', ...)}}`
  - `{{$fromAI('phone', ...)}}`
  - `{{$fromAI('street', ...)}}`
  - `{{$fromAI('city', ...)}}`
  - `{{$fromAI('zip', ...)}}`
  - `{{$fromAI('dic', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Duplicate subject creation if `GET_SUBJECT` is skipped or search fails
  - Missing required business fields in the underlying sub-workflow
  - Validation failures on email/ICO formats
- **Sub-workflow reference:** `Fakturoid_create_subject_TEMPLATE` (`XW7U0YtRiZeMpCRi`)

#### UPDATE_SUBJECT
- **Type and technical role:** `toolWorkflow`; updates an existing Fakturoid subject.
- **Configuration choices:**
  - Invokes `Fakturoid_update_subject_TEMPLATE`
  - Supports partial update semantics: only provided fields are changed
  - Requires `subject_id` from `fakturoid_get_subject`
  - Updatable fields:
    - `name`, `ico`, `dic`, `street`, `city`, `zip`, `email`, `phone`
- **Key expressions or variables used:**
  - `{{$fromAI('subject_id', ...)}}`
  - plus one expression for each optional field
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Confusing `subject_id` with IČO
  - Invalid `subject_id`
  - Partial update may unintentionally leave stale data if user expected full overwrite
- **Sub-workflow reference:** `Fakturoid_update_subject_TEMPLATE` (`jXIz2pM85LjrUfyp`)

#### DELETE_SUBJECT
- **Type and technical role:** `toolWorkflow`; deletes a Fakturoid subject/contact by ID.
- **Configuration choices:**
  - Invokes `Fakturoid_delete_subject_TEMPLATE`
  - Accepts only `subject_id`
- **Key expressions or variables used:**
  - `{{$fromAI('subject_id', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Invalid `subject_id`
  - Deletion blocked by Fakturoid business rules or references
  - Agent prompt does not emphasize confirmation for subject deletion as strongly as for invoice deletion, so behavior depends on model interpretation
- **Sub-workflow reference:** `Fakturoid_delete_subject_TEMPLATE` (`CcYWBnucFh1OoivN`)

---

## 2.4 Invoice Management

### Overview
This block provides all invoice-related operations. It is used to find invoices, create new ones for an existing subject, update invoice fields, delete unpaid invoices, and register payments.

### Nodes Involved
- CREATE_INVOICE
- GET_INVOICE
- DELETE_INVOICE
- UPDATE_INVOICE
- INVOICE_PAYMENT

### Node Details

#### CREATE_INVOICE
- **Type and technical role:** `toolWorkflow`; creates a new invoice in Fakturoid.
- **Configuration choices:**
  - Invokes `Fakturoid_create_invoice_TEMPLATE`
  - Description instructs the agent to show a summary with item breakdown and total amount before creation
  - Inputs:
    - `subject_id` as number
    - `lines` as JSON/array
    - `note` as optional string
  - The line-item guidance is strict: each line must use exactly:
    - `name`
    - `quantity`
    - `unit_price`
- **Key expressions or variables used:**
  - `{{$fromAI('subject_id', ...)}}`
  - `{{$fromAI('lines', ...)}}`
  - `{{$fromAI('note', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Passing `invoice_id` instead of `subject_id`
  - Invalid `lines` structure
  - Missing required fields in any line item
  - Type mismatch if the LLM emits malformed JSON
  - Confirmation may be skipped if model behavior drifts
- **Sub-workflow reference:** `Fakturoid_create_invoice_TEMPLATE` (`Otewvb4hm364Skd2`)

#### GET_INVOICE
- **Type and technical role:** `toolWorkflow`; searches invoices or retrieves invoice details.
- **Configuration choices:**
  - Invokes `Fakturoid_get_invoice_TEMPLATE`
  - Inputs:
    - `Invoice_id` as string
    - `query` as string
    - `limit` as number
  - Description allows searching by invoice number, subject name, or listing recent invoices
- **Key expressions or variables used:**
  - `{{$fromAI('Invoice_id', ...)}}`
  - `{{$fromAI('query', ...)}}`
  - `{{$fromAI('limit', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Note the field is named `Invoice_id` with capital `I`, which differs from `invoice_id` used elsewhere; this can cause confusion in maintenance
  - No match or too many matches
  - String/number mismatch depending on sub-workflow expectation
- **Sub-workflow reference:** `Fakturoid_get_invoice_TEMPLATE` (`oNKwEN15gPaAWSsu`)

#### DELETE_INVOICE
- **Type and technical role:** `toolWorkflow`; deletes an invoice.
- **Configuration choices:**
  - Invokes `Fakturoid_delete_invoice_TEMPLATE`
  - Accepts only `invoice_id`
  - Description explicitly says:
    - can only delete unpaid invoices
    - always ask for confirmation
    - use only for deleting invoices
- **Key expressions or variables used:**
  - `{{$fromAI('invoice_id', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Attempting to delete a paid invoice
  - Using `subject_id` by mistake
  - Invalid or stale `invoice_id`
- **Sub-workflow reference:** `Fakturoid_delete_invoice_TEMPLATE` (`LsmsxYwHnMcbKjet`)

#### UPDATE_INVOICE
- **Type and technical role:** `toolWorkflow`; updates selected invoice fields.
- **Configuration choices:**
  - Invokes `Fakturoid_update_invoice_TEMPLATE`
  - Supports partial updates
  - Inputs:
    - `invoice_id`
    - `due_on`
    - `note`
    - `private_note`
    - `number`
- **Key expressions or variables used:**
  - `{{$fromAI('invoice_id', ...)}}`
  - `{{$fromAI('due_on', ...)}}`
  - `{{$fromAI('note', ...)}}`
  - `{{$fromAI('private_note', ...)}}`
  - `{{$fromAI('number', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Wrong ID type
  - Invalid date format for `due_on`
  - Number conflicts if invoice numbering rules exist downstream
- **Sub-workflow reference:** `Fakturoid_update_invoice_TEMPLATE` (`TpOq2gGQ25lbInlm`)

#### INVOICE_PAYMENT
- **Type and technical role:** `toolWorkflow`; records invoice payment or marks invoice as paid.
- **Configuration choices:**
  - Invokes `Fakturoid_invoice_paymentTEMPLATE`
  - Inputs:
    - `invoice_id`
    - `paid_on`
    - `payment_method`
  - Description says `paid_on` defaults to today if omitted, assuming the sub-workflow implements that default
- **Key expressions or variables used:**
  - `{{$fromAI('invoice_id', ...)}}`
  - `{{$fromAI('paid_on', ...)}}`
  - `{{$fromAI('payment_method', ...)}}`
- **Input and output connections:**
  - Connected to **AI Agent1** as `ai_tool`
- **Version-specific requirements:** Type version `2.2`.
- **Edge cases or potential failure types:**
  - Invalid date format
  - Unsupported payment method string if downstream validation is strict
  - Wrong invoice ID or already-paid state
- **Sub-workflow reference:** `Fakturoid_invoice_paymentTEMPLATE` (`nM6KDKEpkqatSCQc`)

---

## 2.5 Documentation and Visual Guidance Notes

### Overview
This block contains sticky notes used for visual organization and operational guidance in the canvas. They do not affect execution but are important for maintainability and onboarding.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2

### Node Details

#### Sticky Note
- **Type and technical role:** `n8n-nodes-base.stickyNote`; visual label for the contacts tool area.
- **Configuration choices:** Displays heading `## CONTACTS`.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note1
- **Type and technical role:** `stickyNote`; visual label for the invoices tool area.
- **Configuration choices:** Displays heading `## INVOICES`.
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational.
- **Sub-workflow reference:** None.

#### Sticky Note2
- **Type and technical role:** `stickyNote`; high-level workflow guidance and setup instructions.
- **Configuration choices:** Contains:
  - workflow purpose
  - supported actions
  - architecture note about 10 sub-workflows
  - setup steps for Fakturoid token/account slug, LLM credentials, and sub-workflow activation
- **Input and output connections:** None.
- **Version-specific requirements:** Type version `1`.
- **Edge cases or potential failure types:** None operational, but the setup guidance is critical for successful deployment.
- **Sub-workflow reference:** References all 10 tool sub-workflows conceptually.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Simple Memory | `@n8n/n8n-nodes-langchain.memoryBufferWindow` | Stores short conversation history for the AI agent |  | AI Agent1 |  |
| INVOICE_PAYMENT | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to mark invoice as paid / record payment |  | AI Agent1 | ## INVOICES |
| UPDATE_SUBJECT | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to update an existing contact in Fakturoid |  | AI Agent1 | ## CONTACTS |
| CREATE_SUBJECT | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to create a new contact in Fakturoid |  | AI Agent1 | ## CONTACTS |
| GET_SUBJECT | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to search contacts in Fakturoid |  | AI Agent1 | ## CONTACTS |
| ARES LOOKUP | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to query the Czech ARES registry |  | AI Agent1 | ## CONTACTS |
| Sticky Note | `n8n-nodes-base.stickyNote` | Visual label for contacts section |  |  | ## CONTACTS |
| CREATE_INVOICE | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to create an invoice |  | AI Agent1 | ## INVOICES |
| GET_INVOICE | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to search or fetch invoice details |  | AI Agent1 | ## INVOICES |
| DELETE_INVOICE | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to delete an invoice |  | AI Agent1 | ## INVOICES |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Visual label for invoices section |  |  | ## INVOICES |
| UPDATE_INVOICE | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to update selected invoice fields |  | AI Agent1 | ## INVOICES |
| DELETE_SUBJECT | `@n8n/n8n-nodes-langchain.toolWorkflow` | Tool to delete a contact in Fakturoid |  | AI Agent1 | ## CONTACTS |
| AI Agent1 | `@n8n/n8n-nodes-langchain.agent` | Central reasoning/orchestration agent | When chat message received, GPT-5-mini, Simple Memory, ARES LOOKUP, GET_SUBJECT, CREATE_SUBJECT, UPDATE_SUBJECT, DELETE_SUBJECT, CREATE_INVOICE, GET_INVOICE, DELETE_INVOICE, UPDATE_INVOICE, INVOICE_PAYMENT |  | ## How it works<br>AI agent connected to Fakturoid API and ARES registry. Handles invoicing and contact management through natural Czech conversation — no forms, no clicking.<br>**What the agent can do:**<br>- Search, create, update and delete contacts (with ARES lookup for auto-fill)<br>- Search, create, update and delete invoices<br>- Mark invoices as paid<br>**Architecture:**<br>The agent workflow connects to 10 sub-workflows as tools — each handles one Fakturoid operation. Sub-workflows normalize API responses and pass only relevant fields back to the agent.<br>## Setup steps<br>1. Add your **Fakturoid API token** and **account slug** to the Fakturoid HTTP Request nodes<br>2. Connect your **LLM credentials** (OpenAI, Anthropic, or any compatible provider) to the AI Agent node<br>3. Activate all sub-workflows before activating the main agent workflow<br>4. Open the chat and start invoicing |
| When chat message received | `@n8n/n8n-nodes-langchain.chatTrigger` | Chat entry trigger |  | AI Agent1 | ## How it works<br>AI agent connected to Fakturoid API and ARES registry. Handles invoicing and contact management through natural Czech conversation — no forms, no clicking.<br>**What the agent can do:**<br>- Search, create, update and delete contacts (with ARES lookup for auto-fill)<br>- Search, create, update and delete invoices<br>- Mark invoices as paid<br>**Architecture:**<br>The agent workflow connects to 10 sub-workflows as tools — each handles one Fakturoid operation. Sub-workflows normalize API responses and pass only relevant fields back to the agent.<br>## Setup steps<br>1. Add your **Fakturoid API token** and **account slug** to the Fakturoid HTTP Request nodes<br>2. Connect your **LLM credentials** (OpenAI, Anthropic, or any compatible provider) to the AI Agent node<br>3. Activate all sub-workflows before activating the main agent workflow<br>4. Open the chat and start invoicing |
| GPT-5-mini | `@n8n/n8n-nodes-langchain.lmChatOpenAi` | Chat model used by the agent |  | AI Agent1 | ## How it works<br>AI agent connected to Fakturoid API and ARES registry. Handles invoicing and contact management through natural Czech conversation — no forms, no clicking.<br>**What the agent can do:**<br>- Search, create, update and delete contacts (with ARES lookup for auto-fill)<br>- Search, create, update and delete invoices<br>- Mark invoices as paid<br>**Architecture:**<br>The agent workflow connects to 10 sub-workflows as tools — each handles one Fakturoid operation. Sub-workflows normalize API responses and pass only relevant fields back to the agent.<br>## Setup steps<br>1. Add your **Fakturoid API token** and **account slug** to the Fakturoid HTTP Request nodes<br>2. Connect your **LLM credentials** (OpenAI, Anthropic, or any compatible provider) to the AI Agent node<br>3. Activate all sub-workflows before activating the main agent workflow<br>4. Open the chat and start invoicing |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Visual documentation and setup guidance |  |  | ## How it works<br>AI agent connected to Fakturoid API and ARES registry. Handles invoicing and contact management through natural Czech conversation — no forms, no clicking.<br>**What the agent can do:**<br>- Search, create, update and delete contacts (with ARES lookup for auto-fill)<br>- Search, create, update and delete invoices<br>- Mark invoices as paid<br>**Architecture:**<br>The agent workflow connects to 10 sub-workflows as tools — each handles one Fakturoid operation. Sub-workflows normalize API responses and pass only relevant fields back to the agent.<br>## Setup steps<br>1. Add your **Fakturoid API token** and **account slug** to the Fakturoid HTTP Request nodes<br>2. Connect your **LLM credentials** (OpenAI, Anthropic, or any compatible provider) to the AI Agent node<br>3. Activate all sub-workflows before activating the main agent workflow<br>4. Open the chat and start invoicing |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like `AI Invoice Assistant_TEMPLATE`.

2. **Add a Chat Trigger node**
   - Node type: `When chat message received`
   - Technical node: `@n8n/n8n-nodes-langchain.chatTrigger`
   - Leave options at default unless your environment requires chat-specific settings.
   - This will be the main entry point.

3. **Add an AI Agent node**
   - Node type: `AI Agent`
   - Technical node: `@n8n/n8n-nodes-langchain.agent`
   - Connect the Chat Trigger’s main output to the AI Agent’s main input.

4. **Paste the system prompt into the AI Agent**
   - Use a Czech-language prompt for invoice/contact operations.
   - Include these critical behaviors:
     - distinguish `subject_id` from `invoice_id`
     - never invent IDs
     - ask for confirmation before invoice creation, deletion, and payment marking
     - reuse previously fetched entities from conversation memory
     - use contact tools only for contacts and invoice tools only for invoices
   - Include formatting instructions for Czech currency/date rendering if desired.

5. **Add a chat model node**
   - Node type: OpenAI Chat Model
   - Technical node: `@n8n/n8n-nodes-langchain.lmChatOpenAi`
   - Name it `GPT-5-mini` or equivalent.
   - Attach valid **OpenAI credentials**.
   - If your credential allows model selection, choose a lightweight chat-capable model appropriate for agentic tool use.
   - Connect this node to the AI Agent as `ai_languageModel`.

6. **Add a memory node**
   - Node type: Buffer Window Memory
   - Technical node: `@n8n/n8n-nodes-langchain.memoryBufferWindow`
   - Set `Context Window Length` to `10`.
   - Connect it to the AI Agent as `ai_memory`.

7. **Create the ARES lookup tool node**
   - Node type: `Tool Workflow`
   - Name: `ARES LOOKUP`
   - Select the sub-workflow that performs Czech ARES searches.
   - Add description: search Czech business registry by IČO or company name and return structured company details.
   - Define workflow inputs:
     - `ICO` → number
     - `CompanyName` → string
   - Use AI-bound expressions based on `$fromAI(...)` for both fields.
   - Connect it to the AI Agent as `ai_tool`.

8. **Create the contact search tool**
   - Node type: `Tool Workflow`
   - Name: `GET_SUBJECT`
   - Select the sub-workflow that searches Fakturoid contacts.
   - Description should state that it searches by company name, IČO, email, or phone and should be used before creating a contact.
   - Define input:
     - `query` → string
   - Connect it to the AI Agent as `ai_tool`.

9. **Create the contact creation tool**
   - Node type: `Tool Workflow`
   - Name: `CREATE_SUBJECT`
   - Select the sub-workflow that creates a Fakturoid contact.
   - Define inputs:
     - `name` string
     - `ico` string
     - `email` string
     - `phone` string
     - `street` string
     - `city` string
     - `zip` string
     - `dic` string
   - Map each field with `$fromAI(...)`.
   - Connect as `ai_tool`.

10. **Create the contact update tool**
    - Node type: `Tool Workflow`
    - Name: `UPDATE_SUBJECT`
    - Select the sub-workflow that updates a Fakturoid subject.
    - Define inputs:
      - `subject_id` string
      - `name`, `ico`, `dic`, `street`, `city`, `zip`, `email`, `phone`
    - Configure partial updates so omitted fields remain unchanged in the sub-workflow.
    - Connect as `ai_tool`.

11. **Create the contact deletion tool**
    - Node type: `Tool Workflow`
    - Name: `DELETE_SUBJECT`
    - Select the sub-workflow that deletes a Fakturoid subject.
    - Define one input:
      - `subject_id` number
    - Connect as `ai_tool`.

12. **Create the invoice creation tool**
    - Node type: `Tool Workflow`
    - Name: `CREATE_INVOICE`
    - Select the sub-workflow that creates a Fakturoid invoice.
    - Define inputs:
      - `subject_id` number
      - `lines` array/json
      - `note` string
    - In the description, explicitly state that the lines array must use:
      - `name`
      - `quantity`
      - `unit_price`
    - Connect as `ai_tool`.

13. **Create the invoice search/detail tool**
    - Node type: `Tool Workflow`
    - Name: `GET_INVOICE`
    - Select the sub-workflow that searches or fetches invoice details.
    - Define inputs:
      - `Invoice_id` string
      - `query` string
      - `limit` number
    - Keep in mind the original workflow uses `Invoice_id` with capital `I`; if you rebuild from scratch, consider standardizing naming if your sub-workflow also expects it.
    - Connect as `ai_tool`.

14. **Create the invoice deletion tool**
    - Node type: `Tool Workflow`
    - Name: `DELETE_INVOICE`
    - Select the sub-workflow that deletes unpaid invoices.
    - Define input:
      - `invoice_id` number
    - In the description, state clearly that deletion should only happen after confirmation.
    - Connect as `ai_tool`.

15. **Create the invoice update tool**
    - Node type: `Tool Workflow`
    - Name: `UPDATE_INVOICE`
    - Select the sub-workflow that updates invoices.
    - Define inputs:
      - `invoice_id` number
      - `due_on` string in `YYYY-MM-DD`
      - `note` string
      - `private_note` string
      - `number` string
    - Connect as `ai_tool`.

16. **Create the invoice payment tool**
    - Node type: `Tool Workflow`
    - Name: `INVOICE_PAYMENT`
    - Select the sub-workflow that records invoice payment.
    - Define inputs:
      - `invoice_id` number
      - `paid_on` string in `YYYY-MM-DD`
      - `payment_method` string
    - Optionally document accepted values such as:
      - `bank_transfer`
      - `cash`
      - `card`
      - `paypal`
    - Connect as `ai_tool`.

17. **Add visual sticky notes for structure**
    - Add one sticky note labeled `## CONTACTS` around:
      - ARES LOOKUP
      - GET_SUBJECT
      - CREATE_SUBJECT
      - UPDATE_SUBJECT
      - DELETE_SUBJECT
    - Add another labeled `## INVOICES` around:
      - CREATE_INVOICE
      - GET_INVOICE
      - DELETE_INVOICE
      - UPDATE_INVOICE
      - INVOICE_PAYMENT
    - Add a larger setup note describing:
      - what the workflow does
      - that it uses 10 sub-workflows as tools
      - setup steps for Fakturoid credentials, LLM credentials, sub-workflow activation

18. **Configure all sub-workflows before activating the main workflow**
    - Each toolWorkflow node must point to an existing, active sub-workflow.
    - Those sub-workflows should expose inputs matching the fields defined above.
    - They should return concise, normalized outputs that the AI Agent can reason over.

19. **Configure Fakturoid access inside the sub-workflows**
    - In each Fakturoid-related sub-workflow, ensure the HTTP Request or Fakturoid nodes contain:
      - valid **Fakturoid API token**
      - correct **account slug**
    - Verify API authentication independently before testing the main workflow.

20. **Validate sub-workflow I/O contracts**
    - `GET_SUBJECT` should return contact records containing at least a usable `subject_id`.
    - `GET_INVOICE` should return invoice records containing at least a usable `invoice_id`.
    - `CREATE_INVOICE` should return invoice metadata such as ID, invoice number, and possibly a PDF/public URL.
    - `ARES LOOKUP` should return structured company details such as legal name, IČO, DIČ, and address when available.

21. **Test the workflow incrementally**
    - Start with simple lookups:
      - search a company by IČO
      - search a contact
      - search an invoice
    - Then test write operations with confirmation:
      - create contact
      - create invoice
      - update invoice
      - mark as paid
      - delete unpaid invoice

22. **Activate all sub-workflows first, then activate the main agent workflow**
    - This is important because the AI Agent depends on the tool nodes being callable.

23. **Open the chat interface and run end-to-end tests in Czech**
    - Example scenarios:
      - create client from IČO
      - create invoice for an existing client
      - change due date on an invoice
      - mark a known invoice as paid
      - delete an unpaid invoice after confirmation

## Sub-workflow setup expectations

Below is the expected role of each referenced sub-workflow:

1. **ARESCaller**
   - Inputs: `ICO`, `CompanyName`
   - Output: normalized Czech business registry data

2. **Fakturoid_get_subject_TEMPLATE**
   - Inputs: `query`
   - Output: matching subjects with `subject_id`

3. **Fakturoid_create_subject_TEMPLATE**
   - Inputs: contact fields
   - Output: created subject with `subject_id`

4. **Fakturoid_update_subject_TEMPLATE**
   - Inputs: `subject_id` + optional fields
   - Output: updated subject record

5. **Fakturoid_delete_subject_TEMPLATE**
   - Inputs: `subject_id`
   - Output: deletion confirmation/result

6. **Fakturoid_get_invoice_TEMPLATE**
   - Inputs: invoice ID or search query
   - Output: matching invoices with `invoice_id`, number, status, totals, links

7. **Fakturoid_create_invoice_TEMPLATE**
   - Inputs: `subject_id`, `lines`, optional `note`
   - Output: created invoice details including `invoice_id`

8. **Fakturoid_update_invoice_TEMPLATE**
   - Inputs: `invoice_id` + changed fields
   - Output: updated invoice

9. **Fakturoid_delete_invoice_TEMPLATE**
   - Inputs: `invoice_id`
   - Output: deletion result

10. **Fakturoid_invoice_paymentTEMPLATE**
    - Inputs: `invoice_id`, `paid_on`, `payment_method`
    - Output: payment registration result / updated invoice status

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| AI agent connected to Fakturoid API and ARES registry. Handles invoicing and contact management through natural Czech conversation — no forms, no clicking. | Workflow purpose |
| The agent can search, create, update and delete contacts, search/create/update/delete invoices, and mark invoices as paid. | Capability summary |
| The architecture uses 10 sub-workflows as tools; each handles one Fakturoid operation and returns normalized fields to the agent. | Design note |
| Add your Fakturoid API token and account slug to the Fakturoid HTTP Request nodes. | Required setup in sub-workflows |
| Connect your LLM credentials (OpenAI, Anthropic, or any compatible provider) to the AI Agent node. | Credential setup |
| Activate all sub-workflows before activating the main agent workflow. | Deployment requirement |
| Open the chat and start invoicing. | Runtime usage |