Extract Marketing Testimonials from Feedback with Gemini AI and Google Sheets

https://n8nworkflows.xyz/workflows/extract-marketing-testimonials-from-feedback-with-gemini-ai-and-google-sheets-4378


# Extract Marketing Testimonials from Feedback with Gemini AI and Google Sheets

### 1. Workflow Overview

This workflow is designed to automate the extraction of emotionally engaging testimonial quotes from user feedback collected via Google Sheets form responses. When a new response is added to the sheet, the workflow uses an AI language model (Google Gemini) to extract a concise testimonial quote, writes this quote back into the sheet, and then sends an email notification containing the extracted testimonial.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Detects new rows added to the Google Sheet containing user feedback.
- **1.2 AI Processing:** Uses a language model to extract a short, emotional testimonial from the feedback text.
- **1.3 Data Update:** Writes the extracted testimonial back into the Google Sheet.
- **1.4 Notification:** Sends an email alert with the extracted testimonial.
- **1.5 Workflow Assistance:** Provides contact and tutorial information related to the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new rows added to a specific Google Sheet tab that collects form responses. It triggers the workflow every minute to check for new feedback submissions.

**Nodes Involved:**  
- Google Sheets Trigger

**Node Details:**

- **Google Sheets Trigger**  
  - *Type & Role:* Event trigger node that watches for new rows added to a Google Sheet.  
  - *Configuration:*  
    - Event: `rowAdded`  
    - Polling interval: every minute  
    - Sheet Name: "Form Responses 1" (specific sheet tab ID provided)  
    - Document ID: Google Sheet identified by ID "14nmSXdGURNj3a1uQsxNcspdN5HrjGl8TA8t-hdQUF2s"  
  - *Expressions/Variables:* None (static configuration)  
  - *Input/Output:* No input, outputs new row data as JSON for each added row.  
  - *Version Requirements:* Compatible with n8n Google Sheets Trigger node v1.  
  - *Potential Failures:*  
    - Google API rate limits or authentication failures  
    - Sheet or document ID changes or permissions issues  
  - *Sub-workflow:* None

#### 1.2 AI Processing

**Overview:**  
Processes the received user feedback by extracting a short, emotionally engaging testimonial quote using the Google Gemini AI language model.

**Nodes Involved:**  
- Basic LLM Chain  
- Google Gemini Chat Model

**Node Details:**

- **Google Gemini Chat Model**  
  - *Type & Role:* Language model node that calls Google Gemini 2.0 Flash for chat completions.  
  - *Configuration:*  
    - Model: `models/gemini-2.0-flash`  
    - No custom options set  
  - *Input/Output:* Receives prompts from Basic LLM Chain node; outputs AI-generated text.  
  - *Version Requirements:* Requires n8n nodes-langchain v1 or higher supporting Google Gemini models.  
  - *Potential Failures:*  
    - API authentication or quota issues  
    - Model unavailability or network timeout  

- **Basic LLM Chain**  
  - *Type & Role:* Constructs the prompt and manages the interaction with the AI language model.  
  - *Configuration:*  
    - Prompt Type: `define` (using a fixed template)  
    - Prompt Text:  
      ```
      Extract a short, emotionally engaging testimonial quote from the following user feedback. Ignore neutral or irrelevant text. Only return the quote.
      "{{ $json.Feedback }}"
      
      Feedback: "{{ $json["Feedback"] }}"
      ```  
    - Uses the incoming JSON field `Feedback` from the Google Sheets Trigger node.  
  - *Input/Output:* Receives JSON input from trigger; outputs prompt to Google Gemini Chat Model and receives the AI response.  
  - *Version Requirements:* Requires Langchain nodes v1.5+.  
  - *Potential Failures:*  
    - Expression errors if `Feedback` field is missing or empty  
    - AI response errors or format issues  
  - *Sub-workflow:* None

#### 1.3 Data Update

**Overview:**  
Writes the AI-extracted testimonial quote back into the Google Sheet, appending or updating the row with this new data.

**Nodes Involved:**  
- Google Sheets

**Node Details:**

- **Google Sheets**  
  - *Type & Role:* Writes data back to the Google Sheet, updating the "Testimony" column for the corresponding feedback row.  
  - *Configuration:*  
    - Operation: `appendOrUpdate` (updates existing rows or appends new)  
    - Sheet Name: "Form Responses 1"  
    - Document ID: Same as trigger node  
    - Columns mapped: Timestamp, Name, Email, Feedback, Testimony (the last receives the AI output)  
    - Matching Column: `Testimony` (used for identifying existing rows)  
    - Mapping Mode: Auto-map input data to sheet columns  
  - *Input/Output:* Receives JSON with AI-generated testimonial and original data; outputs result of sheet update.  
  - *Version Requirements:* Compatible with Google Sheets node v4.5+  
  - *Potential Failures:*  
    - Permission issues writing to the sheet  
    - Data mapping mismatches or missing columns  
    - Concurrent update conflicts  

#### 1.4 Notification

**Overview:**  
Sends an email notification containing the extracted testimonial quote to a predefined recipient.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - *Type & Role:* Sends an email via Gmail service.  
  - *Configuration:*  
    - Send To: `nataylamesa@gmail.com`  
    - Subject: `New Testimonial Extracted`  
    - Message Body: Dynamic content using the extracted testimonial: `={{ $json.text }}` (assumed to contain the AI output)  
    - OAuth2 credentials for Gmail must be configured.  
  - *Input/Output:* Input from Google Sheets node output; no output beyond success/failure.  
  - *Version Requirements:* Gmail node v2.1+  
  - *Potential Failures:*  
    - Gmail OAuth token expiration or invalid credentials  
    - Email sending limits or quota exceeded  
    - Missing or invalid email address  

#### 1.5 Workflow Assistance

**Overview:**  
Provides users of the workflow with contact information, tutorial links, and author details for support and further learning.

**Nodes Involved:**  
- Sticky Note - Assistance  
- Sticky Note - Description

**Node Details:**

- **Sticky Note - Assistance**  
  - *Type & Role:* Informative sticky note with contact details and tutorial links.  
  - *Content:*  
    ```
    =======================================
                WORKFLOW ASSISTANCE
    =======================================
    For any questions or support, please contact:
        Yaron@nofluff.online
    
    Explore more tips and tutorials here:
       - YouTube: https://www.youtube.com/@YaronBeen/videos
       - LinkedIn: https://www.linkedin.com/in/yaronbeen/
    =======================================
    
    Author:
    Yaron Been
    ![Yaron Been](https://1.gravatar.com/avatar/a4e4dcaa1f76ff5266bbf80e8df86d22efda890474c68f7796e72fd82e3f2375?size=512&d=initials)
    ```  
  - *Input/Output:* None

- **Sticky Note - Description**  
  - *Type & Role:* Describes the workflow purpose and high-level overview.  
  - *Content:*  
    ```
    Workflow Name: Testimonial Extractor
    
    Description:
    This workflow listens for new rows added to a Google Sheet form response, extracts a short emotional testimonial using a language model, writes it back to the sheet, and sends an email notification with the extracted quote.
    ```  
  - *Input/Output:* None

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                         | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                               |
|-----------------------|--------------------------------|---------------------------------------|----------------------|----------------------|---------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Assistance | n8n-nodes-base.stickyNote      | Provides workflow support info        | None                 | None                 | For any questions or support, please contact: Yaron@nofluff.online. Links: YouTube https://www.youtube.com/@YaronBeen/videos, LinkedIn https://www.linkedin.com/in/yaronbeen/ |
| Sticky Note - Description | n8n-nodes-base.stickyNote      | Describes workflow purpose             | None                 | None                 | Workflow listens for new Google Sheet rows, extracts testimonial via AI, writes back, sends email notification.           |
| Google Sheets Trigger  | n8n-nodes-base.googleSheetsTrigger | Detects new Google Sheets form responses | None                 | Basic LLM Chain       |                                                                                                                           |
| Basic LLM Chain        | @n8n/n8n-nodes-langchain.chainLlm | Builds prompt and manages AI call      | Google Sheets Trigger | Google Sheets         |                                                                                                                           |
| Google Gemini Chat Model | @n8n/n8n-nodes-langchain.lmChatGoogleGemini | AI language model generating testimonial | Basic LLM Chain (ai_languageModel) | Basic LLM Chain      |                                                                                                                           |
| Google Sheets          | n8n-nodes-base.googleSheets     | Writes extracted testimonial back to sheet | Basic LLM Chain       | Gmail                 |                                                                                                                           |
| Gmail                  | n8n-nodes-base.gmail            | Sends email notification with testimonial | Google Sheets         | None                  |                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: `Google Sheets Trigger`  
   - Configure:  
     - Event: `rowAdded`  
     - Poll Interval: every minute  
     - Document ID: Set to your Google Sheet ID containing form responses  
     - Sheet Name: Select the specific sheet tab collecting responses (e.g., "Form Responses 1")  
   - Ensure Google Sheets API credentials are configured and authorized.

2. **Create Basic LLM Chain Node**  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Connect input from Google Sheets Trigger node main output.  
   - Parameters:  
     - Prompt Type: `define`  
     - Text:  
       ```
       Extract a short, emotionally engaging testimonial quote from the following user feedback. Ignore neutral or irrelevant text. Only return the quote.
       "{{ $json.Feedback }}"
       
       Feedback: "{{ $json["Feedback"] }}"
       ```  
   - No credentials needed here, but ensure Langchain nodes are installed and updated.

3. **Create Google Gemini Chat Model Node**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatGoogleGemini`  
   - Model Name: `models/gemini-2.0-flash`  
   - No additional options required.  
   - Connect this node as the `ai_languageModel` input for the Basic LLM Chain node.  
   - Configure Google Gemini API credentials as required.

4. **Create Google Sheets Node**  
   - Type: `Google Sheets`  
   - Connect input from Basic LLM Chain node main output.  
   - Operation: `appendOrUpdate`  
   - Document ID and Sheet Name: Same as Google Sheets Trigger node.  
   - Columns: Include at least `Timestamp`, `Name`, `Email`, `Feedback`, and `Testimony`.  
   - Matching Columns: Use `Testimony` to identify rows for update.  
   - Mapping Mode: `autoMapInputData` to map JSON fields to sheet columns.  
   - Configure Google Sheets credentials with write permissions.

5. **Create Gmail Node**  
   - Type: `Gmail`  
   - Connect input from Google Sheets node main output.  
   - Parameters:  
     - Send To: Enter recipient email, e.g., `nataylamesa@gmail.com`  
     - Subject: `New Testimonial Extracted`  
     - Message: Use expression `={{ $json.text }}` to include the extracted testimonial from AI output.  
   - Configure Gmail OAuth2 credentials with send mail permission.

6. **Optional: Create Sticky Note Nodes**  
   - Add two sticky notes with the provided workflow description and assistance content for user reference.

7. **Verify Credentials and Permissions**  
   - Ensure all API credentials (Google Sheets, Google Gemini, Gmail) are authorized and valid.  
   - Test the workflow by adding a new row to the Google Sheet and monitor execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                            | Context or Link                                                                                         |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| For workflow support and questions, contact Yaron Been at Yaron@nofluff.online                                                                                        | Contact information in Sticky Note - Assistance                                                      |
| Explore additional tips and tutorials by Yaron Been on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/       | Workflow assistance links                                                                             |
| Workflow author: Yaron Been; avatar image linked in Sticky Note - Assistance                                                                                           | Author credit                                                                                        |
| The workflow respects content policies and handles only public, legal data                                                                                             | Disclaimer about data and content compliance                                                          |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.