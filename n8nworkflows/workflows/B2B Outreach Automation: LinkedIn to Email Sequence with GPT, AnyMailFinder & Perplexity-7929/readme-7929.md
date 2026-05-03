B2B Outreach Automation: LinkedIn to Email Sequence with GPT, AnyMailFinder & Perplexity

https://n8nworkflows.xyz/workflows/b2b-outreach-automation--linkedin-to-email-sequence-with-gpt--anymailfinder---perplexity-7929


# B2B Outreach Automation: LinkedIn to Email Sequence with GPT, AnyMailFinder & Perplexity

### 1. Workflow Overview

This workflow automates a B2B outreach sequence that bridges LinkedIn prospecting with personalized email campaigns, leveraging AI for content generation and external services for contact enrichment and data validation. It targets sales and marketing professionals who want to streamline their lead generation and outreach processes by integrating LinkedIn messaging, email address discovery, and AI-based message personalization.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Retrieval:** Triggering the workflow manually and retrieving prospect data from a Google Sheet.
- **1.2 Data Preparation & Credential Setup:** Setting up initial parameters and credentials for subsequent API calls.
- **1.3 Contact Enrichment & Validation:** Using HTTP requests and external services (e.g., AnyMailFinder or similar) to find and verify email addresses linked to LinkedIn profiles.
- **1.4 AI Content Generation:** Employing OpenAI and Perplexity AI models to generate personalized outreach messages and email templates.
- **1.5 Outreach Execution:** Sending LinkedIn messages and managing email sequence sending.
- **1.6 Data Update & Loop Control:** Updating the Google Sheet with results and iterating over batches of prospect data with controlled delays.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Retrieval

- **Overview:** This block initiates the workflow manually and fetches the list of leads from a Google Sheet to be processed.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’`  
  - `Get row(s) in sheet1`  
  - `Set input1`

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Starts the workflow on user command.  
    - Configuration: No parameters; manual trigger only.  
    - Input: None  
    - Output: Triggers next node (`Get row(s) in sheet1`)  
    - Edge Cases: Workflow will not run automatically; requires manual start.

  - **Get row(s) in sheet1**  
    - Type: Google Sheets (Read)  
    - Role: Reads prospect data from a configured Google Sheet (e.g., LinkedIn profiles, names).  
    - Configuration: Likely configured with the Google Sheets node to fetch all or filtered rows.  
    - Inputs: Trigger node output  
    - Outputs: Data rows for further processing.  
    - Edge Cases: Possible authentication errors or empty sheet results.

  - **Set input1**  
    - Type: Set node  
    - Role: Prepares or formats the input data for the next steps, possibly extracting specific fields or setting variables.  
    - Configuration: Custom expressions or static values to shape data.  
    - Inputs: Rows from Google Sheets  
    - Outputs: Formatted data for credential setup.

---

#### 2.2 Data Preparation & Credential Setup

- **Overview:** Sets credentials and parameters needed for API calls and subsequent nodes, and prepares an email template.
- **Nodes Involved:**  
  - `Set credentials`  
  - `Set Email Template`  
  - `Limit1`  
  - `Loop Over Items1`

- **Node Details:**

  - **Set credentials**  
    - Type: Set node  
    - Role: Defines API keys, tokens, or other credentials required downstream, likely stored in environment variables or workflow parameters.  
    - Configuration: Contains expressions or static credentials for OpenAI, LinkedIn API, or email finder services.  
    - Inputs: Formatted data from `Set input1`  
    - Outputs: Data enriched with credentials.

  - **Set Email Template**  
    - Type: Set node  
    - Role: Defines the base email template or structure used for AI to personalize messages.  
    - Configuration: Static or dynamic text set in parameters.  
    - Inputs: Credential-enhanced data  
    - Outputs: Data passed to `Limit1` node.

  - **Limit1**  
    - Type: Limit node  
    - Role: Limits the number of items processed per batch (commonly used to control API rate limits).  
    - Configuration: Limit count set (e.g., 1 or N).  
    - Inputs: Email template data  
    - Outputs: Batches data for splitting.

  - **Loop Over Items1**  
    - Type: SplitInBatches  
    - Role: Loops over the limited items one batch at a time for controlled processing.  
    - Configuration: Batch size configured (usually 1).  
    - Inputs: Limited data  
    - Outputs: Single batch item for processing; supports loop with delay.

---

#### 2.3 Contact Enrichment & Validation

- **Overview:** Calls external services to find and verify email addresses associated with LinkedIn profiles and decides next steps based on success.
- **Nodes Involved:**  
  - `Wait`  
  - `HTTP Request`  
  - `If3`  
  - `Update no find email`

- **Node Details:**

  - **Wait**  
    - Type: Wait node  
    - Role: Implements delay between batches or API calls to respect rate limits.  
    - Configuration: Time duration set (e.g., seconds or minutes delay).  
    - Inputs: Empty batch outputs (second output of `Loop Over Items1`)  
    - Outputs: Triggers HTTP Request after delay.

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Role: Calls an email-finding or verification API (e.g., AnyMailFinder or similar) using data from the batch.  
    - Configuration: Method, URL, headers, and body set with API credentials and parameters.  
    - Inputs: Triggered after wait or first batch iteration.  
    - Outputs: API response with email data.

  - **If3**  
    - Type: If node  
    - Role: Checks if an email was found in the HTTP Request response.  
    - Configuration: Condition based on presence or validity of email in previous node output.  
    - Inputs: Response from HTTP Request  
    - Outputs:  
      - True: Email found → proceeds to AI content generation.  
      - False: Email not found → updates Google Sheet accordingly.

  - **Update no find email**  
    - Type: Google Sheets (Update)  
    - Role: Marks or updates the prospect row indicating no valid email was found.  
    - Configuration: Uses row ID or unique key to update relevant column/cell.  
    - Inputs: False branch from `If3`  
    - Outputs: Triggers next batch iteration.

---

#### 2.4 AI Content Generation

- **Overview:** Uses AI models to generate personalized LinkedIn messages and email content based on prospect data and templates.
- **Nodes Involved:**  
  - `Message a model` (Perplexity AI)  
  - `Personal LinkedIn Account POST`  
  - `Code`  
  - `OpenAI Chat Model5`  
  - `Email`  
  - `OpenAI Chat Model4`  
  - `Structured Output Parser2`

- **Node Details:**

  - **Message a model**  
    - Type: Perplexity AI node  
    - Role: Generates a personalized LinkedIn message for the prospect using Perplexity AI.  
    - Configuration: Input prompt dynamically constructed with prospect data.  
    - Inputs: True branch from `If3`  
    - Outputs: Message text to LinkedIn POST node.

  - **Personal LinkedIn Account POST**  
    - Type: HTTP Request  
    - Role: Sends the generated LinkedIn message through LinkedIn API or a proxy endpoint.  
    - Configuration: POST method, with message body and authentication headers.  
    - Inputs: Output from `Message a model`  
    - Outputs: Response to `Code` node.

  - **Code**  
    - Type: Code node (JavaScript)  
    - Role: Processes LinkedIn POST response, possibly extracting status or preparing input for email generation.  
    - Configuration: Custom script to parse response and prepare prompt for OpenAI.  
    - Inputs: POST response  
    - Outputs: Prompt for OpenAI Chat Model5.

  - **OpenAI Chat Model5**  
    - Type: OpenAI Chat model  
    - Role: Generates the personalized email body or sequence content using OpenAI models.  
    - Configuration: Model name, temperature, max tokens, prompt formatting.  
    - Inputs: Prompt from `Code` node  
    - Outputs: Email content to `Email` node.

  - **Email**  
    - Type: LangChain LLM Chain node  
    - Role: Processes and finalizes the email content, possibly applying further formatting or templating.  
    - Configuration: Chain setup with prompt template and model.  
    - Inputs: Output from OpenAI Chat Model5 and from Structured Output Parser2  
    - Outputs: Data to update Google Sheet.

  - **OpenAI Chat Model4**  
    - Type: OpenAI Chat model  
    - Role: May generate structured data or auxiliary content (e.g., subject lines, summaries).  
    - Configuration: Parameters set for output style.  
    - Inputs: Data from previous nodes (not fully shown)  
    - Outputs: To `Structured Output Parser2`.

  - **Structured Output Parser2**  
    - Type: LangChain output parser  
    - Role: Parses structured responses from OpenAI into JSON or other usable formats.  
    - Configuration: Parsing rules, expected output format.  
    - Inputs: Output from OpenAI Chat Model4  
    - Outputs: Structured data to `Email` node.

---

#### 2.5 Outreach Execution & Data Update

- **Overview:** Sends the generated email content, updates the Google Sheet with final statuses, and loops back for batch processing.
- **Nodes Involved:**  
  - `Update Final`  
  - `Loop Over Items1` (second output)

- **Node Details:**

  - **Update Final**  
    - Type: Google Sheets (Update)  
    - Role: Updates the prospect row with the final email content and outreach status.  
    - Configuration: Uses row identifiers and updates relevant columns with results.  
    - Inputs: Output from `Email` node  
    - Outputs: Connects back to `Loop Over Items1` to continue batches.

  - **Loop Over Items1** (second output)  
    - Type: SplitInBatches continuation  
    - Role: Controls looping over the dataset, either continuing or ending the workflow.  
    - Configuration: Uses batch completion signal to trigger next batch or end.

---

### 3. Summary Table

| Node Name                   | Node Type                        | Functional Role                         | Input Node(s)              | Output Node(s)               | Sticky Note                              |
|-----------------------------|---------------------------------|---------------------------------------|----------------------------|-----------------------------|-----------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                  | Starts workflow manually               | None                       | Get row(s) in sheet1        |                                         |
| Get row(s) in sheet1        | Google Sheets                   | Fetch prospect data                    | When clicking ‘Execute workflow’ | Set input1                  |                                         |
| Set input1                  | Set                            | Prepare input data                     | Get row(s) in sheet1        | Set credentials             |                                         |
| Set credentials             | Set                            | Set API credentials                    | Set input1                  | Set Email Template          |                                         |
| Set Email Template          | Set                            | Define email template                  | Set credentials             | Limit1                     |                                         |
| Limit1                     | Limit                          | Limit batch size                       | Set Email Template          | Loop Over Items1            |                                         |
| Loop Over Items1            | SplitInBatches                 | Process batches of items               | Limit1                     | Wait (second output), empty (first output) |                                         |
| Wait                       | Wait                           | Delay between batches                  | Loop Over Items1 (second output) | HTTP Request               |                                         |
| HTTP Request               | HTTP Request                   | Email finder API call                  | Wait                       | If3                        |                                         |
| If3                        | If                             | Check if email found                   | HTTP Request               | Message a model (true), Update no find email (false) |                                         |
| Update no find email       | Google Sheets                   | Update sheet if no email found        | If3 (false)                | Loop Over Items1            |                                         |
| Message a model            | Perplexity AI                   | Generate personalized LinkedIn message| If3 (true)                 | Personal LinkedIn Account POST |                                         |
| Personal LinkedIn Account POST | HTTP Request                  | Send LinkedIn message                  | Message a model            | Code                       |                                         |
| Code                       | Code                           | Process LinkedIn response, prepare prompt | Personal LinkedIn Account POST | OpenAI Chat Model5          |                                         |
| OpenAI Chat Model5         | OpenAI Chat Model              | Generate personalized email content   | Code                       | Email                      |                                         |
| Email                      | LangChain LLM Chain            | Finalize email content                 | OpenAI Chat Model5, Structured Output Parser2 | Update Final               |                                         |
| OpenAI Chat Model4         | OpenAI Chat Model              | Generate structured auxiliary content | (Not explicitly connected) | Structured Output Parser2   |                                         |
| Structured Output Parser2  | Output Parser                  | Parse structured AI output             | OpenAI Chat Model4          | Email                      |                                         |
| Update Final               | Google Sheets                   | Update sheet with final outreach status | Email                      | Loop Over Items1            |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a manual trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No parameters.

2. **Add a Google Sheets node:**  
   - Name: `Get row(s) in sheet1`  
   - Operation: Read rows from a configured Google Sheet containing prospect data.  
   - Connect output of manual trigger to this node.

3. **Add a Set node:**  
   - Name: `Set input1`  
   - Purpose: Format or extract necessary fields from Google Sheets data for API calls.  
   - Connect output of Google Sheets node here.

4. **Add another Set node:**  
   - Name: `Set credentials`  
   - Purpose: Define credentials and tokens for LinkedIn API, email finder services, and OpenAI.  
   - Include environment variables or static keys for API authentication.  
   - Connect output of `Set input1` here.

5. **Add Set node for email template:**  
   - Name: `Set Email Template`  
   - Purpose: Store or define the base email template text for AI to personalize.  
   - Connect output of `Set credentials` here.

6. **Add a Limit node:**  
   - Name: `Limit1`  
   - Purpose: Limit number of items processed per batch (e.g., set to 1).  
   - Connect output of `Set Email Template` here.

7. **Add SplitInBatches node:**  
   - Name: `Loop Over Items1`  
   - Purpose: Process items one batch at a time for rate limiting.  
   - Batch size: 1  
   - Connect output of `Limit1` node here.

8. **Add Wait node:**  
   - Name: `Wait`  
   - Purpose: Delay between batches to respect API limits.  
   - Configure delay time (e.g., 10 seconds).  
   - Connect second output (continue) of `Loop Over Items1` to this node.

9. **Add HTTP Request node:**  
   - Name: `HTTP Request`  
   - Purpose: Call email finder API (e.g., AnyMailFinder) to retrieve email address using LinkedIn profile data.  
   - Configure URL, method, headers (with API key), and body with required parameters.  
   - Connect output of `Wait` node and first output of `Loop Over Items1` here.

10. **Add If node:**  
    - Name: `If3`  
    - Purpose: Check if the email was found in the HTTP Request response.  
    - Condition: Check response field for email presence.  
    - Connect output of `HTTP Request` here.

11. **Add Google Sheets Update node:**  
    - Name: `Update no find email`  
    - Purpose: Update prospect row to mark no email found.  
    - Connect false output of `If3` here.  
    - Connect output back to `Loop Over Items1` to continue looping.

12. **Add Perplexity AI node:**  
    - Name: `Message a model`  
    - Purpose: Generate personalized LinkedIn message via Perplexity AI.  
    - Connect true output of `If3` here.

13. **Add HTTP Request node:**  
    - Name: `Personal LinkedIn Account POST`  
    - Purpose: Post LinkedIn message using LinkedIn API or a proxy.  
    - Configure POST method, headers with auth, message body from Perplexity output.  
    - Connect output of `Message a model` here.

14. **Add Code node:**  
    - Name: `Code`  
    - Purpose: Process LinkedIn API response and prepare prompt for OpenAI email generation.  
    - Write JavaScript to parse response and build prompt text.  
    - Connect output of `Personal LinkedIn Account POST` here.

15. **Add OpenAI Chat Model node:**  
    - Name: `OpenAI Chat Model5`  
    - Purpose: Generate personalized email content using OpenAI.  
    - Configure model parameters (model name, temperature, max tokens).  
    - Connect output of `Code` node here.

16. **Add LangChain LLM Chain node:**  
    - Name: `Email`  
    - Purpose: Format and finalize email content.  
    - Configure chain with prompt template and link with OpenAI Chat Model5 output.  
    - Connect output of OpenAI Chat Model5 here.

17. **Add OpenAI Chat Model node:**  
    - Name: `OpenAI Chat Model4`  
    - Purpose: Generate auxiliary structured content (e.g., subject lines).  
    - Connect as needed based on prompt requirements.

18. **Add Output Parser node:**  
    - Name: `Structured Output Parser2`  
    - Purpose: Parse structured response from OpenAI Chat Model4 to usable JSON.  
    - Connect output of OpenAI Chat Model4 here.  
    - Connect output of this parser to the `Email` node as a secondary input.

19. **Add Google Sheets Update node:**  
    - Name: `Update Final`  
    - Purpose: Update Google Sheet with final email content and outreach status.  
    - Connect output of `Email` node here.  
    - Connect output back to `Loop Over Items1` to continue looping.

20. **Ensure all connections are properly linked:**  
    - Manual trigger → Google Sheets read → data prep → credential setting → email template → limit → loop over items → wait → HTTP request → if email found → AI message generation → LinkedIn post → code → OpenAI email generation → email formatting → Google Sheets update → loop continuation.

21. **Configure credentials for:**  
    - Google Sheets API access  
    - LinkedIn API or proxy service access (OAuth2 or API key)  
    - Email finder API (API key)  
    - OpenAI API (API key)  
    - Perplexity AI credentials

22. **Set workflow parameters and environment variables as needed.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow integrates LinkedIn messaging, AI content creation, and email finder APIs for B2B outreach automation. | Workflow description and use case.                                                                      |
| Ensure API rate limits are respected by adjusting `Wait` node delay and batch size in `Limit1`.     | Prevents request throttling and failures.                                                               |
| Credentials must be securely stored in n8n credentials manager and referenced in respective nodes. | Security best practice for API keys and tokens.                                                         |
| For LinkedIn API or messaging, a proxy or approved app with correct permissions is required.         | LinkedIn API access is restrictive; consider third-party tools or manual approval.                      |
| OpenAI nodes use LangChain integration for structured AI tasks; learn more: https://js.langchain.com/ | Useful for building complex AI workflows with n8n.                                                     |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to valid content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.