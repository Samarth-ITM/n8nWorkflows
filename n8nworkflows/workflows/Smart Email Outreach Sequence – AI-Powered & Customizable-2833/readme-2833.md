Smart Email Outreach Sequence – AI-Powered & Customizable

https://n8nworkflows.xyz/workflows/smart-email-outreach-sequence---ai-powered---customizable-2833


# Smart Email Outreach Sequence – AI-Powered & Customizable

### 1. Workflow Overview

This workflow automates a smart, AI-powered email outreach sequence designed to maximize engagement and conversion rates through personalized, domain-based targeting and follow-up emails. It is ideal for B2B lead generation, partnerships, and cold emailing at scale.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Validation**: Reads leads from Google Sheets, validates email addresses, and batches them for processing.
- **1.2 Domain Extraction & Website Intelligence**: Extracts domains from emails and uses Jina.ai to analyze recipient websites for personalized content.
- **1.3 AI-Powered Email Content Generation**: Uses GPT-3.5 to generate initial and follow-up email content based on website insights and customizable prompts.
- **1.4 Email Sending & Sequencing**: Sends the initial email and two follow-ups at optimized 3-day intervals with randomized delays to mimic human behavior.
- **1.5 Error Handling & Workflow Control**: Implements error-resilient execution and conditional logic to ensure smooth operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

**Overview:**  
This block reads the list of leads from Google Sheets, checks if each entry is a valid email, and splits the data into manageable batches for processing.

**Nodes Involved:**  
- When clicking Test workflow1 (Manual Trigger)  
- Read List of Leads (Google Sheets)  
- Is it an Email ? (If)  
- Loop Over Emails (SplitInBatches)  
- Loop Over Emails1 (SplitInBatches)  
- Loop Over Emails2 (SplitInBatches)

**Node Details:**

- **When clicking Test workflow1**  
  - Type: Manual Trigger  
  - Role: Starts the workflow manually for testing or execution  
  - Inputs: None  
  - Outputs: Connects to Read List of Leads  
  - Edge cases: None (manual trigger)

- **Read List of Leads**  
  - Type: Google Sheets  
  - Role: Reads lead data (emails) from a configured Google Sheet  
  - Configuration: Must specify spreadsheet ID, sheet name, and range containing leads  
  - Inputs: Trigger node  
  - Outputs: Leads data to "Is it an Email ?"  
  - Edge cases: Authentication errors, empty or malformed sheet data

- **Is it an Email ?**  
  - Type: If  
  - Role: Validates if the input string is a valid email address  
  - Configuration: Uses expression or regex to check email format  
  - Inputs: Leads data from Google Sheets  
  - Outputs: True branch to three parallel SplitInBatches nodes (Loop Over Emails, Loop Over Emails1, Loop Over Emails2)  
  - Edge cases: Invalid emails filtered out; no leads to process if all invalid

- **Loop Over Emails / Loop Over Emails1 / Loop Over Emails2**  
  - Type: SplitInBatches  
  - Role: Splits the list of emails into batches for sequential processing  
  - Configuration: Batch size configurable (default recommended small to avoid rate limits)  
  - Inputs: Validated emails from If node  
  - Outputs: Each batch to respective Wait nodes or domain extraction nodes  
  - Edge cases: Empty batches, batch size too large causing timeouts

---

#### 2.2 Domain Extraction & Website Intelligence

**Overview:**  
Extracts the domain from each email address and queries Jina.ai to analyze the recipient’s website, providing data for personalized email content.

**Nodes Involved:**  
- Extract Domain Name  
- Extract Domain Name1  
- Extract Domain Name2  
- Get website with Jina.ai  
- Get website with Jina.ai1  
- Get website with Jina.ai2  
- If1  
- If2  
- If3

**Node Details:**

- **Extract Domain Name / Extract Domain Name1 / Extract Domain Name2**  
  - Type: Code  
  - Role: Extracts domain part from email addresses (e.g., user@example.com → example.com)  
  - Configuration: JavaScript code using string manipulation or regex  
  - Inputs: Batches of emails from SplitInBatches nodes  
  - Outputs: Domain string to respective Jina.ai HTTP Request nodes  
  - Edge cases: Malformed emails, missing domain part

- **Get website with Jina.ai / Get website with Jina.ai1 / Get website with Jina.ai2**  
  - Type: HTTP Request  
  - Role: Sends HTTP requests to Jina.ai API to analyze the extracted domain’s website content  
  - Configuration: API endpoint, authentication headers, query parameters including domain  
  - Inputs: Domain from code nodes  
  - Outputs: Website intelligence data to If nodes  
  - Error Handling: Configured to continue on error to avoid workflow interruption  
  - Edge cases: API rate limits, network timeouts, invalid domain responses

- **If1 / If2 / If3**  
  - Type: If  
  - Role: Conditional checks on Jina.ai response to determine if valid data was retrieved  
  - Configuration: Checks for presence or quality of website data  
  - Inputs: Jina.ai response  
  - Outputs: True branch proceeds to email content generation; False branch to markdown limiting or fallback  
  - Edge cases: Empty or malformed API responses

---

#### 2.3 AI-Powered Email Content Generation

**Overview:**  
Generates personalized email content for initial outreach and two follow-ups using GPT-3.5, leveraging website insights and customizable prompts.

**Nodes Involved:**  
- Limit Markdown  
- Limit Markdown1  
- Limit Markdown2  
- Product link  
- Product Link  
- Product Link1  
- Generate Email content  
- Generate Email Follow-up 1  
- Generate Email Final Follow-up

**Node Details:**

- **Limit Markdown / Limit Markdown1 / Limit Markdown2**  
  - Type: Code  
  - Role: Processes or truncates markdown content from website data to fit prompt constraints  
  - Configuration: JavaScript code to limit length or sanitize content  
  - Inputs: Website intelligence data from If nodes  
  - Outputs: Cleaned data to Product link nodes  
  - Edge cases: Overly long content, markdown syntax errors

- **Product link / Product Link / Product Link1**  
  - Type: Set  
  - Role: Sets or formats product or campaign-specific variables for AI prompt customization  
  - Configuration: Static or dynamic values for product/service links or descriptions  
  - Inputs: Processed website data  
  - Outputs: To respective GPT nodes for prompt injection  
  - Edge cases: Missing or incorrect product info

- **Generate Email content**  
  - Type: OpenAI (GPT-3.5)  
  - Role: Generates the initial outreach email content using AI  
  - Configuration: Prompt includes website insights, product info, and personalization instructions  
  - Inputs: Product link data  
  - Outputs: Email content to Gmail node for sending  
  - Credential: Requires OpenAI API key  
  - Edge cases: API quota exceeded, prompt errors, network issues

- **Generate Email Follow-up 1**  
  - Type: OpenAI (GPT-3.5)  
  - Role: Generates first follow-up email content  
  - Configuration: Similar prompt structure with follow-up context  
  - Inputs: Product Link data  
  - Outputs: Email content to Gmail1 node  
  - Credential: OpenAI API key  
  - Edge cases: Same as above

- **Generate Email Final Follow-up**  
  - Type: OpenAI (GPT-3.5)  
  - Role: Generates final follow-up email content  
  - Configuration: Tailored prompt for last outreach attempt  
  - Inputs: Product Link1 data  
  - Outputs: Email content to Gmail2 node  
  - Credential: OpenAI API key  
  - Edge cases: Same as above

---

#### 2.4 Email Sending & Sequencing

**Overview:**  
Sends the initial and follow-up emails via Gmail with randomized delays to simulate human sending patterns and avoid spam filters.

**Nodes Involved:**  
- Gmail  
- Gmail1  
- Gmail2  
- Wait  
- Wait1  
- Wait2  
- Wait3  
- Wait4  
- Code  
- Code1  
- Code2

**Node Details:**

- **Gmail / Gmail1 / Gmail2**  
  - Type: Gmail  
  - Role: Sends emails (initial, first follow-up, final follow-up) via Gmail SMTP using OAuth2  
  - Configuration: Uses OAuth2 credentials, email fields populated dynamically from AI-generated content and lead data  
  - Inputs: Generated email content nodes  
  - Outputs: To Code nodes for delay handling  
  - Credential: Gmail OAuth2 required  
  - Edge cases: Authentication errors, quota limits, email sending failures

- **Wait / Wait1 / Wait2 / Wait3 / Wait4**  
  - Type: Wait  
  - Role: Implements delays between email sends to space out the sequence (typically 3 days with randomization)  
  - Configuration: Wait time set to 3 days +/- random offset to mimic human behavior  
  - Inputs: From batch processing or code nodes  
  - Outputs: To next processing nodes or domain extraction for next email  
  - Edge cases: Workflow timeouts if wait too long, manual interruption

- **Code / Code1 / Code2**  
  - Type: Code  
  - Role: Executes custom JavaScript for delay calculation, logging, or conditional logic between sends  
  - Configuration: Contains logic for randomized wait times or status updates  
  - Inputs: After Gmail nodes  
  - Outputs: To Wait nodes or batch loops  
  - Edge cases: Script errors, expression failures

---

#### 2.5 Error Handling & Workflow Control

**Overview:**  
Ensures the workflow continues smoothly despite errors in API calls or email sending, using conditional branches and error continuation settings.

**Nodes Involved:**  
- HTTP Request nodes (Jina.ai) with "continue on error" enabled  
- If nodes for conditional branching  
- Code nodes for error logging or fallback logic

**Node Details:**

- HTTP Request nodes to Jina.ai have "continue on error" enabled to avoid halting the workflow if the API fails or returns invalid data.  
- If nodes check for valid data presence and route the flow accordingly.  
- Code nodes may include try-catch blocks or error logging to handle unexpected issues gracefully.  
- Edge cases: Network failures, API downtime, invalid data formats, Gmail sending errors.

---

### 3. Summary Table

| Node Name                  | Node Type                | Functional Role                             | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                      |
|----------------------------|--------------------------|---------------------------------------------|-------------------------------|-------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking Test workflow1| Manual Trigger           | Starts workflow manually                     | None                          | Read List of Leads             |                                                                                                 |
| Read List of Leads          | Google Sheets            | Reads leads from Google Sheets               | When clicking Test workflow1   | Is it an Email ?               |                                                                                                 |
| Is it an Email ?            | If                       | Validates email format                        | Read List of Leads             | Loop Over Emails, Loop Over Emails1, Loop Over Emails2|                                                                                                 |
| Loop Over Emails            | SplitInBatches           | Batches emails for processing                 | Is it an Email ?               | Wait1, Extract Domain Name     | Sticky Note1 (top-left area) covers domain extraction and batching logic                         |
| Loop Over Emails1           | SplitInBatches           | Batches emails for follow-up 1                | Is it an Email ?               | Wait3, Extract Domain Name1    | Sticky Note2 (middle-left area) covers second batch and domain extraction                        |
| Loop Over Emails2           | SplitInBatches           | Batches emails for final follow-up            | Is it an Email ?               | Extract Domain Name2           | Sticky Note3 (bottom-left area) covers final batch and domain extraction                         |
| Extract Domain Name         | Code                     | Extracts domain from email                     | Loop Over Emails               | Get website with Jina.ai       | Sticky Note1                                                                                     |
| Extract Domain Name1        | Code                     | Extracts domain from email                     | Loop Over Emails1              | Get website with Jina.ai1      | Sticky Note2                                                                                     |
| Extract Domain Name2        | Code                     | Extracts domain from email                     | Loop Over Emails2              | Get website with Jina.ai2      | Sticky Note3                                                                                     |
| Get website with Jina.ai    | HTTP Request             | Queries Jina.ai for website intelligence      | Extract Domain Name            | If1                           | Sticky Note1                                                                                     |
| Get website with Jina.ai1   | HTTP Request             | Queries Jina.ai for website intelligence      | Extract Domain Name1           | If2                           | Sticky Note2                                                                                     |
| Get website with Jina.ai2   | HTTP Request             | Queries Jina.ai for website intelligence      | Extract Domain Name2           | If3                           | Sticky Note3                                                                                     |
| If1                        | If                       | Checks Jina.ai response validity              | Get website with Jina.ai       | Loop Over Emails, Limit Markdown| Sticky Note1                                                                                     |
| If2                        | If                       | Checks Jina.ai response validity              | Get website with Jina.ai1      | Loop Over Emails1, Limit Markdown1| Sticky Note2                                                                                     |
| If3                        | If                       | Checks Jina.ai response validity              | Get website with Jina.ai2      | Loop Over Emails2, Limit Markdown2| Sticky Note3                                                                                     |
| Limit Markdown             | Code                     | Limits markdown content length                 | If1                           | Product link                  | Sticky Note (center-top) covers markdown limiting and product link setting                      |
| Limit Markdown1            | Code                     | Limits markdown content length                 | If2                           | Product Link                  | Sticky Note4 (center-right) covers markdown limiting and product link setting                   |
| Limit Markdown2            | Code                     | Limits markdown content length                 | If3                           | Product Link1                 | Sticky Note5 (bottom-right) covers markdown limiting and product link setting                   |
| Product link              | Set                      | Sets product/campaign info for AI prompt      | Limit Markdown                | Generate Email content        | Sticky Note (center-top)                                                                           |
| Product Link              | Set                      | Sets product/campaign info for AI prompt      | Limit Markdown1               | Generate Email Follow-up 1    | Sticky Note4                                                                                     |
| Product Link1             | Set                      | Sets product/campaign info for AI prompt      | Limit Markdown2               | Generate Email Final Follow-up| Sticky Note5                                                                                     |
| Generate Email content     | OpenAI (GPT-3.5)         | Generates initial outreach email content      | Product link                  | Gmail                        |                                                                                                 |
| Generate Email Follow-up 1 | OpenAI (GPT-3.5)         | Generates first follow-up email content        | Product Link                  | Gmail1                       |                                                                                                 |
| Generate Email Final Follow-up| OpenAI (GPT-3.5)       | Generates final follow-up email content        | Product Link1                 | Gmail2                       |                                                                                                 |
| Gmail                     | Gmail                    | Sends initial outreach email                   | Generate Email content        | Code                        |                                                                                                 |
| Gmail1                    | Gmail                    | Sends first follow-up email                     | Generate Email Follow-up 1    | Code1                       |                                                                                                 |
| Gmail2                    | Gmail                    | Sends final follow-up email                     | Generate Email Final Follow-up| Code2                       |                                                                                                 |
| Code                      | Code                     | Handles delay/randomization logic post-send    | Gmail                        | Wait                        |                                                                                                 |
| Code1                     | Code                     | Handles delay/randomization logic post-send    | Gmail1                       | Wait2                       |                                                                                                 |
| Code2                     | Code                     | Handles delay/randomization logic post-send    | Gmail2                       | Wait4                       |                                                                                                 |
| Wait                      | Wait                     | Waits randomized delay before next batch        | Code                         | Loop Over Emails             |                                                                                                 |
| Wait1                     | Wait                     | Waits randomized delay before next batch        | Loop Over Emails              | Loop Over Emails             |                                                                                                 |
| Wait2                     | Wait                     | Waits randomized delay before next batch        | Code1                        | Loop Over Emails1            |                                                                                                 |
| Wait3                     | Wait                     | Waits randomized delay before next batch        | Loop Over Emails1             | Loop Over Emails1            |                                                                                                 |
| Wait4                     | Wait                     | Waits randomized delay before next batch        | Code2                        | Loop Over Emails2            |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node** named "When clicking Test workflow1" to start the workflow manually.

2. **Add a Google Sheets node** named "Read List of Leads":  
   - Configure with your Google Sheets credentials.  
   - Set spreadsheet ID and sheet name containing lead emails.  
   - Define the range to read emails.

3. **Add an If node** named "Is it an Email ?" to validate email addresses:  
   - Use an expression or regex to check if the input string matches email format.  
   - Connect "Read List of Leads" output to this node.

4. **Add three SplitInBatches nodes** named "Loop Over Emails", "Loop Over Emails1", and "Loop Over Emails2":  
   - Connect the true output of "Is it an Email ?" to all three nodes.  
   - Configure batch sizes (e.g., 5-10) to control processing load.

5. **Add Wait nodes** named "Wait1", "Wait3", and "Wait" connected respectively to "Loop Over Emails" and "Loop Over Emails1" for pacing batches.

6. **Add three Code nodes** named "Extract Domain Name", "Extract Domain Name1", and "Extract Domain Name2":  
   - Connect from "Loop Over Emails", "Loop Over Emails1", and "Loop Over Emails2" respectively.  
   - Implement JavaScript to extract domain from each email address, e.g.:  
     ```javascript
     const email = items[0].json.email;
     const domain = email.split('@')[1];
     return [{ json: { domain } }];
     ```

7. **Add three HTTP Request nodes** named "Get website with Jina.ai", "Get website with Jina.ai1", and "Get website with Jina.ai2":  
   - Connect from respective domain extraction nodes.  
   - Configure to call Jina.ai API with domain as parameter.  
   - Set "Continue on Error" to true.  
   - Configure authentication if required.

8. **Add three If nodes** named "If1", "If2", and "If3":  
   - Connect from respective Jina.ai HTTP Request nodes.  
   - Configure to check if the response contains valid website data (e.g., non-empty content).

9. **Add three Code nodes** named "Limit Markdown", "Limit Markdown1", and "Limit Markdown2":  
   - Connect from true output of If nodes.  
   - Implement logic to truncate or sanitize markdown content from website data.

10. **Add three Set nodes** named "Product link", "Product Link", and "Product Link1":  
    - Connect from respective markdown limiting nodes.  
    - Set static or dynamic variables for product or campaign info to be used in AI prompts.

11. **Add three OpenAI nodes** named "Generate Email content", "Generate Email Follow-up 1", and "Generate Email Final Follow-up":  
    - Connect from respective product link nodes.  
    - Configure with OpenAI credentials (API key).  
    - Set model to GPT-3.5.  
    - Customize prompts to generate initial email, first follow-up, and final follow-up content using website data and product info.

12. **Add three Gmail nodes** named "Gmail", "Gmail1", and "Gmail2":  
    - Connect from respective OpenAI nodes.  
    - Configure Gmail OAuth2 credentials.  
    - Map email fields (To, Subject, Body) dynamically from AI-generated content and lead data.

13. **Add three Code nodes** named "Code", "Code1", and "Code2":  
    - Connect from respective Gmail nodes.  
    - Implement JavaScript to calculate randomized delays (e.g., 3 days ± random offset).

14. **Add four Wait nodes** named "Wait", "Wait2", "Wait3", and "Wait4":  
    - Connect from respective Code nodes or batch loops.  
    - Configure wait times to implement delays between emails and batches.

15. **Connect Wait nodes back to SplitInBatches nodes** to continue processing next batches after delays.

16. **Add Sticky Notes** as needed to document logic sections and provide instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow uses Gmail OAuth2 for secure email sending.                                           | Gmail node configuration requires OAuth2 credentials.                                           |
| Jina.ai API is used for website intelligence extraction.                                      | Ensure API keys and endpoints are correctly configured.                                         |
| OpenAI GPT-3.5 is used for AI-powered email content generation.                               | Requires OpenAI API key with sufficient quota.                                                  |
| Randomized delays between emails help avoid spam filters and improve deliverability.          | Implemented via Code and Wait nodes with randomized timing logic.                               |
| Workflow includes error handling with "continue on error" in HTTP Request nodes to ensure robustness. | Prevents workflow interruption due to API failures.                                            |
| For detailed setup, refer to the workflow description section above for prerequisites and instructions. |                                                                                                 |
| Video tutorial and blog post links can be added as sticky notes for user guidance.            | Add URLs in sticky notes for easy access.                                                       |

---

This structured documentation enables advanced users and AI agents to fully understand, reproduce, and modify the Smart Email Outreach Sequence workflow with clarity on each component’s role, configuration, and integration points.