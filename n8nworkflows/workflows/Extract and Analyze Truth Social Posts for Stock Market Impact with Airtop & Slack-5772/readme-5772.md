Extract and Analyze Truth Social Posts for Stock Market Impact with Airtop & Slack

https://n8nworkflows.xyz/workflows/extract-and-analyze-truth-social-posts-for-stock-market-impact-with-airtop---slack-5772


# Extract and Analyze Truth Social Posts for Stock Market Impact with Airtop & Slack

### 1. Workflow Overview

This workflow, named **"Trump-o-meter Template"**, automates the extraction and analysis of posts from Donald Trump's Truth Social profile to estimate their potential impact on the U.S. stock market. It is designed for analysts and trading teams who want to monitor influential social media communications and receive timely alerts via Slack.

The workflow consists of the following logical blocks:

- **1.1 Trigger Block**: Initiates the workflow either manually or on a weekly schedule.
- **1.2 Airtop Session & Browser Setup**: Creates a browser session and navigates to the target Truth Social page using Airtop.
- **1.3 Extraction and Analysis**: Uses an AI-powered prompt within Airtop to extract structured post data and estimate stock market impact.
- **1.4 Data Processing**: Splits the extracted posts into individual items and filters those with meaningful content and market impact.
- **1.5 Notification**: Sends relevant post summaries to a specified Slack channel.
- **1.6 Cleanup**: Terminates the Airtop browser session to release resources.
- **1.7 Documentation**: Includes extensive sticky notes describing usage, setup, and disclaimers.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger Block

- **Overview:** Starts the workflow either manually or on a weekly schedule.
- **Nodes Involved:**  
  - `When clicking ‘Execute workflow’` (Manual Trigger)  
  - `Schedule Trigger` (Weekly recurring trigger)

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or on-demand runs.  
    - Config: No parameters.  
    - Inputs: None  
    - Outputs: Starts the Airtop session creation node.  
    - Edge cases: None specific; manual trigger depends on user action.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every week.  
    - Config: Interval set to weeks (default 1 week).  
    - Inputs: None  
    - Outputs: Starts the Airtop session creation node.  
    - Edge cases: Scheduling failures or n8n downtime could delay execution.

---

#### 2.2 Airtop Session & Browser Setup

- **Overview:** Establishes an Airtop browser session configured to access Truth Social, preparing for data extraction.
- **Nodes Involved:**  
  - `Create Airtop Session`  
  - `Create Airtop Browser`  
  - `Sticky Note1` (Instruction)

- **Node Details:**

  - **Create Airtop Session**  
    - Type: Airtop node (API integration)  
    - Role: Creates a new browser session on Airtop with a specified profile to simulate a US-based browser.  
    - Config:  
      - Proxy: Integrated with country set to "US"  
      - Profile Name: Placeholder `"ENTER AIRTOP PROFILE NAME"` (requires user input)  
    - Inputs: Output from trigger nodes  
    - Outputs: Session ID and session data passed downstream  
    - Edge cases:  
      - Missing or incorrect Airtop profile name leads to session creation failure  
      - API key or credential errors  
      - Network or Airtop service downtime

  - **Create Airtop Browser**  
    - Type: Airtop node (browser window control)  
    - Role: Opens a browser tab/window to `https://truthsocial.com/@realDonaldTrump` within the Airtop session.  
    - Config:  
      - URL: Fixed to Trump's Truth Social page  
      - WaitUntil: `load` event ensures page fully loads before proceeding  
    - Inputs: Session ID from `Create Airtop Session`  
    - Outputs: Signals readiness for extraction  
    - Edge cases:  
      - Page load failures or redirects  
      - Network latency or timeouts  
      - Changes in Truth Social page structure affecting later extraction

  - **Sticky Note1**  
    - Content: Instruction to enter the Airtop profile name in the `Create Airtop Session` node.  
    - Purpose: Ensures correct profile configuration for successful session creation.

---

#### 2.3 Extraction and Analysis

- **Overview:** Uses Airtop’s extraction capability with a natural language prompt and output JSON schema to parse up to 6 recent posts and estimate their stock market impact.
- **Nodes Involved:**  
  - `Extract and Analyze Posts`

- **Node Details:**

  - **Extract and Analyze Posts**  
    - Type: Airtop extraction node  
    - Role: Sends a prompt describing the extraction and analysis task; receives structured JSON data with posts and their market impact estimations.  
    - Config:  
      - Prompt includes:  
        - Extract up to 6 posts from @realDonaldTrump on Truth Social  
        - For each post: author name, image URL, post text, post URL  
        - Estimate stock market impact direction (positive, negative, neutral) and magnitude (None, Small, Medium, Large)  
      - Output schema: Detailed JSON schema validating extracted data format  
      - JSON parsing enabled  
    - Inputs: Browser window context from `Create Airtop Browser`  
    - Outputs: JSON data with an array of posts under `output.posts`  
    - Edge cases:  
      - Changes in page layout or content could break extraction accuracy  
      - Incomplete or malformed JSON output  
      - Airtop API errors or timeouts

---

#### 2.4 Data Processing

- **Overview:** Processes the extracted array of posts by splitting them into individual items, then filtering out posts with empty text or no estimated market impact.
- **Nodes Involved:**  
  - `Split Out`  
  - `Filter`

- **Node Details:**

  - **Split Out**  
    - Type: Split Out node  
    - Role: Splits the `output.posts` array into individual workflow items for granular processing.  
    - Config: Field to split out is `output.posts`  
    - Inputs: JSON array from `Extract and Analyze Posts`  
    - Outputs: Individual posts as separate workflow items  
    - Edge cases: Empty or missing `output.posts` array results in no further processing

  - **Filter**  
    - Type: Filter node  
    - Role: Filters posts based on two conditions:  
      1. `post_text` exists and is not empty  
      2. `stock_market_impact.magnitude` is not "None"  
    - Config:  
      - Condition 1: `post_text` exists (string exists operator)  
      - Condition 2: `stock_market_impact.magnitude` not equals "None"  
      - Both combined with AND  
    - Inputs: Individual post items from `Split Out`  
    - Outputs: Only posts meeting conditions proceed  
    - Edge cases: Posts missing fields or with unexpected values will be excluded

---

#### 2.5 Notification

- **Overview:** Sends filtered posts and their estimated market impact summaries as messages to a Slack channel for team visibility.
- **Nodes Involved:**  
  - `Slack`

- **Node Details:**

  - **Slack**  
    - Type: Slack node (message sending)  
    - Role: Posts a message to a predefined Slack channel with the post text and impact summary.  
    - Config:  
      - Text template:  
        ```
        {{ $json.post_text }}

        Potential impact on the stock market: {{ $json.stock_market_impact.magnitude }}, {{ $json.stock_market_impact.direction }}
        ```  
      - Channel: Fixed to channel ID `C08E83RDJN9`  
      - Authentication: OAuth2 credential configured (`Slack account 3`)  
      - Select: Channel mode  
    - Inputs: Filtered posts from `Filter`  
    - Outputs: None  
    - Edge cases:  
      - Slack API rate limits or permission errors  
      - Invalid channel ID or revoked tokens

---

#### 2.6 Cleanup

- **Overview:** Terminates the Airtop browser session to free resources and maintain session hygiene.
- **Nodes Involved:**  
  - `Terminate Airtop Session`

- **Node Details:**

  - **Terminate Airtop Session**  
    - Type: Airtop node (session termination)  
    - Role: Closes the Airtop session using the session ID from `Create Airtop Session`.  
    - Config: Operation set to `terminate`  
    - Inputs: Session ID from `Create Airtop Session` node output  
    - Outputs: None  
    - Edge cases:  
      - Session already terminated or invalid session ID  
      - Airtop API errors or network issues during termination

---

#### 2.7 Documentation

- **Overview:** Provides comprehensive instructions, disclaimers, and background information as a sticky note for user reference.
- **Nodes Involved:**  
  - `Sticky Note`

- **Node Details:**

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Contains a detailed README-like text covering:  
      - Workflow purpose and use case  
      - Detailed explanation of automation steps  
      - Setup requirements including Airtop API key and Slack permissions  
      - Next steps for enhancement  
      - Legal disclaimer emphasizing no financial advice  
    - Inputs/Outputs: None  
    - Edge cases: None

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                         | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                                                      |
|-----------------------------|--------------------|---------------------------------------|-----------------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger     | Manual start of workflow               | None                        | Create Airtop Session     |                                                                                                                                 |
| Schedule Trigger            | Schedule Trigger   | Weekly automatic trigger               | None                        | Create Airtop Session     |                                                                                                                                 |
| Create Airtop Session       | Airtop             | Create browser session with Airtop    | When clicking ‘Execute workflow’, Schedule Trigger | Create Airtop Browser    | Sticky Note1: "## Enter Airtop Profile name here"                                                                                |
| Sticky Note1                | Sticky Note        | Instruction for Airtop profile config | None                        | None                     | "## Enter Airtop Profile name here"                                                                                            |
| Create Airtop Browser       | Airtop             | Open browser at Trump’s Truth Social  | Create Airtop Session       | Extract and Analyze Posts |                                                                                                                                 |
| Extract and Analyze Posts   | Airtop             | Extract posts and estimate impact     | Create Airtop Browser       | Split Out, Terminate Airtop Session |                                                                                                                                 |
| Split Out                  | Split Out          | Split posts array into individual items | Extract and Analyze Posts   | Filter                   |                                                                                                                                 |
| Filter                     | Filter             | Filter posts with content and impact  | Split Out                  | Slack                    |                                                                                                                                 |
| Slack                      | Slack              | Send posts and impact to Slack channel | Filter                    | None                     |                                                                                                                                 |
| Terminate Airtop Session   | Airtop             | Close Airtop browser session           | Extract and Analyze Posts   | None                     |                                                                                                                                 |
| Sticky Note                | Sticky Note        | Workflow README and disclaimers        | None                        | None                     | README with detailed workflow description, setup, and legal disclaimer                                                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Manual Trigger** node named `When clicking ‘Execute workflow’`. No parameters needed.  
   - Add a **Schedule Trigger** node named `Schedule Trigger`. Configure interval to every 1 week.

2. **Create Airtop Session Node:**  
   - Add an **Airtop** node named `Create Airtop Session`.  
   - Set Proxy to `integrated`, country to `"US"`.  
   - Enter your Airtop profile name in `profileName` (replace `"ENTER AIRTOP PROFILE NAME"`).  
   - Use your Airtop API credentials.

3. **Create Airtop Browser Node:**  
   - Add an **Airtop** node named `Create Airtop Browser`.  
   - Resource: `window`  
   - URL: `https://truthsocial.com/@realDonaldTrump`  
   - Additional fields: WaitUntil set to `"load"`  
   - Use Airtop credentials.

4. **Create Extraction Node:**  
   - Add an **Airtop** node named `Extract and Analyze Posts`.  
   - Resource: `extraction`  
   - Prompt:  
     ```
     This is Truth Social. 
     Extract up to 6 posts from @realDonaldTrump. 
     For each post, extract the name of the author, the image URL, the text and the URL.

     Also try to estimate the potential impact of each post on the public stock market in the US:
     Direction - positive, negative, or neutral
     Magnitude - None, Small, Medium or Large
     ```
   - Enable JSON parsing.  
   - Set output schema exactly as in the provided workflow to enforce structured data.  
   - Use Airtop credentials.

5. **Create Split Out Node:**  
   - Add a **Split Out** node named `Split Out`.  
   - Field to split out: `output.posts`.

6. **Create Filter Node:**  
   - Add a **Filter** node named `Filter`.  
   - Set two conditions combined with AND:  
     - `post_text` exists (string operation: exists)  
     - `stock_market_impact.magnitude` not equals `"None"`

7. **Create Slack Node:**  
   - Add a **Slack** node named `Slack`.  
   - Authentication: OAuth2 with Slack app credentials.  
   - Select channel mode, channel ID set to your target Slack channel (e.g., `C08E83RDJN9`).  
   - Text:  
     ```
     {{ $json.post_text }}

     Potential impact on the stock market: {{ $json.stock_market_impact.magnitude }}, {{ $json.stock_market_impact.direction }}
     ```

8. **Create Terminate Session Node:**  
   - Add an **Airtop** node named `Terminate Airtop Session`.  
   - Operation: `terminate`  
   - Session ID: Expression referencing `Create Airtop Session` node's `sessionId` output (e.g., `={{ $('Create Airtop Session').item.json.sessionId }}`)  
   - Use Airtop credentials.

9. **Connect Nodes:**  
   - Connect `When clicking ‘Execute workflow’` and `Schedule Trigger` both to `Create Airtop Session`.  
   - Connect `Create Airtop Session` → `Create Airtop Browser` → `Extract and Analyze Posts`.  
   - Connect `Extract and Analyze Posts` → `Split Out` → `Filter` → `Slack`.  
   - Also connect `Extract and Analyze Posts` → `Terminate Airtop Session`.

10. **Add Sticky Notes:**  
    - Add a sticky note near `Create Airtop Session` with the text:  
      `## Enter Airtop Profile name here`  
    - Add a large sticky note summarizing the README content with purpose, usage, setup, and disclaimers.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Context or Link                                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| README includes detailed use case, workflow explanation, setup instructions (Airtop API key, profile, Slack), next steps for enhancement, and a legal disclaimer emphasizing no financial advice.                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note node in workflow                                                                                      |
| Airtop API key and profile are required; profile must be connected and logged into Truth Social to enable browsing and extraction. Generate API keys at https://portal.airtop.ai/api-keys and manage profiles at https://portal.airtop.ai/browser-profiles.                                                                                                                                                                                                                                                                                                                                                                                              | Airtop official documentation and portal                                                                           |
| Slack node requires an OAuth2 app with write permissions to the target channel. Create Slack apps with appropriate scopes and authorize before use.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      | Slack API documentation                                                                                            |
| Market impact estimations are speculative and for informational purposes only. Users should not make investment decisions based on this workflow output. Always consult licensed financial advisors.                                                                                                                                                                                                                                                                                                                                                                                                                                                    | Legal disclaimer in Sticky Note                                                                                     |

---

**Disclaimer:**  
The provided text is exclusively from an automated workflow created in n8n, respecting all current content policies and containing no illegal, offensive, or protected elements. All data handled is legal and public.