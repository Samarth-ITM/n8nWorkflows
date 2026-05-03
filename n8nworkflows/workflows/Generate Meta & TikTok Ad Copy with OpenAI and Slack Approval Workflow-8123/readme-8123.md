Generate Meta & TikTok Ad Copy with OpenAI and Slack Approval Workflow

https://n8nworkflows.xyz/workflows/generate-meta---tiktok-ad-copy-with-openai-and-slack-approval-workflow-8123


# Generate Meta & TikTok Ad Copy with OpenAI and Slack Approval Workflow

### 1. Workflow Overview

This workflow automates the generation and approval of advertising copy for Meta and TikTok campaigns using OpenAI's language models and Slack for human review. It is tailored to handle two distinct brand types — Fashion Brands and Problem-Solution Brands — each with specialized AI prompts to generate ad copies that resonate with their unique audiences and brand voices.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Captures brand and product information via a web form with file attachment support for research briefs and references.
- **1.2 AI Copy Generation:** Invokes specialized OpenAI nodes to generate initial ad copies based on the brand type selected.
- **1.3 Slack Review and Approval:** Sends the generated ads to a Slack channel for team review and approval, supporting a double-approval mechanism.
- **1.4 Revision Loop:** Handles revision requests with feedback collection via Slack, triggers AI to revise ad copies, and limits revision rounds to a maximum of three.
- **1.5 Finalization:** Sends congratulations messages upon approval or informs users when revision limits are exceeded.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block collects all necessary input data from users via a customizable form interface, including product details, brand type selection, and supporting documents.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; listens for form submissions containing brand and product data.  
  - Configuration:  
    - Form titled "Create Ad Copies" with required fields including Brand Name dropdown (Fashion Brand / Problem-Solution Brand), file upload for research briefs, product name and link, target audience, pain points, USP, optional customer review, proven winner ads, and competitor ads.  
  - Key expressions: Inputs are mapped directly to JSON fields used downstream (e.g., `Product Name`, `Brand Name`).  
  - Inputs: External user form submission  
  - Outputs: JSON item containing all form data, including uploaded files metadata.  
  - Edge cases: File upload failures, incomplete submissions, invalid field formats.  
  - Version: Supports n8n version 2.2 for form trigger node.  

---

#### 2.2 AI Copy Generation

**Overview:**  
Based on the brand type selected, this block routes input data to one of two specialized OpenAI nodes that generate ad copy variations tailored to the brand’s tone and audience.

**Nodes Involved:**  
- If: Type of Brand  
- For Fashion Brands  
- For Problem Solution Brands  

**Node Details:**  

- **If: Type of Brand**  
  - Type: If Node  
  - Role: Routes workflow depending on whether the brand is "Fashion Brand" or "Problem-Solution Brand."  
  - Configuration: Checks if `Brand Name` equals "Fashion Brand". If false, proceeds to Problem-Solution path.  
  - Inputs: Form submission data  
  - Outputs: One of two branches: Fashion or Problem-Solution.  
  - Edge cases: Unrecognized brand types; no fallback defined.  

- **For Fashion Brands**  
  - Type: OpenAI (Langchain)  
  - Role: Generates 3-5 ad copy variations with a subtle, stylish fashion brand voice.  
  - Configuration:  
    - Model: User-configured GPT model (e.g., GPT-4) via OpenAI credentials.  
    - Prompt: Highly detailed, includes brand/product info, target audience, unique selling points, competitor ads, customer reviews, and strict stylistic guidelines emphasizing editorial and minimalistic tone.  
    - Output: 3-5 ad copies (60 chars to 90 words).  
  - Inputs: JSON data filtered for Fashion Brand path.  
  - Outputs: AI-generated ad copy text.  
  - Edge cases: API rate limits, prompt formatting issues, missing input fields.  

- **For Problem Solution Brands**  
  - Type: OpenAI (Langchain)  
  - Role: Generates 3 ad copies designed for Meta/TikTok with a playful, conversational tone targeting Gen Z/Millennials.  
  - Configuration:  
    - Model: User-configured GPT model.  
    - Prompt: Specifies format (Hook + Benefits + CTA), tone, emojis, and includes product info and competitive ads for inspiration.  
    - Output: 3 ad variations, 60-90 words each.  
  - Inputs: JSON data filtered for Problem-Solution path.  
  - Outputs: AI-generated ad copy text.  
  - Edge cases: Same as Fashion Brands.  

---

#### 2.3 Slack Review and Approval

**Overview:**  
This block sends the AI-generated ad copies to a designated Slack channel for team review. It supports a double-approval mechanism and routes workflow depending on approval or disapproval.

**Nodes Involved:**  
- Slack: Send and Review Ad Copies  
- If: Ad Copy is Approve or not  
- Slack: Congrats on your approve Ad Copies  
- Slack: Receive feedback for revision  

**Node Details:**  

- **Slack: Send and Review Ad Copies**  
  - Type: Slack node  
  - Role: Posts the suggested ad copies into a Slack channel and waits for team approval or rejection.  
  - Configuration:  
    - Operation: Send and wait for approval (double approval required).  
    - Message includes product name, brand type, and AI-generated copies.  
    - Channel: User-configured Slack channel ID.  
    - Approval options: Approve or "Try again!" (disapprove).  
  - Inputs: AI-generated ad copies.  
  - Outputs: JSON with approval status.  
  - Edge cases: Slack API rate limits, missing channel permissions, delayed responses.  

- **If: Ad Copy is Approve or not**  
  - Type: If Node  
  - Role: Checks if the Slack approval status is true (approved) or false (needs revision).  
  - Inputs: Slack approval response.  
  - Outputs: Approval or disapproval branch.  

- **Slack: Congrats on your approve Ad Copies**  
  - Type: Slack node  
  - Role: Sends a congratulatory message confirming approval of ad copies.  
  - Configuration: Message includes product and brand information.  
  - Inputs: Approval branch from If node.  
  - Outputs: None (endpoint).  

- **Slack: Receive feedback for revision**  
  - Type: Slack node  
  - Role: Requests free-text feedback from the team in Slack for revision purposes.  
  - Configuration:  
    - Operation: Send and wait for free-text response.  
    - Message prompts for feedback on what needs to be changed.  
    - Channel: Same Slack channel as above.  
  - Inputs: Disapproval branch from If node.  
  - Outputs: JSON containing feedback text.  
  - Edge cases: No feedback provided, Slack API errors.  

---

#### 2.4 Revision Loop

**Overview:**  
Manages revision cycles based on received feedback, triggers AI to revise ad copies, counts revisions, and enforces a maximum of three revision rounds.

**Nodes Involved:**  
- Slack: Thank you for the feedback  
- Edit Fields: Revision Counter Max 3  
- If: Continue Revise or Stop  
- OpenAI: Revise Ad Copies  
- Slack: Send Revise Ad Copies  
- If  

**Node Details:**  

- **Slack: Thank you for the feedback**  
  - Type: Slack node  
  - Role: Acknowledges receipt of feedback and informs the team that revisions are underway.  
  - Inputs: Feedback text from Slack.  
  - Outputs: Continues workflow.  

- **Edit Fields: Revision Counter Max 3**  
  - Type: Set node  
  - Role: Increments a revision counter stored in JSON (`revisionCount`), initializing it to 1 if absent.  
  - Inputs: After feedback acknowledgment.  
  - Outputs: Updated JSON with revision count.  
  - Edge cases: Persistent counting across executions must be ensured by context or data passing.  

- **If: Continue Revise or Stop**  
  - Type: If node  
  - Role: Checks if the revision count exceeds 3.  
  - Inputs: JSON with revision count.  
  - Outputs:  
    - True branch: Revision count > 3 (stop revisions).  
    - False branch: Allow further revisions.  

- **OpenAI: Revise Ad Copies**  
  - Type: OpenAI (Langchain)  
  - Role: Generates a new batch of 3-5 ad copies incorporating the provided feedback.  
  - Configuration:  
    - Uses previous ad copies, product info, and feedback to prompt the AI to revise text.  
    - Outputs updated ad copy variations.  
  - Inputs: Feedback text and original submission data.  
  - Outputs: Revised ad copies.  
  - Edge cases: API limits, prompt complexity.  

- **Slack: Send Revise Ad Copies**  
  - Type: Slack node  
  - Role: Sends revised ad copies to Slack for another round of approval.  
  - Configuration: Similar to initial Slack approval node, with sendAndWait and approval options.  
  - Inputs: Revised ad copies from OpenAI.  
  - Outputs: Approval status for revised copies.  

- **If**  
  - Type: If node  
  - Role: Checks approval status of revised ad copies, routing to final approval or further feedback.  

---

#### 2.5 Finalization on Revision Limits

**Overview:**  
Handles the case when the maximum number of revisions is reached, notifying the team that no further automated revisions will be made.

**Nodes Involved:**  
- Slack: Maxed Out Revision Count  

**Node Details:**  

- **Slack: Maxed Out Revision Count**  
  - Type: Slack node  
  - Role: Sends message to Slack channel informing that the maximum number of revisions (3) has been reached and further help will not be provided.  
  - Inputs: True branch from revision count check.  
  - Outputs: Ends workflow for that revision cycle.  

---

### 3. Summary Table

| Node Name                        | Node Type                        | Functional Role                        | Input Node(s)               | Output Node(s)                    | Sticky Note                                                                                                   |
|---------------------------------|---------------------------------|-------------------------------------|-----------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger                    | Collects brand/product input        | External trigger            | If: Type of Brand                | Step 1: Collect brand information and requirements                                                             |
| If: Type of Brand              | If                             | Routes by brand type                 | On form submission          | For Fashion Brands, For Problem Solution Brands |                                                                                                               |
| For Fashion Brands             | OpenAI (Langchain)              | Generates fashion brand ad copies   | If: Type of Brand (Fashion branch) | Slack: Send and Review Ad Copies | Step 2: Generate initial ad copies based on brand type                                                         |
| For Problem Solution Brands    | OpenAI (Langchain)              | Generates problem-solution ad copies | If: Type of Brand (Other branch) | Slack: Send and Review Ad Copies | Step 2: Generate initial ad copies based on brand type                                                         |
| Slack: Send and Review Ad Copies | Slack                          | Posts ad copies for approval        | For Fashion/Problem Solution Brands | If: Ad Copy is Approve or not    | Step 3: Team review and approval process                                                                        |
| If: Ad Copy is Approve or not  | If                             | Checks Slack approval status        | Slack: Send and Review Ad Copies | Slack: Congrats on your approve Ad Copies, Slack: Receive feedback for revision | Step 3: Team review and approval process                                                                        |
| Slack: Congrats on your approve Ad Copies | Slack                 | Sends approval confirmation         | If: Ad Copy is Approve or not (approved branch) | None                            |                                                                                                               |
| Slack: Receive feedback for revision | Slack                     | Requests revision feedback          | If: Ad Copy is Approve or not (disapproved branch) | Slack: Thank you for the feedback, Edit Fields: Revision Counter Max 3 | Step 4: Revision loop (max 3 iterations)                                                                        |
| Slack: Thank you for the feedback | Slack                        | Acknowledges feedback receipt       | Slack: Receive feedback for revision | Edit Fields: Revision Counter Max 3 | Step 4: Revision loop (max 3 iterations)                                                                        |
| Edit Fields: Revision Counter Max 3 | Set                        | Increments revision count           | Slack: Thank you for the feedback | If: Continue Revise or Stop       | Step 4: Revision loop (max 3 iterations)                                                                        |
| If: Continue Revise or Stop    | If                             | Checks if revision count > 3        | Edit Fields: Revision Counter Max 3 | Slack: Maxed Out Revision Count, OpenAI: Revise Ad Copies | Step 4: Revision loop (max 3 iterations)                                                                        |
| Slack: Maxed Out Revision Count | Slack                         | Notifies max revisions reached      | If: Continue Revise or Stop (true branch) | None                            | Step 4: Revision loop (max 3 iterations)                                                                        |
| OpenAI: Revise Ad Copies       | OpenAI (Langchain)              | Generates revised ad copies          | If: Continue Revise or Stop (false branch) | Slack: Send Revise Ad Copies      | Step 4: Revision loop (max 3 iterations)                                                                        |
| Slack: Send Revise Ad Copies   | Slack                          | Posts revised ad copies for approval | OpenAI: Revise Ad Copies    | If                             | Step 4: Revision loop (max 3 iterations)                                                                        |
| If                            | If                             | Checks approval status of revised copies | Slack: Send Revise Ad Copies | Slack: Congrats on your approve Ad Copies, Slack: Receive feedback for revision | Step 4: Revision loop (max 3 iterations)                                                                        |
| Sticky Note                   | Sticky Note                    | Workflow overview and setup notes  | None                       | None                            | Contains detailed workflow overview, setup, requirements, and customization instructions                      |
| Sticky Note1                  | Sticky Note                    | Step 1 comment for input collection | None                       | None                            | Step 1: Collect brand information and requirements                                                             |
| Sticky Note2                  | Sticky Note                    | Step 2 comment for AI generation    | None                       | None                            | Step 2: Generate initial ad copies based on brand type                                                         |
| Sticky Note3                  | Sticky Note                    | Step 3 comment for Slack approval   | None                       | None                            | Step 3: Team review and approval process                                                                        |
| Sticky Note4                  | Sticky Note                    | Step 4 comment for revision loop    | None                       | None                            | Step 4: Revision loop (max 3 iterations)                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: "On form submission"  
   - Type: Form Trigger  
   - Configure a form titled "Create Ad Copies" with the following fields:  
     - Dropdown "Brand Name" with options: "Fashion Brand", "Problem-Solution Brand" (required)  
     - File upload "Attach Research Brief and other references" (required)  
     - Text inputs (required unless specified): Product Name, Product Page Link (with placeholder), Target Audience, Pain Points (optional), USP - 1 liner, Customer Review (optional), 'Proven Winners' Ad Samples, Ad Copies from Competitors  
   - Set webhook ID (auto-generated) and activate the node.  

2. **Add an If Node to Route by Brand Type**  
   - Name: "If: Type of Brand"  
   - Condition: Check if `Brand Name` equals "Fashion Brand".  
   - Output branches: True (Fashion Brand), False (Problem-Solution Brand).  

3. **Create OpenAI Node for Fashion Brands**  
   - Name: "For Fashion Brands"  
   - Type: OpenAI (Langchain)  
   - Credentials: Configure OpenAI API with your API key.  
   - Model: Select GPT-3.5 or GPT-4 as preferred.  
   - Prompt: Use the detailed fashion brand prompt that includes brand/product info, USP, audience, pain points, customer reviews, competitor ads, and style guidelines. Include file attachment references if needed.  
   - Connect from "If: Type of Brand" True branch.  

4. **Create OpenAI Node for Problem-Solution Brands**  
   - Name: "For Problem Solution Brands"  
   - Similar setup as above but with the prompt tailored for problem-solution ad copies (hook + benefits + CTA, playful tone, emojis).  
   - Connect from "If: Type of Brand" False branch.  

5. **Add Slack Node to Send and Review Ad Copies**  
   - Name: "Slack: Send and Review Ad Copies"  
   - Credentials: Set Slack OAuth2 credentials with access to the chosen channel.  
   - Operation: Send and wait for approval (choose double approval).  
   - Message: Include product name, brand name, and AI-generated ad copies (`{{ $json.message.content }}`).  
   - Connect outputs of both OpenAI nodes here.  

6. **Add If Node to Check Approval Status**  
   - Name: "If: Ad Copy is Approve or not"  
   - Condition: Check if Slack message approved (`$json.data.approved` == true).  
   - True branch: Approved; False branch: Needs revision.  
   - Connect from Slack send and review node.  

7. **Slack Node for Approval Confirmation**  
   - Name: "Slack: Congrats on your approve Ad Copies"  
   - Sends confirmation message to Slack upon approval.  
   - Connect from If node True branch.  

8. **Slack Node to Receive Feedback for Revision**  
   - Name: "Slack: Receive feedback for revision"  
   - Operation: Send and wait for free-text response asking for revision feedback.  
   - Connect from If node False branch.  

9. **Slack Node to Thank for Feedback**  
   - Name: "Slack: Thank you for the feedback"  
   - Sends acknowledgment message after receiving feedback.  
   - Connect from Slack receive feedback node.  

10. **Set Node to Increment Revision Counter**  
    - Name: "Edit Fields: Revision Counter Max 3"  
    - Logic: `revisionCount = existing revisionCount + 1` or set to 1 if undefined.  
    - Connect from Slack thank you node.  

11. **If Node to Check Revision Limit**  
    - Name: "If: Continue Revise or Stop"  
    - Condition: `revisionCount > 3`  
    - True branch: Stop revisions.  
    - False branch: Continue revisions.  
    - Connect from revision counter node.  

12. **Slack Node to Notify Max Revisions Reached**  
    - Name: "Slack: Maxed Out Revision Count"  
    - Sends message that no more revisions will be made.  
    - Connect from If node True branch.  

13. **OpenAI Node to Revise Ad Copies**  
    - Name: "OpenAI: Revise Ad Copies"  
    - Prompt: Use previous ad copies, product info, and feedback text to generate new ad copies (3-5 variations).  
    - Connect from If node False branch.  

14. **Slack Node to Send Revised Ad Copies for Approval**  
    - Name: "Slack: Send Revise Ad Copies"  
    - Same configuration as initial Slack approval node.  
    - Connect from OpenAI revise node.  

15. **If Node to Check Approval of Revised Copies**  
    - Name: "If" (final approval check)  
    - Same condition as previous approval node.  
    - True branch: Connect to approval confirmation Slack node.  
    - False branch: Connect to feedback request Slack node (loop).  

16. **Connect Nodes Appropriately**  
    - Ensure loops between Slack feedback, thank you, revision counter, revision limit check, OpenAI revise, Slack send revised copies, and approval check are correctly wired to allow up to 3 revision cycles.  

17. **Optional: Add Sticky Notes**  
    - Add notes with overview and step comments for clarity.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| This workflow is designed for marketing teams, copywriters, and agencies to rapidly generate and iterate on ad copies with AI assistance and human approval. It balances automation with control by limiting revisions and using Slack’s interactive messages for feedback.                        | Workflow Overview (Sticky Note)                                                                                       |
| Requires OpenAI API keys with access to GPT-3.5 or GPT-4 models and Slack workspace permissions for posting and reading messages in approval channels.                                                                                                                                          | Setup requirements in Sticky Note                                                                                     |
| File upload in the form requires a self-hosted n8n instance or a setup that supports file uploads.                                                                                                                                                                                             | Technical requirement                                                                                                 |
| Customization tips: Adjust prompts in OpenAI nodes to align with your brand voice; modify revision limit in the Set node; add new brand types by expanding form dropdown and routing logic.                                                                                                     | Customization suggestions                                                                                             |
| For detailed Slack message formatting and interactive approvals, refer to n8n Slack node documentation: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/                                                                                                                  | Official documentation                                                                                               |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.