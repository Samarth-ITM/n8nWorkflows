Smart Shopify Agent: AI-Powered Abandoned Cart Recovery

https://n8nworkflows.xyz/workflows/smart-shopify-agent--ai-powered-abandoned-cart-recovery-4396


# Smart Shopify Agent: AI-Powered Abandoned Cart Recovery

### 1. Workflow Overview

The **Smart Shopify Agent: AI-Powered Abandoned Cart Recovery** workflow automates the process of recovering sales from customers who abandon their shopping carts on a Shopify store. It targets abandoned checkouts, waits to allow customers to complete purchases naturally, then rechecks abandonment status. If the cart is still abandoned, it generates a personalized recovery email using AI, sends the email, and logs the activity for tracking.

The workflow is logically divided into four main blocks:

- **1.1 Initialization & Initial Checkout Fetch:** Triggering the workflow and obtaining the initial list of abandoned checkouts from Shopify.
- **1.2 Grace Period Wait & Recheck:** Pausing for a grace period (1 hour) then fetching the updated abandoned checkout list.
- **1.3 Conditional Decision on Abandonment Status:** Comparing initial and updated lists to determine whether the customer still has an abandoned cart.
- **1.4 Personalized Recovery Email Flow:** If abandonment persists, generating, sending, and logging a personalized AI-crafted recovery email.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Initialization & Initial Checkout Fetch

**Overview:**  
This block starts the workflow manually or on schedule and fetches the current list of abandoned checkouts from the Shopify store via API request. This initial data forms the baseline for subsequent comparison to identify genuine abandoned carts.

**Nodes Involved:**  
- Start Workflow  
- Get Initial Abandoned Checkout  
- Sticky Note4 (documentation)

**Node Details:**

- **Start Workflow**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow. Can be configured to scheduled triggers outside this workflow.  
  - *Connections:* Output → Get Initial Abandoned Checkout  
  - *Failure Modes:* None intrinsic; manual trigger requires human interaction or external schedule.  

- **Get Initial Abandoned Checkout**  
  - *Type:* HTTP Request  
  - *Role:* Fetches abandoned checkouts from Shopify API (endpoint: `/admin/api/2023-10/checkouts.json` with query parameter `status=abandoned`).  
  - *Configuration:*  
    - Method: GET  
    - Headers: Includes `X-Shopify-Access-Token` for authentication (requires valid Shopify API token).  
    - Query: `status=abandoned` filters for abandoned carts only.  
  - *Outputs:* JSON list of abandoned checkouts with customer and cart details.  
  - *Connections:* Output → Wait 1 Hour  
  - *Failure Modes:*  
    - API authentication failure (invalid/expired token)  
    - Network timeout  
    - API rate limiting  
    - Empty or malformed response  
  - *Notes:* Shopify API version is 2023-10; ensure compatibility with Shopify API versioning.  

- **Sticky Note4**  
  - *Type:* Sticky Note (Documentation)  
  - *Content:* Describes the purpose and logic of this block for maintainers.

---

#### 1.2 Grace Period Wait & Recheck

**Overview:**  
This block pauses the workflow for 1 hour to allow customers to complete their purchases naturally. After the wait, it re-fetches the abandoned checkout list from Shopify to identify which carts remain abandoned.

**Nodes Involved:**  
- Wait 1 Hour  
- Recheck Abandoned Checkouts  
- Sticky Note5 (documentation)

**Node Details:**

- **Wait 1 Hour**  
  - *Type:* Wait  
  - *Role:* Delays workflow execution for 1 hour after initial fetch.  
  - *Configuration:* Unit = hours, Amount = 1  
  - *Connections:* Output → Recheck Abandoned Checkouts  
  - *Failure Modes:*  
    - Workflow interruption or restart may cause delay loss  
    - No direct failure in node unless workflow is stopped externally  

- **Recheck Abandoned Checkouts**  
  - *Type:* HTTP Request  
  - *Role:* Performs a second API call to Shopify to get updated abandoned checkouts after waiting period.  
  - *Configuration:* Same as "Get Initial Abandoned Checkout" node: GET request, same endpoint, same auth headers, query `status=abandoned`.  
  - *Outputs:* Updated JSON list of abandoned carts.  
  - *Connections:* Output → Is Customer Still Abandoned?  
  - *Failure Modes:* Same as initial HTTP request node (auth issues, network errors, rate limiting).  

- **Sticky Note5**  
  - *Type:* Sticky Note (Documentation)  
  - *Content:* Explains the purpose of waiting and rechecking in this block.

---

#### 1.3 Conditional Decision on Abandonment Status

**Overview:**  
This block compares the initial abandoned checkout data with the updated data after waiting to determine if the same customer’s cart is still abandoned. If yes, proceed to recovery email flow; if not, end workflow silently.

**Nodes Involved:**  
- Is Customer Still Abandoned? (If node)  
- No Operation, do nothing  
- Sticky Note1 (documentation)

**Node Details:**

- **Is Customer Still Abandoned?**  
  - *Type:* If (Conditional)  
  - *Role:* Checks if the customer’s abandoned cart ID from the initial fetch still exists in the updated list.  
  - *Configuration:*  
    - Condition: Compares the initial checkout ID (`$('Get Initial Abandoned Checkout').item.json.checkouts[0].id`) against the first 5 entries of the updated abandoned checkouts (`$json.checkouts[0].id` to `$json.checkouts[4].id`) using equality operators.  
    - Logic: OR combinator (any match means customer still abandoned).  
  - *Connections:*  
    - True → Generate Recovery Email  
    - False → No Operation, do nothing  
  - *Failure Modes:*  
    - Index out of range if fewer than 5 checkouts returned in updated list (may cause expression errors).  
    - Data format changes in Shopify API response.  
  - *Notes:* This node assumes only the first 5 abandoned checkouts are checked for matching, which might limit scalability or miss matches if abandoned carts exceed 5.

- **No Operation, do nothing**  
  - *Type:* NoOp (No Operation)  
  - *Role:* Ends the workflow without action if customer no longer abandoned.  
  - *Connections:* None (terminal node).  

- **Sticky Note1**  
  - *Type:* Sticky Note (Documentation)  
  - *Content:* Details the decision logic in this conditional block.

---

#### 1.4 Personalized Recovery Email Flow

**Overview:**  
For customers confirmed to still have abandoned carts, this block generates a customized recovery email using AI, sends it via Gmail, and logs the email activity to a Google Sheet for tracking.

**Nodes Involved:**  
- Generate Recovery Email (Langchain Agent)  
- Email Writer (OpenAI GPT-4o-mini)  
- Send Email to Customer (Gmail node)  
- Log Email Activity (Google Sheets)  
- Sticky Note2 (documentation)

**Node Details:**

- **Generate Recovery Email**  
  - *Type:* Langchain Agent Node  
  - *Role:* Creates a prompt for AI to generate a friendly and persuasive abandoned cart recovery email based on customer name and cart contents.  
  - *Configuration:*  
    - Prompt: Uses expressions to inject customer first and last name and cart details from initial fetch.  
    - Instruction: Write a friendly persuasive email encouraging purchase completion, optionally mentioning discount, no subject or extra text included.  
    - PromptType: Define (custom prompt).  
  - *Connections:* Output → Send Email to Customer  
  - *Failure Modes:*  
    - AI service unavailability or API quota exceeded  
    - Incorrect or missing input data causing poor or failed generation  
  - *Notes:* Uses Langchain agent to orchestrate AI prompt generation; depends on OpenAI model configured in "Email Writer" node.

- **Email Writer**  
  - *Type:* Language Model Chat (OpenAI)  
  - *Role:* Executes the AI model call with configured GPT-4o-mini model (small GPT-4 variant) to generate the email content.  
  - *Configuration:* Model selected: "gpt-4o-mini".  
  - *Connections:* Output → Generate Recovery Email (input to agent)  
  - *Failure Modes:*  
    - API key/authentication failures  
    - Rate limiting or quota issues  
    - Network issues  
  - *Notes:* Acts as backend model for Langchain agent.

- **Send Email to Customer**  
  - *Type:* Gmail Node  
  - *Role:* Sends the AI-generated recovery email to the customer’s email address.  
  - *Configuration:*  
    - Recipient: Customer email from initial checkout data.  
    - Subject: Personalized with customer's first name.  
    - Message Body: Generated AI email content.  
    - Credentials: Requires OAuth2 configured Gmail account.  
  - *Connections:* Output → Log Email Activity  
  - *Failure Modes:*  
    - Authentication failure or expired Gmail token  
    - Gmail API rate limits or sending restrictions  
    - Invalid email address format  
  - *Notes:* Uses Gmail OAuth2 credential, ensure proper permissions and token refresh setup.

- **Log Email Activity**  
  - *Type:* Google Sheets Append  
  - *Role:* Logs the email sent event including customer name, email address, and AI response content to a specified Google Sheet document for monitoring and analytics.  
  - *Configuration:*  
    - Document ID and Sheet name fixed (linked to a specific Google Sheet).  
    - Columns: email, customer (first + last name), GPT response.  
    - Append operation, no type conversion.  
  - *Connections:* None (terminal node).  
  - *Failure Modes:*  
    - Google API authentication errors  
    - Rate limits or quota exceeded for Google Sheets API  
    - Incorrect spreadsheet ID or permissions  
  - *Notes:* Useful for campaign tracking and performance analysis.

- **Sticky Note2**  
  - *Type:* Sticky Note (Documentation)  
  - *Content:* Explains the recovery email flow and its purpose.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                             | Input Node(s)                | Output Node(s)               | Sticky Note                                                                                                                    |
|---------------------------|----------------------------------|--------------------------------------------|-----------------------------|-----------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Start Workflow            | Manual Trigger                   | Workflow entry point                        |                             | Get Initial Abandoned Checkout | Sticky Note4: Describes initialization and initial abandoned checkout fetch.                                                  |
| Get Initial Abandoned Checkout | HTTP Request                    | Fetch initial abandoned checkouts from Shopify | Start Workflow              | Wait 1 Hour                  | Sticky Note4: Details initial fetch and data structure.                                                                       |
| Wait 1 Hour               | Wait                            | Grace period delay                          | Get Initial Abandoned Checkout | Recheck Abandoned Checkouts | Sticky Note5: Explains purpose of waiting before rechecking.                                                                  |
| Recheck Abandoned Checkouts | HTTP Request                    | Fetch updated abandoned checkouts          | Wait 1 Hour                 | Is Customer Still Abandoned? | Sticky Note5: Details recheck step.                                                                                           |
| Is Customer Still Abandoned? | If                             | Conditional check on abandonment status     | Recheck Abandoned Checkouts | Generate Recovery Email (true), No Operation, do nothing (false) | Sticky Note1: Explains logic for deciding if customer still abandoned.                                                       |
| No Operation, do nothing  | NoOp                            | Ends workflow without action if not abandoned | Is Customer Still Abandoned? (false) |                             |                                                                                                                               |
| Generate Recovery Email   | Langchain Agent                 | Generate personalized email prompt for AI  | Is Customer Still Abandoned? (true) | Send Email to Customer       | Sticky Note2: Describes AI email generation flow.                                                                              |
| Email Writer             | LM Chat OpenAI                  | Executes AI model call                      |                             | Generate Recovery Email      | Sticky Note2: Backend AI model node.                                                                                           |
| Send Email to Customer   | Gmail Node                     | Sends recovery email to customer            | Generate Recovery Email     | Log Email Activity           | Sticky Note2: Sends personalized email and logs activity.                                                                     |
| Log Email Activity       | Google Sheets                  | Logs email activity in Google Sheets        | Send Email to Customer      |                             | Sticky Note2: Enables tracking of sent emails.                                                                                 |
| Sticky Note1             | Sticky Note                    | Documentation on conditional decision block |                             |                             | See above.                                                                                                                    |
| Sticky Note2             | Sticky Note                    | Documentation on recovery email flow         |                             |                             | See above.                                                                                                                    |
| Sticky Note4             | Sticky Note                    | Documentation on initialization block        |                             |                             | See above.                                                                                                                    |
| Sticky Note5             | Sticky Note                    | Documentation on wait and recheck block      |                             |                             | See above.                                                                                                                    |
| Sticky Note9             | Sticky Note                    | Workflow assistance and contact info         |                             |                             | Contains contact info and resource links: https://www.youtube.com/@YaronBeen/videos, https://www.linkedin.com/in/yaronbeen/   |
| Sticky Note3             | Sticky Note                    | Full workflow write-up (external summary)   |                             |                             | Full polished write-up, useful for documentation.                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: *Start Workflow*  
   - Type: Manual Trigger  
   - No special parameters.

2. **Add HTTP Request Node to Fetch Initial Checkouts**  
   - Name: *Get Initial Abandoned Checkout*  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://your-store.myshopify.com/admin/api/2023-10/checkouts.json`  
   - Query Parameters: `status=abandoned`  
   - Headers: `X-Shopify-Access-Token` = *your Shopify API token* (configure credentials securely)  
   - Connect output of *Start Workflow* → input of this node.

3. **Add Wait Node**  
   - Name: *Wait 1 Hour*  
   - Type: Wait  
   - Parameters: Unit = hours, Amount = 1  
   - Connect output of *Get Initial Abandoned Checkout* → input of this node.

4. **Add HTTP Request Node to Recheck Abandoned Checkouts**  
   - Name: *Recheck Abandoned Checkouts*  
   - Type: HTTP Request  
   - Same configuration as *Get Initial Abandoned Checkout*.  
   - Connect output of *Wait 1 Hour* → input of this node.

5. **Add Conditional Node to Check If Customer Still Abandoned**  
   - Name: *Is Customer Still Abandoned?*  
   - Type: If  
   - Condition: Check if initial abandoned checkout ID (expression: `$('Get Initial Abandoned Checkout').item.json.checkouts[0].id`) equals any of the first five IDs from updated list (expressions: `$json.checkouts[0].id` … `$json.checkouts[4].id`) using OR logic.  
   - Connect output of *Recheck Abandoned Checkouts* → input of this node.

6. **Add No Operation Node for False Branch**  
   - Name: *No Operation, do nothing*  
   - Type: NoOp  
   - Connect *Is Customer Still Abandoned?* False output → this node. Terminal node.

7. **Add Langchain Agent Node for Email Generation**  
   - Name: *Generate Recovery Email*  
   - Type: Langchain Agent  
   - Prompt:  
     ```
     Write a friendly and persuasive abandoned cart recovery email for a customer named {{ $json.checkouts[0].customer.first_name }} {{ $json.checkouts[0].customer.last_name }}. The cart contains: {{ $json.checkouts }}. Encourage them to complete their purchase with a subtle reminder and optionally a discount. do not include subject and don't add extra stuff
     ```  
   - Connect *Is Customer Still Abandoned?* True output → this node.

8. **Add OpenAI LM Chat Node**  
   - Name: *Email Writer*  
   - Type: LM Chat OpenAI  
   - Model: Select `gpt-4o-mini` (ensure OpenAI API key credential configured).  
   - Connect this node as the AI model backend for the Langchain Agent.  
   - Connect output of *Email Writer* → input of *Generate Recovery Email* (via ai_languageModel input).

9. **Add Gmail Node to Send Email**  
   - Name: *Send Email to Customer*  
   - Type: Gmail  
   - Parameters:  
     - Recipient: `={{ $('Get Initial Abandoned Checkout').item.json.checkouts[0].email }}`  
     - Subject: `=You left something behind, {{ $('Get Initial Abandoned Checkout').item.json.checkouts[0].customer.first_name }}`  
     - Message Body: `={{ $json.output }}` (output from *Generate Recovery Email*)  
   - Credentials: Configure Gmail OAuth2 with sending permissions.  
   - Connect output of *Generate Recovery Email* → input of this node.

10. **Add Google Sheets Node to Log Email Activity**  
    - Name: *Log Email Activity*  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: `1H83mXsl_gjgwOWrwNYak6ISihoWsgx-EFxSbPlvQx7M` (replace with your own sheet if needed).  
    - Sheet Name: `gid=0` (Sheet1)  
    - Columns to append:  
      - email: `={{ $('Get Initial Abandoned Checkout').item.json.checkouts[0].email }}`  
      - customer: `={{ $('Get Initial Abandoned Checkout').item.json.checkouts[0].customer.first_name }} {{ $('Get Initial Abandoned Checkout').item.json.checkouts[0].customer.last_name }}`  
      - GPT response: `={{ $('Generate Recovery Email').item.json.output }}`  
    - Connect output of *Send Email to Customer* → input of this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Workflow assistance contact: Yaron@nofluff.online | Contact email for support |
| Explore more tips and tutorials here: YouTube channel https://www.youtube.com/@YaronBeen/videos | Video resources |
| Professional profile: https://www.linkedin.com/in/yaronbeen/ | LinkedIn profile for author |
| This workflow respects Shopify API versioning and requires valid API tokens and OAuth2 credentials for Gmail and Google Sheets integration. | Credential requirement note |
| The workflow currently checks only first 5 abandoned checkouts on recheck step, which may require adjustment for stores with many abandoned carts. | Scalability consideration |

---

**Disclaimer:** The above documentation is derived exclusively from an automated workflow created with n8n, strictly adhering to content policies. All data handled is legal and public.