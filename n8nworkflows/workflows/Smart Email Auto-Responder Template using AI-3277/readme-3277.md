Smart Email Auto-Responder Template using AI

https://n8nworkflows.xyz/workflows/smart-email-auto-responder-template-using-ai-3277


# Smart Email Auto-Responder Template using AI

### 1. Workflow Overview

This workflow is a **Smart Email Auto-Responder Template using AI** designed to automatically manage incoming emails, classify their content using AI, and send personalized HTML email responses accordingly. It integrates Gmail for email reception and management, LangChain’s Text Classifier for categorizing emails, Google Gemini (PaLM 2) for optional AI enhancement, SMTP for sending customized replies, and Brevo CRM for contact management.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Filtering**: Polls Gmail for new emails hourly, filters out internal or already-responded emails.
- **1.2 AI Classification**: Uses LangChain Text Classifier (optionally enhanced by Google Gemini) to categorize emails into predefined inquiry types.
- **1.3 Response Dispatch**: Sends tailored HTML email replies via SMTP based on the classified category.
- **1.4 Post-Processing & CRM Update**: Marks emails as read, applies Gmail labels, and adds sender contacts to Brevo CRM.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

- **Overview:**  
  This block triggers the workflow on new Gmail messages every hour, filters out internal emails (e.g., from `@syncbricks.com`), and excludes emails that are replies (subject starting with "Re:") to avoid duplicate responses.

- **Nodes Involved:**  
  - Gmail Trigger  
  - Emails from Existing Contracts (If node)  
  - Reply (If node)

- **Node Details:**

  - **Gmail Trigger**  
    - *Type:* Gmail Trigger node  
    - *Role:* Polls Gmail every hour for new emails  
    - *Configuration:* Poll mode set to every hour; no simple mode (fetches full message data)  
    - *Credentials:* Gmail OAuth2  
    - *Input:* None (trigger)  
    - *Output:* New email items with full metadata  
    - *Edge Cases:* OAuth token expiration, Gmail API rate limits, network failures  
    - *Notes:* Filters are empty here; filtering is done downstream.

  - **Emails from Existing Contracts**  
    - *Type:* If node  
    - *Role:* Filters out emails from internal addresses (e.g., `@syncbricks.com`)  
    - *Configuration:* Condition checks if the `from` header contains `@syncbricks.com`  
    - *Input:* Output from Gmail Trigger  
    - *Output:*  
      - True branch: Emails from internal addresses (filtered out, no further processing)  
      - False branch: External emails proceed to classification  
    - *Edge Cases:* Emails with missing or malformed `from` headers may cause condition failure.

  - **Reply**  
    - *Type:* If node  
    - *Role:* Filters out emails that are replies (subject starts with "Re:") to avoid responding again  
    - *Configuration:* Condition checks if subject does NOT start with "Re:"  
    - *Input:* Emails passing internal filter  
    - *Output:*  
      - True branch: New inquiries proceed to classification  
      - False branch: Ignored to prevent duplicate replies  
    - *Edge Cases:* Subject line variations or missing subject may affect filtering accuracy.

---

#### 2.2 AI Classification

- **Overview:**  
  This block classifies the email content into one of three categories (Guest Post, Youtube, Udemy Courses) using LangChain’s Text Classifier. Optionally, Google Gemini Chat Model can enhance classification or provide summarization.

- **Nodes Involved:**  
  - Text Classifier (LangChain)  
  - Google Gemini Chat Model (optional)

- **Node Details:**

  - **Text Classifier**  
    - *Type:* LangChain Text Classifier node  
    - *Role:* Categorizes email content based on subject and body text  
    - *Configuration:*  
      - Input text combines email subject and body text from Gmail Trigger  
      - Categories defined with descriptions:  
        - Guest Post  
        - Youtube  
        - Udemy Courses  
    - *Input:* Email data from Reply node (filtered new emails)  
    - *Output:* Classification result with category label  
    - *Edge Cases:*  
      - Misclassification due to ambiguous content  
      - API call failures or timeouts  
      - Input text missing or malformed  
    - *Version:* Requires LangChain integration support in n8n

  - **Google Gemini Chat Model**  
    - *Type:* LangChain Google Gemini Chat Model node  
    - *Role:* Optional AI model to enhance classification or provide summarization  
    - *Configuration:* Uses model `models/gemini-2.0-flash-exp`  
    - *Credentials:* Google PaLM API  
    - *Input:* Can receive classification input or email content  
    - *Output:* Feeds back into Text Classifier node for improved accuracy  
    - *Edge Cases:* API quota limits, latency, or failures  
    - *Note:* This node is optional and can be disabled to reduce API usage.

---

#### 2.3 Response Dispatch

- **Overview:**  
  Based on the classification, this block sends a customized HTML email reply using SMTP. There are three separate email send nodes, each with a tailored template for the respective inquiry type.

- **Nodes Involved:**  
  - GuestPost Inquiry (Email Send)  
  - Youtube Video Inquiry (Email Send)  
  - Send Email (Udemy Course Inquiry)  

- **Node Details:**

  - **GuestPost Inquiry**  
    - *Type:* Email Send node  
    - *Role:* Sends guest post inquiry response email  
    - *Configuration:*  
      - HTML email template with pricing, submission guidelines, payment info  
      - Subject prefixed with "Re:" and original subject  
      - Recipient set dynamically from email sender name and address  
      - From email set as "Sophia Mitchell <info@syncbricks.com>"  
    - *Credentials:* SMTP account  
    - *Input:* Classification result indicating "Guest Post"  
    - *Output:* Passes to Mark as Read node  
    - *Edge Cases:* SMTP authentication failure, invalid recipient email, template rendering errors

  - **Youtube Video Inquiry**  
    - *Type:* Email Send node  
    - *Role:* Sends YouTube review video inquiry response  
    - *Configuration:*  
      - HTML template with service offerings, pricing, sample video link  
      - Subject prefixed with "Re:" and original subject  
      - Recipient and sender configured similarly to GuestPost Inquiry  
    - *Credentials:* SMTP account  
    - *Input:* Classification result indicating "Youtube"  
    - *Output:* Passes to Mark as Read node  
    - *Edge Cases:* Same as GuestPost Inquiry

  - **Send Email**  
    - *Type:* Email Send node  
    - *Role:* Sends Udemy course inquiry response  
    - *Configuration:*  
      - HTML template listing courses, free resources, and contact info  
      - Subject prefixed with "Re:" and original subject  
      - Recipient and sender configured similarly  
    - *Credentials:* SMTP account  
    - *Input:* Classification result indicating "Udemy Courses"  
    - *Output:* Passes to Mark as Read node  
    - *Edge Cases:* Same as above

---

#### 2.4 Post-Processing & CRM Update

- **Overview:**  
  After sending the response, this block marks the email as read, applies a Gmail label for organization, and adds or updates the sender as a contact in Brevo CRM.

- **Nodes Involved:**  
  - Mark as Read (Gmail node)  
  - Apply Label (Gmail node)  
  - Create Contact in Brevo (SendInBlue node)

- **Node Details:**

  - **Mark as Read**  
    - *Type:* Gmail node  
    - *Role:* Marks the processed email as read to prevent reprocessing  
    - *Configuration:* Uses message ID from Gmail Trigger node  
    - *Credentials:* Gmail OAuth2  
    - *Input:* Output from any of the Email Send nodes  
    - *Output:* Passes to Apply Label and Create Contact nodes  
    - *Edge Cases:* Gmail API failures, invalid message ID

  - **Apply Label**  
    - *Type:* Gmail node  
    - *Role:* Adds a Gmail label (e.g., "Handled by Bot") to the email  
    - *Configuration:* Label ID hardcoded (e.g., `Label_6332648012153150222`)  
    - *Credentials:* Gmail OAuth2  
    - *Input:* Output from Mark as Read node  
    - *Output:* None (end node)  
    - *Edge Cases:* Label ID mismatch, API errors

  - **Create Contact in Brevo**  
    - *Type:* SendInBlue node  
    - *Role:* Upserts the sender’s email into Brevo contact list for CRM purposes  
    - *Configuration:* Uses sender email from Text Classifier node output  
    - *Credentials:* Brevo API key  
    - *Input:* Output from Mark as Read node  
    - *Output:* None (end node)  
    - *Edge Cases:* API key invalid, network errors, duplicate contacts

---

### 3. Summary Table

| Node Name                | Node Type                      | Functional Role                          | Input Node(s)           | Output Node(s)                      | Sticky Note                                                                                         |
|--------------------------|--------------------------------|----------------------------------------|-------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------|
| Gmail Trigger            | Gmail Trigger                  | Poll new emails every hour              | None                    | Emails from Existing Contracts     | ## Get the and Validate  New Email                                                                |
| Emails from Existing Contracts | If                           | Filter internal emails                   | Gmail Trigger           | Reply (False branch)               |                                                                                                   |
| Reply                    | If                            | Filter out reply emails (subject starts with "Re:") | Emails from Existing Contracts | Text Classifier                   |                                                                                                   |
| Text Classifier          | LangChain Text Classifier      | Classify email content into categories  | Reply                    | GuestPost Inquiry, Youtube Video Inquiry, Send Email | ## Classify the Email                                                                             |
| Google Gemini Chat Model | LangChain Google Gemini Chat   | Optional AI enhancement for classification | (Optional input)         | Text Classifier                   |                                                                                                   |
| GuestPost Inquiry        | Email Send                    | Send guest post inquiry response        | Text Classifier          | Mark as Read                     | ## Email Templates for Services                                                                   |
| Youtube Video Inquiry    | Email Send                    | Send YouTube video inquiry response     | Text Classifier          | Mark as Read                     | ## Email Templates for Services                                                                   |
| Send Email               | Email Send                    | Send Udemy course inquiry response      | Text Classifier          | Mark as Read                     | ## Email Templates for Services                                                                   |
| Mark as Read             | Gmail                         | Mark email as read                       | GuestPost Inquiry, Youtube Video Inquiry, Send Email | Apply Label, Create Contact in Brevo       | ## mark as read, apply label and add to contact                                                  |
| Apply Label              | Gmail                         | Apply Gmail label to processed email    | Mark as Read             | None                            | ## mark as read, apply label and add to contact                                                  |
| Create Contact in Brevo  | SendInBlue                    | Add/update sender contact in Brevo CRM  | Mark as Read             | None                            | ## mark as read, apply label and add to contact                                                  |
| Sticky Note11            | Sticky Note                   | Attribution and credits                  | None                    | None                            | ## Developed by Amjid Ali ... [Includes links to book, courses, and contact info]                 |
| Sticky Note              | Sticky Note                   | Label for Gmail Trigger block            | None                    | None                            | ## Get the and Validate  New Email                                                                |
| Sticky Note1             | Sticky Note                   | Label for Classification block           | None                    | None                            | ## Classify the Email                                                                             |
| Sticky Note2             | Sticky Note                   | Label for Email Templates block           | None                    | None                            | ## Email Templates for Services                                                                   |
| Sticky Note3             | Sticky Note                   | Label for Post-Processing block           | None                    | None                            | ## mark as read, apply label and add to contact                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set to poll every hour (pollTimes: everyHour)  
   - Use OAuth2 credentials for Gmail  
   - No filters applied here; full message data fetched

2. **Add If Node: Emails from Existing Contracts**  
   - Type: If  
   - Condition: Check if `headers.from` contains `@syncbricks.com`  
   - True branch: Stop workflow (no output)  
   - False branch: Continue to next node

3. **Add If Node: Reply**  
   - Type: If  
   - Condition: Subject does NOT start with "Re:"  
   - True branch: Continue to classification  
   - False branch: Stop workflow

4. **Add LangChain Text Classifier Node**  
   - Input text: Combine email subject and body text from Gmail Trigger  
   - Define categories:  
     - Guest Post (description: guest post inquiries)  
     - Youtube (description: YouTube review inquiries)  
     - Udemy Courses (description: course inquiries)  
   - Connect output of Reply node (True branch) to this node  
   - Requires LangChain credentials/configuration

5. **(Optional) Add Google Gemini Chat Model Node**  
   - Model: `models/gemini-2.0-flash-exp`  
   - Credentials: Google PaLM API  
   - Connect output to Text Classifier node’s AI language model input  
   - This node can be disabled if not needed

6. **Add Email Send Nodes for Each Category**  
   - Create three Email Send nodes with SMTP credentials:  
     - GuestPost Inquiry: Use provided HTML template with guest post details  
     - Youtube Video Inquiry: Use provided HTML template with YouTube review details  
     - Send Email: Use provided HTML template for Udemy course inquiries  
   - Set subject to "Re: {{ original subject }}" dynamically  
   - Set recipient to sender’s name and email from incoming email  
   - From email: "Sophia Mitchell <info@syncbricks.com>"  
   - Connect Text Classifier outputs to respective Email Send nodes based on category

7. **Add Gmail Mark as Read Node**  
   - Type: Gmail  
   - Operation: markAsRead  
   - Use message ID from Gmail Trigger node  
   - Connect output of all Email Send nodes to this node

8. **Add Gmail Apply Label Node**  
   - Type: Gmail  
   - Operation: addLabels  
   - Label ID: Use your Gmail label ID (e.g., "Label_6332648012153150222")  
   - Use message ID from Gmail Trigger node  
   - Connect output of Mark as Read node to this node

9. **Add Brevo Create/Update Contact Node**  
   - Type: SendInBlue (Brevo)  
   - Operation: upsert contact  
   - Email: Use sender’s email from Text Classifier node output  
   - Credentials: Brevo API key  
   - Connect output of Mark as Read node to this node

10. **Add Sticky Notes for Documentation**  
    - Add notes to label each logical block for clarity (optional but recommended)

11. **Test Workflow**  
    - Send test emails matching each category  
    - Verify classification, email responses, Gmail label application, and Brevo contact creation

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Developed by Amjid Ali. Please credit original author when sharing this workflow.                                           | Email: amjid@amjidali.com, LinkedIn: https://linkedin.com/in/amjidali, Website: https://syncbricks.com |
| Buy N8N Mastery Book: https://www.amazon.com/dp/B0F23GYCFW                                                                  | Book link                                                                                          |
| Full courses on ERPNext and AI Automation: http://lms.syncbricks.com                                                        | Online courses platform                                                                            |
| YouTube Channel with tutorials and reviews: https://youtube.com/@syncbricks                                                 | Video tutorials and workflow examples                                                             |
| Step-by-step tutorial video for this workflow: https://youtu.be/OAu6tOwubgE                                                  | Video walkthrough                                                                                  |
| AI Automation Mastery Udemy course: https://www.udemy.com/course/mastering-n8n-ai-agents-api-automation-webhooks-no-code/?referralCode=0309FD70BE2D72630C09 | Udemy course                                                                                       |

---

This structured documentation provides a comprehensive understanding of the workflow’s architecture, node configurations, and operational logic, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.