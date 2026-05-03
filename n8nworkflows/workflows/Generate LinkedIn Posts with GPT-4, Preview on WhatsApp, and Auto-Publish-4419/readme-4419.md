Generate LinkedIn Posts with GPT-4, Preview on WhatsApp, and Auto-Publish

https://n8nworkflows.xyz/workflows/generate-linkedin-posts-with-gpt-4--preview-on-whatsapp--and-auto-publish-4419


# Generate LinkedIn Posts with GPT-4, Preview on WhatsApp, and Auto-Publish

### 1. Workflow Overview

This workflow automates the generation, preview, approval, and publishing of LinkedIn posts using GPT-4 and WhatsApp integration. It is designed to create unique content around trending AI topics, allow preview and approval through WhatsApp messaging, and finally auto-publish approved posts to LinkedIn. The workflow is split into six logical blocks:

- **1.1 Scheduling:** Periodically triggers the content generation process.
- **1.2 AI Generation:** Uses GPT-4 to generate LinkedIn post content based on selected AI topics.
- **1.3 Processing:** Formats and prepares the AI-generated content for preview.
- **1.4 Approval System:** Sends the content preview via WhatsApp and waits for approval or rejection.
- **1.5 Publishing:** Formats the approved content and posts it automatically to LinkedIn, followed by a success notification.
- **1.6 Smart Restart:** Handles declined content by notifying via WhatsApp and restarting content generation.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduling

- **Overview:**  
  This block initiates the workflow every 6 hours (adjustable) to kickstart the LinkedIn content generation process.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Prepare Search Topics

- **Node Details:**

  - **Schedule Trigger**  
    - *Type & Role:* Schedule trigger node to start the workflow periodically.  
    - *Configuration:* Runs every 6 hours by default (modifiable schedule).  
    - *Connections:* Outputs to "Prepare Search Topics".  
    - *Potential Failures:* Misconfiguration of the schedule may cause missed triggers.  
    - *Notes:* Runs every 6 hours - adjust timing as needed for your posting schedule.

  - **Prepare Search Topics**  
    - *Type & Role:* Function node selecting random AI-related topics for content generation.  
    - *Configuration:* Uses a predefined array of search terms representing AI topics, selects one or multiple randomly.  
    - *Key Variables:* `searchTerms` array containing niche topics.  
    - *Connections:* Takes input from "Schedule Trigger" and outputs to "AI Content Generator".  
    - *Potential Failures:* Empty or malformed `searchTerms` array; function errors if random selection fails or no input.  
    - *Notes:* Modify `searchTerms` array for your specific niche or trending topics.

---

#### 2.2 AI Generation

- **Overview:**  
  Generates unique LinkedIn post content based on selected AI topics by leveraging GPT-4 powered nodes.

- **Nodes Involved:**  
  - AI Content Generator  
  - OpenAI GPT-4 Model

- **Node Details:**

  - **AI Content Generator**  
    - *Type & Role:* Langchain Agent node that drives content generation logic.  
    - *Configuration:* Uses GPT-4 as underlying model, builds prompt around trending AI topics.  
    - *Connections:* Receives input from "Prepare Search Topics", sends output to "Process AI Content".  
    - *Version:* 1.9 (Langchain agent version)  
    - *Potential Failures:* API quota limits, prompt errors, invalid input format.  
    - *Notes:* Customize the prompt to fit your industry or target audience.

  - **OpenAI GPT-4 Model**  
    - *Type & Role:* Language model node providing GPT-4 capabilities.  
    - *Configuration:* Requires valid OpenAI API key credential.  
    - *Connections:* Linked as the language model target for "AI Content Generator".  
    - *Version:* 1.2  
    - *Potential Failures:* Authentication errors, API timeouts, rate limits.

---

#### 2.3 Processing

- **Overview:**  
  Processes the raw AI-generated text, parsing and formatting it for LinkedIn posting.

- **Nodes Involved:**  
  - Process AI Content

- **Node Details:**

  - **Process AI Content**  
    - *Type & Role:* Function node to parse and format AI response into LinkedIn-ready content.  
    - *Configuration:* Extracts relevant text from AI output, cleans formatting, applies any required text transformations.  
    - *Connections:* Receives from "AI Content Generator", outputs to "Send Content Preview".  
    - *Potential Failures:* Parsing errors if AI output is unexpected; expression failures if text is malformed.

---

#### 2.4 Approval System

- **Overview:**  
  Sends the formatted content preview to the user's phone via WhatsApp for review and waits for approval before proceeding.

- **Nodes Involved:**  
  - Send Content Preview  
  - WhatsApp Approval Gateway  
  - Check Approval Status

- **Node Details:**

  - **Send Content Preview**  
    - *Type & Role:* WhatsApp node that sends the generated content preview.  
    - *Configuration:* Uses WhatsApp API credentials, configured to send a message to a predefined phone number.  
    - *Connections:* Receives from "Process AI Content", outputs to "WhatsApp Approval Gateway".  
    - *Potential Failures:* Messaging API errors, invalid phone numbers, connectivity issues.  
    - *Webhook:* Has webhook ID for WhatsApp messaging.

  - **WhatsApp Approval Gateway**  
    - *Type & Role:* WhatsApp node waiting for user’s approval reply.  
    - *Configuration:* Listens for WhatsApp responses with a webhook, designed to capture approval or decline input.  
    - *Connections:* Receives from "Send Content Preview", outputs to "Check Approval Status".  
    - *Potential Failures:* Delayed or missing user responses, webhook failures.  
    - *Webhook:* Has webhook ID for WhatsApp message reception.

  - **Check Approval Status**  
    - *Type & Role:* If node routing workflow based on approval input.  
    - *Configuration:* Checks the WhatsApp reply; routes to publish if approved, or decline path otherwise.  
    - *Connections:* Receives from "WhatsApp Approval Gateway", outputs to "Format LinkedIn Post" on approval, or "Send Decline Notification" on rejection.  
    - *Potential Failures:* Misinterpretation of input, expression errors.

---

#### 2.5 Publishing

- **Overview:**  
  Formats the approved content and posts it automatically to LinkedIn, then sends a success notification via WhatsApp.

- **Nodes Involved:**  
  - Format LinkedIn Post  
  - Publish to LinkedIn  
  - Send Success Notification

- **Node Details:**

  - **Format LinkedIn Post**  
    - *Type & Role:* Function node to finalize content format for LinkedIn.  
    - *Configuration:* Applies any final text adjustments, hashtags, or metadata required by LinkedIn.  
    - *Connections:* Receives from "Check Approval Status" (approval branch), outputs to "Publish to LinkedIn".  
    - *Potential Failures:* Formatting errors, text length issues.

  - **Publish to LinkedIn**  
    - *Type & Role:* LinkedIn node that posts the content to the user’s LinkedIn profile.  
    - *Configuration:* Requires LinkedIn OAuth2 credentials with posting permissions.  
    - *Connections:* Receives from "Format LinkedIn Post", outputs to "Send Success Notification".  
    - *Potential Failures:* Authentication failures, API rate limits, invalid post content.

  - **Send Success Notification**  
    - *Type & Role:* WhatsApp node notifying user of successful post publication.  
    - *Configuration:* Sends confirmation message to predefined phone number.  
    - *Connections:* Receives from "Publish to LinkedIn".  
    - *Potential Failures:* Messaging issues, webhook errors.

---

#### 2.6 Smart Restart

- **Overview:**  
  Handles declined posts by notifying the user and restarting the content generation process automatically.

- **Nodes Involved:**  
  - Send Decline Notification  
  - Restart Content Generation

- **Node Details:**

  - **Send Decline Notification**  
    - *Type & Role:* WhatsApp node that informs the user that the content was rejected and the process will restart.  
    - *Configuration:* Sends a decline message to the user’s phone.  
    - *Connections:* Receives from "Check Approval Status" (decline branch), outputs to "Restart Content Generation".  
    - *Potential Failures:* Messaging API errors.

  - **Restart Content Generation**  
    - *Type & Role:* Execute Workflow node that triggers the workflow again from the "Prepare Search Topics" node.  
    - *Configuration:* Linked to the same workflow, starting at the content preparation stage.  
    - *Connections:* Receives from "Send Decline Notification", outputs to "Prepare Search Topics".  
    - *Potential Failures:* Recursive loop risk if approvals are never given; system resource exhaustion.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                            | Input Node(s)               | Output Node(s)                | Sticky Note                                               |
|-------------------------|----------------------------------|--------------------------------------------|-----------------------------|------------------------------|-----------------------------------------------------------|
| 📋 Workflow Overview    | Sticky Note                      | Overview of the workflow                    |                             |                              |                                                           |
| ⚙️ Setup Guide          | Sticky Note                      | Setup instructions                          |                             |                              |                                                           |
| 🕐 Stage 1: Scheduling   | Sticky Note                      | Scheduling block marker                      |                             |                              |                                                           |
| Schedule Trigger        | Schedule Trigger                 | Triggers workflow every 6 hours             |                             | Prepare Search Topics         | 🕐 Runs every 6 hours - adjust timing as needed for your posting schedule |
| Prepare Search Topics   | Function                        | Selects random AI topics                     | Schedule Trigger             | AI Content Generator          | 🎯 Selects random AI topics for content generation. Modify searchTerms array for your niche. |
| 🧠 Stage 2: AI Generation | Sticky Note                      | AI generation block marker                   |                             |                              |                                                           |
| AI Content Generator    | Langchain Agent (GPT-4)          | Generates LinkedIn content using GPT-4      | Prepare Search Topics        | Process AI Content            | 🤖 GPT-4 generates unique LinkedIn content based on trending AI topics. Customize the prompt for your industry/niche. |
| OpenAI GPT-4 Model      | Langchain OpenAI Chat Model      | GPT-4 language model                         |                             | AI Content Generator (LM)    | 🧠 GPT-4 language model for content generation. Requires OpenAI API key. |
| ⚙️ Stage 3: Processing  | Sticky Note                      | Processing block marker                      |                             |                              |                                                           |
| Process AI Content      | Function                        | Parse and format AI response                 | AI Content Generator         | Send Content Preview          | ⚙️ Parses AI response and formats content for LinkedIn posting |
| 📲 Stage 4: Approval System | Sticky Note                      | Approval system block marker                  |                             |                              |                                                           |
| Send Content Preview    | WhatsApp                        | Sends content preview to phone               | Process AI Content           | WhatsApp Approval Gateway     | 📱 Sends generated content to your phone for preview before publishing |
| WhatsApp Approval Gateway | WhatsApp                        | Waits for approval reply via WhatsApp        | Send Content Preview         | Check Approval Status         | ⏳ Waits for your approval via WhatsApp before proceeding |
| Check Approval Status   | If                             | Routes based on approval decision             | WhatsApp Approval Gateway    | Format LinkedIn Post, Send Decline Notification | ✅ Routes workflow based on your approval decision |
| 🎯 Stage 5: Publishing  | Sticky Note                      | Publishing block marker                      |                             |                              |                                                           |
| Format LinkedIn Post    | Function                        | Final formatting for LinkedIn post           | Check Approval Status (Yes)  | Publish to LinkedIn           | ✨ Final formatting of approved content for LinkedIn posting |
| Publish to LinkedIn     | LinkedIn                       | Posts content automatically                   | Format LinkedIn Post         | Send Success Notification     | 🔗 Auto-posts approved content to your LinkedIn profile |
| Send Success Notification | WhatsApp                        | Sends success confirmation                    | Publish to LinkedIn          |                              | 🎉 Confirms successful LinkedIn posting                   |
| ♻️ Stage 6: Smart Restart | Sticky Note                      | Smart restart block marker                   |                             |                              |                                                           |
| Send Decline Notification | WhatsApp                        | Notifies content decline and restart         | Check Approval Status (No)   | Restart Content Generation    | ❌ Notifies that content was declined and workflow will restart |
| Restart Content Generation | Execute Workflow               | Restarts content generation process           | Send Decline Notification    | Prepare Search Topics         | 🔄 Restarts workflow to generate new content after decline |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger every 6 hours (adjust as needed).

2. **Create Prepare Search Topics Function Node**  
   - Connect input from "Schedule Trigger".  
   - Define an array `searchTerms` with AI or niche topics.  
   - Randomly select a topic to output for content generation.

3. **Set up OpenAI GPT-4 Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Configure with valid OpenAI API credentials.  
   - Version should be 1.2 or compatible.

4. **Create AI Content Generator Node**  
   - Type: Langchain Agent (GPT-4)  
   - Connect input from "Prepare Search Topics".  
   - Configure to use the OpenAI GPT-4 Model node as its language model.  
   - Customize the prompt template to generate LinkedIn posts based on input topics.

5. **Add Process AI Content Function Node**  
   - Connect input from "AI Content Generator".  
   - Implement parsing logic to extract and format the AI-generated text for LinkedIn.

6. **Create Send Content Preview WhatsApp Node**  
   - Connect input from "Process AI Content".  
   - Configure WhatsApp API credentials and target phone number.  
   - Set to send the formatted content as a message.

7. **Create WhatsApp Approval Gateway Node**  
   - Connect input from "Send Content Preview".  
   - Configure webhook to listen for approval or decline responses from WhatsApp.

8. **Add Check Approval Status If Node**  
   - Connect input from "WhatsApp Approval Gateway".  
   - Define condition to check if the reply is approval (e.g., message contains "yes" or "approve").  
   - Output true branch to "Format LinkedIn Post"; false branch to "Send Decline Notification".

9. **Create Format LinkedIn Post Function Node**  
   - Connect input from true branch of "Check Approval Status".  
   - Implement final formatting for LinkedIn post (hashtags, mentions, etc.).

10. **Add Publish to LinkedIn Node**  
    - Connect input from "Format LinkedIn Post".  
    - Configure LinkedIn OAuth2 credentials with posting rights.

11. **Create Send Success Notification WhatsApp Node**  
    - Connect input from "Publish to LinkedIn".  
    - Configure WhatsApp to notify success confirmation.

12. **Create Send Decline Notification WhatsApp Node**  
    - Connect input from false branch of "Check Approval Status".  
    - Configure WhatsApp to notify content rejection.

13. **Add Restart Content Generation Execute Workflow Node**  
    - Connect input from "Send Decline Notification".  
    - Set to call the same workflow, starting from "Prepare Search Topics".  
    - Ensure parameters and context are reset appropriately.

14. **Test entire workflow end-to-end**  
    - Validate scheduling triggers.  
    - Confirm AI content generation and formatting.  
    - Verify WhatsApp preview sending and approval response handling.  
    - Check LinkedIn posting and notification delivery.  
    - Test decline path and workflow restart.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                        |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Workflow runs every 6 hours by default; adjust schedule to fit your posting frequency needs.         | Scheduling recommendations                             |
| Customize AI prompt in "AI Content Generator" to better match your brand tone or niche.              | AI content customization                               |
| WhatsApp integration requires webhook setup and valid API credentials.                              | WhatsApp API documentation                             |
| LinkedIn node requires OAuth2 credentials with permissions to post content on your profile.          | LinkedIn Developer Portal                              |
| Recursive workflow restart may cause infinite loops if never approved; implement safeguards if needed.| Workflow stability advice                              |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. It complies strictly with content policies, contains no illegal or offensive elements, and manipulates only legal and public data.