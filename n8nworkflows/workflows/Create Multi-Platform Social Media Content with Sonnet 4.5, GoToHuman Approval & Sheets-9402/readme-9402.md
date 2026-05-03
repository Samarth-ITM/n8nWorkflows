Create Multi-Platform Social Media Content with Sonnet 4.5, GoToHuman Approval & Sheets

https://n8nworkflows.xyz/workflows/create-multi-platform-social-media-content-with-sonnet-4-5--gotohuman-approval---sheets-9402


# Create Multi-Platform Social Media Content with Sonnet 4.5, GoToHuman Approval & Sheets

### 1. Workflow Overview

This workflow automates the creation, human review, and management of social media content ideas across multiple platforms using AI and integration with Google Sheets and GoToHuman. It is designed for marketing teams or social media managers who want to streamline content generation and approval processes, optimizing content for various platforms like Instagram, Facebook, LinkedIn, and X.com.

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Triggering the workflow manually or on a schedule and retrieving content ideas from a Google Sheet editorial plan.
- **1.2 AI Content Generation**: Using Anthropic Claude Sonnet 4.5 via LangChain agent to generate platform-optimized social media posts from ideas.
- **1.3 Human Review via GoToHuman**: Sending generated content for human approval or rejection and handling the response.
- **1.4 Post-Review Processing**: Updating the Google Sheet based on approval status or regenerating content if rejected.
- **1.5 Control Flow & Looping**: Managing batch processing of multiple content ideas and branching logic based on review outcomes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow either manually or by schedule, then fetches unapproved content ideas from a Google Sheet to process them one by one.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger  
  - Get ideas (Google Sheets)  
  - Loop Over Items (SplitInBatches)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow on-demand.  
    - *Config:* No parameters; simple manual execution.  
    - *Connections:* Outputs to ‘Get ideas’.  
    - *Edge cases:* None significant; user-triggered.

  - **Schedule Trigger**  
    - *Type:* Scheduled Trigger  
    - *Role:* Enables periodic automatic start (interval set but empty, meaning defaults to every hour).  
    - *Config:* Interval empty, set to default.  
    - *Connections:* Not connected in the given flow, likely optional or alternative trigger.

  - **Get ideas**  
    - *Type:* Google Sheets node (read)  
    - *Role:* Reads rows from the editorial plan sheet filtering for entries where 'APPROVED' is empty (pending ideas).  
    - *Config:* Reads from specific Google Sheet and sheet tab (gid=0). Filters applied on the 'APPROVED' column to only get unapproved ideas.  
    - *Credentials:* Google Sheets OAuth2.  
    - *Connections:* Outputs to ‘Loop Over Items’.  
    - *Edge cases:* API quota limits, authentication failures, empty results.

  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Processes each idea individually in batches to handle one item at a time downstream.  
    - *Config:* Default batch size (likely 1).  
    - *Connections:* First output empty (no-op), second output leads to ‘Social Media Content Creator’.  
    - *Edge cases:* Large datasets may cause execution delays or timeouts.

#### 2.2 AI Content Generation

- **Overview:**  
  Uses AI (Anthropic Claude Sonnet 4.5) via LangChain to generate platform-specific social media post content based on the idea and platform.

- **Nodes Involved:**  
  - Anthropic Chat Model (Claude Sonnet 4.5)  
  - Social Media Content Creator (LangChain Agent)  
  - Set Text  
  - Set new prompt

- **Node Details:**

  - **Anthropic Chat Model**  
    - *Type:* AI Language Model Node (Anthropic)  
    - *Role:* Provides AI language model capabilities (Claude Sonnet 4.5) to the LangChain agent.  
    - *Config:* Uses ‘claude-sonnet-4-5-20250929’ model.  
    - *Credentials:* Anthropic API key.  
    - *Connections:* AI model input for ‘Social Media Content Creator’.  
    - *Edge cases:* API failures, rate limits, model unavailability.

  - **Social Media Content Creator**  
    - *Type:* LangChain Agent Node  
    - *Role:* Transforms post idea and platform into optimized social media text.  
    - *Config:*  
      - Input text is the ‘IDEA’ field from Google Sheets.  
      - System Message defines detailed instructions on style, tone, platform adaptations, and content rules.  
      - Output is only the post text, no explanations.  
    - *Connections:* Outputs to ‘Set Text’.  
    - *Edge cases:* Expression errors if input fields missing, AI content not aligned with requirements.

  - **Set Text**  
    - *Type:* Set Node  
    - *Role:* Maps AI output text into a ‘text’ property for GoToHuman review input.  
    - *Config:* Sets ‘text’ = AI-generated output.  
    - *Connections:* Outputs to ‘Send review request and wait for response’.  
    - *Edge cases:* Empty or malformed AI output causing review to fail.

  - **Set new prompt**  
    - *Type:* Set Node  
    - *Role:* When content is rejected, updates the prompt to regenerate content with latest feedback.  
    - *Config:* Sets ‘IDEA’ and ‘PLATFORM’ from response values and metadata to feed back into content creator.  
    - *Connections:* Outputs to ‘Social Media Content Creator’.  
    - *Edge cases:* Missing meta data or response values causing failure.

#### 2.3 Human Review via GoToHuman

- **Overview:**  
  Sends generated posts to GoToHuman platform for human review, waits for the approval or rejection response, then routes accordingly.

- **Nodes Involved:**  
  - Send review request and wait for response (GoToHuman)  
  - Switch

- **Node Details:**

  - **Send review request and wait for response**  
    - *Type:* GoToHuman node  
    - *Role:* Sends the AI-generated post text for human review using a configured review template, waits for response.  
    - *Config:*  
      - Sends ‘text’ field from AI output.  
      - Metadata includes the ‘PLATFORM’ for contextual info.  
      - Uses a predefined Review Template ID.  
    - *Credentials:* GoToHuman API key.  
    - *Connections:* Outputs to ‘Switch’ node.  
    - *Edge cases:* API errors, webhook delays, invalid template ID.

  - **Switch**  
    - *Type:* Switch (conditional branching)  
    - *Role:* Routes the workflow based on reviewer’s response: 'approved' or 'rejected'.  
    - *Config:* Checks the ‘response’ field for exact string matches ‘approved’ or ‘rejected’ (case-sensitive).  
    - *Connections:*  
      - ‘approved’ branch leads to ‘Update row in sheet’.  
      - ‘rejected’ branch leads to ‘Set new prompt’ (to regenerate).  
    - *Edge cases:* Unexpected or missing response values causing dead ends.

#### 2.4 Post-Review Processing

- **Overview:**  
  Updates the Google Sheet with the approved content or loops back if rejected for regeneration.

- **Nodes Involved:**  
  - Update row in sheet  
  - Loop Over Items (loop back)

- **Node Details:**

  - **Update row in sheet**  
    - *Type:* Google Sheets node (update)  
    - *Role:* Updates the row corresponding to the content idea with approved post text and marks it as approved.  
    - *Config:*  
      - Updates ‘POST’ column with approved text.  
      - Sets ‘APPROVED’ column to ‘x’.  
      - Uses ‘row_number’ to match the correct row.  
      - Targets specific Google Sheet and tab (gid=0).  
    - *Credentials:* Google Sheets OAuth2.  
    - *Connections:* Outputs back to ‘Loop Over Items’ to process next item.  
    - *Edge cases:* Write permission errors, concurrent update conflicts.

  - **Loop Over Items** (second output)  
    - *Role:* Allows repeating the content generation for rejected posts by looping back to ‘Social Media Content Creator’.  
    - *Edge cases:* Infinite loop if rejection never resolved.

#### 2.5 Control Flow & Looping

- **Overview:**  
  Manages processing multiple ideas sequentially and controlling the flow based on review results.

- **Nodes Involved:**  
  - Loop Over Items  
  - Switch

- **Node Details:**

  - **Loop Over Items**  
    - Controls batch processing, one content idea at a time to ensure orderly review and update.  
    - Has two outputs:  
      - First output: empty (no action)  
      - Second output: feeds content to AI generator.  
    - Loops back from update node after approval for next idea.

  - **Switch**  
    - Routes based on review status to either update the sheet or regenerate the content.

---

### 3. Summary Table

| Node Name                          | Node Type                     | Functional Role                                  | Input Node(s)                     | Output Node(s)                           | Sticky Note                                                                                               |
|-----------------------------------|-------------------------------|-------------------------------------------------|----------------------------------|-----------------------------------------|-----------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’  | Manual Trigger                | Manual start of workflow                         | -                                | Get ideas                              |                                                                                                           |
| Schedule Trigger                  | Schedule Trigger             | Optional scheduled start                         | -                                | -                                       |                                                                                                           |
| Get ideas                        | Google Sheets                | Fetch unapproved content ideas                   | When clicking ‘Execute workflow’ | Loop Over Items                         |                                                                                                           |
| Loop Over Items                  | SplitInBatches               | Processes ideas individually                      | Get ideas                       | Social Media Content Creator, empty output |                                                                                                           |
| Anthropic Chat Model             | AI Language Model (Anthropic) | Provides AI model for content generation         | -                                | Social Media Content Creator (AI input) |                                                                                                           |
| Social Media Content Creator     | LangChain Agent              | Generates platform-optimized social posts        | Loop Over Items, Set new prompt  | Set Text                               |                                                                                                           |
| Set Text                        | Set                         | Maps AI output to ‘text’ for review              | Social Media Content Creator     | Send review request and wait for response |                                                                                                           |
| Send review request and wait for response | GoToHuman                  | Sends content for human review & waits response | Set Text                       | Switch                                |                                                                                                           |
| Switch                         | Switch                      | Routes based on approval / rejection             | Send review request and wait for response | Update row in sheet, Set new prompt   |                                                                                                           |
| Update row in sheet             | Google Sheets (update)       | Updates approved post in Google Sheet            | Switch (approved branch)          | Loop Over Items                         |                                                                                                           |
| Set new prompt                 | Set                         | Prepares prompt to regenerate content if rejected | Switch (rejected branch)          | Social Media Content Creator            |                                                                                                           |
| Sticky Note                    | Sticky Note                 | Workflow description and instructions            | -                                | -                                       | ## AI-Powered Multiplatform Social Media Content Creator with Human-in-the-Loop (GoToHuman) Review         |
| Sticky Note1                   | Sticky Note                 | Instruction for step 1: Google Sheet setup       | -                                | -                                       | ## STEP 1 - Clone [this Google Sheet](https://docs.google.com/spreadsheets/d/1WLdKRpPxwLzvD27uHZIlY5Lcc5i-79PmapcWyYzJevI/edit?usp=sharing) - Fill in the "Date" and "Idea" fields and select the social media platform. |
| Sticky Note2                   | Sticky Note                 | Instruction for step 2: GoToHuman setup          | -                                | -                                       | ## STEP 2 - Install GoToHuman node in your n8n instance - Go to [GoToHuman](https://www.gotohuman.com/) website - Create a "New Review Template" - Get your API KEY - Add the template in n8n "GoToHuman" loop |
| Sticky Note3                   | Sticky Note                 | Reminder to set review template                   | -                                | -                                       | Set your review template                                                                                   |
| Sticky Note4                   | Sticky Note                 | Reminder to set new prompt                         | -                                | -                                       | Set your new prompt                                                                                         |
| Sticky Note5                   | Sticky Note                 | Reminder about approval check                      | -                                | -                                       | If the content generated is approved or not                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Purpose: Manual start of the workflow.

2. **(Optional) Create a Schedule Trigger node**  
   - Name: `Schedule Trigger`  
   - Set interval as desired (default runs hourly if left empty).

3. **Add a Google Sheets node (Read operation)**  
   - Name: `Get ideas`  
   - Connect manual trigger output to this node.  
   - Configure credentials with Google Sheets OAuth2 account.  
   - Document ID: Use or clone the editorial plan spreadsheet.  
   - Sheet Name: Select the first tab (`gid=0`).  
   - Add filter on the ‘APPROVED’ column to only retrieve rows where it is empty.  
   - Output all columns including ‘IDEA’, ‘PLATFORM’, and ‘row_number’.

4. **Add a SplitInBatches node**  
   - Name: `Loop Over Items`  
   - Connect from ‘Get ideas’.  
   - Default batch size (1) to process each idea individually.

5. **Add an Anthropic Chat Model node**  
   - Name: `Anthropic Chat Model`  
   - Configure with Anthropic API credentials.  
   - Model: Select ‘claude-sonnet-4-5-20250929’.  
   - No extra parameters needed.

6. **Add a LangChain Agent node**  
   - Name: `Social Media Content Creator`  
   - Connect second output of `Loop Over Items` to this node.  
   - Set the text input to the ‘IDEA’ field from the item.  
   - Choose ‘Define’ prompt type.  
   - Paste the detailed system message provided for platform-specific content creation and style instructions.  
   - Set the Anthropic node as the language model for this agent.

7. **Add a Set node**  
   - Name: `Set Text`  
   - Connect from ‘Social Media Content Creator’.  
   - Set a new field ‘text’ with value from AI output (e.g., `{{$json.output}}`).

8. **Add GoToHuman node**  
   - Name: `Send review request and wait for response`  
   - Connect from ‘Set Text’.  
   - Configure GoToHuman API credentials.  
   - Set the review template ID (create on GoToHuman platform beforehand).  
   - Map the ‘text’ field to send.  
   - Add metadata key ‘PLATFORM’ from current item platform value.  
   - Enable webhook to wait for response.

9. **Add a Switch node**  
   - Name: `Switch`  
   - Connect from ‘Send review request and wait for response’.  
   - Add two outputs:  
     - Condition ‘approved’: when response equals string ‘approved’ (case-sensitive).  
     - Condition ‘rejected’: when response equals string ‘rejected’.

10. **Add Google Sheets (Update) node**  
    - Name: `Update row in sheet`  
    - Connect from Switch ‘approved’ output.  
    - Configure with Google Sheets OAuth2 credentials.  
    - Target the editorial plan sheet.  
    - Update the row using ‘row_number’ to:  
      - Set ‘POST’ column to the approved text (`{{$json.responseValues.text.value}}`).  
      - Set ‘APPROVED’ column to ‘x’.  
    - Connect output back to ‘Loop Over Items’ input to continue batch processing.

11. **Add a Set node**  
    - Name: `Set new prompt`  
    - Connect from Switch ‘rejected’ output.  
    - Set fields ‘IDEA’ and ‘PLATFORM’ from current response to regenerate content.  
    - Connect output back to ‘Social Media Content Creator’ to retry content generation.

12. **Connect the first output of `Loop Over Items` to empty or no further action node** (optional, can be left unconnected).

13. **Add Sticky Notes** (optional but recommended)  
    - Add notes with instructions for:  
      - Cloning the Google Sheet editorial plan.  
      - Setting up GoToHuman review templates and API keys.  
      - Descriptions of each step for user clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                      | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Clone [this Google Sheet](https://docs.google.com/spreadsheets/d/1WLdKRpPxwLzvD27uHZIlY5Lcc5i-79PmapcWyYzJevI/edit?usp=sharing) for the editorial plan.                           | Editorial plan sheet template for content ideas input.                                                          |
| Install GoToHuman node in your n8n instance and create a review template on [GoToHuman](https://www.gotohuman.com/). Get the API key and template ID for integration.           | GoToHuman platform for human-in-the-loop content review.                                                        |
| The AI model used is Anthropic Claude Sonnet 4.5 (`claude-sonnet-4-5-20250929`) accessed via LangChain agent node for advanced content generation with specific platform instructions. | AI content generation model and setup.                                                                           |
| Workflow enables seamless collaboration between AI-generated content and human approval, optimizing social media posts for multiple platforms with brand safety and engagement. | Overall workflow purpose and benefits.                                                                           |

---

**Disclaimer:** The provided text is extracted exclusively from an automated workflow created with n8n, respecting all applicable content policies. All data handled is legal and public.