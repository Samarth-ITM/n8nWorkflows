Sentiment Analytics Visualizer

https://n8nworkflows.xyz/workflows/sentiment-analytics-visualizer-5204


# Sentiment Analytics Visualizer

### 1. Workflow Overview

The **Sentiment Analytics Visualizer** workflow automates the process of analyzing customer reviews' sentiment from a Google Sheet, updating the sheet with sentiment results, generating a summary chart, and emailing the results to a designated recipient. It is designed for businesses and analysts who want to monitor customer feedback sentiment efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger and Google Sheets selection to load customer reviews.
- **1.2 Sentiment Analysis Loop:** Processing each review individually through sentiment analysis using Langchain’s sentimentAnalysis node with OpenAI GPT-4o-mini or similar.
- **1.3 Update Data Source:** Writing back the sentiment results into the Google Sheet.
- **1.4 Aggregate Sentiment Data:** Counting the number of reviews per sentiment category.
- **1.5 Visualization Generation:** Creating a chart image summarizing sentiment distribution.
- **1.6 Notification Delivery:** Sending the sentiment summary chart via Gmail to a specified recipient.
- **1.7 Configuration Notes:** Sticky notes guiding setup for Google Sheets, OpenAI, and Gmail credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually and reads review data from a specified Google Sheet.
- **Nodes Involved:**  
  - When clicking 'Test workflow'  
  - Select Google Sheet

- **Node Details:**

  - **When clicking 'Test workflow'**  
    - *Type:* Manual Trigger  
    - *Role:* Starts the workflow on user command.  
    - *Configuration:* Default manual trigger, no parameters.  
    - *Connections:* Outputs to Select Google Sheet node.  
    - *Edge Cases:* None specific; user must trigger manually.

  - **Select Google Sheet**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Fetches review data (title and text) from Google Sheets.  
    - *Configuration:*  
      - Document ID: User’s Google Sheet ID (must be replaced).  
      - Sheet Name: Defaults to first sheet (gid=0).  
    - *Key Expressions:* None dynamic beyond configured documentId and sheetName.  
    - *Connections:* Outputs data to Loop Over Items node.  
    - *Edge Cases:*  
      - Invalid or missing document ID causes read failure.  
      - Empty or malformed sheet may cause no data output.

#### 2.2 Sentiment Analysis Loop

- **Overview:** Processes each review individually to analyze sentiment using Langchain’s sentimentAnalysis node with OpenAI’s GPT model.
- **Nodes Involved:**  
  - Loop Over Items  
  - Sentiment Analysis  
  - OpenAI Chat Model (used as language model backend)

- **Node Details:**

  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Iterates over each review item sequentially.  
    - *Configuration:* Default batch size (1 by default), no reset on completion.  
    - *Connections:* Input from Select Google Sheet; outputs to Sentiment Analysis.  
    - *Edge Cases:* Large batch sizes may cause timeouts; single-item batches safer.

  - **Sentiment Analysis**  
    - *Type:* Langchain Sentiment Analysis  
    - *Role:* Analyzes sentiment of each review.  
    - *Configuration:*  
      - Categories: Positive, Neutral, Negative  
      - System prompt instructs to output JSON formatted sentiment only.  
      - Input text dynamically composed as:  
        ```
        Title: {{ $json['Review title'] }}
        Text: {{ $json['Review text'] }}
        ```  
      - Auto-fixing enabled to improve response accuracy.  
      - Detailed results disabled for simplicity.  
    - *Connections:* Input from Loop Over Items; output to Update Google Sheet.  
    - *Version:* TypeVersion 1.  
    - *Edge Cases:*  
      - API quota limits or invalid credentials cause failure.  
      - Unexpected response formatting may cause JSON parsing errors.

  - **OpenAI Chat Model**  
    - *Type:* Langchain OpenAI Chat Model  
    - *Role:* Provides the GPT-4o-mini model backend for sentiment analysis.  
    - *Configuration:*  
      - Model: gpt-4o-mini (cost-effective)  
    - *Connections:* Invoked internally by Sentiment Analysis node (ai_languageModel connection).  
    - *Edge Cases:*  
      - API key invalid or exhausted credits lead to errors.

#### 2.3 Update Data Source

- **Overview:** Writes the sentiment category results back into the Google Sheet corresponding to each review.
- **Nodes Involved:**  
  - Update Google Sheet

- **Node Details:**

  - **Update Google Sheet**  
    - *Type:* Google Sheets (Update)  
    - *Role:* Updates the ‘Sentiment’ column for each processed review row.  
    - *Configuration:*  
      - Document ID and Sheet Name must match the original sheet.  
      - Matching done by 'row_number' to ensure correct row update.  
      - Columns updated: Sentiment set to `={{ $json.sentimentAnalysis.category }}` — the sentiment category from the analysis.  
      - Cell Format: USER_ENTERED to allow formulas if needed.  
    - *Connections:* Input from Sentiment Analysis. Output loops back to Loop Over Items to continue iteration.  
    - *Edge Cases:*  
      - Incorrect or missing row_number can cause wrong updates.  
      - Sheet permission issues cause update failure.

#### 2.4 Aggregate Sentiment Data

- **Overview:** After all reviews are analyzed and updated, aggregates sentiment counts per category for visualization.
- **Nodes Involved:**  
  - Read Data from Google Sheet  
  - Extract Number of Answers per Sentiment

- **Node Details:**

  - **Read Data from Google Sheet**  
    - *Type:* Google Sheets (Read)  
    - *Role:* Reads the full sheet data again to gather sentiment results.  
    - *Configuration:* Same Document ID and Sheet Name.  
    - *Execute Once:* Enabled to avoid repeated execution in loops.  
    - *Connections:* Outputs data to Extract Number of Answers per Sentiment.  
    - *Edge Cases:* Same as previous Google Sheet read nodes.

  - **Extract Number of Answers per Sentiment**  
    - *Type:* Code (JavaScript)  
    - *Role:* Counts how many reviews fall into Positive, Neutral, and Negative categories.  
    - *Configuration:* Custom JS code iterates all inputs, counts sentiment matches, and outputs labels and values arrays.  
    - *Key Code Snippet:*  
      ```javascript
      let sentimentCount = { Positive: 0, Negative: 0, Neutral: 0 };
      comments.forEach((comment) => {
        const sentiment = comment?.json?.sentimentAnalysis;
        if (sentiment.includes("Positive")) sentimentCount.Positive++;
        else if (sentiment.includes("Negative")) sentimentCount.Negative++;
        else if (sentiment.includes("Neutral")) sentimentCount.Neutral++;
      });
      return {
        labels: ["Positive", "Neutral", "Negative"],
        values: [sentimentCount.Positive, sentimentCount.Neutral, sentimentCount.Negative],
      };
      ```
    - *Connections:* Outputs aggregated data to Generate QuickChart.  
    - *Edge Cases:*  
      - Missing or malformed sentimentAnalysis field causes miscounts.

#### 2.5 Visualization Generation

- **Overview:** Generates a chart image summarizing the sentiment distribution for email attachment.
- **Nodes Involved:**  
  - Generate QuickChart

- **Node Details:**

  - **Generate QuickChart**  
    - *Type:* QuickChart  
    - *Role:* Creates a chart image from input labels and values.  
    - *Configuration:*  
      - Data: Sentiment counts from previous node.  
      - Labels Mode: Array with labels "Positive", "Neutral", "Negative".  
      - Output: Image data (PNG or other format).  
      - Chart and dataset options left default (can be customized).  
    - *Connections:* Outputs chart binary attachment to Gmail node.  
    - *Edge Cases:*  
      - Invalid data arrays cause chart generation errors.

#### 2.6 Notification Delivery

- **Overview:** Sends an email with the sentiment chart attached to a specified recipient.
- **Nodes Involved:**  
  - Send Gmail with Sentiment Chart

- **Node Details:**

  - **Send Gmail with Sentiment Chart**  
    - *Type:* Gmail  
    - *Role:* Sends email with sentiment summary and attached chart.  
    - *Configuration:*  
      - Recipient email address must be replaced with actual target.  
      - OAuth2 credentials configured for Gmail access.  
      - Subject: "Sentiment Analysis Summary"  
      - Message body: Polite notification with chart attachment.  
      - Attachment: Chart image from Generate QuickChart node.  
    - *Connections:* Input from Generate QuickChart.  
    - *Edge Cases:*  
      - Incorrect OAuth2 setup causes send failure.  
      - Invalid recipient address causes bounce.

#### 2.7 Configuration Notes

- **Overview:** Sticky notes provide setup instructions for users.
- **Nodes Involved:**  
  - 📋 Setup Instructions  
  - 🔧 Configure Google Sheets  
  - 🤖 Configure OpenAI  
  - 📧 Configure Gmail

- **Node Details:**

  - **📋 Setup Instructions**  
    - Provides overall setup steps: Google Sheets ID, OpenAI API key, Gmail config, and expected sheet structure.

  - **🔧 Configure Google Sheets**  
    - Details on obtaining Google Sheet ID and required columns.

  - **🤖 Configure OpenAI**  
    - Instructions on adding OpenAI credentials and model choice.

  - **📧 Configure Gmail**  
    - Instructions on setting up Gmail OAuth2 credentials and recipient email.

---

### 3. Summary Table

| Node Name                      | Node Type                              | Functional Role                         | Input Node(s)              | Output Node(s)               | Sticky Note                                                              |
|-------------------------------|--------------------------------------|---------------------------------------|----------------------------|-----------------------------|--------------------------------------------------------------------------|
| When clicking 'Test workflow'  | Manual Trigger                       | Start workflow on demand               |                            | Select Google Sheet          |                                                                          |
| Select Google Sheet            | Google Sheets (Read)                 | Read reviews data from sheet           | When clicking 'Test workflow' | Loop Over Items           | 🔧 CUSTOMIZE THIS NODE: Replace documentId & sheet name with your own   |
| Loop Over Items                | Split In Batches                    | Process each review individually       | Select Google Sheet          | Sentiment Analysis           |                                                                          |
| Sentiment Analysis             | Langchain Sentiment Analysis         | Analyze sentiment of each review       | Loop Over Items              | Update Google Sheet          | 🤖 CUSTOMIZE THIS NODE: Add OpenAI credentials, model set to gpt-4o-mini |
| OpenAI Chat Model              | Langchain OpenAI Chat Model          | GPT model backend for sentiment analysis | Sentiment Analysis (ai_languageModel) | Sentiment Analysis | 🤖 CUSTOMIZE THIS NODE: Add OpenAI credentials, model set to gpt-4o-mini |
| Update Google Sheet            | Google Sheets (Update)               | Write sentiment results back to sheet | Sentiment Analysis           | Loop Over Items              | 🔧 CUSTOMIZE THIS NODE: Replace documentId & sheet name with your own   |
| Read Data from Google Sheet    | Google Sheets (Read)                 | Reload entire sheet for aggregation    | Loop Over Items              | Extract Number of Answers per Sentiment | 🔧 CUSTOMIZE THIS NODE: Replace documentId & sheet name with your own   |
| Extract Number of Answers per Sentiment | Code (JavaScript)             | Count sentiments per category          | Read Data from Google Sheet  | Generate QuickChart          |                                                                          |
| Generate QuickChart            | QuickChart                          | Generate chart image from sentiment counts | Extract Number of Answers per Sentiment | Send Gmail with Sentiment Chart |                                                                          |
| Send Gmail with Sentiment Chart| Gmail                              | Email sentiment chart summary          | Generate QuickChart          |                             | 📧 CUSTOMIZE THIS NODE: Replace recipient email, configure OAuth2       |
| 📋 Setup Instructions          | Sticky Note                        | Explains overall required setup        |                            |                             |                                                                          |
| 🔧 Configure Google Sheets     | Sticky Note                        | Detailed Google Sheets setup guide     |                            |                             |                                                                          |
| 🤖 Configure OpenAI            | Sticky Note                        | Instructions for OpenAI setup          |                            |                             |                                                                          |
| 📧 Configure Gmail             | Sticky Note                        | Gmail OAuth2 and recipient setup       |                            |                             |                                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node**  
   - Name: "When clicking 'Test workflow'"  
   - Purpose: Manual start trigger.

2. **Create Google Sheets (Read) node**  
   - Name: "Select Google Sheet"  
   - Operation: Read  
   - Document ID: Replace with your Google Sheets document ID.  
   - Sheet Name: Use the exact name or gid (e.g., "Sheet1" or "gid=0").  
   - Output: All rows with columns "Review title" and "Review text".  
   - Connect: Output of Manual Trigger → Input of this node.

3. **Create Split In Batches node**  
   - Name: "Loop Over Items"  
   - Default batch size (1) recommended for API calls.  
   - Connect: Output of "Select Google Sheet" → Input of this node.

4. **Create Langchain Sentiment Analysis node**  
   - Name: "Sentiment Analysis"  
   - Configure categories: Positive, Neutral, Negative.  
   - System prompt: "You are highly intelligent and accurate sentiment analyzer. Analyze the sentiment of the provided text. Categorize it into one of the following: {categories}. Use the provided formatting instructions. Only output the JSON."  
   - Input Text:  
     ```
     Title: {{ $json['Review title'] }}
     Text: {{ $json['Review text'] }}
     ```  
   - Enable Auto Fixing; disable detailed results.  
   - Connect: Input from "Loop Over Items".  

5. **Create Langchain OpenAI Chat Model node**  
   - Name: "OpenAI Chat Model"  
   - Model: gpt-4o-mini (or gpt-4o for higher accuracy).  
   - Connect: Link as AI language model backend to "Sentiment Analysis" node.

6. **Create Google Sheets (Update) node**  
   - Name: "Update Google Sheet"  
   - Operation: Update  
   - Document ID & Sheet Name: Same as "Select Google Sheet".  
   - Mapping: Update the "Sentiment" column with `={{ $json.sentimentAnalysis.category }}`.  
   - Matching column: Use "row_number" to link to correct row.  
   - Cell format: USER_ENTERED.  
   - Connect: Input from "Sentiment Analysis" node.  
   - Output: Connect back to "Loop Over Items" to continue looping.

7. **Create Google Sheets (Read) node**  
   - Name: "Read Data from Google Sheet"  
   - Operation: Read  
   - Document ID & Sheet Name: Same as above.  
   - Execute once: Enabled.  
   - Connect: Output of "Loop Over Items" (after completion).

8. **Create Code node (JavaScript)**  
   - Name: "Extract Number of Answers per Sentiment"  
   - Code:  
     ```javascript
     const comments = $input.all();
     let sentimentCount = { Positive: 0, Negative: 0, Neutral: 0 };
     comments.forEach((comment) => {
       const sentiment = comment?.json?.sentimentAnalysis;
       if (sentiment.includes("Positive")) sentimentCount.Positive++;
       else if (sentiment.includes("Negative")) sentimentCount.Negative++;
       else if (sentiment.includes("Neutral")) sentimentCount.Neutral++;
     });
     return {
       labels: ["Positive", "Neutral", "Negative"],
       values: [sentimentCount.Positive, sentimentCount.Neutral, sentimentCount.Negative],
     };
     ```  
   - Connect: Input from "Read Data from Google Sheet".

9. **Create QuickChart node**  
   - Name: "Generate QuickChart"  
   - Data: `={{ $json.values }}`  
   - Labels Array: `={{ $json.labels }}`  
   - Labels Mode: Array  
   - Connect: Input from "Extract Number of Answers per Sentiment".

10. **Create Gmail node**  
    - Name: "Send Gmail with Sentiment Chart"  
    - Authentication: Configure Gmail OAuth2 credentials.  
    - Recipient: Replace with your email address.  
    - Subject: "Sentiment Analysis Summary"  
    - Message:  
      ```
      Dear Team,

      Please find attached the sentiment analysis summary of recent customer reviews.

      Best regards,
      Automated Sentiment Analysis System
      ```  
    - Attachments: Attach binary output from "Generate QuickChart".  
    - Connect: Input from "Generate QuickChart".

11. **Add Sticky Notes** (optional but recommended)  
    - Add notes for setup instructions on Google Sheets ID, OpenAI API key, OAuth2 Gmail credentials, and expected sheet structure.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Setup requires Google Sheets with columns 'Review title' and 'Review text'.                      | Sheet structure requirement                                                                                 |
| Replace all placeholder IDs and email addresses before running.                                  | Important to avoid connection errors                                                                       |
| OpenAI API key must have sufficient credits and permission to use GPT-4o-mini or GPT-4o models.  | OpenAI setup instructions                                                                                   |
| Gmail node must be configured with OAuth2 credentials for sending emails securely.                | Gmail setup instructions                                                                                     |
| The workflow uses Langchain nodes for sentiment analysis leveraging OpenAI models.               | https://n8n.io/integrations/n8n-nodes-langchain                                                             |
| QuickChart node generates charts dynamically based on JSON input data arrays.                    | https://quickchart.io/                                                                                       |

---

**Disclaimer:** The provided content is generated exclusively from an automated n8n workflow export. It fully complies with content policies and contains no illegal or protected data. All used data and API calls must respect respective service terms and user privacy.