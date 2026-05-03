Qualify TikTok Influencers from Username with Dumpling AI + GPT-4

https://n8nworkflows.xyz/workflows/qualify-tiktok-influencers-from-username-with-dumpling-ai---gpt-4-9126


# Qualify TikTok Influencers from Username with Dumpling AI + GPT-4

### 1. Workflow Overview

This workflow automates the qualification of TikTok influencers based on their username input. It is designed for marketing or influencer outreach teams who want to quickly assess if a TikTok user meets specific popularity criteria. The workflow includes two main logical blocks:

- **1.1 Data Retrieval and AI Qualification:**  
  Collect TikTok profile data using Dumpling AI’s API, then process and evaluate that data against predefined influencer criteria using GPT-4.

- **1.2 Data Storage and Update in Google Sheets:**  
  Check if the TikTok user already exists in a Google Sheet by User ID and either update the existing record or append a new one with the latest evaluation results.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval and AI Qualification

**Overview:**  
This block receives a TikTok username via a form, fetches profile data from Dumpling AI, and then uses GPT-4 to determine if the user qualifies as an influencer based on video count, follower count, and heart count.

**Nodes Involved:**  
- Trigger on TikTok Username Form  
- Get TikTok Profile (Dumpling AI)  
- Evaluate Profile Qualification (GPT-4)  
- Sticky Note (explains this block)

**Node Details:**

- **Trigger on TikTok Username Form**  
  - Type: Form Trigger  
  - Role: Entry point; listens for TikTok username submissions via a form  
  - Configuration: Single form field labeled "Tik Tok Username"  
  - Input/Output: Outputs form data containing the username  
  - Failure Modes: Form not submitted, malformed input, webhook errors  
  - Version: 2.3

- **Get TikTok Profile (Dumpling AI)**  
  - Type: HTTP Request  
  - Role: Calls Dumpling AI API to retrieve TikTok user profile data based on username  
  - Configuration:  
    - POST request to `https://app.dumplingai.com/api/v1/get-tiktok-profile`  
    - JSON body includes "handle" set dynamically from the submitted TikTok username  
    - Uses HTTP Header Authentication with stored Dumpling AI credentials  
  - Key Expression: `={{ $json['Tik Tok Username'] }}`  
  - Input: Form submission data  
  - Output: JSON with TikTok user profile including stats like videoCount, followerCount, heartCount  
  - Failure Modes: Network errors, invalid API key, user not found, malformed responses  
  - Version: 4.2

- **Evaluate Profile Qualification (GPT-4)**  
  - Type: OpenAI (LangChain)  
  - Role: Uses GPT-4 to analyze profile stats and decide qualification status  
  - Configuration:  
    - Model: GPT-4.1  
    - System prompt instructs GPT-4 to check three numeric conditions (≥40 videos, ≥100,000 followers, ≥300,000 hearts)  
    - Input variables: video_count, follower_count, heart_count from Dumpling AI response  
    - Output: Exactly "Qualified for influencer outreach" or "Not qualified"  
  - Key Expression: Template injecting stats from previous node’s JSON  
  - Input: TikTok profile data JSON  
  - Output: Qualification message in JSON (`message.content`)  
  - Failure Modes: API key/auth issues, rate limits, malformed input, unexpected GPT output format  
  - Version: 1.8

- **Sticky Note (Block Explanation)**  
  - Content summarizes the data retrieval and AI evaluation steps for clarity.

---

#### 2.2 Data Storage and Update in Google Sheets

**Overview:**  
This block checks if the TikTok user already exists in a Google Sheet (using User ID as key). Depending on existence, it updates the user’s data or appends a new row, including the qualification result.

**Nodes Involved:**  
- Check if User Already Exists (Google Sheets)  
- Route Based on Existence (Switch)  
- Add New TikTok User to Sheet (Google Sheets)  
- Update Existing TikTok User in Sheet (Google Sheets)  
- Sticky Note (explains this block)

**Node Details:**

- **Check if User Already Exists**  
  - Type: Google Sheets (Read with Filter)  
  - Role: Filters the sheet rows by User ID to determine if the user data is already stored  
  - Configuration:  
    - Document and sheet specified (Google Sheets OAuth2 credential required)  
    - Filter: Matches `User ID` column exactly with `user.id` from Dumpling AI response  
  - Input: Profile data JSON from GPT-4 evaluation node  
  - Output: Rows matching the User ID (empty if not found)  
  - Failure Modes: Credential/authentication errors, sheet not found, quota limits  
  - Version: 4.7

- **Route Based on Existence**  
  - Type: Switch  
  - Role: Branches workflow based on whether the user exists in the sheet  
  - Configuration:  
    - If no matching rows (length 0): route to "Does not exist" (add new user)  
    - If one or more rows (length >= 1): route to "Exists" (update user)  
  - Input: Result of Google Sheets filter  
  - Output: Two branches for add or update  
  - Failure Modes: Expression errors if input is malformed

- **Add New TikTok User to Sheet**  
  - Type: Google Sheets (Append)  
  - Role: Appends a new row with the TikTok user data and qualification result  
  - Configuration:  
    - Maps columns: User ID, Qualified?, Heart Count, Video Count, Tik Tok user, Follower Count, Following Count  
    - Sheet and document IDs configured to target the correct Google Sheet and tab  
  - Input: Data from previous nodes (profile and qualification)  
  - Output: Confirmation of row addition  
  - Failure Modes: Sheet permissions, quota, malformed data, API limits  
  - Version: 4.7

- **Update Existing TikTok User in Sheet**  
  - Type: Google Sheets (AppendOrUpdate)  
  - Role: Updates the existing row matching User ID with latest data  
  - Configuration:  
    - Similar column mapping as Add New node  
    - Matching column set as User ID  
    - Sheet and document IDs same as above  
  - Input: Data from previous nodes  
  - Output: Confirmation of row update  
  - Failure Modes: Sheet permissions issues, concurrency conflicts, malformed data  
  - Version: 4.7

- **Sticky Note (Block Explanation)**  
  - Content explains the logic and requirements of Google Sheets integration, including required columns.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                                  | Input Node(s)                          | Output Node(s)                                   | Sticky Note                                                                                  |
|-------------------------------|----------------------------------|-------------------------------------------------|--------------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------|
| Trigger on TikTok Username Form| Form Trigger                     | Entry point; captures TikTok username via form | -                                    | Get TikTok Profile (Dumpling AI)                 | 🧠 Get TikTok Profile & Evaluate with AI (covers nodes 1-3)                                 |
| Get TikTok Profile (Dumpling AI)| HTTP Request                   | Fetch TikTok profile data from Dumpling AI API | Trigger on TikTok Username Form       | Evaluate Profile Qualification (GPT-4)           | 🧠 Get TikTok Profile & Evaluate with AI                                                   |
| Evaluate Profile Qualification (GPT-4)| OpenAI (LangChain)         | Evaluate if user meets influencer criteria      | Get TikTok Profile (Dumpling AI)      | Check if User Already Exists                       | 🧠 Get TikTok Profile & Evaluate with AI                                                   |
| Check if User Already Exists   | Google Sheets                   | Check if user already exists in Google Sheet    | Evaluate Profile Qualification (GPT-4)| Route Based on Existence                          | 📄 Check Google Sheet & Save Result (covers nodes 4-7)                                    |
| Route Based on Existence       | Switch                         | Branch workflow based on user existence          | Check if User Already Exists           | Add New TikTok User to Sheet, Update Existing TikTok User in Sheet | 📄 Check Google Sheet & Save Result                                                        |
| Add New TikTok User to Sheet   | Google Sheets                   | Append new user data row                          | Route Based on Existence (Does not exist)| -                                               | 📄 Check Google Sheet & Save Result                                                        |
| Update Existing TikTok User in Sheet| Google Sheets               | Update existing user data row                     | Route Based on Existence (Exists)      | -                                               | 📄 Check Google Sheet & Save Result                                                        |
| Sticky Note                   | Sticky Note                    | Block explanation for data retrieval and AI     | -                                    | -                                               | Covers nodes 1-3: Data retrieval and AI evaluation                                         |
| Sticky Note1                  | Sticky Note                    | Block explanation for Google Sheets interaction | -                                    | -                                               | Covers nodes 4-7: Google Sheets check and update logic                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Node Type: **Form Trigger**  
   - Name: `Trigger on TikTok Username Form`  
   - Configuration:  
     - Form Title: "Tiktok User"  
     - Form Field: Single text input labeled "Tik Tok Username"  
   - This node listens for incoming form submissions containing TikTok usernames.

2. **Create HTTP Request Node:**  
   - Node Type: **HTTP Request**  
   - Name: `Get TikTok Profile (Dumpling AI)`  
   - Connection: Connect from Trigger node output  
   - Configuration:  
     - Method: POST  
     - URL: `https://app.dumplingai.com/api/v1/get-tiktok-profile`  
     - Body Content Type: JSON  
     - JSON Body:  
       ```json
       { "handle": "={{ $json['Tik Tok Username'] }}" }
       ```  
     - Authentication: HTTP Header Auth using Dumpling AI API credentials  
   - Credentials: Create or select existing Dumpling AI HTTP Header authentication

3. **Create OpenAI Node:**  
   - Node Type: **OpenAI (LangChain)**  
   - Name: `Evaluate Profile Qualification (GPT-4)`  
   - Connection: Connect from Dumpling AI HTTP Request node output  
   - Configuration:  
     - Model: GPT-4.1  
     - Messages:  
       - System message: Explains criteria: user must have ≥40 videos, ≥100000 followers, ≥300000 hearts. Output "Qualified for influencer outreach" or "Not qualified" exactly.  
       - User message: Inject variables from previous node:  
         ```
         video_count: {{ $json.stats.videoCount }},  
         follower_count: {{ $json.stats.followerCount }},  
         heart_count: {{ $json.stats.heartCount }}
         ```  
   - Credentials: OpenAI API key configured

4. **Create Google Sheets Read Node:**  
   - Node Type: **Google Sheets**  
   - Name: `Check if User Already Exists`  
   - Connection: Connect from GPT-4 node output  
   - Configuration:  
     - Operation: Read rows with filter  
     - Document: Select your Google Sheet document with TikTok data  
     - Sheet: Select the sheet/tab (usually the first tab)  
     - Filter:  
       - Lookup Column: `User ID`  
       - Lookup Value: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.user.id }}`  
   - Credentials: Google Sheets OAuth2 account linked

5. **Create Switch Node:**  
   - Node Type: **Switch**  
   - Name: `Route Based on Existence`  
   - Connection: Connect from Google Sheets Read node output  
   - Configuration:  
     - Rule 1: If the number of rows returned (`Object.keys($json).length`) equals 0 → output "Does not exist"  
     - Rule 2: If the number of rows returned ≥1 → output "Exists"

6. **Create Google Sheets Append Node:**  
   - Node Type: **Google Sheets**  
   - Name: `Add New TikTok User to Sheet`  
   - Connection: Connect from Switch node "Does not exist" output  
   - Configuration:  
     - Operation: Append row  
     - Sheet and document: Same as Read node  
     - Columns mapping:  
       - `User ID`: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.user.id }}`  
       - `Qualifed?`: `={{ $('Evaluate Profile Qualification (GPT-4)').item.json.message.content }}`  
       - `Heart Count`: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.stats.heartCount }}`  
       - `Video Count`: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.stats.videoCount }}`  
       - `Tik Tok user`: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.user.uniqueId }}`  
       - `Follower Count`: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.stats.followerCount }}`  
       - `Following Count`: `={{ $('Get TikTok Profile (Dumpling AI)').item.json.stats.followingCount }}`

7. **Create Google Sheets AppendOrUpdate Node:**  
   - Node Type: **Google Sheets**  
   - Name: `Update Existing TikTok User in Sheet`  
   - Connection: Connect from Switch node "Exists" output  
   - Configuration:  
     - Operation: Append or update row  
     - Matching column: `User ID`  
     - Sheet and document: Same as above  
     - Columns mapping: same as Add New node (ensure all fields are mapped)  
     - This updates the row matching the User ID with fresh data

8. **(Optional) Add Sticky Note Nodes:**  
   - Add sticky notes to explain blocks and logic for clarity

9. **Credentials Setup:**  
   - Dumpling AI HTTP Header Auth: API key for Dumpling AI  
   - OpenAI API: API key for GPT-4  
   - Google Sheets OAuth2: Authorized Google account with access to the Google Sheet

10. **Testing:**  
    - Trigger the form with test TikTok usernames  
    - Monitor workflow executions for errors or data mismatches  
    - Verify Google Sheets rows are appended or updated correctly with qualification status

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow uses Dumpling AI API for TikTok profile retrieval. Requires valid API key and network.  | Dumpling AI API docs (not included, refer to https://app.dumplingai.com/docs if needed)                          |
| GPT-4 is used with a strict system prompt to enforce exact output for qualification result.      | OpenAI API best practices for prompt design                                                                    |
| Google Sheets document must have columns exactly as specified: `Tik Tok user`, `User ID`, etc.  | Google Sheets structure required for correct data mapping                                                      |
| Sticky notes provide block explanations and important instructions within the workflow editor.  | Enhances maintainability and onboarding for users modifying the workflow                                        |
| The workflow expects the TikTok username as input from an external form; ensure access to form. | n8n form trigger node allows embedding forms or integration with web frontends                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated workflow created with n8n, a no-code automation tool. The content respects all current content policies and contains no illegal, offensive, or protected material. All processed data is legal and public.