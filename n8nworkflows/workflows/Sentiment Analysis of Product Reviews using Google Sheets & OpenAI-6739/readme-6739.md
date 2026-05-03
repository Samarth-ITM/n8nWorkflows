Sentiment Analysis of Product Reviews using Google Sheets & OpenAI

https://n8nworkflows.xyz/workflows/sentiment-analysis-of-product-reviews-using-google-sheets---openai-6739


# Sentiment Analysis of Product Reviews using Google Sheets & OpenAI

### 1. Workflow Overview

This workflow automates the sentiment analysis of product reviews stored in a Google Sheet, leveraging OpenAI’s language models via n8n’s Langchain integration. Upon detection of new or updated reviews in the spreadsheet, the workflow triggers and performs sentiment analysis on each review text. The analyzed sentiment category is then updated back into the same Google Sheet alongside the original review. 

The workflow is organized into the following logical blocks:

- **1.1 Input Reception via Google Sheets Trigger:** Detects new or updated rows in the product reviews Google Sheet.
- **1.2 Sentiment Analysis Processing:** Applies OpenAI’s GPT-based model to classify the sentiment of each review.
- **1.3 Output Update to Google Sheets:** Writes the sentiment result back into the original Google Sheet, matched by review text.

Two sticky notes provide descriptive context and purpose for the workflow but have no functional role.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception via Google Sheets Trigger

- **Overview:**  
  This block monitors the specified Google Sheet for new or changed product reviews. It triggers the workflow whenever data changes are detected, polling every minute.

- **Nodes Involved:**  
  - Google Sheets Trigger

- **Node Details:**

  - **Google Sheets Trigger**  
    - **Type:** Trigger node, monitors Google Sheets for changes.  
    - **Configuration:**  
      - Polls every minute to detect changes.  
      - Targets the document ID `1HOq0GIqZ80rMhlDaU9B4d4Vq9BPj1WsTzfnB1kAmY2g`.  
      - Sheet name set to the first sheet (gid=0, named "Sheet1").  
    - **Input/Output:**  
      - No input (trigger node).  
      - Outputs data rows representing individual product reviews.  
    - **Credentials:** Requires OAuth2 credentials for Google Sheets API.  
    - **Edge Cases:**  
      - Polling frequency may lead to delayed detection if edits happen faster than one minute.  
      - Authentication errors if credentials expire or are invalid.  
      - Large sheets or rapid changes might cause rate limiting from Google Sheets API.  
    - **Version specific:** Uses n8n version supporting `googleSheetsTrigger` node (1.x+).  

#### 2.2 Sentiment Analysis Processing

- **Overview:**  
  This block receives the review text from the trigger and performs sentiment classification using OpenAI’s GPT-4o-mini model wrapped by Langchain’s Sentiment Analysis node.

- **Nodes Involved:**  
  - Sentiment Analysis  
  - OpenAI Chat Model

- **Node Details:**

  - **Sentiment Analysis**  
    - **Type:** Langchain Sentiment Analysis node (custom AI processing node).  
    - **Configuration:**  
      - Input text is dynamically set to the review content from the previous node (`{{ $json.Review }}`).  
      - Uses default options for sentiment analysis.  
    - **Input/Output:**  
      - Input: Review text from Google Sheets Trigger.  
      - Output: Sentiment analysis result, including a sentiment category field (`category`).  
    - **Edge Cases:**  
      - Failures if review text is empty or malformed.  
      - Latency or timeouts from OpenAI API calls.  
      - Potential API key/authentication errors with OpenAI.  
    - **Version:** Langchain integration version 1.  

  - **OpenAI Chat Model**  
    - **Type:** Langchain OpenAI chat model node.  
    - **Configuration:**  
      - Uses `gpt-4o-mini` model for processing.  
      - No special prompt or options specified beyond the model.  
    - **Input/Output:**  
      - Input: Receives requests from Sentiment Analysis node (as AI language model backend).  
      - Output: Returns AI-generated sentiment classification data.  
    - **Credentials:** Requires valid OpenAI API key credentials.  
    - **Edge Cases:**  
      - API rate limits or quota exceeded errors.  
      - Model unavailability or version deprecations.  
      - Network connectivity issues.  
    - **Version:** Node version 1.2.  

#### 2.3 Output Update to Google Sheets

- **Overview:**  
  This block updates the Google Sheet row with the analyzed sentiment category, matching on the review text to ensure correct row update.

- **Nodes Involved:**  
  - Updated Sentiment of Product Review (Google Sheets node)

- **Node Details:**

  - **Updated Sentiment of Product Review**  
    - **Type:** Google Sheets node (write/update operation).  
    - **Configuration:**  
      - Operation: Update existing rows.  
      - Matching column: `Review` - the review text is used to find the corresponding row.  
      - Columns updated:  
        - `Review`: The original review text (passed through).  
        - `Sentiment`: The sentiment category returned by the Sentiment Analysis node (`{{ $json.sentimentAnalysis.category }}`).  
      - Targets the same document and sheet as trigger node.  
    - **Input/Output:**  
      - Input: JSON containing review and sentiment category.  
      - Output: Confirmation of the update operation.  
    - **Credentials:** OAuth2 credentials for Google Sheets API.  
    - **Edge Cases:**  
      - Matching may fail if review texts are not unique or contain minor variations, potentially leading to no update or wrong row updated.  
      - Authentication errors or API rate limits.  
      - Data type mismatches if fields are not strings.  
    - **Version:** Google Sheets node version 4.5.  

#### 2.4 Informational Sticky Notes

- **Overview:**  
  Two sticky notes provide human-readable descriptions and workflow title for clarity.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Displays the title: "Sentiment Analysis of Product Review Using OpenAI".  
    - No inputs or outputs.  

  - **Sticky Note1**  
    - Contains the workflow description:  
      > "Whenever a product review is submitted, the workflow is initiated. Each review is then sent for sentiment analysis using OpenAI. Once the sentiment is determined, the result is updated alongside the review."  
    - No inputs or outputs.  

---

### 3. Summary Table

| Node Name                      | Node Type                               | Functional Role                         | Input Node(s)            | Output Node(s)                     | Sticky Note                                                                                         |
|-------------------------------|---------------------------------------|---------------------------------------|-------------------------|----------------------------------|---------------------------------------------------------------------------------------------------|
| Google Sheets Trigger          | Google Sheets Trigger                  | Input reception from Google Sheets    | —                       | Sentiment Analysis                |                                                                                                   |
| Sentiment Analysis             | Langchain Sentiment Analysis           | Perform sentiment classification      | Google Sheets Trigger    | Updated Sentiment of Product Review |                                                                                                   |
| OpenAI Chat Model              | Langchain OpenAI Chat Model            | Backend AI model for sentiment analysis | Sentiment Analysis       | Sentiment Analysis               |                                                                                                   |
| Updated Sentiment of Product Review | Google Sheets (Update operation)    | Update sentiment back to Google Sheet | Sentiment Analysis       | —                                |                                                                                                   |
| Sticky Note                   | Sticky Note                           | Display workflow title                 | —                       | —                                | ## Sentiment Analysis of Product Review Using OpenAI                                              |
| Sticky Note1                  | Sticky Note                           | Display workflow description           | —                       | —                                | ## Description Whenever a product review is submitted, the workflow is initiated. Each review is then sent for sentiment analysis using OpenAI. Once the sentiment is determined, the result is updated alongside the review. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Type: Google Sheets Trigger  
   - Configure:  
     - Set to poll every minute.  
     - Specify Document ID: `1HOq0GIqZ80rMhlDaU9B4d4Vq9BPj1WsTzfnB1kAmY2g`.  
     - Sheet Name: use `Sheet1` (gid=0).  
   - Credentials: Assign Google Sheets OAuth2 credentials with read access.

2. **Add Sentiment Analysis Node:**  
   - Type: Langchain Sentiment Analysis  
   - Parameters:  
     - Input text expression: `={{ $json.Review }}` to dynamically pass review text from trigger.  
     - Use default options.  
   - Connect output of Google Sheets Trigger to input of this node.

3. **Add OpenAI Chat Model Node:**  
   - Type: Langchain OpenAI Chat Model  
   - Parameters:  
     - Model: `gpt-4o-mini`.  
     - No additional options needed.  
   - Credentials: Assign valid OpenAI API Key credentials.  
   - Connect this node as AI model backend for Sentiment Analysis node (usually handled internally in Langchain nodes).

4. **Add Google Sheets Node for Updating Rows:**  
   - Type: Google Sheets  
   - Operation: Update  
   - Parameters:  
     - Document ID: same as trigger node.  
     - Sheet Name: `Sheet1`.  
     - Matching Columns: `Review` to locate the row.  
     - Columns to update:  
       - `Review`: set to `={{ $json.Review }}` (pass-through).  
       - `Sentiment`: set to `={{ $json.sentimentAnalysis.category }}` (from sentiment analysis output).  
   - Credentials: Use Google Sheets OAuth2 credentials with write access.  
   - Connect output of Sentiment Analysis node to input of this update node.

5. **(Optional) Add Sticky Notes:**  
   - Add one sticky note with content:  
     `## Sentiment Analysis of Product Review Using OpenAI`  
   - Add a second sticky note with description:  
     ```
     ## Description

     Whenever a product review is submitted, the workflow is initiated. Each review is then sent for sentiment analysis using OpenAI. Once the sentiment is determined, the result is updated alongside the review.
     ```

6. **Activate the Workflow:**  
   - Ensure all credentials are correctly set and tested.  
   - Activate the workflow to start polling and processing reviews.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                              |
|----------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| This workflow integrates Google Sheets with OpenAI via Langchain nodes, demonstrating AI-powered sentiment analysis at scale. | n8n official documentation for Google Sheets and Langchain nodes: https://docs.n8n.io/    |
| Ensure OpenAI API usage complies with rate limits to avoid workflow failures or delays.            | https://platform.openai.com/docs/guides/rate-limits                                         |
| For better matching accuracy when updating rows, consider adding an explicit unique row ID column. | Best practice to avoid ambiguity in Google Sheets row updates.                              |

---

**Disclaimer:** The content above is derived exclusively from an automated workflow created with n8n, fully compliant with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.