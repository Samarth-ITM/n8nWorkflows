Automatic Email Categorization & Labeling in Zoho Mail with GPT-4o-mini

https://n8nworkflows.xyz/workflows/automatic-email-categorization---labeling-in-zoho-mail-with-gpt-4o-mini-8212


# Automatic Email Categorization & Labeling in Zoho Mail with GPT-4o-mini

---

### 1. Workflow Overview

This workflow automates the categorization and labeling of incoming emails in Zoho Mail by leveraging AI text classification with GPT-4o-mini via OpenRouter. Its primary purpose is to streamline email management by automatically detecting the email type (e.g., support, billing, HR) and applying the corresponding Zoho Mail label. This is particularly useful for businesses needing to route customer requests, separate financial communications, or organize recruitment emails efficiently.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Fetch incoming emails from Zoho Mail using IMAP.
- **1.2 Zoho Authentication and Label Fetching:** Obtain OAuth token for Zoho and retrieve available labels.
- **1.3 Label Mapping:** Convert Zoho label display names to their internal IDs for easy referencing.
- **1.4 AI Email Classification:** Use GPT-4o-mini to classify the email content into predefined categories.
- **1.5 Category Routing & Label Assignment:** Map predicted categories to Zoho label IDs, then apply the label to the email.
- **1.6 Merge Results & Finalize:** Aggregate the results to perform the labeling API call.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the workflow on new incoming emails from the Zoho Mail inbox via IMAP protocol.

- **Nodes Involved:**  
  - Incoming Email

- **Node Details:**

  - **Incoming Email**  
    - Type: IMAP Email Read  
    - Role: Listens and fetches new emails arriving in the Zoho Mail "Inbox" mailbox.  
    - Configuration: Uses IMAP credentials for Zoho Mail configured in n8n. Fetches all emails in Inbox without additional filters.  
    - Inputs: Trigger node (start)  
    - Outputs: Passes the email data (headers, body, metadata) downstream.  
    - Edge Cases: Possible IMAP connection issues, invalid credentials, empty inbox.  
    - Version: v2.1  

---

#### 2.2 Zoho Authentication and Label Fetching

- **Overview:**  
  Performs OAuth token refresh to authenticate with Zoho and fetches all available email labels from the account.

- **Nodes Involved:**  
  - Set Account ID  
  - Get Access token  
  - Get labels

- **Node Details:**

  - **Set Account ID**  
    - Type: Set  
    - Role: Statically defines the Zoho Mail Account ID for API calls.  
    - Configuration: Assigns a string variable `accountID` with the user’s Zoho account ID (must be replaced with actual ID).  
    - Inputs: From Incoming Email  
    - Outputs: Passes accountID downstream.  
    - Edge Cases: Missing or incorrect account ID will cause API failures.  
    - Version: v3.4  

  - **Get Access token**  
    - Type: HTTP Request  
    - Role: Retrieves a fresh OAuth access token from Zoho using client credentials and refresh token.  
    - Configuration: POST request to Zoho OAuth token endpoint using query parameters `client_id`, `grant_type=refresh_token`, and `refresh_token`. Uses `httpQueryAuth` credentials type with query parameters for authentication.  
    - Inputs: From Set Account ID  
    - Outputs: Provides access token JSON with `access_token`.  
    - Edge Cases: Invalid client ID, refresh token expiry or revocation, network errors.  
    - Version: v4.2  

  - **Get labels**  
    - Type: HTTP Request  
    - Role: Fetches the list of all labels available in Zoho Mail for the given account.  
    - Configuration: GET request to Zoho Mail API `/api/accounts/{accountID}/labels` with Authorization header `Zoho-oauthtoken {access_token}`. Accepts JSON responses.  
    - Inputs: From Get Access token and Set Account ID (for accountID and token)  
    - Outputs: JSON array of labels with `labelId` and `displayName`.  
    - Edge Cases: Expired access token, malformed account ID, API rate limiting.  
    - Version: v4.2  

---

#### 2.3 Label Mapping

- **Overview:**  
  Converts the array of Zoho labels into a map for quick lookup of label IDs by label display names, enabling dynamic category-to-label resolution.

- **Nodes Involved:**  
  - Label name to ID map

- **Node Details:**

  - **Label name to ID map**  
    - Type: Code (JavaScript)  
    - Role: Parses the label list JSON, builds a case-insensitive map from label display names to their IDs, and optionally resolves a category to labelId.  
    - Configuration:  
      - Reads the label data from either `json.body` or `json` depending on the input shape.  
      - Builds `labelMap` object: keys are lowercase label names, values are label IDs.  
      - If a category is provided in input JSON, outputs `selectedLabelId`.  
    - Inputs: From Get labels  
    - Outputs: JSON with `labelMap` and optional `selectedLabelId`  
    - Edge Cases: Unexpected API response structure, missing label fields, case sensitivity issues.  
    - Version: v2  

---

#### 2.4 AI Email Classification

- **Overview:**  
  Processes the plain text content of the incoming email through an AI text classifier based on GPT-4o-mini to predict the email category.

- **Nodes Involved:**  
  - Text Classifier  
  - OpenRouter Chat Model

- **Node Details:**

  - **OpenRouter Chat Model**  
    - Type: Language Model Chat (OpenRouter)  
    - Role: Provides the GPT-4o-mini model integration as the AI backend for classification.  
    - Configuration: Uses OpenRouter API credentials. Model is set to `openai/gpt-4o-mini`. No extra options set.  
    - Inputs: None direct, connected as `ai_languageModel` input to Text Classifier.  
    - Outputs: Passes AI response to Text Classifier.  
    - Edge Cases: Model API key invalid, API rate limiting, timeout, or network failure.  
    - Version: v1  

  - **Text Classifier**  
    - Type: LangChain Text Classifier  
    - Role: Uses the AI model to classify input text into one of several predefined categories with descriptions.  
    - Configuration:  
      - Input text taken from incoming email's plain text body (`$node["Incoming Email"].json.textPlain`).  
      - Categories defined explicitly with descriptions for support, leads, account management, billing and finance, HR and recruitment, and Other.  
    - Inputs: Receives AI model from OpenRouter Chat Model node.  
    - Outputs: Directs classification result to one of six outputs, each corresponding to a category or "Other".  
    - Edge Cases: Misclassification, empty or malformed email text, AI service errors.  
    - Version: v1.1  

---

#### 2.5 Category Routing & Label Assignment

- **Overview:**  
  Maps the predicted category from the classifier to the corresponding Zoho label ID and prepares the data to apply the label.

- **Nodes Involved:**  
  - Set Support Category  
  - Set Leads Category  
  - Set Account Management Category  
  - Set Billing & Finance Category  
  - Set HR & Recruitment Category  
  - No Operation, do nothing  
  - Merge

- **Node Details:**

  - **Set Support Category**  
    - Type: Set  
    - Role: Assigns the Zoho label ID for "support" category to the field `Category` using the label map.  
    - Configuration: Uses expression `{{ $json.labelMap.support }}` to retrieve label ID.  
    - Inputs: From Text Classifier output for "support" category.  
    - Outputs: To Merge node input 0.  
    - Edge Cases: Missing labelMap or missing "support" label.  
    - Version: v3.4  

  - **Set Leads Category**  
    - Type: Set  
    - Role: Assigns label ID for "leads" category similarly.  
    - Configuration: `{{ $json.labelMap.leads }}`  
    - Inputs: From Text Classifier output for "leads".  
    - Outputs: To Merge node input 1.  
    - Edge Cases: Same as above.  
    - Version: v3.4  

  - **Set Account Management Category**  
    - Type: Set  
    - Role: Assigns label ID for "account management".  
    - Configuration: `{{ $json.labelMap['account management'] }}`  
    - Inputs: From Text Classifier output "account management".  
    - Outputs: To Merge node input 2.  
    - Edge Cases: Same as above.  
    - Version: v3.4  

  - **Set Billing & Finance Category**  
    - Type: Set  
    - Role: Assigns label ID for "billing and finance".  
    - Configuration: `{{ $json.labelMap['billing and finance'] }}`  
    - Inputs: From Text Classifier output "billing and finance".  
    - Outputs: To Merge node input 3.  
    - Edge Cases: Same as above.  
    - Version: v3.4  

  - **Set HR & Recruitment Category**  
    - Type: Set  
    - Role: Assigns label ID for "hr and recruitment".  
    - Configuration: `{{ $json.labelMap['hr and recruitment'] }}`  
    - Inputs: From Text Classifier output "hr and recruitment".  
    - Outputs: To Merge node input 4.  
    - Edge Cases: Same as above.  
    - Version: v3.4  

  - **No Operation, do nothing**  
    - Type: NoOp  
    - Role: Handles the "Other" category where no label change is applied.  
    - Inputs: From Text Classifier output "Other".  
    - Outputs: To Merge node input 5 (not connected further).  
    - Version: v1  

  - **Merge**  
    - Type: Merge  
    - Role: Combines all category branches into one output stream for label application.  
    - Configuration: Accepts 5 inputs from category Set nodes.  
    - Inputs: From all Set Category nodes except No Operation.  
    - Outputs: To "Add Label to the email" node.  
    - Edge Cases: No matching category outputs, multiple inputs arriving simultaneously.  
    - Version: v3.2  

---

#### 2.6 Apply Label to Email

- **Overview:**  
  Applies the determined Zoho label to the original incoming email via Zoho Mail API.

- **Nodes Involved:**  
  - Add Label to the email

- **Node Details:**

  - **Add Label to the email**  
    - Type: HTTP Request  
    - Role: Sends a PUT request to Zoho Mail API to apply the selected label to the message ID of the incoming email.  
    - Configuration:  
      - URL: `https://mail.zoho.com/api/accounts/{accountID}/updatemessage`  
      - Method: PUT  
      - Headers: Accept JSON, Authorization with fresh Zoho OAuth token.  
      - Body (JSON): Contains mode `"applyLabel"`, labelId array with selected category label ID, messageId array with email’s Zoho message ID, and `isFolderSpecific` false.  
      - Uses expressions to fill `accountID`, `labelId` from merged category output, and messageId from incoming email metadata.  
    - Inputs: From Merge node  
    - Outputs: None (terminal node)  
    - Edge Cases: Token expiry, invalid messageId, labelId mismatch, API errors, concurrent updates.  
    - Version: v4.2  

---

### 3. Summary Table

| Node Name                     | Node Type                      | Functional Role                                | Input Node(s)                | Output Node(s)             | Sticky Note                                                                                                                                                    |
|-------------------------------|--------------------------------|------------------------------------------------|------------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Incoming Email                | Email Read IMAP                | Fetch incoming emails from Zoho Mail Inbox     | Trigger (start)              | Set Account ID             | See "Try It Out!" note: Demonstrates AI-based auto-classification and labeling in Zoho Mail. Use cases and setup instructions included.                      |
| Set Account ID               | Set                           | Defines Zoho Mail Account ID                    | Incoming Email               | Get Access token           | Same sticky note applies                                                                                                                                        |
| Get Access token             | HTTP Request                  | Refreshes OAuth access token for Zoho API      | Set Account ID              | Get labels                 | Same sticky note applies                                                                                                                                        |
| Get labels                  | HTTP Request                  | Retrieves Zoho Mail labels for the account     | Get Access token            | Label name to ID map       | Same sticky note applies                                                                                                                                        |
| Label name to ID map         | Code                          | Maps Zoho label display names to label IDs     | Get labels                  | Text Classifier            | Same sticky note applies                                                                                                                                        |
| Text Classifier             | LangChain Text Classifier      | Classifies email text into categories           | Label name to ID map, OpenRouter Chat Model | Multiple Set Category nodes | Same sticky note applies                                                                                                                                        |
| OpenRouter Chat Model        | Language Model Chat            | Provides GPT-4o-mini AI model for classification| None (standalone AI node)   | Text Classifier (ai_languageModel) | Same sticky note applies                                                                                                                                        |
| Set Support Category         | Set                           | Sets label ID for "support" category             | Text Classifier (support output) | Merge                    | Same sticky note applies                                                                                                                                        |
| Set Leads Category           | Set                           | Sets label ID for "leads" category               | Text Classifier (leads output)   | Merge                    | Same sticky note applies                                                                                                                                        |
| Set Account Management Category | Set                        | Sets label ID for "account management" category | Text Classifier (account management output) | Merge                    | Same sticky note applies                                                                                                                                        |
| Set Billing & Finance Category | Set                         | Sets label ID for "billing and finance" category | Text Classifier (billing and finance output) | Merge                    | Same sticky note applies                                                                                                                                        |
| Set HR & Recruitment Category | Set                          | Sets label ID for "hr and recruitment" category  | Text Classifier (hr and recruitment output) | Merge                    | Same sticky note applies                                                                                                                                        |
| No Operation, do nothing      | NoOp                          | Handles "Other" category (no label applied)       | Text Classifier (Other output) | None                     | Same sticky note applies                                                                                                                                        |
| Merge                       | Merge                         | Merges all categorized branches into one output | Set Category nodes           | Add Label to the email     | Same sticky note applies                                                                                                                                        |
| Add Label to the email       | HTTP Request                  | Applies the selected label to the email in Zoho | Merge                       | None                      | Sticky Note2: Output screenshot of labeled emails in Zoho Mail UI. ![](https://ik.imagekit.io/tscnqj8zf/Zoho_email_labelling.png?updatedAt=1756894481455)       |
| Sticky Note                 | Sticky Note                   | Explains workflow purpose, usage, and requirements | None                        | None                      | See sticky note content in section 5                                                                                                                           |
| Sticky Note2                | Sticky Note                   | Visual output example of labeled emails           | None                        | None                      | See sticky note content in section 5                                                                                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Incoming Email" node**  
   - Type: Email Read IMAP  
   - Credentials: Configure Zoho Mail IMAP credentials  
   - Parameters: Mailbox set to "Inbox"  
   - Triggers workflow on new emails  

2. **Create "Set Account ID" node**  
   - Type: Set  
   - Create string variable `accountID` with your Zoho Mail Account ID (replace placeholder)  
   - Connect Incoming Email → Set Account ID  

3. **Create "Get Access token" node**  
   - Type: HTTP Request (POST)  
   - URL: `https://accounts.zoho.com/oauth/v2/token`  
   - Authentication: Use generic HTTP query auth with credentials containing `client_id`, `refresh_token`  
   - Query parameters: `client_id`, `grant_type=refresh_token`, `refresh_token`  
   - Connect Set Account ID → Get Access token  

4. **Create "Get labels" node**  
   - Type: HTTP Request (GET)  
   - URL expression: `https://mail.zoho.com/api/accounts/{{ $json.accountID }}/labels`  
   - Headers: Accept: application/json  
   - Authorization header: `Zoho-oauthtoken {{ $json.access_token }}` (from Get Access token)  
   - Connect Get Access token → Get labels  

5. **Create "Label name to ID map" node**  
   - Type: Code (JavaScript)  
   - Paste the provided JavaScript code to parse labels and build labelMap  
   - Connect Get labels → Label name to ID map  

6. **Create "OpenRouter Chat Model" node**  
   - Type: LangChain LM Chat OpenRouter  
   - Credentials: Configure with OpenRouter API key  
   - Set model to `openai/gpt-4o-mini`  
   - No extra options needed  

7. **Create "Text Classifier" node**  
   - Type: LangChain Text Classifier  
   - Input Text: Expression `{{ $node["Incoming Email"].json.textPlain }}`  
   - Categories: Set categories with names and descriptions exactly as in the workflow (support, leads, account management, billing and finance, hr and recruitment, Other)  
   - Connect Label name to ID map → Text Classifier input  
   - Connect OpenRouter Chat Model → Text Classifier via `ai_languageModel` input  

8. **Create six "Set" nodes for categories**  
   - For each category node (Support, Leads, Account Management, Billing & Finance, HR & Recruitment):  
     - Type: Set  
     - Assign field `Category` with expression `{{ $json.labelMap.<category> }}` where `<category>` matches the lowercase label name in labelMap  
     - Connect corresponding Text Classifier output branch to each Set node  

9. **Create "No Operation, do nothing" node**  
   - Type: NoOp  
   - Connect Text Classifier output "Other" branch to this node  

10. **Create "Merge" node**  
    - Type: Merge  
    - Number of inputs: 5 (for the five Set category nodes)  
    - Connect all Set category nodes outputs to Merge inputs (one per input)  

11. **Create "Add Label to the email" node**  
    - Type: HTTP Request (PUT)  
    - URL expression: `https://mail.zoho.com/api/accounts/{{ $json.accountID }}/updatemessage`  
    - Headers: Accept: application/json, Authorization: `Zoho-oauthtoken {{ $('Get Access token').item.json.access_token }}`  
    - Body (JSON):  
      ```json
      {
        "mode": "applyLabel",
        "labelId": ["{{ $json.Category }}"],
        "messageId": ["{{ $('Incoming Email').item.json.metadata['x-zm-messageid'] }}"],
        "isFolderSpecific": false
      }
      ```  
    - Connect Merge → Add Label to the email  

12. **Activate and test the workflow**  
    - Replace all placeholder credentials and IDs with real values.  
    - Run with sample emails to verify classification and labeling.  
    - Initially, keep "Add Label" node disabled to prevent unintended label application until confident.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                 | Context or Link                                                                                             |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This n8n template demonstrates how to use AI to automatically classify and organize incoming emails in Zoho Mail by applying the correct label (e.g., Support, Billing, HR, etc.). Use cases include routing support requests, separating billing emails, streamlining HR, and reducing inbox clutter. | Sticky Note node at workflow start                                                                         |
| How it works: Trigger on new email → OAuth token refresh → fetch labels → build label map → classify with GPT-4o-mini → route by category → apply Zoho label.                                                                                                                                 | Sticky Note node                                                                                            |
| Requirements: Zoho Mail account with API enabled, OAuth credentials configured in n8n, AI model access via OpenRouter or equivalent.                                                                                                                                                          | Sticky Note node                                                                                            |
| Output screenshot example: Shows labeled emails in Zoho Mail UI with applied categories.                                                                                                                                                                                                      | Sticky Note2 node: ![Example](https://ik.imagekit.io/tscnqj8zf/Zoho_email_labelling.png?updatedAt=1756894481455) |

---

**Disclaimer:** The provided text is based exclusively on an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.