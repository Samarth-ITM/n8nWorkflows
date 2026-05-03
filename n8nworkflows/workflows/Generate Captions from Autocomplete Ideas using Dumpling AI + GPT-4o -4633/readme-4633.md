Generate Captions from Autocomplete Ideas using Dumpling AI + GPT-4o 

https://n8nworkflows.xyz/workflows/generate-captions-from-autocomplete-ideas-using-dumpling-ai---gpt-4o--4633


# Generate Captions from Autocomplete Ideas using Dumpling AI + GPT-4o 

### 1. Workflow Overview

This workflow automates the generation of engaging social media captions from trending autocomplete keyword suggestions. It is designed to run daily at noon, fetching seed keywords from a Google Sheet, retrieving autocomplete suggestions via Dumpling AI, and then generating concise captions using OpenAI’s GPT-4o model. The captions alongside their corresponding keywords are saved back into a Google Sheet for content repurposing or marketing use.

**Logical Blocks:**

- **1.1 Scheduled Trigger:** Initiates the workflow daily at a fixed time.
- **1.2 Input Reception:** Reads base search keywords from a Google Sheet.
- **1.3 Autocomplete Fetching:** Calls Dumpling AI API to retrieve autocomplete suggestions for each keyword.
- **1.4 Suggestions Processing:** Converts API suggestions into an iterable array.
- **1.5 Caption Generation:** Loops through each suggestion and uses GPT-4o to generate a short, engaging caption.
- **1.6 Output Storage:** Appends the keyword-caption pairs into a designated Google Sheet.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Scheduled Trigger

**Overview:**  
This block triggers the entire workflow automatically once every day at 12 PM, ensuring timely and regular execution without manual intervention.

**Nodes Involved:**  
- Run Every Day at 12 PM

**Node Details:**

- **Run Every Day at 12 PM**  
  - Type: Schedule Trigger  
  - Configuration: Set to trigger daily at 12:00 PM (hour 12).  
  - Input: None (trigger node)  
  - Output: Passes trigger event to downstream nodes.  
  - Edge Cases: Workflow will not run outside this time; if the server or n8n instance is down at trigger time, execution may be missed or delayed.

---

#### 2.2 Input Reception

**Overview:**  
Fetches a list of seed search keywords from a specified Google Sheet tab. These keywords act as the starting point for fetching autocomplete suggestions.

**Nodes Involved:**  
- Get Search Keywords from Google Sheet

**Node Details:**

- **Get Search Keywords from Google Sheet**  
  - Type: Google Sheets  
  - Configuration:  
    - Document ID and Sheet Name point to a Google Sheet containing search keywords (Sheet2).  
    - Reads all rows from the sheet (no filters applied).  
  - Input: Trigger from scheduled trigger node.  
  - Output: JSON items each containing a `Keywords` field.  
  - Credentials: Google Sheets OAuth2 API credentials configured.  
  - Edge Cases:  
    - Invalid or revoked OAuth2 credentials may cause authentication errors.  
    - Empty or missing sheet/tab will yield no keywords and halt downstream processing.

---

#### 2.3 Autocomplete Fetching

**Overview:**  
For each keyword, calls Dumpling AI’s autocomplete endpoint to get trending search autocomplete suggestions. This provides contextually relevant phrases to generate captions from.

**Nodes Involved:**  
- Fetch Autocomplete Suggestions (Dumpling AI)

**Node Details:**

- **Fetch Autocomplete Suggestions (Dumpling AI)**  
  - Type: HTTP Request  
  - Configuration:  
    - Method: POST  
    - URL: `https://app.dumplingai.com/api/v1/get-autocomplete`  
    - Body: JSON containing:  
      - `query`: keyword from Google Sheet (`{{$json.Keywords}}`)  
      - `country`: "US"  
      - `language`: "en"  
    - Authentication: HTTP Header Authentication with API key credential (Dumpling AI-n8n)  
  - Input: Keywords from Google Sheets node.  
  - Output: JSON response containing an array of `suggestions`.  
  - Edge Cases:  
    - API key invalidation or quota exceeded causing 401/429 errors.  
    - Network or timeout issues.  
    - Unexpected response format breaking downstream parsing.

---

#### 2.4 Suggestions Processing

**Overview:**  
Transforms the raw suggestions array from Dumpling AI response into a format suitable for iteration in subsequent nodes.

**Nodes Involved:**  
- Format Suggestions into Array  
- Loop Through Each Autocomplete Suggestion

**Node Details:**

- **Format Suggestions into Array**  
  - Type: Set  
  - Configuration: Assigns a new field `suggestions` equal to the `suggestions` array extracted from the HTTP response JSON.  
  - Input: HTTP response JSON from Dumpling AI node.  
  - Output: JSON item with `suggestions` array field.  
  - Edge Cases: If `suggestions` field is missing or empty, the array will be empty and loop will not execute.

- **Loop Through Each Autocomplete Suggestion**  
  - Type: SplitOut  
  - Configuration: Splits the `suggestions` array into individual JSON items, each containing one suggestion string.  
  - Input: Output from the Set node with `suggestions` array.  
  - Output: Stream of individual suggestion items to downstream node.  
  - Edge Cases: Empty input array results in no iterations.

---

#### 2.5 Caption Generation

**Overview:**  
For each individual autocomplete suggestion, this block calls the OpenAI GPT-4o model via n8n’s LangChain node to generate a short, human-like social media caption optimized for engagement.

**Nodes Involved:**  
- Generate Caption from Suggestion (GPT-4o)

**Node Details:**

- **Generate Caption from Suggestion (GPT-4o)**  
  - Type: LangChain OpenAI node  
  - Configuration:  
    - Model: chatgpt-4o-latest  
    - Message System Prompt:  
      - Defines role as a creative social media content writer.  
      - Instructions for tone (friendly, inspiring), length (<280 chars), engagement hooks, non-verbatim repetition.  
      - Output limited to caption text only.  
    - Message User Prompt: Uses the current suggestion string `"{{ $json.value }}"` as the input search phrase.  
  - Input: Each autocomplete suggestion from the looping node.  
  - Output: GPT-4o generated caption text in `message.content`.  
  - Credentials: OpenAI API key configured.  
  - Edge Cases:  
    - API rate limits or invalid key cause errors.  
    - Model downtime or timeouts.  
    - Unexpected output format or empty response.

---

#### 2.6 Output Storage

**Overview:**  
Appends the generated caption paired with its corresponding autocomplete keyword into a target Google Sheet for content management or later use.

**Nodes Involved:**  
- Save Keyword & Generated Caption to Google Sheet

**Node Details:**

- **Save Keyword & Generated Caption to Google Sheet**  
  - Type: Google Sheets  
  - Configuration:  
    - Operation: Append new rows.  
    - Sheet Name: Points to a "content" sheet tab.  
    - Columns:  
      - Keyword: from current suggestion (`$('Loop Through Each Autocomplete Suggestion').item.json.value`)  
      - Caption: from GPT-4o response (`$json.message.content`)  
    - No matching or update, pure append mode.  
  - Input: Captions generated node.  
  - Credentials: Google Sheets OAuth2 API credentials reused.  
  - Edge Cases:  
    - Authentication failure.  
    - Sheet permission or quota issues.  
    - Data type mismatches or invalid characters.

---

### 3. Summary Table

| Node Name                               | Node Type                     | Functional Role                        | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                                                   |
|----------------------------------------|-------------------------------|-------------------------------------|---------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Run Every Day at 12 PM                  | Schedule Trigger              | Initiates workflow daily at noon    | None                            | Get Search Keywords from Google Sheet |                                                                                                                               |
| Get Search Keywords from Google Sheet   | Google Sheets                | Reads seed keywords from sheet       | Run Every Day at 12 PM           | Fetch Autocomplete Suggestions (Dumpling AI) |                                                                                                                               |
| Fetch Autocomplete Suggestions (Dumpling AI) | HTTP Request                 | Fetches autocomplete suggestions    | Get Search Keywords from Google Sheet | Format Suggestions into Array        |                                                                                                                               |
| Format Suggestions into Array            | Set                          | Extracts suggestions array           | Fetch Autocomplete Suggestions  | Loop Through Each Autocomplete Suggestion |                                                                                                                               |
| Loop Through Each Autocomplete Suggestion | SplitOut                    | Splits suggestions for iteration    | Format Suggestions into Array    | Generate Caption from Suggestion (GPT-4o) |                                                                                                                               |
| Generate Caption from Suggestion (GPT-4o) | LangChain OpenAI             | Generates social media caption       | Loop Through Each Autocomplete Suggestion | Save Keyword & Generated Caption to Google Sheet |                                                                                                                               |
| Save Keyword & Generated Caption to Google Sheet | Google Sheets                | Saves keyword-caption pair           | Generate Caption from Suggestion | None                              |                                                                                                                               |
| Sticky Note                             | Sticky Note                   | Workflow purpose and summary note   | None                            | None                              | ### ✨ Auto-Generate Social Captions from Trending Google Autocomplete\n\nThis workflow uses a search term from Google Sheets and fetches trending autocomplete suggestions using Dumpling AI’s `/get-autocomplete` endpoint. Each suggestion is looped and processed by GPT-4o to generate a short social media caption. The keyword and caption pair is saved to another sheet for content reuse. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node: "Run Every Day at 12 PM"**  
   - Node Type: Schedule Trigger  
   - Set trigger time to daily at 12:00 PM (hour = 12).  
   - No credentials needed.

2. **Add Google Sheets Node: "Get Search Keywords from Google Sheet"**  
   - Node Type: Google Sheets  
   - Set Operation to “Read Rows” (default) on the sheet named "Sheet2".  
   - Configure Document ID to your Google Sheet containing keywords.  
   - Authenticate with Google Sheets OAuth2 credentials.

3. **Connect "Run Every Day at 12 PM" → "Get Search Keywords from Google Sheet"**

4. **Add HTTP Request Node: "Fetch Autocomplete Suggestions (Dumpling AI)"**  
   - Method: POST  
   - URL: `https://app.dumplingai.com/api/v1/get-autocomplete`  
   - Body Content-Type: JSON  
   - Body:  
     ```json
     {
       "query": "{{ $json.Keywords }}",
       "country": "US",
       "language": "en"
     }
     ```  
   - Authentication: HTTP Header Authentication with Dumpling AI API key credential.  
   - Connect from "Get Search Keywords from Google Sheet".

5. **Add Set Node: "Format Suggestions into Array"**  
   - Add field `suggestions` of type Array set to `={{ $json.suggestions }}`.  
   - Input from "Fetch Autocomplete Suggestions (Dumpling AI)".

6. **Add SplitOut Node: "Loop Through Each Autocomplete Suggestion"**  
   - Field to split out: `suggestions`.  
   - Connect from "Format Suggestions into Array".

7. **Add LangChain OpenAI Node: "Generate Caption from Suggestion (GPT-4o)"**  
   - Model: chatgpt-4o-latest  
   - Messages:  
     - System Role:  
       ```
       You are a creative content writer for social media. Write a short, engaging, and relevant caption based on the search phrase below. The tone should be friendly and inspiring, and the caption should be optimized to grab attention on platforms like Instagram, LinkedIn, or Twitter.

       Make sure the caption:
       - Feels human and conversational
       - Includes a subtle hook or curiosity element
       - Stays under 280 characters
       - Does not repeat the phrase verbatim

       Respond with just the caption, nothing else.
       ```  
     - User Role: `Search phrase: "{{ $json.value }}"`  
   - Credentials: OpenAI API key.  
   - Connect from "Loop Through Each Autocomplete Suggestion".

8. **Add Google Sheets Node: "Save Keyword & Generated Caption to Google Sheet"**  
   - Operation: Append  
   - Sheet Name: "content" (or your chosen sheet for captions)  
   - Columns mapping:  
     - Keyword: `={{ $('Loop Through Each Autocomplete Suggestion').item.json.value }}`  
     - Caption: `={{ $json.message.content }}`  
   - Authenticate with Google Sheets OAuth2 credentials.  
   - Connect from "Generate Caption from Suggestion (GPT-4o)".

9. **Validate all credentials and test the workflow by manual trigger once before enabling the schedule.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| This workflow leverages Dumpling AI’s `/get-autocomplete` API endpoint to obtain trending search terms, which helps generate relevant, timely captions for social media content.                                                  | Dumpling AI API documentation or website (https://app.dumplingai.com)                                         |
| The GPT-4o model is accessed via n8n’s LangChain OpenAI node, allowing advanced prompt engineering and multi-role message setup for creative caption generation.                                                                | OpenAI GPT-4o API and LangChain integration in n8n documentation.                                              |
| Google Sheets OAuth2 credentials must have appropriate permissions to read and append data in the designated sheets.                                                                                                            | Google Cloud Console for OAuth2 setup and permission scopes.                                                   |
| Design considerations include limiting captions to under 280 characters to suit platforms like Twitter and ensuring captions do not repeat the keyword verbatim to improve engagement.                                          | Best practices for social media content writing.                                                               |
| Workflow is scheduled to run once daily at noon; consider adjusting schedule or adding error handling for production use (e.g., retries, alerts on failure).                                                                     | n8n workflow scheduling and error workflow features.                                                           |

---

**Disclaimer:** The content provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.