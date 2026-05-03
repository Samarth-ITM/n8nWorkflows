Summarize Text with RapidAPI and Log Results to Google Sheets

https://n8nworkflows.xyz/workflows/summarize-text-with-rapidapi-and-log-results-to-google-sheets-6304


# Summarize Text with RapidAPI and Log Results to Google Sheets

### 1. Workflow Overview

This workflow automates the process of summarizing user-submitted text and logging the results into Google Sheets. It is designed to:

- Receive input text and related parameters via a user web form.
- Call an external Text Summarizer API from RapidAPI to generate a summary.
- Check if the summary was successfully generated.
- Log either the successful summary or an error message into a Google Sheet.
- Handle success and failure cases gracefully with brief wait periods before logging.

The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Capture user input from a web form submission.
- **1.2 Data Preparation:** Transform and map form inputs to API-compatible formats.
- **1.3 API Interaction:** Send the formatted data to the summarization API and receive the summary.
- **1.4 Decision Making:** Evaluate API response for success or failure.
- **1.5 Success Path:** Wait briefly then log the summary in Google Sheets.
- **1.6 Failure Path:** Wait briefly then log an error message in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon user submission of a web form that collects text summarization parameters.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: `formTrigger` (Trigger node)  
    - Role: Presents a web form to the user for input; triggers workflow when submitted.  
    - Configuration:  
      - Form titled *Text Summerizer*  
      - Fields:  
        - Title (text input, required)  
        - Content (textarea, required)  
        - Mode (dropdown: Paragraph or Bullet, required)  
        - Length (dropdown: Short, Medium, Long, required)  
      - Response mode: returns response from the last node executed  
    - Inputs: None (trigger node)  
    - Outputs: Triggers next node with JSON containing form data  
    - Edge cases: User submits empty or invalid data (mitigated by required fields)  
    - Version: 2.2  

#### 1.2 Data Preparation

- **Overview:**  
  Transforms form inputs to formats compatible with the API: converts mode to lowercase and length to numeric values.

- **Nodes Involved:**  
  - Mapping

- **Node Details:**

  - **Mapping**  
    - Type: `set` (Data transformation node)  
    - Role: Prepare and normalize input data for the API request.  
    - Configuration:  
      - Converts `Mode` value to lowercase string (e.g., "Paragraph" → "paragraph")  
      - Maps `Length` string to numeric: Short → 1, Medium → 2, Long → 3  
      - Includes all other original fields unchanged  
    - Expressions:  
      - `Mode` = `{{$json.Mode.toLowerCase()}}`  
      - `Length` = ternary mapping expression based on input string  
    - Inputs: From On form submission node  
    - Outputs: Prepared JSON with mapped fields  
    - Edge cases: Unexpected or missing values for Mode or Length (no explicit validation)  
    - Version: 3.4  

#### 1.3 API Interaction

- **Overview:**  
  Sends a POST request with the user’s text and parameters to the RapidAPI Text Summarizer endpoint and receives the summarized text.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**

  - **HTTP Request**  
    - Type: `httpRequest`  
    - Role: Perform external API call to summarize text.  
    - Configuration:  
      - URL: `https://text-summarizer-ai.p.rapidapi.com/text-summarizer.php`  
      - Method: POST  
      - Content-Type: multipart/form-data  
      - Body Parameters:  
        - text: user content  
        - mode: mapped mode (lowercase)  
        - length: mapped length (numeric)  
      - Headers:  
        - `x-rapidapi-host`: `text-summarizer-ai.p.rapidapi.com`  
        - `x-rapidapi-key`: user must supply their RapidAPI key (credential)  
      - On Error: Continue workflow with error output (does not stop workflow)  
    - Inputs: Mapped data from previous node  
    - Outputs: API response with summary or error  
    - Edge cases:  
      - API key invalid or missing → authentication error  
      - Network timeout or API unavailability  
      - Unexpected API response format or missing summary field  
    - Version: 4.2  

#### 1.4 Decision Making

- **Overview:**  
  Checks if the API response contains a non-empty `summary` field to determine success or failure path.

- **Nodes Involved:**  
  - If

- **Node Details:**

  - **If**  
    - Type: `if` (Conditional routing)  
    - Role: Evaluate if the summarization was successful.  
    - Configuration:  
      - Condition: `summary` field in JSON is not empty (`notEmpty` operator)  
      - Case-sensitive and strict type validation enabled  
    - Inputs: API response from HTTP Request node  
    - Outputs:  
      - True branch: summary exists (success)  
      - False branch: summary missing or empty (failure)  
    - Edge cases: summary field missing or empty string, malformed response  
    - Version: 2.2  

#### 1.5 Success Path

- **Overview:**  
  Adds a wait period before writing successful summary data to Google Sheets.

- **Nodes Involved:**  
  - Wait  
  - Google Sheets (success logging)

- **Node Details:**

  - **Wait**  
    - Type: `wait`  
    - Role: Pause workflow briefly before proceeding to logging  
    - Configuration: No specific wait time configured (defaults to immediate or minimal delay)  
    - Inputs: True output from If node  
    - Outputs: Next node to Google Sheets success logging  
    - Edge cases: None significant  
    - Version: 1.1  

  - **Google Sheets**  
    - Type: `googleSheets`  
    - Role: Append or update row in Google Sheets with summary data  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Sheet: Sheet1 (gid=0)  
      - Document ID: user must specify Google Sheet URL or ID  
      - Authentication: Service Account OAuth2 credentials  
      - Columns mapped:  
        - Mode (from form)  
        - Title (from form)  
        - Length (from form)  
        - Summary (API response summary)  
        - Generated date (form submission timestamp)  
    - Inputs: From Wait node  
    - Outputs: None (end node)  
    - Edge cases:  
      - Google Sheets API errors (permission, quota)  
      - Missing or invalid document ID or sheet name  
    - Version: 4.6  

#### 1.6 Failure Path

- **Overview:**  
  Adds a wait period before logging error information to Google Sheets when summarization fails.

- **Nodes Involved:**  
  - Wait1  
  - Google Sheets1 (error logging)

- **Node Details:**

  - **Wait1**  
    - Type: `wait`  
    - Role: Pause workflow briefly before logging error message  
    - Configuration: No specific wait time configured  
    - Inputs: False output from If node or error output from HTTP Request node  
    - Outputs: Google Sheets1 node  
    - Edge cases: None significant  
    - Version: 1.1  

  - **Google Sheets1**  
    - Type: `googleSheets`  
    - Role: Append or update row in Google Sheets with error message  
    - Configuration:  
      - Operation: appendOrUpdate  
      - Sheet: Sheet1 (gid=0)  
      - Document ID: same as success path (user provided)  
      - Authentication: Service Account OAuth2 credentials  
      - Columns mapped:  
        - Mode (from form)  
        - Title (from form)  
        - Length (from form)  
        - Summary: fixed string `"Error occured. Please Try later"`  
        - Generated date (form submission timestamp)  
    - Inputs: From Wait1 node or error branch from HTTP Request  
    - Outputs: None (end node)  
    - Edge cases: Same as Google Sheets success node (permissions, quota)  
    - Version: 4.6  

---

### 3. Summary Table

| Node Name            | Node Type          | Functional Role                       | Input Node(s)         | Output Node(s)              | Sticky Note                                                                                                   |
|----------------------|--------------------|------------------------------------|-----------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission   | formTrigger        | Trigger workflow on form submission | None                  | Mapping                    | Acts as the trigger node for the workflow. Presents a form with Title, Content, Mode, and Length.            |
| Mapping             | set                | Transform input data for API         | On form submission    | HTTP Request               | Prepares and transforms input data: Mode to lowercase, Length to numeric.                                   |
| HTTP Request        | httpRequest        | Send text and params to summarizer API | Mapping               | If, Wait1                  | Sends POST request to RapidAPI summarizer with text, mode, length; handles errors without stopping flow.    |
| If                  | if                 | Check if summary returned            | HTTP Request          | Wait (success), Wait1 (fail) | Checks if API returned a non-empty summary for routing success or failure path.                               |
| Wait                | wait               | Pause before logging success         | If (true)             | Google Sheets              | Adds a short delay before storing successful summaries.                                                      |
| Google Sheets       | googleSheets       | Log successful summaries             | Wait                  | None                      | Appends or updates Google Sheet with summary data and metadata.                                             |
| Wait1               | wait               | Pause before logging failure         | If (false), HTTP Request (error) | Google Sheets1             | Adds a short delay before storing error information.                                                        |
| Google Sheets1      | googleSheets       | Log error message                    | Wait1                 | None                      | Appends or updates Google Sheet with error message in summary column.                                       |
| Sticky Note         | stickyNote         | Documentation                       | None                  | None                      | Text summarizing workflow features, nodes, logic, and benefits.                                             |
| Sticky Note1        | stickyNote         | Documentation                       | None                  | None                      | Describes "On form submission" node purpose and function.                                                   |
| Sticky Note2        | stickyNote         | Documentation                       | None                  | None                      | Describes "Mapping" node purpose and function.                                                              |
| Sticky Note3        | stickyNote         | Documentation                       | None                  | None                      | Describes "HTTP Request" node purpose and function.                                                         |
| Sticky Note4        | stickyNote         | Documentation                       | None                  | None                      | Describes "If" node purpose and function.                                                                    |
| Sticky Note5        | stickyNote         | Documentation                       | None                  | None                      | Describes "Wait" (success path) node purpose and function.                                                  |
| Sticky Note6        | stickyNote         | Documentation                       | None                  | None                      | Describes "Google Sheets" (success path) node purpose and function.                                         |
| Sticky Note7        | stickyNote         | Documentation                       | None                  | None                      | Describes "Wait1" (error path) node purpose and function.                                                   |
| Sticky Note8        | stickyNote         | Documentation                       | None                  | None                      | Describes "Google Sheets1" (error path) node purpose and function.                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a `formTrigger` node:**
   - Name: `On form submission`
   - Form Title: `Text Summerizer`
   - Add the following form fields (all required):
     - Title (Text)
     - Content (Textarea)
     - Mode (Dropdown) with options: `Paragraph`, `Bullet`
     - Length (Dropdown) with options: `Short`, `Medium`, `Long`
   - Set `Response Mode` to `lastNode`.
   - Position: at workflow start.
   - Save.

3. **Add a `set` node:**
   - Name: `Mapping`
   - Purpose: prepare data for API
   - Add two new fields:
     - `Mode` (string) with expression: `{{$json.Mode.toLowerCase()}}`
     - `Length` (string or number) with expression:
       ```
       {{$json.Length === 'Short' ? 1 : $json.Length === 'Medium' ? 2 : $json.Length === 'Long' ? 3 : $json.Length}}
       ```
   - Enable "Include Other Fields" to pass original data downstream.
   - Connect output of `On form submission` to input of `Mapping`.

4. **Add an `httpRequest` node:**
   - Name: `HTTP Request`
   - Method: POST
   - URL: `https://text-summarizer-ai.p.rapidapi.com/text-summarizer.php`
   - Content type: `multipart/form-data`
   - Body parameters (multipart):
     - `text` = `{{$json.Content}}`
     - `mode` = `{{$json.Mode}}`
     - `length` = `{{$json.Length}}`
   - Header parameters:
     - `x-rapidapi-host` = `text-summarizer-ai.p.rapidapi.com`
     - `x-rapidapi-key` = your RapidAPI key (replace `"your key"`)
   - Set "On Error" to `Continue with Error Output`.
   - Connect `Mapping` output to `HTTP Request` input.

5. **Add an `if` node:**
   - Name: `If`
   - Condition: Check if `$json.summary` is not empty (`notEmpty` operator).
   - Case sensitive: true, type validation: strict.
   - Connect `HTTP Request` main output to `If` input.

6. **Add a `wait` node:**
   - Name: `Wait`
   - No delay configured (default)
   - Connect `If` true output to this `Wait` node.

7. **Add a `googleSheets` node:**
   - Name: `Google Sheets`
   - Operation: Append or Update
   - Sheet Name: `Sheet1` (gid=0)
   - Document ID: provide your Google Sheet URL or ID
   - Authentication: Service Account OAuth2 (setup credentials accordingly)
   - Columns to map:
     - Mode = `{{$node["On form submission"].json.Mode}}`
     - Title = `{{$node["On form submission"].json.Title}}`
     - Length = `{{$node["On form submission"].json.Length}}`
     - Summary = `{{$json.summary}}`
     - Generated date = `{{$node["On form submission"].json.submittedAt}}`
   - Connect `Wait` output to `Google Sheets`.

8. **Add a second `wait` node:**
   - Name: `Wait1`
   - No delay configured (default)
   - Connect `If` false output and `HTTP Request` error output to `Wait1`.

9. **Add a second `googleSheets` node:**
   - Name: `Google Sheets1`
   - Operation: Append or Update
   - Sheet Name: `Sheet1` (gid=0)
   - Document ID: same as success node
   - Authentication: Service Account OAuth2
   - Columns to map:
     - Mode = `{{$node["On form submission"].json.Mode}}`
     - Title = `{{$node["On form submission"].json.Title}}`
     - Length = `{{$node["On form submission"].json.Length}}`
     - Summary = fixed string `"Error occured. Please Try later"`
     - Generated date = `{{$node["On form submission"].json.submittedAt}}`
   - Connect `Wait1` output to `Google Sheets1`.

10. **Save workflow and activate it.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                 | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| This workflow enables automated AI-powered text summarization with error handling and result logging to Google Sheets.                                       | Workflow description                                                                                     |
| Ensure you have valid RapidAPI credentials to access the Text Summarizer API: https://rapidapi.com/                                                          | RapidAPI Text Summarizer documentation                                                                   |
| Google Sheets authentication uses Service Account credentials; ensure the account has edit permissions on the target spreadsheet.                            | Google Sheets API and OAuth2 configuration                                                               |
| The workflow includes brief wait nodes to improve sequencing and avoid race conditions when writing to Google Sheets.                                        | Best practice for API integrations and Google Sheets update consistency                                  |
| For customization, you can modify form fields, API endpoint, or Google Sheets columns as needed.                                                             | Workflow flexibility                                                                                      |
| Sticky notes within the workflow provide detailed explanations of each node’s purpose and configuration.                                                     | In-workflow documentation                                                                                |
| The formTrigger node requires the workflow to be deployed and accessible via a webhook URL for users to submit their data.                                  | n8n form trigger deployment specifics                                                                    |

---

**Disclaimer:** The text and workflow described here are fully compliant with all content policies and only process legal and public data. No illegal or offensive content is involved.