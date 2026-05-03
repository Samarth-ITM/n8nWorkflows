Triage customer support emails and draft Gmail replies with IONOS AI Model Hub

https://n8nworkflows.xyz/workflows/triage-customer-support-emails-and-draft-gmail-replies-with-ionos-ai-model-hub-14966


# Triage customer support emails and draft Gmail replies with IONOS AI Model Hub

# Workflow Documentation: Triage customer support emails and draft Gmail replies with IONOS AI Model Hub

## 1. Workflow Overview
This workflow automates the initial handling of customer support emails. It monitors a Gmail inbox, filters for specific support addresses, uses a high-capacity AI model (Llama 3.3-70B via IONOS AI Model Hub) to categorize the request and draft a response, and finally creates a Gmail draft for human review.

The logic is organized into the following functional blocks:
- **1.1 Input Reception & Filtering:** Monitors the inbox and ensures only relevant support emails are processed.
- **1.2 Data Preparation:** Cleans HTML and strips email threads to provide the AI with a concise input.
- **1.3 AI Processing:** Classifies the email by category and priority, performs sentiment analysis, and generates a draft reply.
- **1.4 Response Parsing & Routing:** Converts the AI's text response into a structured data object and routes it based on whether the email is spam or a legitimate inquiry.
- **1.5 Final Action:** Either archives the email (if spam) or creates a structured Gmail draft.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception & Filtering
**Overview:** This block acts as the entry point, polling Gmail for new messages and filtering them to avoid processing irrelevant emails.
- **Nodes Involved:** `New Email (info@ / support@)`, `Only support@ or info@`
- **Node Details:**
    - **New Email (info@ / support@)**: Gmail Trigger. Configured to poll every minute for "unread" messages.
    - **Only support@ or info@**: Filter node. Uses an OR condition to check if the `From` field contains "support@" or "info@".
    - **Edge Cases:** If the `From` field is empty or improperly formatted, the filter will fail and the email will be ignored.

### 2.2 Data Preparation
**Overview:** Transforms raw email data into a clean string to save AI tokens and improve accuracy.
- **Nodes Involved:** `Prepare Email Data`
- **Node Details:**
    - **Prepare Email Data**: Code node (JavaScript).
    - **Configuration:**
        - `stripHtml()`: Removes CSS, scripts, and HTML tags.
        - `removeQuotedText()`: Strips email signatures and previous thread history (e.g., "On... wrote:").
        - Limits the `cleanBody` to 3,500 characters.
    - **Output:** A clean JSON object containing `from`, `fromName`, `subject`, `date`, `threadId`, `messageId`, `toAddress`, and `cleanBody`.

### 2.3 AI Processing
**Overview:** Uses a Large Language Model (LLM) to analyze the email and generate a response.
- **Nodes Involved:** `AI Triage & Draft Reply`, `IONOS Cloud Chat Model`
- **Node Details:**
    - **AI Triage & Draft Reply**: Chain LLM node. It provides a strict system prompt requiring a JSON output.
        - **Logic:** Assigns one of six categories (Spam, Sales Lead, Tech Support, FAQ, Billing, Other) and one of four priority levels.
        - **Language:** Instructs the AI to reply in the same language as the incoming email.
    - **IONOS Cloud Chat Model**: AI Model node.
        - **Model:** `meta-llama/Llama-3.3-70B-Instruct`.
        - **Credential:** Requires `ionosCloudApi` token.
    - **Potential Failures:** AI might occasionally return markdown (e.g., ```json ... ```) instead of raw JSON, which is handled in the next block.

### 2.4 Response Parsing & Routing
**Overview:** Validates the AI's output and decides the next operational step.
- **Nodes Involved:** `Parse AI Response`, `Route by Category`
- **Node Details:**
    - **Parse AI Response**: Code node (JavaScript).
        - **Configuration:** Uses a Regular Expression (`/\{[\s\S]*\}/`) to extract JSON from the AI response even if the AI added conversational filler.
        - **Fallback:** If parsing fails, it provides a default "Other" category and a generic polite reply.
    - **Route by Category**: Switch node.
        - **Logic:** Routes to "Spam" if `category === 'Spam'`, otherwise routes to the default output (all other categories).

### 2.5 Final Action
**Overview:** Executes the final state change in Gmail.
- **Nodes Involved:** `Archive Spam (mark as read)`, `Create Draft Reply`
- **Node Details:**
    - **Archive Spam (mark as read)**: Gmail node. Marks the specific `messageId` as read to remove it from the unread queue.
    - **Create Draft Reply**: Gmail node.
        - **Resource:** Draft.
        - **Content:** Prepends a "🤖 AI TRIAGE" header containing the category, priority, and sentiment, followed by the AI-generated `draft_reply`.
        - **Thread Management:** Uses `threadId` to ensure the draft is linked to the original conversation.

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| New Email (info@ / support@) | Gmail Trigger | Polls for unread emails | - | Only support@ or info@ | Polls Gmail every minute for new unread messages. |
| Only support@ or info@ | Filter | Limits processing to support emails | New Email | Prepare Email Data | Polls Gmail every minute for new unread messages. Update Filter node with full addresses. |
| Prepare Email Data | Code | Cleans HTML and strips threads | Only support@ or info@ | AI Triage & Draft Reply | - |
| AI Triage & Draft Reply | Chain LLM | Analyzes email and drafts response | Prepare Email Data | Parse AI Response | Single AI call returns structured JSON (category, priority, summary, sentiment, language, draft). |
| IONOS Cloud Chat Model | IONOS Cloud Model | Provides Llama 3.3-70B intelligence | - | AI Triage & Draft Reply | Single AI call returns structured JSON. |
| Parse AI Response | Code | Validates and cleans AI JSON | AI Triage & Draft Reply | Route by Category | - |
| Route by Category | Switch | Directs flow based on category | Parse AI Response | Archive Spam / Create Draft | - |
| Archive Spam (mark as read) | Gmail | Marks spam as read | Route by Category | - | - |
| Create Draft Reply | Gmail | Creates a reviewable Gmail draft | Route by Category | - | - |

---

## 4. Reproducing the Workflow from Scratch

### Step 1: Initial Setup & Credentials
1. **Install Node Package:** Ensure the `@ionos-cloud/n8n-nodes-ionos-cloud` package is installed in your n8n instance.
2. **Gmail Credentials:** Set up an OAuth2 connection for the Gmail account used for `support@` or `info@`.
3. **IONOS Credentials:** Create a credential for `ionosCloudApi` using your IONOS Cloud API token.

### Step 2: Input & Filtering
1. Create a **Gmail Trigger** node: Set "Read Status" to `unread` and "Poll Times" to every 1 minute.
2. Create a **Filter** node: Add two conditions using the `OR` combinator.
    - Condition 1: `{{ $json.From }}` contains `support@`
    - Condition 2: `{{ $json.From }}` contains `info@`

### Step 3: Data Cleaning
1. Create a **Code** node named "Prepare Email Data". 
2. Paste the JavaScript provided in the workflow to handle HTML stripping and quoted text removal. Ensure it outputs the `cleanBody`, `fromName`, and `threadId`.

### Step 4: AI Configuration
1. Create a **Chain LLM** node.
2. Attach an **IONOS Cloud Chat Model** node to it.
    - Select Model: `meta-llama/Llama-3.3-70B-Instruct`.
3. In the Chain LLM "Text" field, enter the prompt defining the categories (Spam, Sales Lead, etc.), priority levels, and the required JSON structure.

### Step 5: Parsing and Routing
1. Create a **Code** node named "Parse AI Response". Use JavaScript to extract the JSON object from the AI output using regex and merge it with the data from the "Prepare Email Data" node.
2. Create a **Switch** node. 
    - Set a rule: if `{{ $json.category }}` equals `Spam`, output to Route 0.
    - All other cases go to the "fallback" (extra) output.

### Step 6: Gmail Actions
1. **For Route 0 (Spam):** Create a **Gmail** node. Operation: `markAsRead`. Use `{{ $json.messageId }}`.
2. **For Fallback (Legit):** Create a **Gmail** node. Resource: `draft`. 
    - **Subject:** `Re: {{ $json.subject }}`
    - **Thread ID:** `{{ $json.threadId }}`
    - **Message:** Create a template including the AI metadata (Category, Priority, etc.) and the `{{ $json.draft_reply }}`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Sovereign AI — IONOS AI Model Hub | The workflow utilizes Llama 3.3-70B for high-quality reasoning and data privacy. |
| Model Hub Installation | Requires the installation of the community node package `@ionos-cloud/n8n-nodes-ionos-cloud`. |
| Domain Update | Remember to replace the generic `support@` and `info@` in the Filter node with your specific corporate email addresses. |