Create AI-Summarized Email Digests from Gmail Labels with OpenAI O4-Mini

https://n8nworkflows.xyz/workflows/create-ai-summarized-email-digests-from-gmail-labels-with-openai-o4-mini-5839


# Create AI-Summarized Email Digests from Gmail Labels with OpenAI O4-Mini

### 1. Workflow Overview

This workflow automates the creation and sending of a daily email digest containing AI-generated summaries of emails labeled in a Gmail account. It is designed to help users efficiently consume multiple emails from specific categories (such as newsletters or news feeds) by summarizing their content into concise, well-formatted HTML digests. The workflow runs on a daily schedule, fetches emails from the last 24 hours under a specific Gmail label, processes each email with OpenAI's "o4-mini" model to generate summaries, combines these summaries, and sends the digest as one consolidated email.

Logical blocks:

- **1.1 Scheduled Email Retrieval:** Trigger the workflow daily, fetch labeled emails from Gmail received in the last 24 hours, and conditionally proceed if emails exist.
- **1.2 AI-Powered Email Summarization:** Process email content using LangChain nodes with OpenAI’s "o4-mini" chat model, including document loading, text splitting, and summarization chains.
- **1.3 Content Formatting and Email Delivery:** Extract key fields, combine all summaries into a single HTML digest with structured headings, and send the digest via Gmail.
- **1.4 Conditional Processing:** Skip summarization and sending if no emails are found to avoid empty digests.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Email Retrieval

**Overview:**  
Runs daily at 9 AM, retrieves up to 20 emails from a specified Gmail label received in the last 24 hours, and filters the flow based on availability of emails.

**Nodes Involved:**  
- Schedule Trigger  
- Get mails (last 24h)  
- If  
- No Operation, do nothing

**Node Details:**

- **Schedule Trigger**  
  - Type: Core Trigger  
  - Role: Starts workflow daily at 9:00 AM  
  - Config: Interval trigger set to trigger at 9 AM every day  
  - Inputs: None  
  - Outputs: Connects to "Get mails (last 24h)"  
  - Edge cases: Workflow only triggers at scheduled time; no fallback for missed runs.

- **Get mails (last 24h)**  
  - Type: Gmail node  
  - Role: Fetch emails with a specific Gmail label received in the last 24 hours  
  - Config:  
    - Limit: 20 emails  
    - Filters: Label IDs set to `YOUR_LABEL_ID` (user must replace with actual label)  
    - Received after: dynamic expression for current time minus 1 day  
    - Operation: getAll (fetch all matching emails up to limit)  
  - Inputs: From Schedule Trigger  
  - Outputs: Connects to "If" node  
  - Requirements: Gmail OAuth2 credentials configured  
  - Edge cases: Invalid label ID or authentication failure; no emails found.

- **If**  
  - Type: Conditional node  
  - Role: Checks if any emails were retrieved (length > 0)  
  - Config: Condition: number of items greater than 0  
  - Inputs: From "Get mails (last 24h)"  
  - Outputs:  
    - True: proceeds to "Summarization Mails"  
    - False: proceeds to "No Operation, do nothing"  
  - Edge cases: Expression evaluation failures; empty email list.

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Role: Terminates workflow path gracefully if no emails found  
  - Inputs: From "If" (false branch)  
  - Outputs: None  
  - Edge cases: None

---

#### 1.2 AI-Powered Email Summarization

**Overview:**  
Processes each retrieved email’s content by loading it as a document, splitting the text optimally, and feeding it through a summarization chain powered by OpenAI's "o4-mini" chat model with a custom prompt that generates HTML summaries preserving external links.

**Nodes Involved:**  
- OpenAI Chat Model  
- Recursive Character Text Splitter  
- Default Data Loader  
- Summarization Mails

**Node Details:**

- **OpenAI Chat Model**  
  - Type: LangChain AI Language Model node  
  - Role: Provides access to OpenAI’s "o4-mini" chat model for text generation  
  - Config: Model set to "o4-mini" (optimized smaller model)  
  - Inputs: Connected as ai_languageModel input to "Summarization Mails"  
  - Outputs: AI-generated summaries  
  - Credentials: OpenAI API key required  
  - Edge cases: API rate limits, authentication errors, model errors.

- **Recursive Character Text Splitter**  
  - Type: LangChain Text Splitter node  
  - Role: Splits email text recursively into chunks sized to the whole text length (effectively no splitting)  
  - Config: Chunk size set dynamically to the length of the email text (disables splitting)  
  - Inputs: From "Summarization Mails" (ai_textSplitter input)  
  - Outputs: Passes split text to "Default Data Loader"  
  - Edge cases: Large text may cause timeouts or memory issues.

- **Default Data Loader**  
  - Type: LangChain Document Data Loader node  
  - Role: Loads the JSON email text into a document format for processing  
  - Config: Uses dynamic expression to load text from email JSON field `text`  
  - Inputs: From "Recursive Character Text Splitter" (ai_document input)  
  - Outputs: Document passed to "Summarization Mails"  
  - Edge cases: Missing or malformed text field.

- **Summarization Mails**  
  - Type: LangChain Summarization Chain node  
  - Role: Executes summarization on loaded documents using OpenAI chat model and a detailed prompt  
  - Config:  
    - Summarization method: "stuff" (concatenate all texts)  
    - Prompt includes instructions for:  
      - Concise summaries  
      - Bullet points for multiple topics  
      - Preservation and numbering of external links in HTML format  
      - Output formatted with `<p>`, `<ul><li>`, `<a href>` tags  
    - Example output provided in prompt for clarity  
  - Inputs: Receives ai_languageModel, ai_document, and ai_textSplitter from respective nodes  
  - Outputs: Summarized text passed to "Edit Fields"  
  - Edge cases: Prompt misinterpretation, API failures, empty input.

---

#### 1.3 Content Formatting and Email Delivery

**Overview:**  
Extracts necessary fields from summarized emails, combines all summaries into a single HTML digest with headings identifying each email’s subject and sender, and sends the digest via Gmail.

**Nodes Involved:**  
- Edit Fields  
- Combine Subject and Body  
- Send Digested mail

**Node Details:**

- **Edit Fields**  
  - Type: Set node  
  - Role: Extracts and formats key fields for each summarized email item  
  - Config:  
    - Assigns `output.text` from summary text in JSON output  
    - Assigns `subject` from the original email’s subject field (`Get mails (last 24h)`)  
    - Assigns `headers.from` from the original email’s sender field (`If` node)  
  - Inputs: From "Summarization Mails"  
  - Outputs: To "Combine Subject and Body"  
  - Edge cases: Missing fields in original email JSON.

- **Combine Subject and Body**  
  - Type: Code node  
  - Role: Merges all processed email summaries into one HTML string digest  
  - Config: JavaScript code that:  
    - Iterates all input items  
    - Extracts and cleans sender (removes "From: " prefix if present)  
    - Wraps each summary in `<h3>` with subject and sender, followed by summary text  
    - Concatenates all into one combinedContent string  
    - Outputs combined content and itemCount as JSON  
  - Inputs: From "Edit Fields"  
  - Outputs: To "Send Digested mail"  
  - Edge cases: HTML injection risks if input is not sanitized; empty inputs.

- **Send Digested mail**  
  - Type: Gmail node  
  - Role: Sends the final digest email to the user’s email address  
  - Config:  
    - Recipient email: user must replace `"your-email@example.com"` with their address  
    - Subject: dynamically set to `"Daily Tech-News Digest for YYYY-MM-DD"` (current date)  
    - Message body: uses combinedContent HTML from previous node  
  - Inputs: From "Combine Subject and Body"  
  - Outputs: None  
  - Credentials: Gmail OAuth2 required  
  - Edge cases: Authentication failures, email sending limits, invalid recipient.

---

#### 1.4 Conditional Processing

**Overview:**  
Ensures no empty digests are sent by skipping summarization and sending if no emails were retrieved.

**Nodes Involved:**  
- If  
- No Operation, do nothing

**Node Details:**  
Covered in section 1.1 but important as a separate logical guard.

---

### 3. Summary Table

| Node Name              | Node Type                                   | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                            |
|------------------------|---------------------------------------------|----------------------------------------|-----------------------------|----------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger       | n8n-nodes-base.scheduleTrigger              | Trigger workflow daily at 9 AM          | None                        | Get mails (last 24h)        | ## 1. Scheduled Email Retrieval [Read more about Schedule Trigger](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/) [Read more about Gmail node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) The workflow runs daily at 9 AM and fetches emails from a specific Gmail label received in the last 24 hours. Configure your desired label in the Gmail node to categorize which emails should be included in your digest. |
| Get mails (last 24h)    | n8n-nodes-base.gmail                        | Fetch labeled emails from last 24h     | Schedule Trigger            | If                         | See above                                                                                                             |
| If                     | n8n-nodes-base.if                           | Check if emails were found              | Get mails (last 24h)         | Summarization Mails, No Operation, do nothing | ## Smart Processing [Read more about If node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/) The workflow intelligently checks if any emails were found. If no emails match the criteria, it skips processing to avoid sending empty digests. |
| No Operation, do nothing| n8n-nodes-base.noOp                         | Skip processing if no emails            | If (false branch)            | None                       | See above                                                                                                             |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi      | Provides OpenAI "o4-mini" model         | None (ai_languageModel input) | Summarization Mails (ai_languageModel input) | ## 2. AI-Powered Email Summarization [Read more about LangChain nodes](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainSummarization/) Each email is processed through OpenAI's language model to create concise, readable summaries. The workflow loads email content, splits text, uses a custom prompt to generate HTML-formatted summaries preserving links. |
| Recursive Character Text Splitter | @n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter | Splits email text chunks for AI processing | Summarization Mails (ai_textSplitter input) | Default Data Loader (ai_textSplitter output) | See above                                                                                                             |
| Default Data Loader     | @n8n/n8n-nodes-langchain.documentDefaultDataLoader | Loads email text as document for AI     | Recursive Character Text Splitter (ai_document input) | Summarization Mails (ai_document input) | See above                                                                                                             |
| Summarization Mails     | @n8n/n8n-nodes-langchain.chainSummarization | Generates AI summaries from email content | If (true branch) and AI nodes | Edit Fields                 | See above                                                                                                             |
| Edit Fields             | n8n-nodes-base.set                          | Extracts and formats key fields from summaries | Summarization Mails          | Combine Subject and Body    | ## 3. Content Formatting and Email Delivery [Read more about Code node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/) Summarized emails are combined into a single digest with proper headers and formatting. |
| Combine Subject and Body| n8n-nodes-base.code                         | Combines all summaries into one HTML digest | Edit Fields                 | Send Digested mail          | See above                                                                                                             |
| Send Digested mail      | n8n-nodes-base.gmail                        | Sends final summarized digest email    | Combine Subject and Body     | None                       | See above                                                                                                             |
| Main Workflow Explanation | n8n-nodes-base.stickyNote                 | Documentation and overview              | None                        | None                       | ## Daily Email Digest with AI Summarization This n8n workflow automatically creates AI-powered summaries of labeled emails and sends them as a daily digest. Perfect for staying on top of newsletters, news feeds, or any categorized emails without information overload! … [Discord link](https://discord.com/invite/XPKeKXeB7d) [Forum link](https://community.n8n.io/) |
| Email Retrieval Section | n8n-nodes-base.stickyNote                   | Documentation for scheduled email retrieval | None                        | None                       | See Schedule Trigger note                                                                                              |
| AI Processing Section   | n8n-nodes-base.stickyNote                   | Documentation for AI summarization block | None                        | None                       | See OpenAI Chat Model note                                                                                            |
| Formatting and Output Section | n8n-nodes-base.stickyNote               | Documentation for formatting and email sending | None                        | None                       | See Edit Fields note                                                                                                  |
| Conditional Processing  | n8n-nodes-base.stickyNote                   | Documentation for conditional execution | None                        | None                       | See If node note                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node**  
   - Type: `Schedule Trigger`  
   - Set interval to daily at 9:00 AM  
   - This triggers the workflow once per day.

3. **Add a Gmail node named "Get mails (last 24h)"**  
   - Operation: `getAll`  
   - Limit: 20  
   - Filters:  
     - Label IDs: Replace `"YOUR_LABEL_ID"` with the actual Gmail label ID you want to summarize  
     - Received After: Set to expression `{{$now.minus({days: 1}).toISO()}}` to fetch emails from the last 24 hours  
   - Connect Schedule Trigger output to this node’s input.  
   - Configure Gmail OAuth2 credentials.

4. **Add an If node named "If"**  
   - Condition: Check if the number of incoming items is greater than 0  
     - Use expression: `{{$items().length}} > 0`  
   - Connect "Get mails (last 24h)" output to "If".  
   - True branch proceeds to summarization nodes; False branch proceeds to a No Operation node.

5. **Add a No Operation node named "No Operation, do nothing"**  
   - Connect "If" node’s false output to this node.  
   - This ends processing gracefully if no emails found.

6. **Add LangChain nodes for AI summarization:**  
   a. **OpenAI Chat Model node**  
      - Model: Select `o4-mini` as the language model  
      - Configure OpenAI API credentials.  
   b. **Recursive Character Text Splitter node**  
      - Chunk Size: Set expression to `{{$json.text.length}}` (disables splitting)  
   c. **Default Data Loader node**  
      - JSON Data: Set expression to `{{$json.text}}` to load email text  
   d. **Summarization Mails node (Chain Summarization)**  
      - Summarization method: "stuff"  
      - Prompt: Use the detailed HTML summarization prompt provided, instructing bullet points, link preservation, and HTML formatting  
      - Connect inputs:  
        - ai_languageModel input from "OpenAI Chat Model"  
        - ai_textSplitter input from "Recursive Character Text Splitter"  
        - ai_document input from "Default Data Loader"  
      - Connect "If" node’s true output to this node.

7. **Add a Set node named "Edit Fields"**  
   - Assign fields:  
     - `output.text` = `{{$json.output.text}}` (summary text)  
     - `subject` = `{{$node["Get mails (last 24h)"].item.json.subject}}` (original email subject)  
     - `headers.from` = `{{$node["If"].item.json.headers.from}}` (original email sender)  
   - Connect "Summarization Mails" output to this node.

8. **Add a Code node named "Combine Subject and Body"**  
   - Paste the provided JavaScript code that:  
     - Iterates over all input items  
     - Extracts and cleans sender info  
     - Creates HTML `<h3>` headings with subject and sender  
     - Combines all summaries with headings into a single HTML string  
     - Outputs combinedContent and itemCount  
   - Connect "Edit Fields" output to this node.

9. **Add a Gmail node named "Send Digested mail"**  
   - Operation: Send Email  
   - To: Replace `"your-email@example.com"` with your email address  
   - Subject: Use expression `Daily Tech-News Digest for {{$now.toISODate()}}`  
   - Message: Use expression `{{$json.combinedContent}}` (HTML body)  
   - Configure Gmail OAuth2 credentials.  
   - Connect "Combine Subject and Body" output to this node.

10. **Verify all connections:**  
    - Schedule Trigger → Get mails (last 24h) → If  
    - If (true) → OpenAI Chat Model, Recursive Character Text Splitter, Default Data Loader → Summarization Mails → Edit Fields → Combine Subject and Body → Send Digested mail  
    - If (false) → No Operation, do nothing

11. **Test the workflow with actual Gmail label and OpenAI API credentials.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is ideal for content creators, professionals, or anyone needing to reduce email overload by generating concise daily digests of labeled emails.                                                                                                                                                                                                                                                                                                                                                                | General use case                                                                                   |
| For help and community support, join the n8n Discord: https://discord.com/invite/XPKeKXeB7d or visit the n8n Forum: https://community.n8n.io/                                                                                                                                                                                                                                                                                                                                                                                    | Support links                                                                                      |
| The summarization prompt is designed to produce clean HTML output including bullet points and preserved external links, making the digest easily readable and navigable. Modifying the prompt allows customization of summary style and format.                                                                                                                                                                                                                                                                               | Summarization customization                                                                        |
| Ensure Gmail OAuth2 credentials have appropriate permissions to read emails and send mail. Label ID for Gmail must be obtained from Gmail settings or API.                                                                                                                                                                                                                                                                                                                                                                    | Credentials and Gmail setup                                                                        |
| Relevant Documentation: Schedule Trigger: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.scheduletrigger/ Gmail Node: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/ LangChain Summarization Node: https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.chainSummarization/ Code Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.code/ If Node: https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.if/ | Official n8n Documentation Links                                                                   |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, adhering strictly to applicable content policies, containing no illegal or offensive elements. All processed data is legal and public.