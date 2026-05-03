Extract Facebook Group Posts with Airtop

https://n8nworkflows.xyz/workflows/extract-facebook-group-posts-with-airtop-5138


# Extract Facebook Group Posts with Airtop

### 1. Workflow Overview

This workflow, titled **Facebook Group Scraper**, automates the extraction of non-sponsored posts from a specified Facebook Group feed using the Airtop browser automation platform integrated within n8n. It targets community managers, marketers, and researchers who need structured insights and engagement metrics from Facebook Groups.

The workflow consists of two main logical blocks:

- **1.1 Input Reception:** Captures user input via a form trigger collecting the Facebook Group URL and Airtop Profile.
- **1.2 Data Extraction and Processing:** Uses Airtop to initiate a browser session, navigate to the Facebook Group, and extract detailed post data structured in a predefined JSON schema.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block collects necessary parameters from the user through an interactive form to drive the scraping process.

**Nodes Involved:**  
- On form submission

**Node Details:**  

- **On form submission**  
  - **Type:** Form Trigger  
  - **Role:** Entry point; receives user input via an HTTP form submission (webhook).  
  - **Configuration:**  
    - Form title: "Facebook Group Scraper"  
    - Two required fields:  
      - *Facebook Group URL*: URL of the Facebook Group feed to scrape.  
      - *Airtop Profile*: Airtop browser profile authenticated to Facebook.  
    - Form description explains the automation purpose.  
  - **Expressions/Variables:** None (static form fields).  
  - **Input Connections:** None (trigger node).  
  - **Output Connections:** Connected to the "Airtop" node.  
  - **Version Requirements:** Uses webhook and form trigger functionality available in n8n v2.2+ for formTrigger node.  
  - **Potential Failures:**  
    - Missing or invalid input (form fields are required, but user may submit incorrect URLs or invalid profile names).  
    - Webhook connectivity errors or delays.  
  - **Sub-workflow:** None.

---

#### 1.2 Data Extraction and Processing

**Overview:**  
This block interacts with Airtop, a browser automation tool, to open a browser session using the supplied Airtop Profile, navigate to the Facebook Group URL, and extract up to 5 non-sponsored posts with detailed attributes returned as structured JSON.

**Nodes Involved:**  
- Airtop

**Node Details:**  

- **Airtop**  
  - **Type:** Airtop node (browser automation/extraction)  
  - **Role:** Executes browser automation to scrape Facebook Group posts.  
  - **Configuration:**  
    - URL to navigate: bound dynamically to the "Facebook Group URL" from form input.  
    - Prompt instructs Airtop to extract up to 5 non-sponsored posts including post text, URLs, timestamps, engagement metrics (likes, shares, comments), profile details, and thumbnails.  
    - Uses the "Airtop Profile" to launch an authenticated browser session.  
    - Session mode: "new" (starts a fresh browser session per request).  
    - Output schema: Strict JSON schema defining expected fields for posts array, including types and required properties, ensuring structured, predictable output.  
    - Pagination mode: "infinite-scroll" to handle dynamic loading if necessary.  
    - JSON output parsing enabled.  
  - **Expressions/Variables:**  
    - URL: `={{ $json['Facebook Group URL'] }}`  
    - ProfileName: `={{ $json['Airtop Profile'] }}`  
  - **Input Connections:** Receives data from "On form submission" node.  
  - **Output Connections:** None (end of chain).  
  - **Version Requirements:** Requires Airtop integration available in n8n; API credentials configured.  
  - **Potential Failures:**  
    - Authentication errors if Airtop profile is invalid or expired.  
    - Network or timeout issues while loading Facebook Group page.  
    - Changes in Facebook page structure that break extraction prompt logic.  
    - API rate limits or quota exhaustion on Airtop.  
    - JSON parsing errors if output schema is violated.  
  - **Sub-workflow:** None.

---

#### Additional Node - Documentation Sticky Note

- **Sticky Note**  
  - **Type:** Documentation/Annotation  
  - **Role:** Provides comprehensive README-like documentation embedded in the workflow editor for user reference.  
  - **Content Highlights:**  
    - Explains use case, what the automation does, setup requirements, and next steps for integration or extension.  
    - Contains links to Airtop profiles and API key resources.  
  - **Position:** Detached from main execution flow, purely informational.  
  - **Potential Issues:** None; purely static content.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role                 | Input Node(s)       | Output Node(s) | Sticky Note                                                                                                             |
|---------------------|---------------------|--------------------------------|---------------------|----------------|-------------------------------------------------------------------------------------------------------------------------|
| On form submission  | Form Trigger        | Collects Facebook URL and Airtop Profile | None                | Airtop         |                                                                                                                         |
| Airtop              | Airtop Automation   | Extracts Facebook Group posts using Airtop | On form submission  | None           |                                                                                                                         |
| Sticky Note         | Sticky Note         | Documentation and usage notes  | None                | None           | README: Explains use case, automation details, setup requirements, and next steps. Links: https://portal.airtop.ai/    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Node Type: Form Trigger  
   - Configure:  
     - Form Title: "Facebook Group Scraper"  
     - Add two required fields:  
       - "Facebook Group URL" (string, placeholder: "The URL of the Facebook Group feed you want to scrape")  
       - "Airtop Profile" (string, placeholder: "Airtop Profile authenticated to Facebook")  
     - Form Description: "This automation will extract information from a Facebook group"  
   - This node acts as a webhook and starts the workflow upon form submission.

2. **Create an Airtop Node**  
   - Node Type: Airtop (requires Airtop API credentials setup)  
   - Set credentials: Link to your Airtop API key credential.  
   - Parameters:  
     - URL: Set expression to `{{$json["Facebook Group URL"]}}` to use input from form.  
     - Prompt: Paste the following multi-line prompt:  
       ```
       This is a Facebook group feed. 
       Extract up to 5 non-sponsored posts, for each post extract:
        - Post text
        - Post URL
        - Page/profile URL
        - Timestamp	
        - Number of likes 
        - Number if shares
        - Number of comments
        - Page or profile details	
        - Post thumbnail
       ```  
     - Profile Name: Expression `{{$json["Airtop Profile"]}}`  
     - Session Mode: "new"  
     - Additional fields:  
       - Output Schema: Paste the JSON schema defining the posts array with all required properties and types as shown in the original workflow.  
       - Pagination Mode: "infinite-scroll"  
       - Parse JSON Output: Enabled (true)  
   - Connect this node's input to the output of the Form Trigger node.

3. **(Optional) Add a Sticky Note for Documentation**  
   - Type: Sticky Note  
   - Paste the README content explaining the use case, setup, and operation details.  
   - Position it separately for clarity; no connections needed.

4. **Credential Setup**  
   - Configure Airtop API credentials in n8n with your Airtop API key.  
   - Ensure Airtop Profile is created and logged into Facebook via the Airtop portal.

5. **Activate and Test**  
   - Activate the workflow.  
   - Submit the form with a valid Facebook Group URL and Airtop Profile.  
   - Verify that data for up to 5 posts is returned with all specified attributes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                    |
|-----------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Airtop API key generation and profile setup is required to run browser automation.                                                | https://portal.airtop.ai/api-keys                  |
| Airtop Browser Profiles must be authenticated with Facebook for the scraper to work.                                              | https://portal.airtop.ai/browser-profiles          |
| n8n instance must have Airtop integration enabled to use the Airtop node.                                                         | https://n8n.io/                                    |
| This automation is ideal for extracting community insights and engagement metrics from Facebook Groups for marketing or research.| Workflow use case                                  |
| Consider extending the workflow by connecting output to analytics or notification systems.                                         | Workflow next steps                                |

---

**Disclaimer:**  
The provided text originates exclusively from an automated n8n workflow created with n8n integration and automation tools. This process strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.