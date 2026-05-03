Route Feedback by Sentiment Using Typeform, GCP Natural Language, Notion & Trello

https://n8nworkflows.xyz/workflows/route-feedback-by-sentiment-using-typeform--gcp-natural-language--notion---trello-7243


# Route Feedback by Sentiment Using Typeform, GCP Natural Language, Notion & Trello

### 1. Workflow Overview

This workflow automates the routing and handling of user feedback collected via Typeform by analyzing sentiment with Google Cloud Natural Language, storing data in Notion, notifying a Slack channel of positive feedback, and creating Trello cards for follow-up actions. It is designed for organizations seeking to streamline feedback management by automatically categorizing and distributing user comments based on sentiment.

Logical blocks:

- **1.1 Input Reception:** Capture new feedback submissions from Typeform.
- **1.2 Sentiment Analysis:** Evaluate the sentiment score of user feedback using Google Cloud Natural Language.
- **1.3 Sentiment-based Routing:** Use conditional logic to branch the workflow based on whether the sentiment is positive or non-positive.
- **1.4 Data Persistence:** Store feedback details and sentiment scores in a Notion database.
- **1.5 Notifications and Task Creation:**
  - Notify a Slack channel for positive feedback.
  - Create a Trello card for follow-up on neutral or negative feedback.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for new form submissions from Typeform, triggering the workflow as soon as a user submits feedback.

- **Nodes Involved:**  
  - Typeform: New Submission

- **Node Details:**  
  - **Type:** Typeform Trigger  
  - **Role:** Initiates workflow on new submission to a specific Typeform form.  
  - **Configuration:**  
    - Uses a webhook linked to a specified Typeform form ID.  
    - Requires Typeform API credentials for authentication.  
  - **Key Expressions:** None (raw submission data used downstream).  
  - **Input:** None (trigger node).  
  - **Output:** JSON object with all submitted fields, notably `"Name"` and `"Any suggestions for us? "`.  
  - **Version:** Compatible with n8n typeVersion 1.  
  - **Potential Failures:**  
    - Webhook misconfiguration or invalid form ID.  
    - Typeform API authentication errors.  
    - Delays if webhook is not triggered properly.  
  - **Sub-workflow:** None.

#### 1.2 Sentiment Analysis

- **Overview:**  
  Extracts the sentiment score of the feedback text to determine if the feedback is positive, negative, or neutral.

- **Nodes Involved:**  
  - Analyze Feedback Sentiment

- **Node Details:**  
  - **Type:** Google Cloud Natural Language  
  - **Role:** Performs sentiment analysis on the feedback text.  
  - **Configuration:**  
    - Analyzes the content field extracted from the Typeform submission’s `"Any suggestions for us? "` field.  
    - Uses OAuth2 credentials for Google Cloud Natural Language API.  
  - **Key Expressions:**  
    - Content: `={{$json["Any suggestions for us? "]}}`  
  - **Input:** Receives submission JSON from Typeform node.  
  - **Output:** JSON with sentiment analysis result, especially `documentSentiment.score` (range roughly -1 to 1).  
  - **Version:** n8n typeVersion 1.  
  - **Potential Failures:**  
    - API authentication or quota limits.  
    - Empty or malformed feedback text causing analysis errors.  
  - **Sub-workflow:** None.

#### 1.3 Sentiment-based Routing

- **Overview:**  
  Determines the workflow path based on whether the sentiment score is positive (greater than 0) or not.

- **Nodes Involved:**  
  - Check Sentiment Score

- **Node Details:**  
  - **Type:** If  
  - **Role:** Conditional branching based on sentiment score.  
  - **Configuration:**  
    - Condition: checks if `documentSentiment.score` from previous node is greater than 0.  
  - **Key Expressions:**  
    - `={{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}} > 0`  
  - **Input:** Sentiment analysis JSON.  
  - **Output:**  
    - True branch: positive sentiment.  
    - False branch: zero or negative sentiment.  
  - **Version:** n8n typeVersion 1.  
  - **Potential Failures:**  
    - Missing or undefined sentiment score.  
    - Expression evaluation errors due to unexpected JSON structure.  
  - **Sub-workflow:** None.

#### 1.4 Data Persistence

- **Overview:**  
  Saves all feedback data including sentiment score to a Notion database for record-keeping and further analysis.

- **Nodes Involved:**  
  - Add Feedback to Notion

- **Node Details:**  
  - **Type:** Notion  
  - **Role:** Creates a new page in a Notion database with feedback details.  
  - **Configuration:**  
    - Target database by ID.  
    - Maps Typeform fields and sentiment score to respective Notion database columns with appropriate types (title, rich_text, number, date).  
    - Adds a static "Source" field with value "Typeform".  
    - Uses current timestamp for "Submitted At".  
    - Requires Notion API credentials.  
  - **Key Expressions:**  
    - Title: `={{$node["Typeform: New Submission"].json["Name"]}}`  
    - Feedback: `={{$node["Typeform: New Submission"].json["Any suggestions for us? "]}}`  
    - Sentiment Score: `={{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}`  
    - Submitted At: `={{$now}}`  
  - **Input:** Data from Typeform and sentiment analysis nodes.  
  - **Output:** Confirmation JSON from Notion API.  
  - **Version:** n8n typeVersion 1.  
  - **Potential Failures:**  
    - Notion API authentication or permission errors.  
    - Incorrect database ID or schema mismatch.  
    - Rate limits or API downtime.  
  - **Sub-workflow:** None.

#### 1.5 Notifications and Task Creation

- **Overview:**  
  Depending on sentiment, either notify a Slack channel or create a Trello card for follow-up.

- **Nodes Involved:**  
  - Notify Slack with Positive Feedback  
  - Create Trello Card for Follow-up

- **Node Details:**  

  **Notify Slack with Positive Feedback**  
  - **Type:** Slack  
  - **Role:** Sends a message to a Slack channel with positive feedback details.  
  - **Configuration:**  
    - Target channel by ID.  
    - Message attachment containing feedback text, submitter name, and sentiment score.  
    - OAuth2 or token-based Slack API credentials required.  
  - **Key Expressions:**  
    - Text: `={{$node["Typeform: New Submission"].json["Any suggestions for us? "]}}`  
    - Title: `={{$node["Typeform: New Submission"].json["Name"]}} | Score: {{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}`  
  - **Input:** From "Add Feedback to Notion" node (true branch).  
  - **Output:** Slack API response JSON.  
  - **Version:** n8n typeVersion 1.  
  - **Potential Failures:**  
    - Slack API authentication errors.  
    - Channel not found or insufficient permissions.  
    - Network or rate limit issues.  

  **Create Trello Card for Follow-up**  
  - **Type:** Trello  
  - **Role:** Creates a Trello card for feedback requiring follow-up (neutral/negative sentiment).  
  - **Configuration:**  
    - Target list by ID.  
    - Card name includes sentiment score.  
    - Description includes score, feedback text, and submitter name.  
    - Requires Trello API credentials.  
  - **Key Expressions:**  
    - Name: `=Score: {{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}`  
    - Description:  
      ```
      Score: {{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}
      Feedback: {{$node["Typeform: New Submission"].json["Any suggestions for us? "]}}
      User: {{$node["Typeform: New Submission"].json["Name"]}}
      ```  
  - **Input:** From "Check Sentiment Score" node (false branch).  
  - **Output:** Confirmation JSON from Trello API.  
  - **Version:** n8n typeVersion 1.  
  - **Potential Failures:**  
    - Trello API authentication or permission issues.  
    - Invalid list ID or card creation errors.  
    - Network problems or rate limits.  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                            | Input Node(s)                  | Output Node(s)                               | Sticky Note                                                                                   |
|-------------------------------|-------------------------|-------------------------------------------|--------------------------------|----------------------------------------------|-----------------------------------------------------------------------------------------------|
| Typeform: New Submission       | Typeform Trigger        | Captures new form submissions from Typeform | None                           | Analyze Feedback Sentiment                    |                                                                                               |
| Analyze Feedback Sentiment     | Google Cloud Natural Language | Performs sentiment analysis on feedback text | Typeform: New Submission       | Check Sentiment Score                         |                                                                                               |
| Check Sentiment Score          | If                      | Routes workflow based on sentiment score   | Analyze Feedback Sentiment      | Add Feedback to Notion (true), Create Trello Card for Follow-up (false) |                                                                                               |
| Add Feedback to Notion         | Notion                  | Stores feedback and sentiment in Notion database | Check Sentiment Score (true)   | Notify Slack with Positive Feedback           |                                                                                               |
| Notify Slack with Positive Feedback | Slack                   | Sends positive feedback notification to Slack | Add Feedback to Notion          | None                                         |                                                                                               |
| Create Trello Card for Follow-up | Trello                  | Creates Trello card for non-positive feedback | Check Sentiment Score (false)  | None                                         |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Typeform Trigger Node**  
   - Node Type: Typeform Trigger  
   - Configure webhook with your Typeform form ID (replace placeholder).  
   - Set Typeform API credentials.  
   - This node triggers the workflow on new submissions.

2. **Add Google Cloud Natural Language Node**  
   - Node Type: Google Cloud Natural Language  
   - Set OAuth2 credentials for Google Cloud.  
   - Configure content to analyze with expression: `={{$json["Any suggestions for us? "]}}` (feedback text).  
   - Connect output of Typeform Trigger node to this node’s input.

3. **Add If Node for Sentiment Score Check**  
   - Node Type: If  
   - Condition: Number operation, check if `={{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}} > 0`.  
   - Connect output of Google Cloud node to this node.

4. **Add Notion Node to Store Feedback**  
   - Node Type: Notion  
   - Set Notion API credentials.  
   - Configure to create a new page in your Notion database (replace database ID placeholder).  
   - Map properties:  
     - Title: `={{$node["Typeform: New Submission"].json["Name"]}}`  
     - Feedback (rich_text): `={{$node["Typeform: New Submission"].json["Any suggestions for us? "]}}`  
     - Sentiment Score (number): `={{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}`  
     - Source (rich_text): "Typeform" (static value)  
     - Submitted At (date): `={{$now}}`  
   - Connect true output of If node to this node.

5. **Add Slack Node for Positive Feedback Notification**  
   - Node Type: Slack  
   - Set Slack API credentials.  
   - Configure to post in your Slack channel (replace channel placeholder).  
   - Message attachments:  
     - Title: `={{$node["Typeform: New Submission"].json["Name"]}} | Score: {{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}`  
     - Text: `={{$node["Typeform: New Submission"].json["Any suggestions for us? "]}}`  
   - Connect output of Notion node to this Slack node.

6. **Add Trello Node for Follow-up Cards**  
   - Node Type: Trello  
   - Set Trello API credentials.  
   - Configure to create a card in your Trello list (replace list ID placeholder).  
   - Card name: `=Score: {{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}`  
   - Description:  
     ```
     Score: {{$node["Analyze Feedback Sentiment"].json["documentSentiment"]["score"]}}
     Feedback: {{$node["Typeform: New Submission"].json["Any suggestions for us? "]}}
     User: {{$node["Typeform: New Submission"].json["Name"]}}
     ```  
   - Connect false output of If node to this Trello node.

7. **Validate all connections:**  
   - Typeform Trigger → Analyze Feedback Sentiment → Check Sentiment Score  
   - Check Sentiment Score (true) → Add Feedback to Notion → Notify Slack with Positive Feedback  
   - Check Sentiment Score (false) → Create Trello Card for Follow-up  

8. **Test the workflow with sample submissions to confirm functionality.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow requires proper credential setup for Typeform, Google Cloud Natural Language, Notion, Slack, and Trello APIs. | Credential configuration is critical for seamless API calls and requires OAuth2 or API keys as per service. |
| Sentiment score from Google Cloud Natural Language typically ranges from -1 (negative) to 1 (positive).     | Useful for interpreting branching logic in the If node.                                                    |
| Make sure the Notion database schema matches the property keys and types configured in the node.           | See Notion API documentation for database property configuration: https://developers.notion.com/reference/database |
| Slack messages use attachments for richer formatting, ensure your Slack app has permissions to post messages. | https://api.slack.com/messaging/composing/layouts                                                            |
| Trello API requires the list ID to add cards; retrieve this from Trello board via API or UI.                | https://developer.atlassian.com/cloud/trello/rest/api-group-cards/#api-cards-post                            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.