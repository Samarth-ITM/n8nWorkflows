Automated Nicereply Feedback Routing to MS Teams by Team and Sentiment

https://n8nworkflows.xyz/workflows/automated-nicereply-feedback-routing-to-ms-teams-by-team-and-sentiment-10113


# Automated Nicereply Feedback Routing to MS Teams by Team and Sentiment

### 1. Workflow Overview

This workflow automates the retrieval and routing of customer feedback collected via Nicereply to Microsoft Teams channels based on team assignment and sentiment analysis. It is designed to streamline feedback processing by categorizing responses by sentiment (happiness rating) and the presence or absence of comments, then directing the feedback to the appropriate internal teams (e.g., Support, Team Lead) in MS Teams.

Logical blocks:

- **1.1 Scheduled Trigger & Data Retrieval:** Periodically fetch new feedback from Nicereply API.
- **1.2 Data Preparation & Enrichment:** Split feedback into individual entries, normalize fields, set defaults, and remap survey IDs.
- **1.3 Filtering:** Filter feedback to only include submissions from the last 24 hours.
- **1.4 Sentiment & Comment Analysis:** Adjust happiness values for readability, then separate feedback by presence/absence of customer comments.
- **1.5 Routing & Notification:** Route feedback to distinct MS Teams channels or chats based on team type and sentiment, with customized messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

- **Overview:** This block initiates the workflow on a schedule and retrieves the latest feedback data from Nicereply.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Feedback  
  - Split Out
- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Purpose: Runs the workflow automatically every hour at the 1st minute mark.  
    - Config: Interval set to trigger at minute 1 every hour.  
    - Inputs: None (trigger node).  
    - Outputs: Triggers "Get Feedback" node.  
    - Edge Cases: Workflow will not run if n8n scheduler is disabled or misconfigured.

  - **Get Feedback**  
    - Type: HTTP Request  
    - Purpose: Fetches feedback responses from Nicereply API.  
    - Config:  
      - URL: `https://api.nicereply.com/responses`  
      - Authentication: HTTP Basic Auth (credentials must be configured).  
    - Inputs: Trigger from Schedule Trigger.  
    - Outputs: JSON array of feedback responses passed to "Split Out".  
    - Edge Cases: Authentication failures, network timeouts, API rate limits.

  - **Split Out**  
    - Type: Split Out  
    - Purpose: Splits the array of feedback data into individual items for granular processing.  
    - Config: Splits on the "data" field from the API response.  
    - Inputs: Output from Get Feedback node.  
    - Outputs: Individual feedback entries to "Edit Feedbacks".  
    - Edge Cases: Empty data arrays or unexpected data formats may cause failures.

---

#### 2.2 Data Preparation & Enrichment

- **Overview:** This block normalizes and enriches each feedback entry, remaps survey IDs to readable team names, and sets default values for missing data.  
- **Nodes Involved:**  
  - Edit Feedbacks  
  - Change survey ID according Nicereply (AI Transform)  
  - Filter  
  - Set Empty Respondent to "Unknown Client" (Code)  
  - Change happiness value (AI Transform)
- **Node Details:**

  - **Edit Feedbacks**  
    - Type: Set  
    - Purpose: Extracts and renames relevant fields from raw feedback JSON into a simplified format.  
    - Config: Assigns fields: Respondent, Ticket Number, Survey Date, Happiness (from scale value), Long Answer (open-ended), SurveyID, AgentID, Agent Name (empty for now).  
    - Inputs: Individual feedback items from Split Out.  
    - Outputs: Normalized JSON to "Change survey ID according Nicereply".  
    - Edge Cases: Missing fields in input JSON may result in undefined values.

  - **Change survey ID according Nicereply**  
    - Type: AI Transform  
    - Purpose: Remaps raw Nicereply SurveyID values to friendly team names (e.g., "Documentation", "Support").  
    - Config: Placeholder for AI instructions to convert SurveyID codes to readable names.  
    - Inputs: Output from "Edit Feedbacks".  
    - Outputs: Remapped SurveyID to "Filter".  
    - Edge Cases: Incorrect mapping rules or unrecognized SurveyIDs may cause routing errors.

  - **Filter**  
    - Type: Filter  
    - Purpose: Filters feedback entries to only those received in the last 24 hours.  
    - Config: Checks if "Survey Date" is after current time minus 24 hours.  
    - Inputs: Output from "Change survey ID according Nicereply".  
    - Outputs: Passing items to "Set Empty Respondent to \"Unknown Client\"" node.  
    - Edge Cases: Timezone issues, invalid date formats could cause incorrect filtering.

  - **Set Empty Respondent to "Unknown Client"**  
    - Type: Code (JavaScript)  
    - Purpose: Ensures that any feedback with a null Respondent field is set to "Unknown Client" to avoid missing data.  
    - Config: JavaScript code maps through items and replaces null Respondent values.  
    - Inputs: Filtered feedback entries.  
    - Outputs: Enriched feedback to "Change happiness value".  
    - Edge Cases: Code execution errors if input format changes unexpectedly.

  - **Change happiness value**  
    - Type: AI Transform  
    - Purpose: Converts numeric happiness scores (1-3) into descriptive sentiment labels such as "Great!", "OK", or "Bad".  
    - Config: AI instructions to map numeric values to textual descriptions.  
    - Inputs: Output from code node.  
    - Outputs: Routed to "Without Comment" node.  
    - Edge Cases: Unexpected numeric values or missing happiness scores.

---

#### 2.3 Sentiment & Comment Analysis

- **Overview:** This block branches feedback entries based on presence or absence of a customer comment and routes them accordingly.  
- **Nodes Involved:**  
  - Without Comment (IF)  
  - Switch - Type of Team  
  - Happiness Value  
  - Switch - Type of Team1  
  - Happiness Value2  
- **Node Details:**

  - **Without Comment**  
    - Type: IF  
    - Purpose: Checks if the "Long Answer" field (customer comment) is empty or not.  
    - Config: Condition checks if "Long Answer" is empty string.  
    - Inputs: Output from "Change happiness value".  
    - Outputs:  
      - True: routes to "Switch - Type of Team" (feedback without comments).  
      - False: routes to "Switch - Type of Team1" (feedback with comments).  
    - Edge Cases: Null or undefined fields might cause misclassification.

  - **Switch - Type of Team**  
    - Type: Switch  
    - Purpose: Routes feedback without comments to appropriate team branches based on remapped SurveyID.  
    - Config: Conditions for Team A, Team B, Team C by matching SurveyID field.  
    - Inputs: True branch of "Without Comment".  
    - Outputs: To "Happiness Value" node.  
    - Edge Cases: Missing or unmapped SurveyID leads to no output route.

  - **Happiness Value**  
    - Type: Switch  
    - Purpose: Routes feedback without comments further by sentiment label.  
    - Config: Checks happiness values equal to "Great!", "OK", or "Bad".  
    - Inputs: From "Switch - Type of Team".  
    - Outputs: To Teams message nodes ("Send to Support", "Send to Team Lead1").  
    - Edge Cases: Unexpected happiness labels.

  - **Switch - Type of Team1**  
    - Type: Switch  
    - Purpose: Routes feedback with comments to appropriate teams per SurveyID.  
    - Config: Same SurveyID-based routing as "Switch - Type of Team".  
    - Inputs: False branch of "Without Comment".  
    - Outputs: To "Happiness Value2".  
    - Edge Cases: Same as above.

  - **Happiness Value2**  
    - Type: Switch  
    - Purpose: Routes feedback with comments further by happiness sentiment.  
    - Config: Same as "Happiness Value".  
    - Inputs: From "Switch - Type of Team1".  
    - Outputs: To Teams message nodes ("Send to Support1", "Send to Team Lead").  
    - Edge Cases: Same as above.

---

#### 2.4 Routing & Notification

- **Overview:** This block sends formatted feedback messages to specific Microsoft Teams channels or chats based on team type and sentiment. There are separate nodes for feedback with and without comments.
- **Nodes Involved:**  
  - Send to Support  
  - Send to Support1  
  - Send to Team Lead  
  - Send to Team Lead1  
- **Node Details:**

  - **Send to Support**  
    - Type: Microsoft Teams  
    - Purpose: Sends feedback without comments to the Support MS Teams channel.  
    - Config:  
      - Team ID and Channel ID set to appropriate MS Teams identifiers for Support.  
      - Message formatted in HTML to include respondent name and happiness rating.  
    - Inputs: From "Happiness Value" node (no comment branch).  
    - Outputs: None.  
    - Edge Cases: Authentication failures, invalid team/channel IDs, API rate limits.

  - **Send to Support1**  
    - Type: Microsoft Teams  
    - Purpose: Sends feedback with comments to the Support MS Teams channel.  
    - Config: Similar to "Send to Support", message includes the long comment.  
    - Inputs: From "Happiness Value2" node (with comment branch).  
    - Outputs: None.  
    - Edge Cases: Same as above.

  - **Send to Team Lead**  
    - Type: Microsoft Teams  
    - Purpose: Sends feedback with comments to a specific Team Lead chat.  
    - Config:  
      - Uses a chatId (private chat) rather than a channel.  
      - Message includes respondent, happiness, and comment.  
    - Inputs: From "Happiness Value2" (with comment branch).  
    - Outputs: None.  
    - Edge Cases: Chat ID invalid or user permissions missing.

  - **Send to Team Lead1**  
    - Type: Microsoft Teams  
    - Purpose: Sends feedback without comments to Team Lead chat.  
    - Config: Similar to "Send to Team Lead", message omits long comment.  
    - Inputs: From "Happiness Value" (no comment branch).  
    - Outputs: None.  
    - Edge Cases: Same as above.

---

#### 2.5 Documentation & Notes

- **Overview:** Provides user documentation, instructions, and workflow description embedded as sticky notes.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3
- **Node Details:**

  - **Sticky Note**  
    - Content: "## Sending feedback with comment"  
    - Position: Near nodes handling feedback with comments.

  - **Sticky Note1**  
    - Content: "## Sending feedback without comment"  
    - Position: Near nodes handling feedback without comments.

  - **Sticky Note2**  
    - Content: Detailed project description, use cases, instructions, and resource links.  
    - Position: Top-left, overview area.

  - **Sticky Note3**  
    - Content: Workflow step summary and AI Transform explanation.  
    - Position: Top-left, near initial nodes.

---

### 3. Summary Table

| Node Name                         | Node Type                 | Functional Role                         | Input Node(s)                                  | Output Node(s)                            | Sticky Note                           |
|----------------------------------|---------------------------|---------------------------------------|-----------------------------------------------|------------------------------------------|-------------------------------------|
| Schedule Trigger                 | Schedule Trigger           | Initiates workflow on schedule        | None                                          | Get Feedback                            |                                     |
| Get Feedback                    | HTTP Request              | Retrieves feedback data from API      | Schedule Trigger                              | Split Out                              |                                     |
| Split Out                      | Split Out                 | Splits array into individual entries  | Get Feedback                                  | Edit Feedbacks                         |                                     |
| Edit Feedbacks                 | Set                       | Normalizes and extracts key fields    | Split Out                                     | Change survey ID according Nicereply    |                                     |
| Change survey ID according Nicereply | AI Transform              | Remaps survey IDs to team names       | Edit Feedbacks                                | Filter                                |                                     |
| Filter                        | Filter                    | Filters feedback from last 24 hours   | Change survey ID according Nicereply          | Set Empty Respondent to "Unknown Client" |                                     |
| Set Empty Respondent to "Unknown Client" | Code                      | Sets default respondent if missing    | Filter                                         | Change happiness value                |                                     |
| Change happiness value         | AI Transform              | Converts numeric scores to descriptive| Set Empty Respondent to "Unknown Client"       | Without Comment                       |                                     |
| Without Comment               | IF                        | Determines if comment is present      | Change happiness value                         | Switch - Type of Team, Switch - Type of Team1 |                                     |
| Switch - Type of Team          | Switch                    | Routes no-comment feedback by team    | Without Comment (true branch)                  | Happiness Value                       |                                     |
| Happiness Value               | Switch                    | Routes no-comment feedback by sentiment| Switch - Type of Team                          | Send to Support, Send to Team Lead1   |                                     |
| Switch - Type of Team1         | Switch                    | Routes feedback with comment by team  | Without Comment (false branch)                 | Happiness Value2                      |                                     |
| Happiness Value2              | Switch                    | Routes feedback with comment by sentiment| Switch - Type of Team1                        | Send to Support1, Send to Team Lead   |                                     |
| Send to Support               | Microsoft Teams           | Sends no-comment feedback to Support channel| Happiness Value                           | None                                 | Sticky Note1 (covers feedback without comment) |
| Send to Support1              | Microsoft Teams           | Sends feedback with comment to Support channel| Happiness Value2                         | None                                 | Sticky Note (covers feedback with comment) |
| Send to Team Lead             | Microsoft Teams           | Sends feedback with comment to Team Lead chat| Happiness Value2                        | None                                 | Sticky Note (covers feedback with comment) |
| Send to Team Lead1            | Microsoft Teams           | Sends no-comment feedback to Team Lead chat| Happiness Value                          | None                                 | Sticky Note1 (covers feedback without comment) |
| Sticky Note                  | Sticky Note               | Documentation: Sending feedback with comment| None                                        | None                                 |                                     |
| Sticky Note1                 | Sticky Note               | Documentation: Sending feedback without comment| None                                      | None                                 |                                     |
| Sticky Note2                 | Sticky Note               | Workflow overview and instructions    | None                                          | None                                 |                                     |
| Sticky Note3                 | Sticky Note               | Step summary and AI Transform notes   | None                                          | None                                 |                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set the trigger interval to run every hour at the 1st minute.  
   - No input required.

2. **Add HTTP Request Node "Get Feedback"**  
   - Connect input from Schedule Trigger.  
   - URL: `https://api.nicereply.com/responses`  
   - Authentication: HTTP Basic Auth using Nicereply API credentials (configure in Credentials).  
   - Method: GET (default).  
   - No body or query parameters needed.

3. **Add Split Out Node**  
   - Connect input from "Get Feedback".  
   - Field to split out: `data` (the array of feedback responses).  
   - This splits array into individual feedback items.

4. **Add Set Node "Edit Feedbacks"**  
   - Connect input from "Split Out".  
   - Configure assignments:  
     - Respondent = `{{$json.from}}`  
     - Ticket Number = `{{$json.ticket_id}}`  
     - Survey Date = `{{$json.created_at}}`  
     - Happiness = `{{$json.answers[0].scale.value}}`  
     - Long Answer = `{{$json.answers[1].open_ended.value}}`  
     - SurveyID = `{{$json.survey_id}}`  
     - AgentID = `{{$json.feedback_object_id}}`  
     - Agent Name = (empty string)  

5. **Add AI Transform Node "Change survey ID according Nicereply"**  
   - Connect input from "Edit Feedbacks".  
   - Provide AI instructions to remap raw SurveyIDs to friendly team names (e.g., "Documentation", "Support", "Consulting").  
   - Example: Input SurveyID `abcd-123-abcd-123` → Output `"Documentation"`.

6. **Add Filter Node**  
   - Connect input from the AI Transform.  
   - Set condition to only allow feedback where "Survey Date" is within last 24 hours (e.g., `={{ $json['Survey Date'] }} > {{$now.minus({hours: 24})}}`).

7. **Add Code Node "Set Empty Respondent to \"Unknown Client\""**  
   - Connect input from Filter.  
   - Use JavaScript code to replace any `null` in Respondent field with `"Unknown Client"`.  
   - Example code:  
     ```js
     const items = $input.all();
     return items.map(item => {
       if (item.json.Respondent === null) item.json.Respondent = "Unknown Client";
       return item;
     });
     ```

8. **Add AI Transform Node "Change happiness value"**  
   - Connect input from Code node.  
   - Provide AI instructions to map numeric happiness values to text:  
     `3 → "Great!"`, `2 → "OK"`, `1 → "Bad"`.

9. **Add IF Node "Without Comment"**  
   - Connect input from AI Transform.  
   - Condition: Check if `Long Answer` is empty (string is empty or null).  
   - True branch: feedback without comments.  
   - False branch: feedback with comments.

10. **Create Switch Node "Switch - Type of Team"**  
    - Connect input from "Without Comment" true branch.  
    - Create rules to match `SurveyID` to team names (Team A, Team B, Team C).  
    - Output routes to "Happiness Value" node.

11. **Create Switch Node "Happiness Value"**  
    - Connect input from "Switch - Type of Team".  
    - Branch by happiness text: "Great!", "OK", "Bad".  
    - Outputs connect to Microsoft Teams nodes for no-comment feedback.

12. **Create Switch Node "Switch - Type of Team1"**  
    - Connect input from "Without Comment" false branch.  
    - Same SurveyID team routing logic as "Switch - Type of Team".  
    - Output routes to "Happiness Value2".

13. **Create Switch Node "Happiness Value2"**  
    - Connect input from "Switch - Type of Team1".  
    - Branch by happiness text as above.  
    - Outputs connect to Microsoft Teams nodes for feedback with comments.

14. **Create Microsoft Teams Node "Send to Support"**  
    - Connect input from "Happiness Value" (no comment branch).  
    - Configure Team ID and Channel ID corresponding to Support team.  
    - Message content example (HTML):  
      `We’ve received feedback for <b>Support</b>!<br>{{ $json.Respondent }} thinks our service was <b>{{ $json.Happiness }}</b>`.

15. **Create Microsoft Teams Node "Send to Support1"**  
    - Connect input from "Happiness Value2" (with comment branch).  
    - Configure Team ID and Channel ID for Support.  
    - Message includes respondent, happiness, and long comment.

16. **Create Microsoft Teams Node "Send to Team Lead"**  
    - Connect input from "Happiness Value2".  
    - Configure chatId for private Team Lead chat.  
    - Message includes respondent, happiness, and comment.

17. **Create Microsoft Teams Node "Send to Team Lead1"**  
    - Connect input from "Happiness Value" (no comment).  
    - Configure chatId for Team Lead chat.  
    - Message includes respondent and happiness only.

18. **Add Sticky Note Nodes**  
    - Add documentation and usage notes as described, positioning near relevant nodes.

19. **Configure Credentials**  
    - Set up HTTP Basic Auth credentials for Nicereply API in n8n.  
    - Set up Microsoft Teams OAuth2 credentials with permissions to post messages to channels and chats.

20. **Test Workflow**  
    - Run manually or wait for scheduled trigger.  
    - Verify messages appear correctly in MS Teams channels and chats.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This workflow uses native n8n nodes combined with AI Transform nodes to automate and customize feedback routing. The AI Transform nodes require proper AI instructions and may need access to an AI service configured in n8n.                                                                                                                                                                                                                     | AI Transform details: https://docs.n8n.io/nodes/n8n-nodes-base.aiTransform/                              |
| The workflow is designed to be scalable: users can add more teams in the Switch nodes and expand routing logic accordingly.                                                                                                                                                                                                                                                                                                                  |                                                                                                          |
| Microsoft Teams nodes require valid team IDs, channel IDs, and chat IDs. These can be obtained from Teams UI or via Microsoft Graph API.                                                                                                                                                                                                                                                                                                    | MS Teams developer docs: https://learn.microsoft.com/en-us/microsoftteams/platform/                       |
| Nicereply API credentials must have read permissions for responses and be set up as HTTP Basic Auth credentials in n8n.                                                                                                                                                                                                                                                                                                                     | Nicereply API documentation: https://www.nicereply.com/api/                                              |
| For best results, test with small datasets and verify message formatting in Teams before full deployment.                                                                                                                                                                                                                                                                                                                                    |                                                                                                          |
| Workflow includes detailed sticky notes explaining logic and usage, plus links to n8n community and Easy8.ai support resources.                                                                                                                                                                                                                                                                                                             | n8n community: https://community.n8n.io/u/easy8.ai, YouTube: https://www.youtube.com/@easy8ai             |

---

This completes the comprehensive analysis and documentation of the "Automated Nicereply Feedback Routing to MS Teams by Team and Sentiment" workflow.