Automated Space-Themed Birthday Emails with NASA Images, GPT-4, Gmail & Slack

https://n8nworkflows.xyz/workflows/automated-space-themed-birthday-emails-with-nasa-images--gpt-4--gmail---slack-10645


# Automated Space-Themed Birthday Emails with NASA Images, GPT-4, Gmail & Slack

### 1. Workflow Overview

This workflow automates the process of sending personalized, space-themed birthday greetings every morning at 7:00 AM. It targets teams, communities, or organizations that want to celebrate birthdays with a unique cosmic touch by integrating NASA’s Earth images, AI-generated messages, Gmail for email delivery, and Slack for team notifications.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Data Input:** Triggers the workflow daily and fetches birthday data from a Google Sheet.
- **1.2 Birthday Filtering & Conditional Branching:** Filters the list to identify birthdays occurring on the current day and decides whether to proceed.
- **1.3 NASA Image Retrieval:** Fetches the latest NASA EPIC Earth image to include in birthday messages.
- **1.4 Data Preparation & AI Message Generation:** Combines birthday data with NASA image info and generates personalized space-themed birthday messages using GPT-4.
- **1.5 Email Formatting & Sending:** Formats the combined data into HTML emails and sends them via Gmail.
- **1.6 Slack Notification:** Posts a summary notification in Slack about the number of birthdays celebrated that day.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Data Input

**Overview:**  
This block initiates the workflow every morning at 7:00 AM and retrieves the birthday roster from a Google Sheet.

**Nodes Involved:**  
- Every Morning at 7:00  
- Get Birthday Roster

**Node Details:**

- **Every Morning at 7:00**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 7:00 AM using a cron expression (`0 7 * * *`).  
  - Input: None (trigger node)  
  - Output: Triggers the next node to fetch birthday data.  
  - Edge Cases: Cron misconfiguration could cause missed triggers.

- **Get Birthday Roster**  
  - Type: Google Sheets  
  - Role: Reads birthday data from a specified Google Sheet document and sheet.  
  - Configuration: Requires Google Sheets credentials, Document ID, and Sheet Name. The sheet must have columns named `Name`, `Email`, and `Birthday`.  
  - Input: Trigger from schedule node  
  - Output: Outputs all rows from the sheet for further processing.  
  - Edge Cases:  
    - Missing or incorrect credentials cause authentication errors.  
    - Incorrect Document ID or Sheet Name results in no data or errors.  
    - Sheet missing required columns causes downstream failures.

---

#### 1.2 Birthday Filtering & Conditional Branching

**Overview:**  
Filters the birthday list to those whose birthday matches the current date and conditionally proceeds only if there are birthdays today.

**Nodes Involved:**  
- Filter Today's Birthdays  
- Any Birthdays Today?

**Node Details:**

- **Filter Today's Birthdays**  
  - Type: Code (JavaScript)  
  - Role: Iterates over all input rows, extracts the birthday date, and filters those matching today’s month and day.  
  - Key Logic:  
    - Parses birthday strings into Date objects.  
    - Compares month and day with current date.  
    - Returns a single JSON object containing an array of today’s birthdays or empty to stop workflow.  
  - Input: Birthday roster from Google Sheets node  
  - Output: Either a JSON object with `birthdays` array or empty to halt workflow.  
  - Edge Cases:  
    - Missing or malformed birthday fields are skipped.  
    - Timezone differences could affect date matching.

- **Any Birthdays Today?**  
  - Type: If  
  - Role: Checks if the filtered birthday list length is greater than zero.  
  - Condition: `{{$input.all().length}} > 0`  
  - Input: Output from Filter Today's Birthdays  
  - Output:  
    - True branch proceeds to fetch NASA images and count birthdays.  
    - False branch stops workflow (no birthdays today).  
  - Edge Cases: Empty input leads to workflow termination.

---

#### 1.3 NASA Image Retrieval

**Overview:**  
Fetches the latest natural color Earth images from NASA’s EPIC API to include in birthday greetings.

**Nodes Involved:**  
- Fetch NASA EPIC Images2

**Node Details:**

- **Fetch NASA EPIC Images2**  
  - Type: HTTP Request  
  - Role: Calls NASA’s EPIC API endpoint (`https://epic.gsfc.nasa.gov/api/natural`) to retrieve metadata about recent Earth images.  
  - Configuration:  
    - HTTP GET request expecting JSON response.  
  - Input: True branch from Any Birthdays Today? node  
  - Output: JSON array of image metadata objects.  
  - Edge Cases:  
    - API downtime or rate limiting causes request failures.  
    - Network timeouts or malformed responses.  
  - Version: Requires n8n version supporting HTTP Request node v4.3 or later.

---

#### 1.4 Data Preparation & AI Message Generation

**Overview:**  
Prepares combined data for each birthday person, including NASA image info, then generates personalized space-themed birthday messages using GPT-4.

**Nodes Involved:**  
- Prepare Birthday Data  
- Generate Space Birthday Message  
- OpenAI Chat Model  
- Format for Gmail

**Node Details:**

- **Prepare Birthday Data**  
  - Type: Code (JavaScript)  
  - Role:  
    - Extracts the first NASA EPIC image metadata.  
    - Constructs the full image URL based on the image date and name.  
    - Combines each birthday person’s data with the image info (caption, date, URL).  
  - Input: NASA EPIC images and filtered birthdays  
  - Output: Array of JSON objects, each containing personal and image data.  
  - Edge Cases:  
    - No images returned from NASA API leads to undefined data.  
    - Date parsing errors.  

- **Generate Space Birthday Message**  
  - Type: LangChain Agent (AI)  
  - Role: Uses GPT-4 to generate a creative, space-themed birthday message incorporating the NASA EPIC photo caption.  
  - Configuration:  
    - System message instructs the AI to produce a ~120 character celebratory message referencing Earth and the image caption.  
    - Input variables: Person’s name and EPIC photo caption.  
  - Input: Prepared birthday data  
  - Output: AI-generated birthday message text only.  
  - Edge Cases:  
    - AI service downtime or quota exceeded.  
    - Unexpected AI output format.  
  - Version: Requires LangChain integration and OpenAI credentials.

- **OpenAI Chat Model**  
  - Type: LangChain LM Chat OpenAI  
  - Role: Provides the underlying GPT-4 model for the LangChain Agent node.  
  - Configuration: Uses model `gpt-4.1-mini` with OpenAI credentials.  
  - Input: From Generate Space Birthday Message node (AI language model input)  
  - Output: AI-generated text response.  
  - Edge Cases: API errors, authentication failures.

- **Format for Gmail**  
  - Type: Code (JavaScript)  
  - Role:  
    - Combines personal info and AI-generated messages into a single JSON object per recipient.  
    - Validates that the number of personal info items matches the number of AI messages to avoid mismatches.  
  - Input: Outputs from Prepare Birthday Data and Generate Space Birthday Message  
  - Output: Final formatted data ready for email sending.  
  - Edge Cases:  
    - Mismatched array lengths throw errors.  
    - Missing fields cause incomplete emails.

---

#### 1.5 Email Formatting & Sending

**Overview:**  
Sends personalized HTML birthday emails featuring the AI message and NASA Earth photo via Gmail.

**Nodes Involved:**  
- Send Birthday Email

**Node Details:**

- **Send Birthday Email**  
  - Type: Gmail  
  - Role: Sends an HTML email to each birthday person.  
  - Configuration:  
    - Uses Gmail OAuth2 credentials.  
    - Email subject includes birthday emoji and recipient’s name.  
    - Email body includes:  
      - Personalized greeting with name  
      - AI-generated birthday message  
      - NASA EPIC photo caption and image embedded with alt text  
      - Date of the photo formatted as local date string  
      - Closing wishes  
  - Input: Formatted data from Format for Gmail node  
  - Output: Email sent confirmation  
  - Edge Cases:  
    - Authentication errors with Gmail credentials.  
    - Invalid email addresses cause send failures.  
    - HTML rendering issues in email clients.

---

#### 1.6 Slack Notification

**Overview:**  
Posts a summary message in Slack indicating how many birthdays are celebrated today.

**Nodes Involved:**  
- Count Birthdays  
- Post Slack Notification

**Node Details:**

- **Count Birthdays**  
  - Type: Code (JavaScript)  
  - Role: Counts the number of birthdays in the filtered list.  
  - Input: Filtered birthdays data  
  - Output: JSON object with `birthdayCount` property  
  - Edge Cases: Empty list results in zero count.

- **Post Slack Notification**  
  - Type: Slack  
  - Role: Posts a message to a specified Slack channel with the count of birthdays.  
  - Configuration:  
    - Uses Slack OAuth2 credentials.  
    - Message text: "🎂 Birthdays today: X" where X is the count.  
    - Requires channel ID configuration.  
  - Input: Output from Count Birthdays node  
  - Output: Slack message confirmation  
  - Edge Cases:  
    - Authentication errors.  
    - Invalid or missing channel ID.  
    - Slack API rate limits.

---

### 3. Summary Table

| Node Name                  | Node Type                         | Functional Role                          | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                   |
|----------------------------|----------------------------------|----------------------------------------|------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| Every Morning at 7:00       | Schedule Trigger                 | Triggers workflow daily at 7:00 AM     | None                         | Get Birthday Roster            |                                                                                               |
| Get Birthday Roster         | Google Sheets                   | Fetches birthday data from Google Sheet| Every Morning at 7:00         | Filter Today's Birthdays       | ## 1. Configure Birthday Roster<br>- Select Google Sheets credentials<br>- Enter Document ID and Sheet Name<br>- Sheet must have `Name`, `Email`, `Birthday` columns |
| Filter Today's Birthdays    | Code                           | Filters birthdays matching today’s date| Get Birthday Roster           | Any Birthdays Today?           |                                                                                               |
| Any Birthdays Today?        | If                             | Checks if there are birthdays today    | Filter Today's Birthdays      | Fetch NASA EPIC Images2, Count Birthdays |                                                                                               |
| Fetch NASA EPIC Images2     | HTTP Request                   | Fetches latest NASA Earth images       | Any Birthdays Today?          | Prepare Birthday Data          |                                                                                               |
| Prepare Birthday Data       | Code                           | Combines birthday and NASA image data  | Fetch NASA EPIC Images2       | Generate Space Birthday Message|                                                                                               |
| Generate Space Birthday Message | LangChain Agent (AI)          | Generates space-themed birthday messages| Prepare Birthday Data         | Format for Gmail               | ## 2. Configure OpenAI Model<br>- Select OpenAI credentials<br>- Choose AI model (e.g., GPT-4) |
| OpenAI Chat Model           | LangChain LM Chat OpenAI        | Provides GPT-4 model for AI message    | Generate Space Birthday Message| Generate Space Birthday Message|                                                                                               |
| Format for Gmail            | Code                           | Merges personal info with AI messages  | Prepare Birthday Data, Generate Space Birthday Message | Send Birthday Email           |                                                                                               |
| Send Birthday Email         | Gmail                          | Sends personalized birthday emails     | Format for Gmail              | None                         | ## 4. Configure Email Sender<br>- Select Gmail credentials<br>- Email content pre-filled with data |
| Count Birthdays             | Code                           | Counts number of birthdays today        | Any Birthdays Today?          | Post Slack Notification        |                                                                                               |
| Post Slack Notification     | Slack                          | Posts birthday count notification       | Count Birthdays              | None                         | ## 3. Configure Slack Notification<br>- Select Slack credentials<br>- Set Slack channel ID     |
| Sticky Note                 | Sticky Note                    | Instructional note                      | None                         | None                         | ## 4. Configure Email Sender<br>- Select your Gmail credentials.<br>The email content is pre-filled with data from the previous steps. |
| Sticky Note1                | Sticky Note                    | Workflow overview and purpose          | None                         | None                         | ## Send daily space-themed birthday greetings via Gmail and Slack<br>This workflow automates daily birthday celebrations with a unique, cosmic twist! It checks a Google Sheet for birthdays each morning, fetches a stunning new image of Earth from NASA, uses AI to write a personalized space-themed message, and sends it as a celebratory email. It also posts a summary notification in Slack.<br><br>### Who it's for<br>Perfect for teams, communities, or anyone who wants a fun, automated way to remember and celebrate birthdays. If you love space and want to make someone's special day a little more cosmic, this is for you.<br><br>### What it does<br>1.  **Triggers Daily:** The workflow runs every morning at 7:00 AM.<br>2.  **Reads Birthdays:** It fetches a list of names and birthdays from a specified Google Sheet.<br>3.  **Checks for Today's Birthdays:** It filters the list to see if anyone has a birthday today.<br>4.  **Gets NASA Image:** If there's a birthday, it fetches the latest "Earth Polychromatic Imaging Camera" (EPIC) image from NASA's API.<br>5.  **Generates AI Message:** It uses an AI model to craft a unique, space-themed birthday message.<br>6.  **Sends Email:** It sends a personalized HTML email to the birthday person, featuring the message and the NASA Earth photo.<br>7.  **Notifies Slack:** It posts a message in a designated Slack channel to let the team know who is celebrating their birthday.<br><br>### How to set up<br>Follow the instructions in the sticky notes below to configure the workflow nodes. |
| Sticky Note2                | Sticky Note                    | Google Sheets configuration instructions| None                         | None                         | ## 1. Configure Birthday Roster<br>- Select your Google Sheets credentials.<br>- Enter the ID of the Google Sheet containing birthdays.<br>- Enter the name of the specific sheet.<br><br>**IMPORTANT:** Your sheet must have columns named `Name`, `Email`, and `Birthday`. |
| Sticky Note3                | Sticky Note                    | Slack notification configuration       | None                         | None                         | ## 3. Configure Slack Notification<br>- Select your Slack credentials.<br>- Choose the Slack channel where you want to post birthday notifications. |
| Sticky Note4                | Sticky Note                    | OpenAI model configuration instructions| None                         | None                         | ## 2. Configure OpenAI Model<br>- Select your OpenAI credentials.<br>- Choose the AI model you wish to use for generating messages. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set cron expression to `0 7 * * *` (every day at 7:00 AM)  
   - No credentials needed.

2. **Create a Google Sheets Node**  
   - Type: Google Sheets  
   - Credentials: Select or create Google Sheets OAuth2 credentials.  
   - Set Document ID to your Google Sheet containing birthdays.  
   - Set Sheet Name to the specific sheet name.  
   - Ensure the sheet has columns named `Name`, `Email`, and `Birthday`.  
   - Connect Schedule Trigger output to this node’s input.

3. **Create a Code Node to Filter Today's Birthdays**  
   - Type: Code (JavaScript)  
   - Paste the following logic:  
     ```javascript
     const today = new Date();
     const todayMonth = today.getMonth() + 1;
     const todayDay = today.getDate();

     const birthdaysToday = [];

     for (const item of $input.all()) {
       const birthdayStr = item.json.birthday || item.json.Birthday;
       if (!birthdayStr) continue;

       const birthday = new Date(birthdayStr);
       const birthMonth = birthday.getMonth() + 1;
       const birthDay = birthday.getDate();

       if (birthMonth === todayMonth && birthDay === todayDay) {
         birthdaysToday.push(item.json);
       }
     }

     if (birthdaysToday.length > 0) {
       return [{ json: { birthdays: birthdaysToday } }];
     }
     return [];
     ```  
   - Connect Google Sheets node output to this node.

4. **Create an If Node to Check for Birthdays**  
   - Type: If  
   - Condition: Check if input length > 0 (`{{$input.all().length}} > 0`)  
   - Connect Code node output to this node.

5. **Create an HTTP Request Node to Fetch NASA EPIC Images**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://epic.gsfc.nasa.gov/api/natural`  
   - Response Format: JSON  
   - Connect If node’s True output to this node.

6. **Create a Code Node to Prepare Birthday Data**  
   - Type: Code (JavaScript)  
   - Paste the following logic:  
     ```javascript
     const epicImages = $('Fetch NASA EPIC Images2').all().map(item => item.json);
     const allBirthdays = $('Filter Today\'s Birthdays').first().json.birthdays;

     const latestImage = epicImages[0];

     const results = [];

     const d = new Date(latestImage.date);
     const year = d.getFullYear();
     const month = ('0' + (d.getMonth() + 1)).slice(-2);
     const day = ('0' + d.getDate()).slice(-2);
     const imageName = latestImage.image;
     const imageUrl = `https://epic.gsfc.nasa.gov/archive/natural/${year}/${month}/${day}/png/${imageName}.png`;

     for (const person of allBirthdays) {
       results.push({
         json: {
           ...person,
           epicCaption: latestImage.caption,
           epicDate: latestImage.date,
           epicImageUrl: imageUrl
         }
       });
     }

     return results;
     ```  
   - Connect HTTP Request node output to this node.

7. **Create a LangChain Agent Node to Generate Birthday Messages**  
   - Type: LangChain Agent (AI)  
   - Credentials: Select OpenAI credentials.  
   - System Message:  
     ```
     You are a creative birthday message writer specializing in space-themed greetings.

     Your task is to:
     1. Generate a fun, space-themed birthday message for the person, mentioning our home planet Earth.
     2. Incorporate the NASA EPIC photo's caption into the message.
     3. Keep the message to approximately 120 characters.
     4. Make it celebratory and cosmic!
     5. Return ONLY the birthday message text, nothing else.
     ```  
   - Input Text Template:  
     ```
     Person's name: {{ $json.name || $json.Name }}
     Today's Earth photo caption: {{ $json.epicCaption }}
     ```  
   - Connect Prepare Birthday Data node output to this node.

8. **Create an OpenAI Chat Model Node**  
   - Type: LangChain LM Chat OpenAI  
   - Credentials: Use the same OpenAI credentials.  
   - Model: Select `gpt-4.1-mini` or preferred GPT-4 variant.  
   - Connect LangChain Agent node’s AI language model input to this node.

9. **Create a Code Node to Format Data for Gmail**  
   - Type: Code (JavaScript)  
   - Paste the following logic:  
     ```javascript
     const personalInfoItems = $('Prepare Birthday Data').all();
     const birthdayMessageItems = $('Generate Space Birthday Message').all();

     if (personalInfoItems.length !== birthdayMessageItems.length) {
       throw new Error(`Mismatch in item counts: personal info ${personalInfoItems.length}, AI messages ${birthdayMessageItems.length}`);
     }

     const finalPackages = [];

     for (let i = 0; i < personalInfoItems.length; i++) {
       const personalInfo = personalInfoItems[i].json;
       const birthdayMessage = birthdayMessageItems[i].json.output;

       finalPackages.push({
         json: {
           ...personalInfo,
           birthdayMessage: birthdayMessage,
         }
       });
     }

     return finalPackages;
     ```  
   - Connect LangChain Agent node output to this node.

10. **Create a Gmail Node to Send Birthday Emails**  
    - Type: Gmail  
    - Credentials: Select Gmail OAuth2 credentials.  
    - Send To: `={{ $json.Email }}`  
    - Subject: `=🎂🌏 Happy Birthday {{ $json.Name }} ！`  
    - Message (HTML):  
      ```html
      <h2>Happy Birthday {{ $json.name || $json.Name }}! 🎂🌏</h2>

      <p>{{ $json.birthdayMessage }}</p>

      <h3>A View of Our Home Planet For You:</h3>
      <p><strong>{{ $json.epicCaption }}</strong></p>
      <img src="{{ $json.epicImageUrl }}" alt="NASA EPIC Earth Photo" style="max-width: 600px; height: auto;" />

      <p>This photo of Earth was taken on {{ new Date($json.epicDate).toLocaleDateString() }}.</p>

      <p>Wishing you a wonderful year ahead! 🌟</p>
      ```  
    - Connect Format for Gmail node output to this node.

11. **Create a Code Node to Count Birthdays**  
    - Type: Code (JavaScript)  
    - Logic:  
      ```javascript
      const birthdays = $input.item.json.birthdays || [];
      return [{ json: { birthdayCount: birthdays.length } }];
      ```  
    - Connect If node’s True output (parallel to NASA fetch) to this node.

12. **Create a Slack Node to Post Notification**  
    - Type: Slack  
    - Credentials: Select Slack OAuth2 credentials.  
    - Channel ID: Set the Slack channel ID where notifications should be posted.  
    - Message Text: `=🎂 Birthdays today: {{ $json.birthdayCount }}`  
    - Connect Count Birthdays node output to this node.

---

This completes the step-by-step reconstruction of the workflow, enabling automated daily space-themed birthday greetings with NASA images, AI-generated messages, Gmail email delivery, and Slack notifications.