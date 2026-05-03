Send personalized birthday and anniversary emails with Google Sheets, Gemini, and Gmail

https://n8nworkflows.xyz/workflows/send-personalized-birthday-and-anniversary-emails-with-google-sheets--gemini--and-gmail-14802


# Send personalized birthday and anniversary emails with Google Sheets, Gemini, and Gmail

# 1. Workflow Overview

This workflow automatically sends personalized birthday or anniversary emails to clients. It runs once per day, reads a client list from Google Sheets, checks whether today matches each client’s birthday or anniversary, generates a context-aware celebratory message with Google Gemini, and sends the final email through Gmail.

Typical use cases:
- Wealth advisors sending relationship-building messages at scale
- Client success or account management teams maintaining regular personalized outreach
- Firms wanting semi-automated communications with light personalization logic

## 1.1 Trigger and Data Retrieval
The workflow starts on a daily schedule and reads all rows from a Google Sheet containing client data.

## 1.2 Configuration and Date Filtering
A settings node defines reusable labels, defaults, and email phrasing. The workflow then checks whether each row corresponds to a birthday or anniversary happening today.

## 1.3 Variable Preparation
For matching rows, the workflow compiles all message variables into one normalized object, including fallback values for age and risk profile.

## 1.4 Suggestion Selection Logic
A Code node selects a suitable “special idea” based on occasion, age group, risk profile, and premium/normal relationship type.

## 1.5 AI Message Generation
Gemini is prompted to write a warm message using both client metadata and the selected suggestion, while following strict tone and formatting rules.

## 1.6 Email Formatting and Delivery
The Gemini response is converted into HTML-friendly line breaks, recipient and subject fields are assembled, and Gmail sends the final message.

---

# 2. Block-by-Block Analysis

## Block 1 — Documentation and Visual Grouping

### Overview
This block contains sticky notes used for human-readable guidance inside the n8n canvas. These notes do not affect execution but are important for understanding setup requirements and workflow structure.

### Nodes Involved
- Overview — Read This First
- Group — Trigger and Fetch
- Group — Filter and Prep
- Group — AI and Email

### Node Details

#### Overview — Read This First
- **Type and role:** Sticky Note; onboarding and setup guidance
- **Configuration choices:** Contains a summary of the workflow behavior and required setup steps
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** Standard sticky note behavior
- **Edge cases / failures:** None at runtime
- **Sub-workflow reference:** None

Content summary:
- Runs daily at 9:01 AM
- Reads rows from Google Sheets
- Checks birthday/anniversary
- Picks a personalized investment suggestion
- Uses Gemini to generate a message
- Sends via Gmail
- Setup requires Google Sheets OAuth2, Gemini API key, Gmail OAuth2, correct Sheet ID, and required columns

#### Group — Trigger and Fetch
- **Type and role:** Sticky Note; visual grouping for schedule and Google Sheets read
- **Configuration choices:** Describes that the schedule triggers daily and fetches client rows
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Group — Filter and Prep
- **Type and role:** Sticky Note; visual grouping for filtering and variable preparation
- **Configuration choices:** Describes date matching, settings, and variable assembly
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Group — AI and Email
- **Type and role:** Sticky Note; visual grouping for AI generation and Gmail sending
- **Configuration choices:** Describes suggestion selection, Gemini generation, and Gmail delivery
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** None
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

## Block 2 — Trigger and Fetch

### Overview
This block starts the workflow on a daily schedule and retrieves all client rows from Google Sheets. Each row then flows individually through downstream logic.

### Nodes Involved
- Run Every Day at 9 AM
- Read Client List from Google Sheet

### Node Details

#### Run Every Day at 9 AM
- **Type and role:** `Schedule Trigger`; entry point for timed execution
- **Configuration choices:** Configured to run daily at 9:01 AM despite the node name saying “9 AM”
- **Key expressions or variables used:** None
- **Input and output connections:** No input; outputs to `Read Client List from Google Sheet`
- **Version-specific requirements:** `typeVersion 1.3`
- **Edge cases / failures:**
  - Workflow must be activated for the trigger to run
  - Timezone depends on the n8n instance settings
  - Node name may mislead operators because actual trigger minute is `01`
- **Sub-workflow reference:** None

#### Read Client List from Google Sheet
- **Type and role:** `Google Sheets`; reads all client records
- **Configuration choices:**
  - Uses a fixed Google Sheet document ID
  - Reads from sheet `gid=0`, cached as `Sheet1`
  - No extra options configured
- **Key expressions or variables used:** None directly in parameters
- **Input and output connections:** Input from `Run Every Day at 9 AM`; output to `Settings — Change These Before Running`
- **Version-specific requirements:** `typeVersion 4.5`; requires Google Sheets OAuth2 credential
- **Edge cases / failures:**
  - OAuth credential missing or expired
  - Wrong sheet/document ID
  - Missing required columns
  - Empty rows or inconsistent date formats
  - Permission/access issues on the Google Sheet
- **Sub-workflow reference:** None

Important expected columns from the sticky note:
- Client Name
- Email
- Advisor Name
- Birthday
- Anniversary
- Relationship Type
- Client Age
- Risk Profile

Actual expressions in later nodes expect some exact column names that differ slightly:
- `Relationship Type (Premium / Normal)`
- `Client_Age`

This mismatch is a key implementation risk.

---

## Block 3 — Settings and Date Filtering

### Overview
This block centralizes reusable workflow constants, then filters the incoming rows so only clients with a birthday or anniversary matching today continue.

### Nodes Involved
- Settings — Change These Before Running
- Does Today Match Birthday or Anniversary?
- No Special Date Today — Stop Here

### Node Details

#### Settings — Change These Before Running
- **Type and role:** `Set`; defines shared configuration values and defaults
- **Configuration choices:**
  - Stores labels, subject prefixes, age thresholds, tone values, firm name, fallback age/risk profile, and current date values
  - Includes computed values:
    - `Today Date = {{ new Date().toISOString().slice(0,10) }}`
    - `Today Month Day = {{ new Date().toISOString().slice(5,10) }}`
- **Key expressions or variables used:**
  - `new Date().toISOString()...`
- **Input and output connections:** Input from `Read Client List from Google Sheet`; output to `Does Today Match Birthday or Anniversary?`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases / failures:**
  - Timezone may produce unexpected “today” value if server timezone differs from business timezone
  - Several settings are defined but not fully reused everywhere; some downstream expressions hardcode logic instead of relying on the settings
- **Sub-workflow reference:** None

Main stored values:
- Trigger Hour: `9`
- Advisor Firm Name: `Wealth Advisory Services`
- Premium Tone: `formal + warm`
- Normal Tone: `friendly`
- Premium Label: `premium`
- Birthday Label: `Birthday`
- Anniversary Label: `Anniversary`
- Default Risk Profile: `moderate`
- Default Client Age: `35`
- Young Age Limit: `30`
- Senior Age Limit: `50`
- Birthday subject prefix: `Happy Birthday`
- Anniversary subject prefix: `Happy Anniversary`
- Subject suffix: `A Special Gift Awaits Inside`
- AI message paragraph count: `3-4 short paragraphs`

#### Does Today Match Birthday or Anniversary?
- **Type and role:** `If`; filters rows based on month-day match
- **Configuration choices:**
  - OR combinator with two conditions:
    1. Birthday month-day equals today month-day
    2. Anniversary month-day equals today month-day
- **Key expressions or variables used:**
  - `new Date($('Read Client List from Google Sheet').item.json.Birthday).toISOString().slice(5,10)`
  - `new Date($('Read Client List from Google Sheet').item.json.Anniversary).toISOString().slice(5,10)`
  - `new Date().toISOString().slice(5,10)`
- **Input and output connections:**
  - Input from `Settings — Change These Before Running`
  - True output to `Prepare All Client and Message Variables`
  - False output to `No Special Date Today — Stop Here`
- **Version-specific requirements:** `typeVersion 2.3`
- **Edge cases / failures:**
  - Invalid date strings can cause expression errors or `Invalid Date`
  - Empty Birthday/Anniversary fields may break `toISOString()`
  - Date parsing may vary depending on source format
  - UTC conversion can shift the date near midnight depending on locale and sheet formatting
- **Sub-workflow reference:** None

#### No Special Date Today — Stop Here
- **Type and role:** `No Operation`; explicit dead-end for non-matching records
- **Configuration choices:** None
- **Key expressions or variables used:** None
- **Input and output connections:** Input from false branch of `Does Today Match Birthday or Anniversary?`; no output
- **Version-specific requirements:** `typeVersion 1`
- **Edge cases / failures:** None significant
- **Sub-workflow reference:** None

---

## Block 4 — Variable Preparation

### Overview
This block consolidates client data, configuration defaults, derived values, and the full AI prompt into a single item structure for downstream processing.

### Nodes Involved
- Prepare All Client and Message Variables

### Node Details

#### Prepare All Client and Message Variables
- **Type and role:** `Set`; normalizes all working variables into one payload
- **Configuration choices:**
  - Copies client data from the Google Sheets row
  - Applies fallback defaults for age and risk profile
  - Computes occasion, message tone, premium flag, birthday flag, subject line, and AI prompt
- **Key expressions or variables used:**
  - `$('Read Client List from Google Sheet').item.json[...]`
  - `$('Settings — Change These Before Running').first().json[...]`
  - Date comparison using `new Date(...).toISOString().slice(5,10)`
  - Fallback example:
    - `Client_Age || Default Client Age`
    - `Risk Profile || Default Risk Profile`
- **Input and output connections:** Input from true branch of `Does Today Match Birthday or Anniversary?`; output to `Pick Smart Investment Suggestion by Age and Risk`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases / failures:**
  - The expression expects exact Google Sheet headers:
    - `Relationship Type (Premium / Normal)`
    - `Client_Age`
  - `.toLowerCase()` on an empty relationship field can fail
  - Empty or malformed Birthday values can break all date-derived fields
  - Hardcoded `'premium'`, `'formal + warm'`, and `'friendly'` are partly duplicated in the AI prompt logic rather than consistently sourced from settings
- **Sub-workflow reference:** None

Important fields created:
- `Client Name`
- `Client Email`
- `Advisor Name`
- `Client Birthday`
- `Client Anniversary`
- `Relationship Type`
- `Client Age`
- `Risk Profile`
- `Occasion`
- `Message Tone`
- `Is Premium Client`
- `Is Birthday Today`
- `Today Date`
- `Email Subject`
- `AI Prompt`
- `Advisor Firm`

The `AI Prompt` instructs Gemini to:
- Write a birthday or anniversary message
- Use 3–4 short paragraphs
- Keep each paragraph 1–2 lines
- Add line breaks
- Keep warm and professional tone
- Add a subtle future-planning touch
- End only once with:
  - `Warm regards,`
  - advisor name
- Avoid duplicate signature lines

---

## Block 5 — Suggestion Selection Logic

### Overview
This block uses JavaScript to choose a suitable commemorative financial idea based on occasion, age bucket, risk profile, and premium status, then appends that context to the working payload.

### Nodes Involved
- Pick Smart Investment Suggestion by Age and Risk

### Node Details

#### Pick Smart Investment Suggestion by Age and Risk
- **Type and role:** `Code`; business logic for recommendation selection
- **Configuration choices:**
  - Reads the prepared variables
  - Separates birthday suggestions into three age groups:
    - young: under 30
    - mid: 30–50
    - senior: above 50
  - Uses a separate anniversary suggestion pool
  - Selects suggestion index by a deterministic “risk offset”:
    - high → 0
    - moderate → 1
    - otherwise → 2
- **Key expressions or variables used:**
  - `$('Prepare All Client and Message Variables').item.json`
  - `parseInt(v['Client Age']) || 35`
  - `(v['Risk Profile'] || 'moderate').toLowerCase()`
  - booleans interpreted defensively as both boolean and string values
- **Input and output connections:** Input from `Prepare All Client and Message Variables`; output to `Ask Gemini to Write Personalised Wish`
- **Version-specific requirements:** `typeVersion 2`
- **Edge cases / failures:**
  - Invalid birthday values can break `new Date(v['Client Birthday']).toISOString()`
  - Unknown risk profiles fall into the “else” branch and use offset `2`
  - Age thresholds are hardcoded in code and do not use the earlier settings values `Young Age Limit` and `Senior Age Limit`
  - The deterministic selection is not random; clients with same age/risk/occasion always get the same suggestion
- **Sub-workflow reference:** None

Output adds:
- `Investment Suggestion`
- `Investment Instrument`
- `Investment Why`
- `Investment Intro`
- `Age Group`

Behavior notes:
- Premium clients get:
  - `As your trusted advisor, I would love to suggest something meaningful for this special day —`
- Normal clients get:
  - `Here is a little idea to make this day even more memorable —`

---

## Block 6 — AI Generation

### Overview
This block asks Gemini to write the final personalized message. It combines the prebuilt prompt with an additional “special suggestion” section and constrains wording so it sounds warm rather than sales-oriented.

### Nodes Involved
- Ask Gemini to Write Personalised Wish

### Node Details

#### Ask Gemini to Write Personalised Wish
- **Type and role:** `Google Gemini` (`@n8n/n8n-nodes-langchain.googleGemini`); text generation
- **Configuration choices:**
  - Uses model `models/gemini-2.5-flash`
  - Sends one composed message containing:
    - the base AI prompt
    - the chosen intro/instrument/suggestion/reason
    - formatting and vocabulary constraints
- **Key expressions or variables used:**
  - `$json['AI Prompt']`
  - `$json['Investment Intro']`
  - `$json['Investment Instrument']`
  - `$json['Investment Suggestion']`
  - `$json['Investment Why']`
- **Input and output connections:** Input from `Pick Smart Investment Suggestion by Age and Risk`; output to `Format Email Body and Delivery Fields`
- **Version-specific requirements:**
  - `typeVersion 1.1`
  - Requires Google Gemini/PaLM credential configured in n8n
  - The exact response schema may vary across node versions or model integrations
- **Edge cases / failures:**
  - Invalid or missing API credentials
  - Model/API quota issues
  - Safety filter or output formatting deviations
  - Response structure may differ from what the next node expects
  - Prompt instructions can still be partially ignored by the model
- **Sub-workflow reference:** None

Special additional prompt rules:
- Include the suggestion as one short paragraph
- Make it feel like a meaningful memory idea, not advice
- Avoid terms such as:
  - invest
  - portfolio
  - returns
  - market
  - profit
  - financial planning
  - wealth management

---

## Block 7 — Email Formatting and Delivery

### Overview
This block converts the AI response into an HTML email body, builds delivery metadata, and sends the email with Gmail.

### Nodes Involved
- Format Email Body and Delivery Fields
- Send Personalised Wish Email to Client

### Node Details

#### Format Email Body and Delivery Fields
- **Type and role:** `Set`; transforms AI output into email-ready fields
- **Configuration choices:**
  - Extracts Gemini text from `$json['content']['parts'][0]['text']`
  - Replaces line breaks with HTML `<br>` tags
  - Builds recipient, subject, sender display name, and metadata
- **Key expressions or variables used:**
  - `$json['content']['parts'][0]['text'].replace(/\n\n/g, '<br><br>').replace(/\n/g, '<br>')`
  - values sourced from `Pick Smart Investment Suggestion by Age and Risk`
- **Input and output connections:** Input from `Ask Gemini to Write Personalised Wish`; output to `Send Personalised Wish Email to Client`
- **Version-specific requirements:** `typeVersion 3.4`
- **Edge cases / failures:**
  - If Gemini response schema differs, `content.parts[0].text` may be undefined
  - Empty text output causes formatting issues
  - If Gmail node expects plain text vs HTML handling, rendering may vary
- **Sub-workflow reference:** None

Generated fields:
- `Final Email Body`
- `Send To`
- `Subject Line`
- `Display Sender Name`
- `Client Name`
- `Occasion`
- `Investment Instrument Used`
- `Age Group`
- `Sent On Date`

#### Send Personalised Wish Email to Client
- **Type and role:** `Gmail`; sends the email
- **Configuration choices:**
  - Recipient from `Send To`
  - Subject from `Subject Line`
  - Body from `Final Email Body`
  - Sender display name from `Display Sender Name`
- **Key expressions or variables used:**
  - `={{ $json['Send To'] }}`
  - `={{ $json['Subject Line'] }}`
  - `={{ $json['Final Email Body'] }}`
  - `={{ $json['Display Sender Name'] }}`
- **Input and output connections:** Input from `Format Email Body and Delivery Fields`; no further output
- **Version-specific requirements:** `typeVersion 2.2`; requires Gmail OAuth2 credential
- **Edge cases / failures:**
  - Gmail OAuth token expired or revoked
  - Invalid recipient email
  - Sending quota or account restrictions
  - Sender name formatting may not display as expected in some email clients
  - HTML rendering depends on Gmail node handling of message content
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Overview — Read This First | Sticky Note | Canvas documentation and setup guidance |  |  | ## Birthday / Milestone Smart Message Generator<br>## How it works<br>Runs daily at 9:01 AM, reads all client rows from Google Sheets and checks if today matches anyone's Birthday or Anniversary. If yes, it picks a personalised investment suggestion based on age, risk profile and occasion, then asks Gemini to write a warm human message. Finally it sends the email directly to the client.<br>## Setup steps<br>1. Connect Google Sheets OAuth2 credential<br>2. Add your Google Gemini API key<br>3. Connect Gmail OAuth2 credential<br>4. Update the Sheet ID in the Read Client List node<br>5. Confirm your sheet has columns: Client Name, Email, Advisor Name, Birthday, Anniversary, Relationship Type, Client Age, Risk Profile<br>6. Adjust trigger time in the Schedule node if needed |
| Group — Trigger and Fetch | Sticky Note | Visual grouping for trigger and Google Sheets fetch |  |  | ## Trigger and Data Fetch<br>Schedule fires daily at 9:01 AM and reads every client row from the Google Sheet. All client data flows into the next step for date checking. |
| Group — Filter and Prep | Sticky Note | Visual grouping for filtering and variable prep |  |  | ## Date Check and Variable Preparation<br>Filters only clients whose Birthday or Anniversary matches today. Settings node holds all config. Prepare Variables builds every field needed for AI and email in one place. |
| Group — AI and Email | Sticky Note | Visual grouping for AI generation and email delivery |  |  | ## AI Generation and Email Delivery<br>Investment Suggestion picks the right idea by age, risk and occasion. Gemini writes a warm personalised message. Gmail sends it directly to the client. |
| Run Every Day at 9 AM | Schedule Trigger | Daily workflow entry point |  | Read Client List from Google Sheet | ## Trigger and Data Fetch<br>Schedule fires daily at 9:01 AM and reads every client row from the Google Sheet. All client data flows into the next step for date checking. |
| Read Client List from Google Sheet | Google Sheets | Fetch all client rows from spreadsheet | Run Every Day at 9 AM | Settings — Change These Before Running | ## Trigger and Data Fetch<br>Schedule fires daily at 9:01 AM and reads every client row from the Google Sheet. All client data flows into the next step for date checking. |
| Settings — Change These Before Running | Set | Central configuration and defaults | Read Client List from Google Sheet | Does Today Match Birthday or Anniversary? | ## Date Check and Variable Preparation<br>Filters only clients whose Birthday or Anniversary matches today. Settings node holds all config. Prepare Variables builds every field needed for AI and email in one place. |
| Does Today Match Birthday or Anniversary? | If | Filter rows whose birthday or anniversary matches today | Settings — Change These Before Running | Prepare All Client and Message Variables; No Special Date Today — Stop Here | ## Date Check and Variable Preparation<br>Filters only clients whose Birthday or Anniversary matches today. Settings node holds all config. Prepare Variables builds every field needed for AI and email in one place. |
| No Special Date Today — Stop Here | No Operation | End branch for non-matching rows | Does Today Match Birthday or Anniversary? |  | ## Date Check and Variable Preparation<br>Filters only clients whose Birthday or Anniversary matches today. Settings node holds all config. Prepare Variables builds every field needed for AI and email in one place. |
| Prepare All Client and Message Variables | Set | Normalize client data and construct prompt fields | Does Today Match Birthday or Anniversary? | Pick Smart Investment Suggestion by Age and Risk | ## Date Check and Variable Preparation<br>Filters only clients whose Birthday or Anniversary matches today. Settings node holds all config. Prepare Variables builds every field needed for AI and email in one place. |
| Pick Smart Investment Suggestion by Age and Risk | Code | Select occasion-specific suggestion based on age/risk | Prepare All Client and Message Variables | Ask Gemini to Write Personalised Wish | ## AI Generation and Email Delivery<br>Investment Suggestion picks the right idea by age, risk and occasion. Gemini writes a warm personalised message. Gmail sends it directly to the client. |
| Ask Gemini to Write Personalised Wish | Google Gemini | Generate personalized email text with AI | Pick Smart Investment Suggestion by Age and Risk | Format Email Body and Delivery Fields | ## AI Generation and Email Delivery<br>Investment Suggestion picks the right idea by age, risk and occasion. Gemini writes a warm personalised message. Gmail sends it directly to the client. |
| Format Email Body and Delivery Fields | Set | Convert AI output to HTML body and delivery fields | Ask Gemini to Write Personalised Wish | Send Personalised Wish Email to Client | ## AI Generation and Email Delivery<br>Investment Suggestion picks the right idea by age, risk and occasion. Gemini writes a warm personalised message. Gmail sends it directly to the client. |
| Send Personalised Wish Email to Client | Gmail | Send final email to client | Format Email Body and Delivery Fields |  | ## AI Generation and Email Delivery<br>Investment Suggestion picks the right idea by age, risk and occasion. Gemini writes a warm personalised message. Gmail sends it directly to the client. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n.
   - Suggested name: `N-0016 Birthday / Milestone Smart Message Generator`

2. **Add a Schedule Trigger node**.
   - Node type: `Schedule Trigger`
   - Name it: `Run Every Day at 9 AM`
   - Configure it to run every day at:
     - Hour: `9`
     - Minute: `1`
   - Note: the node name says 9 AM, but the actual configured time is 9:01.

3. **Add a Google Sheets node** after the trigger.
   - Node type: `Google Sheets`
   - Name it: `Read Client List from Google Sheet`
   - Connect `Run Every Day at 9 AM` → `Read Client List from Google Sheet`
   - Select Google Sheets OAuth2 credentials
   - Set the document ID to your client spreadsheet
   - Select sheet `gid=0` / first sheet as appropriate
   - Configure the node to read rows from the sheet

4. **Prepare the spreadsheet columns**.
   - Ensure your sheet includes at least these columns:
     - `Client Name`
     - `Email`
     - `Advisor Name`
     - `Birthday`
     - `Anniversary`
     - `Relationship Type (Premium / Normal)`
     - `Client_Age`
     - `Risk Profile`
   - Important: use these exact headers if you want the workflow to work without expression changes.

5. **Add a Set node for configuration**.
   - Node type: `Set`
   - Name it: `Settings — Change These Before Running`
   - Connect `Read Client List from Google Sheet` → `Settings — Change These Before Running`
   - Add these fields:
     - `Trigger Hour` = number `9`
     - `Advisor Firm Name` = `Wealth Advisory Services`
     - `Premium Tone` = `formal + warm`
     - `Normal Tone` = `friendly`
     - `Premium Label` = `premium`
     - `Birthday Label` = `Birthday`
     - `Anniversary Label` = `Anniversary`
     - `Default Risk Profile` = `moderate`
     - `Default Client Age` = `35`
     - `Young Age Limit` = `30`
     - `Senior Age Limit` = `50`
     - `Email Subject Birthday Prefix` = `Happy Birthday`
     - `Email Subject Anniversary Prefix` = `Happy Anniversary`
     - `Email Subject Suffix` = `A Special Gift Awaits Inside`
     - `AI Message Paragraph Count` = `3-4 short paragraphs`
     - `Today Date` = expression `{{ new Date().toISOString().slice(0,10) }}`
     - `Today Month Day` = expression `{{ new Date().toISOString().slice(5,10) }}`

6. **Add an If node for date filtering**.
   - Node type: `If`
   - Name it: `Does Today Match Birthday or Anniversary?`
   - Connect `Settings — Change These Before Running` → `Does Today Match Birthday or Anniversary?`
   - Set combinator to `OR`
   - Add condition 1:
     - Left value: `{{ new Date($('Read Client List from Google Sheet').item.json.Birthday).toISOString().slice(5,10) }}`
     - Operator: `equals`
     - Right value: `{{ new Date().toISOString().slice(5,10) }}`
   - Add condition 2:
     - Left value: `{{ new Date($('Read Client List from Google Sheet').item.json.Anniversary).toISOString().slice(5,10) }}`
     - Operator: `equals`
     - Right value: `{{ new Date().toISOString().slice(5,10) }}`

7. **Add a No Operation node for the false branch**.
   - Node type: `No Operation`
   - Name it: `No Special Date Today — Stop Here`
   - Connect the false output of `Does Today Match Birthday or Anniversary?` to this node

8. **Add a Set node for variable preparation**.
   - Node type: `Set`
   - Name it: `Prepare All Client and Message Variables`
   - Connect the true output of `Does Today Match Birthday or Anniversary?` to this node
   - Add the following fields using expressions:

   - `Client Name`  
     `{{ $('Read Client List from Google Sheet').item.json['Client Name'] }}`

   - `Client Email`  
     `{{ $('Read Client List from Google Sheet').item.json['Email'] }}`

   - `Advisor Name`  
     `{{ $('Read Client List from Google Sheet').item.json['Advisor Name'] }}`

   - `Client Birthday`  
     `{{ $('Read Client List from Google Sheet').item.json['Birthday'] }}`

   - `Client Anniversary`  
     `{{ $('Read Client List from Google Sheet').item.json['Anniversary'] }}`

   - `Relationship Type`  
     `{{ $('Read Client List from Google Sheet').item.json['Relationship Type (Premium / Normal)'] }}`

   - `Client Age`  
     `{{ $('Read Client List from Google Sheet').item.json['Client_Age'] || $('Settings — Change These Before Running').first().json['Default Client Age'] }}`

   - `Risk Profile`  
     `{{ $('Read Client List from Google Sheet').item.json['Risk Profile'] || $('Settings — Change These Before Running').first().json['Default Risk Profile'] }}`

   - `Occasion`  
     `{{ new Date($('Read Client List from Google Sheet').item.json['Birthday']).toISOString().slice(5,10) === new Date().toISOString().slice(5,10) ? $('Settings — Change These Before Running').first().json['Birthday Label'] : $('Settings — Change These Before Running').first().json['Anniversary Label'] }}`

   - `Message Tone`  
     `{{ $('Read Client List from Google Sheet').item.json['Relationship Type (Premium / Normal)'].toLowerCase() === $('Settings — Change These Before Running').first().json['Premium Label'] ? $('Settings — Change These Before Running').first().json['Premium Tone'] : $('Settings — Change These Before Running').first().json['Normal Tone'] }}`

   - `Is Premium Client`  
     `{{ $('Read Client List from Google Sheet').item.json['Relationship Type (Premium / Normal)'].toLowerCase() === $('Settings — Change These Before Running').first().json['Premium Label'] }}`

   - `Is Birthday Today`  
     `{{ new Date($('Read Client List from Google Sheet').item.json['Birthday']).toISOString().slice(5,10) === new Date().toISOString().slice(5,10) }}`

   - `Today Date`  
     `{{ $('Settings — Change These Before Running').first().json['Today Date'] }}`

   - `Email Subject`  
     `{{ (new Date($('Read Client List from Google Sheet').item.json['Birthday']).toISOString().slice(5,10) === new Date().toISOString().slice(5,10) ? $('Settings — Change These Before Running').first().json['Email Subject Birthday Prefix'] : $('Settings — Change These Before Running').first().json['Email Subject Anniversary Prefix']) + ', ' + $('Read Client List from Google Sheet').item.json['Client Name'] + '! ' + $('Settings — Change These Before Running').first().json['Email Subject Suffix'] }}`

   - `AI Prompt`  
     Build one long expression that:
     - identifies whether it is birthday or anniversary
     - includes client name
     - sets tone based on relationship type
     - requests 3–4 short paragraphs
     - asks for line breaks
     - requests a subtle future-planning touch
     - enforces closing with `Warm regards,` and advisor name only once

   - `Advisor Firm`  
     `{{ $('Settings — Change These Before Running').first().json['Advisor Firm Name'] }}`

9. **Add a Code node for suggestion logic**.
   - Node type: `Code`
   - Name it: `Pick Smart Investment Suggestion by Age and Risk`
   - Connect `Prepare All Client and Message Variables` → `Pick Smart Investment Suggestion by Age and Risk`
   - Paste logic that:
     - reads values from the previous node
     - groups birthday clients into:
       - `young` if age < 30
       - `mid` if age <= 50
       - `senior` otherwise
     - uses separate anniversary suggestions
     - maps risk profile to index:
       - `high` → 0
       - `moderate` → 1
       - other/low → 2
     - outputs all original fields plus:
       - `Investment Suggestion`
       - `Investment Instrument`
       - `Investment Why`
       - `Investment Intro`
       - `Age Group`

   - Use the exact suggestion texts if you want behavior identical to the provided workflow.

10. **Add a Google Gemini node**.
    - Node type: `Google Gemini` / `@n8n/n8n-nodes-langchain.googleGemini`
    - Name it: `Ask Gemini to Write Personalised Wish`
    - Connect `Pick Smart Investment Suggestion by Age and Risk` → `Ask Gemini to Write Personalised Wish`
    - Configure credentials:
      - Google Gemini / PaLM API credential
    - Set model:
      - `models/gemini-2.5-flash`
    - Add one message whose content is an expression combining:
      - `$json['AI Prompt']`
      - a section labeled internally as special suggestion context
      - intro line
      - instrument
      - suggestion
      - why it is meaningful
      - formatting rules:
        - one short paragraph
        - emotional tone
        - no sales pitch
        - avoid restricted financial words
        - place suggestion before the closing signature

11. **Add a Set node to format the email**.
    - Node type: `Set`
    - Name it: `Format Email Body and Delivery Fields`
    - Connect `Ask Gemini to Write Personalised Wish` → `Format Email Body and Delivery Fields`
    - Add these fields:

    - `Final Email Body`  
      `{{ $json['content']['parts'][0]['text'].replace(/\n\n/g, '<br><br>').replace(/\n/g, '<br>') }}`

    - `Send To`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Client Email'] }}`

    - `Subject Line`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Email Subject'] }}`

    - `Display Sender Name`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Advisor Name'] + ' — ' + $('Pick Smart Investment Suggestion by Age and Risk').item.json['Advisor Firm'] }}`

    - `Client Name`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Client Name'] }}`

    - `Occasion`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Occasion'] }}`

    - `Investment Instrument Used`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Investment Instrument'] }}`

    - `Age Group`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Age Group'] }}`

    - `Sent On Date`  
      `{{ $('Pick Smart Investment Suggestion by Age and Risk').item.json['Today Date'] }}`

12. **Add a Gmail node**.
    - Node type: `Gmail`
    - Name it: `Send Personalised Wish Email to Client`
    - Connect `Format Email Body and Delivery Fields` → `Send Personalised Wish Email to Client`
    - Configure Gmail OAuth2 credentials
    - Set:
      - `To` = `{{ $json['Send To'] }}`
      - `Subject` = `{{ $json['Subject Line'] }}`
      - `Message` = `{{ $json['Final Email Body'] }}`
      - Sender name option = `{{ $json['Display Sender Name'] }}`

13. **Optionally add the same sticky notes** for clarity.
    - Add one general overview note
    - Add one note for trigger/fetch
    - Add one note for filter/prep
    - Add one note for AI/email

14. **Test with sample rows** in Google Sheets.
    - Include at least:
      - one row with today as birthday
      - one row with today as anniversary
      - one row with no matching date
    - Verify formatting and subject lines

15. **Validate date formatting before production use**.
    - Prefer Google Sheets date cells, not inconsistent text dates
    - Confirm n8n parses them consistently in your locale/timezone

16. **Activate the workflow** after successful test runs.
    - The workflow in the JSON is currently inactive
    - Activation is required for the schedule trigger to run automatically

### Credential setup summary
- **Google Sheets OAuth2**
  - Must have access to the source spreadsheet
- **Google Gemini / PaLM API**
  - Must be valid and enabled for the selected model
- **Gmail OAuth2**
  - Must permit send access from the connected mailbox

### No sub-workflow setup required
This workflow does not invoke any sub-workflows and has only one entry point: the Schedule Trigger.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow is inactive in the provided export. | Activate it in n8n after testing. |
| The trigger node name says “9 AM”, but the actual schedule is 9:01 AM. | Review the Schedule Trigger configuration. |
| The sheet column names described in the note do not exactly match the expressions used later. The workflow expects `Relationship Type (Premium / Normal)` and `Client_Age`. | Align Google Sheets headers or update expressions accordingly. |
| Several age and label settings are centralized in the Settings node, but some logic is still hardcoded in downstream expressions and code. | Consider refactoring for consistency. |
| Date parsing relies heavily on `new Date(...).toISOString()`. | Watch for timezone shifts, empty cells, and inconsistent sheet date formats. |