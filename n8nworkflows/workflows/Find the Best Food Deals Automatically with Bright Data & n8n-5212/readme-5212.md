Find the Best Food Deals Automatically with Bright Data & n8n

https://n8nworkflows.xyz/workflows/find-the-best-food-deals-automatically-with-bright-data---n8n-5212


# Find the Best Food Deals Automatically with Bright Data & n8n

### 1. Workflow Overview

This workflow, titled **"Find the Best Food Deals Automatically with Bright Data & n8n"**, is designed to automatically scrape Uber Eats for pizza deals, extract the relevant restaurant names and discount offers, and then email the best deals to a specified recipient. It leverages Bright Data’s Web Unlocker API to bypass JavaScript-heavy content and anti-bot protections on Uber Eats. The workflow is modular and organized into three logical blocks:

- **1.1 Trigger & Fetch Deals Page:** Initiates the workflow manually or on schedule and fetches the Uber Eats search results page using Bright Data API.
- **1.2 Extract Names & Deals:** Parses the raw HTML content to extract pizza restaurant names and their associated deals using CSS selectors.
- **1.3 Send Best Deals via Email:** Formats and sends an email with the extracted pizza deals using Gmail integration.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Fetch Deals Page

- **Overview:**  
  This block triggers the workflow execution and retrieves the Uber Eats pizza deals page HTML through Bright Data’s proxy API, overcoming geo-restrictions and bot-blocking mechanisms.

- **Nodes Involved:**  
  - Trigger Workflow  
  - Fetch Uber Eats Search Page (Bright Data)

- **Node Details:**

  **Trigger Workflow**  
  - Type: Manual Trigger  
  - Role: Starts the workflow either on demand or can be configured for scheduling.  
  - Configuration: Default manual trigger without parameters; can be replaced or supplemented by a schedule trigger.  
  - Inputs: None  
  - Outputs: Connects to the HTTP Request node  
  - Edge Cases: No input dependencies; workflow will not start without manual trigger or schedule configuration.

  **Fetch Uber Eats Search Page (Bright Data)**  
  - Type: HTTP Request  
  - Role: Sends a POST request to Bright Data’s Web Unlocker API to retrieve the full HTML of the Uber Eats pizza delivery page.  
  - Configuration:  
    - URL: `https://api.brightdata.com/request`  
    - Method: POST  
    - Body Parameters:  
      - `zone`: `"n8n_unblocker"` (Bright Data zone for unblocking)  
      - `url`: Uber Eats pizza delivery URL (URL encoded and includes location and query parameters)  
      - `country`: `"us"` (geolocation parameter)  
      - `format`: `"raw"` (request raw HTML response)  
    - Headers: Authorization with Bearer token set to `"YOUR_API_KEY"` (user must replace with their Bright Data API key)  
  - Inputs: Trigger Workflow output  
  - Outputs: HTML content forwarded to the extraction node  
  - Edge Cases:  
    - Authentication failure (invalid or missing API key)  
    - Bright Data service downtime or rate limiting  
    - URL format errors or page structure changes  
  - Version Requirements: HTTP Request node version 4.2 or higher recommended for full parameter support

---

#### 1.2 Extract Names & Deals

- **Overview:**  
  This block parses the raw HTML to extract pizza restaurant names and their respective deals using CSS selectors, converting unstructured HTML into structured JSON data.

- **Nodes Involved:**  
  - Extract Names & Deals

- **Node Details:**

  **Extract Names & Deals**  
  - Type: HTML Extract  
  - Role: Parses HTML to extract arrays of restaurant names and deals.  
  - Configuration:  
    - Operation: Extract HTML content  
    - Extraction Values:  
      - Key: `deak` (likely a typo for "deal" or "deak" used consistently)  
        - CSS Selector: `div.be.f3.bg.f2.ck.bp.bn.k2` (targets restaurant names)  
        - Return Array: true (extract multiple matches)  
      - Key: `Deal`  
        - CSS Selector: `div.hr.bn.k2` (targets deal descriptions)  
        - Return Array: true  
  - Inputs: Raw HTML from Bright Data node  
  - Outputs: JSON with arrays of names and deals passed to email node  
  - Edge Cases:  
    - Page structure changes breaking selectors  
    - No matches found (empty arrays)  
    - Inconsistent data alignment between names and deals arrays  
  - Version Requirements: HTML Extract node version 1.2 or newer for array extraction support

---

#### 1.3 Send Best Deals via Email

- **Overview:**  
  This block formats the extracted pizza deals into an email message and sends it to the recipient using Gmail’s OAuth2 credentials.

- **Nodes Involved:**  
  - Send Pizza Deals via Gmail

- **Node Details:**

  **Send Pizza Deals via Gmail**  
  - Type: Gmail  
  - Role: Sends an email containing the pizza deals information to a specified recipient.  
  - Configuration:  
    - Send To: `shahkar.genai@gmail.com` (modifiable recipient email)  
    - Subject: `"Best Pizza Deals Near You Today!"` (template expression allowed)  
    - Message: Text format with template expressions:  
      ```
      Hey there! 👋

      Here are the best pizza deals we found near your location:

      🍱 Title: {{ $json.deak[0] }}
      🎉 Deal: {{ $json.Deal[0] }}

      We’ve scanned the menu data for you using our smart scraper automation.
      ```  
    - Email Type: Text (non-HTML)  
    - Credentials: Gmail OAuth2 credentials configured under `gmailOAuth2`  
  - Inputs: JSON data from Extract Names & Deals node  
  - Outputs: None (final node)  
  - Edge Cases:  
    - OAuth token expiration or invalid credentials  
    - Missing data in input JSON (empty deals)  
    - Email sending limits by Gmail  
  - Version Requirements: Gmail node version 2.1 or above for OAuth2 and template support

---

### 3. Summary Table

| Node Name                                | Node Type          | Functional Role                         | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                   |
|-----------------------------------------|--------------------|---------------------------------------|---------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------|
| Trigger Workflow                        | Manual Trigger     | Starts workflow execution              | None                            | Fetch Uber Eats Search Page (Bright Data) |                                                                                               |
| Fetch Uber Eats Search Page (Bright Data) | HTTP Request       | Retrieves Uber Eats pizza deals page HTML | Trigger Workflow                | Extract Names & Deals            | See Sticky Note: Section 1 explanation of trigger and fetch steps                              |
| Extract Names & Deals                   | HTML Extract       | Parses HTML to extract restaurant names and deals | Fetch Uber Eats Search Page (Bright Data) | Send Pizza Deals via Gmail       | See Sticky Note: Section 2 explanation of extraction with CSS selectors                        |
| Send Pizza Deals via Gmail              | Gmail              | Sends email with best pizza deals      | Extract Names & Deals           | None                            | See Sticky Note: Section 3 explanation of email formatting and sending                        |
| Sticky Note9                           | Sticky Note        | Workflow assistance and contacts info  | None                            | None                            | Workflow assistance and contact details with YouTube and LinkedIn links                       |
| Sticky Note3                           | Sticky Note        | Full workflow breakdown and explanation | None                            | None                            | Comprehensive workflow breakdown with sections and explanations                               |
| Sticky Note                           | Sticky Note        | Section 1 detailed explanation          | None                            | None                            | Details on Trigger & Fetch block                                                              |
| Sticky Note1                           | Sticky Note        | Section 2 detailed explanation          | None                            | None                            | Details on Extract Names & Deals block                                                        |
| Sticky Note2                           | Sticky Note        | Section 3 detailed explanation          | None                            | None                            | Details on Send Email block                                                                   |
| Sticky Note4                           | Sticky Note        | Affiliate link disclosure               | None                            | None                            | Bright Data affiliate link for support                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a **Manual Trigger** node named `"Trigger Workflow"`.  
   - No parameters needed; optionally configure a schedule trigger if periodic runs are desired.

2. **Add HTTP Request Node for Bright Data:**  
   - Add an **HTTP Request** node named `"Fetch Uber Eats Search Page (Bright Data)"`.  
   - Set Method to `POST`.  
   - URL: `https://api.brightdata.com/request`  
   - In **Body Parameters** (set as JSON or parameters array):  
     - `zone`: `"n8n_unblocker"`  
     - `url`: Uber Eats pizza delivery URL encoded properly, for example:  
       `https://www.ubereats.com/feed?diningMode=DELIVERY&pl=...&scq=Pizza` (use your target location and query parameters)  
     - `country`: `"us"`  
     - `format`: `"raw"`  
   - In **Headers**, add:  
     - `Authorization`: `Bearer YOUR_API_KEY` (replace `YOUR_API_KEY` with your Bright Data API key)  
   - Connect the output of the Trigger Workflow node to this node’s input.

3. **Add HTML Extract Node:**  
   - Add an **HTML Extract** node named `"Extract Names & Deals"`.  
   - Set Operation to `extractHtmlContent`.  
   - Configure extraction values:  
     - Key: `deak`  
       - CSS Selector: `div.be.f3.bg.f2.ck.bp.bn.k2`  
       - Return Array: true  
     - Key: `Deal`  
       - CSS Selector: `div.hr.bn.k2`  
       - Return Array: true  
   - Connect the output of the Bright Data HTTP Request node to this node.

4. **Add Gmail Node for Sending Email:**  
   - Add a **Gmail** node named `"Send Pizza Deals via Gmail"`.  
   - Set recipient email in **Send To** field, e.g., `shahkar.genai@gmail.com` (change as needed).  
   - Email subject: `"Best Pizza Deals Near You Today!"` (can be a static string or expression).  
   - Email message body (Text format):  
     ```
     Hey there! 👋

     Here are the best pizza deals we found near your location:

     🍱 Title: {{ $json.deak[0] }}
     🎉 Deal: {{ $json.Deal[0] }}

     We’ve scanned the menu data for you using our smart scraper automation.
     ```  
   - Select **Email Type** as Text.  
   - Configure Gmail OAuth2 credentials by adding your Google account OAuth credentials in n8n credentials manager and selecting them here.  
   - Connect the output of the HTML Extract node to this Gmail node.

5. **Optional - Add Sticky Notes for Documentation:**  
   - Add sticky notes to document each section and overall workflow assistance as per the original workflow for easier maintenance.

6. **Test the Workflow:**  
   - Execute manually to verify that the Bright Data API returns the HTML.  
   - Confirm the HTML Extract node successfully parses restaurant names and deals.  
   - Verify the Gmail node sends the email with the correct content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                            | Context or Link                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| For any questions or support, contact: Yaron@nofluff.online. Explore more tips and tutorials on YouTube: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                   | Workflow Assistance and Author Contact              |
| Bright Data affiliate link: https://get.brightdata.com/1tndi4600b25 - using this helps support free content creation.                                                                                                                                   | Affiliate Disclosure & Support Link                  |
| The workflow uses Bright Data’s Web Unlocker API to bypass JavaScript rendering and bot protection on Uber Eats, which is critical for reliable scraping.                                                                                                | Technical Note on Web Unlocker API                   |
| CSS selectors used are specific and may break if Uber Eats updates their site layout; users should verify selectors periodically and update accordingly.                                                                                                  | Maintenance Note on CSS Selectors                    |
| OAuth2 credentials for Gmail must be properly set up with appropriate scopes (send email) to enable email sending without errors.                                                                                                                      | Credential Setup Note                                |
| This workflow can be adapted to other food types, locations, or platforms (e.g., DoorDash, Grubhub) by changing the API URL and CSS selectors accordingly.                                                                                                | Adaptation Advice                                    |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.