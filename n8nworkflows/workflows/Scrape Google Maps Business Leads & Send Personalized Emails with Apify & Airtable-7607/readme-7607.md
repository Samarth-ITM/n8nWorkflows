Scrape Google Maps Business Leads & Send Personalized Emails with Apify & Airtable

https://n8nworkflows.xyz/workflows/scrape-google-maps-business-leads---send-personalized-emails-with-apify---airtable-7607


# Scrape Google Maps Business Leads & Send Personalized Emails with Apify & Airtable

### 1. Workflow Overview

This workflow automates the process of scraping business leads from Google Maps using Apify, organizing the data in Airtable, and sending personalized outreach emails via Gmail. It is designed for users who want to efficiently gather contact information for training centers (or similar businesses) and initiate automated, customized email campaigns.

The workflow is logically divided into six blocks:

- **1.1 Trigger and Start**: Manual or alternative trigger to launch the workflow.
- **1.2 Google Maps Scraping via Apify**: Sending a request to Apify’s Google Maps scraper actor.
- **1.3 Wait for Scraper Completion**: Introducing a delay to allow Apify to complete data scraping.
- **1.4 Data Cleaning and Mapping**: Restructuring raw scraped data into a clean schema.
- **1.5 Airtable Record Creation**: Storing each lead as a record in Airtable.
- **1.6 Personalized Email Sending**: Sending customized outreach emails to each lead via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Start

- **Overview:**  
  Initiates the workflow manually. This block allows the user to start the scraping and outreach process on demand.

- **Nodes Involved:**  
  - GO (Manual Trigger)

- **Node Details:**

  - **GO**  
    - Type: Manual Trigger  
    - Role: Starts the workflow when the user clicks manually.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None (start node).  
    - Outputs: Triggers the “APIFY SCRAPT GOOGLE MAPS” node.  
    - Edge Cases: None typical; manual start means user control over execution timing.

---

#### 1.2 Google Maps Scraping via Apify

- **Overview:**  
  Sends a POST request to Apify’s Google Maps Email Leads Fast Scraper actor to retrieve business leads matching a specified query.

- **Nodes Involved:**  
  - APIFY SCRAPT GOOGLE MAPS

- **Node Details:**

  - **APIFY SCRAPT GOOGLE MAPS**  
    - Type: HTTP Request  
    - Role: Calls Apify API to scrape Google Maps data.  
    - Configuration:  
      - Method: POST  
      - URL: Placeholder “YOUR URL” which should be replaced with the actual Apify API endpoint URL including the API token.  
      - Body (JSON): Contains parameters such as area height and width, whether to get emails only, the Google Maps URL to search, max results, and search query terms.  
    - Key Expressions: None beyond static JSON body with embedded parameters.  
    - Inputs: Triggered by GO node.  
    - Outputs: Sends results to the “Wait” node.  
    - Edge Cases:  
      - Invalid or expired Apify token or URL leads to auth or 404 errors.  
      - API rate limits or network timeouts.  
      - Empty or malformed responses if search query is invalid or no results found.  
    - Notes: The URL must be obtained from Apify’s console as described in Sticky Note2.

---

#### 1.3 Wait for Scraper Completion

- **Overview:**  
  Waits for a fixed delay to ensure Apify has completed scraping before data is processed further.

- **Nodes Involved:**  
  - Wait

- **Node Details:**

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution for a duration (default 10 seconds).  
    - Configuration: Default wait with no custom duration set, but recommended to adjust if scraping takes longer.  
    - Inputs: Receives output from “APIFY SCRAPT GOOGLE MAPS”.  
    - Outputs: Passes flow to “CLEAN DATA MAPPING”.  
    - Edge Cases:  
      - Insufficient wait time may cause incomplete data downstream.  
      - Excessive wait time delays workflow unnecessarily.  
    - No authentication needed.

---

#### 1.4 Data Cleaning and Mapping

- **Overview:**  
  Reformats and cleans the raw JSON data from Apify into a structured format matching Airtable columns for easy storage and use.

- **Nodes Involved:**  
  - CLEAN DATA MAPPING

- **Node Details:**

  - **CLEAN DATA MAPPING**  
    - Type: Set  
    - Role: Maps raw scraped fields into a clean, standardized schema with meaningful keys.  
    - Configuration:  
      - Assigns fields such as Société (Company), Mail (Email), Téléphone (Phone), Site Web (Website), Linkedin, Facebook, Ville (City), Catégorie centre formation (Category), Notes google maps (Reviews + rating), Lien Google MAP (Maps URL).  
      - Uses expressions like `={{ $json.name }}`, `={{ $json.email }}`, etc. to map each field from the input JSON to a new key.  
    - Inputs: Receives raw data after “Wait”.  
    - Outputs: Passes cleaned data to “AIRTABLE CREATE RECORD”.  
    - Edge Cases:  
      - Missing or null fields in raw data may result in blank outputs. Expressions should handle undefined gracefully.  
      - Data type mismatches (e.g., numbers vs. strings) are managed by forced string conversion.  
    - No external credentials required.

---

#### 1.5 Airtable Record Creation

- **Overview:**  
  Creates new records in Airtable for each cleaned lead, storing all relevant contact and business information.

- **Nodes Involved:**  
  - AIRTABLE CREATE RECORD

- **Node Details:**

  - **AIRTABLE CREATE RECORD**  
    - Type: Airtable  
    - Role: Inserts cleaned lead data into a specific Airtable base and table.  
    - Configuration:  
      - Base ID: `appA6eMHOoquiTCeO` (example base)  
      - Table ID: `tblZFszM5ubwwSYDK` (example table)  
      - Operation: Create Record  
      - Columns manually mapped to cleaned data fields, e.g., Mail, Ville, Facebook, Linkedin, Site Web, Société, Téléphone, Lien Google Maps, Notes google maps, Catégorie centre formation.  
    - Credentials: Airtable Personal Access Token with access to the base.  
    - Inputs: Receives data from “CLEAN DATA MAPPING”.  
    - Outputs: Sends each created record to “SEND EMAIL AUTOMATIC”.  
    - Edge Cases:  
      - Auth errors if token is invalid or lacks access.  
      - Rate limits or request errors from Airtable API.  
      - Data mismatch if Airtable fields are misconfigured or missing.  
    - Notes: Base and Table IDs can be found from Airtable URL; token creation instructions in Sticky Note5.

---

#### 1.6 Personalized Email Sending

- **Overview:**  
  Sends a personalized HTML email to each lead using Gmail with details from the Airtable record.

- **Nodes Involved:**  
  - SEND EMAIL AUTOMATIC

- **Node Details:**

  - **SEND EMAIL AUTOMATIC**  
    - Type: Gmail  
    - Role: Sends outreach emails to each business lead with customized content.  
    - Configuration:  
      - To: dynamically set to lead’s email `={{ $json.fields.Mail }}`  
      - Subject: lead’s company name `={{ $json.fields['Société'] }}`  
      - Message: Rich HTML email including company name, city, website link, Google Maps reviews, and a Calendly booking link.  
    - Credentials: Gmail OAuth2 account authorized to send emails.  
    - Inputs: Receives records created in Airtable node.  
    - Outputs: None (end node).  
    - Edge Cases:  
      - Invalid or missing email addresses cause send failures.  
      - Gmail API limits or credential errors can block sending.  
      - HTML rendering issues if some fields are empty or malformed.  
    - Notes: The email content is fully customizable with embedded expressions.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                      | Input Node(s)              | Output Node(s)             | Sticky Note                                                                                                   |
|-------------------------|---------------------|------------------------------------|----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| GO                      | Manual Trigger      | Start workflow manually             | None                       | APIFY SCRAPT GOOGLE MAPS    | Step 1 – GO Trigger: Manual trigger to start workflow; can be replaced by webhook or cron trigger.          |
| APIFY SCRAPT GOOGLE MAPS | HTTP Request       | Scrape Google Maps via Apify API    | GO                         | Wait                       | Step 2 – Scrape Google Maps: POST request to Apify API using JSON body with search parameters.               |
| Wait                    | Wait                | Delay to ensure scraper completion  | APIFY SCRAPT GOOGLE MAPS    | CLEAN DATA MAPPING          | Step 3 – Wait: Recommended 10 seconds delay to let Apify finish processing data.                            |
| CLEAN DATA MAPPING       | Set                 | Clean and map raw data fields       | Wait                       | AIRTABLE CREATE RECORD      | Step 4 – Mapping: Map raw JSON fields to structured keys matching Airtable columns.                         |
| AIRTABLE CREATE RECORD   | Airtable            | Create records in Airtable          | CLEAN DATA MAPPING          | SEND EMAIL AUTOMATIC        | Step 5 – Airtable Storage: Insert structured leads into Airtable base/table; requires Airtable PAT token.    |
| SEND EMAIL AUTOMATIC     | Gmail               | Send personalized outreach emails   | AIRTABLE CREATE RECORD      | None                       | Step 6 – Automatic Email: Send HTML email with lead's details; uses Gmail OAuth2 credentials.                |
| Sticky Note              | Sticky Note         | Visual notes/guidance                | None                       | None                       | ![gif](https://i.gifer.com/4aG.gif)                                                                         |
| Sticky Note1             | Sticky Note         | Explanation of Step 1 (Trigger)     | None                       | None                       | Step 1 – GO Trigger: Manual trigger details.                                                                 |
| Sticky Note2             | Sticky Note         | Explanation of Step 2 (Apify Scrape)| None                       | None                       | Step 2 – Scrape Google Maps: Instructions to get Apify URL and example JSON body.                            |
| Sticky Note3             | Sticky Note         | Explanation of Step 3 (Wait)         | None                       | None                       | Step 3 – Wait: Purpose and recommended delay.                                                                |
| Sticky Note4             | Sticky Note         | Explanation of Step 4 (Mapping)      | None                       | None                       | Step 4 – Mapping: Field assignments example and purpose.                                                    |
| Sticky Note5             | Sticky Note         | Explanation of Step 5 (Airtable)     | None                       | None                       | Step 5 – Airtable Storage: Credentials, Base/Table IDs, and mapping instructions. See https://airtable.com/create/tokens |
| Sticky Note6             | Sticky Note         | Explanation of Step 6 (Email)        | None                       | None                       | Step 6 – Automatic Email: Email content and personalization details.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Add a "Manual Trigger" node named `GO`.
   - No special configuration needed.
   - This will start the workflow manually.

2. **Add HTTP Request Node for Apify Scraping**
   - Create an "HTTP Request" node named `APIFY SCRAPT GOOGLE MAPS`.
   - Set HTTP Method to `POST`.
   - Enter the Apify API endpoint URL for the Google Maps Email Leads Fast Scraper actor (include your Apify API token).
     - To get this URL:  
       1. Visit [Apify Google Maps Email Leads Fast Scraper](https://console.apify.com/actors/j66N0LgqJT3a7fSzu/input)  
       2. Click API → API Endpoints → Copy the “Run Actor synchronously and get dataset items” URL.  
   - Set "Body Content Type" to JSON.
   - Use the following JSON as the request body, adjusting parameters as needed:
     ```json
     {
       "area_height": 10,
       "area_width": 10,
       "emails_only": true,
       "gmaps_url": "https://www.google.com/maps/search/centres+de+formation+à+proximité+de+Amiens/",
       "max_results": 200,
       "search_query": "centre de formation"
     }
     ```
   - Connect the `GO` node output to this node.

3. **Add Wait Node**
   - Add a "Wait" node named `Wait`.
   - Set wait time to approximately 10 seconds (adjust if scraping takes longer).
   - Connect `APIFY SCRAPT GOOGLE MAPS` output to `Wait`.

4. **Add Set Node for Data Cleaning and Mapping**
   - Add a "Set" node named `CLEAN DATA MAPPING`.
   - Disable "Keep only set" (default).
   - Add fields with the following assignments (expression mode):
     - Société = `{{$json.name}}`
     - Mail = `{{$json.email}}`
     - Téléphone = `{{$json.phone_number}}`
     - Site Web = `{{$json.website_url}}`
     - Linkedin = `{{$json.linkedin}}`
     - Facebook = `{{$json.facebook}}`
     - Ville = `{{$json.city}}`
     - Catégorie centre formation = `{{$json.google_business_categories}}`
     - Notes google maps = `{{$json.reviews_number}} avis avec une note de {{$json.review_score}} / 5`
     - Lien Google MAP  = `{{$json.google_maps_url}}`
   - Connect `Wait` node output to this node.

5. **Add Airtable Node to Create Records**
   - Add an "Airtable" node named `AIRTABLE CREATE RECORD`.
   - Set Operation to "Create Record".
   - Select your Airtable base (Base ID) and table (Table ID).
     - You can find these IDs from the Airtable URL, e.g., `https://airtable.com/appXXXXXX/tblXXXXXX`.
   - Authenticate with your Airtable Personal Access Token credential.
     - Create token at https://airtable.com/create/tokens with access to the base.
   - Map each Airtable column to corresponding fields from the Set node:
     - Mail → `{{$json.Mail}}`
     - Ville → `{{$json.Ville}}`
     - Facebook → `{{$json.Facebook}}`
     - Linkedin → `{{$json.Linkedin}}`
     - Site Web → `{{$json['Site Web']}}`
     - Société → `{{$json.Société}}`
     - Téléphone → `{{$json['Téléphone']}}`
     - Lien Google Maps → `{{$json['Lien Google MAP ']}}`
     - Notes google maps → `{{$json['Notes google maps']}}`
     - Catégorie centre formation → `{{$json['Catégorie centre formation']}}`
   - Connect `CLEAN DATA MAPPING` output to this node.

6. **Add Gmail Node to Send Personalized Emails**
   - Add a "Gmail" node named `SEND EMAIL AUTOMATIC`.
   - Authenticate using Gmail OAuth2 credentials authorized to send emails.
   - Set "To" field to `={{ $json.fields.Mail }}`.
   - Set "Subject" field to `={{ $json.fields['Société'] }}`.
   - Set "Message" to the following HTML content (adjust URLs and text as needed):
     ```html
     <div style="font-family:Arial,Helvetica,sans-serif;font-size:15px;line-height:1.6;color:#111;max-width:620px;margin:0 auto;padding:18px">
       <p>Hello {{ $json.fields['Société'] }} team,</p>
       <p>I design custom automations for training centers. Goal: <strong>zero repetitive manual tasks</strong>, from registration to invoicing.</p>
       <ul style="padding-left:18px;margin:10px 0">
         <li>Registrations → Airtable + automatic file generation (Drive/PDF).</li>
         <li>Invitations & reminders → scheduled email/SMS.</li>
         <li>Attendance (e.g. EduSign) → automatic export & archiving.</li>
         <li>Invoicing & OPCO files → generation + storage in the right folders.</li>
         <li>Weekly reporting → KPIs sent by email/Slack.</li>
       </ul>
       <p style="margin-top:12px">
         <em>Info:</em> {{ $json.fields['Société'] }} in {{ $json.fields.Ville }} — website: <a href="{{ $json.fields['Site Web'] }}" style="color:#0b5fff">{{ $json.fields['Site Web'] }}</a> {{ $json.fields['Notes google maps'] ? ('— ' + $json.fields['Notes google maps']) : '' }}
       </p>
       <p>Would you be open to a quick <strong>15-min call</strong> (video) where I show you a mini-workflow adapted to {{ $json.fields['Société'] }}?</p>
       <p style="margin:16px 0">
         <a href="https://calendly.com/ton-lien/15min" style="background:#0b5fff;color:#fff;text-decoration:none;padding:10px 14px;border-radius:8px;display:inline-block">Book a slot</a>
       </p>
       <p>Talk soon,<br/>Baptiste Fort</p>
     </div>
     ```
   - Connect `AIRTABLE CREATE RECORD` output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| To obtain the Apify API endpoint URL, visit Apify's Google Maps Email Leads Fast Scraper page, click API, then API Endpoints, and copy the synchronous run URL which includes your token.                                           | https://console.apify.com/actors/j66N0LgqJT3a7fSzu/input                                           |
| Airtable Personal Access Tokens must be created with access to the correct base. You can create and manage tokens here.                                                                                                          | https://airtable.com/create/tokens                                                                |
| The email template includes a Calendly booking link; replace the placeholder URL with your actual Calendly link to enable lead scheduling.                                                                                      | https://calendly.com/ton-lien/15min (replace with your link)                                       |
| The workflow uses Gmail OAuth2 credentials; ensure the Gmail account has appropriate API access and sending limits to avoid failures.                                                                                            | Gmail API documentation                                                                            |
| The workflow's modular design allows replacing the trigger node with a webhook or cron for automation instead of manual starts.                                                                                                | n8n trigger node documentation                                                                    |
| This automation is ideal for lead generation targeting training centers or similar businesses with Google Maps presence and email data available via Apify.                                                                      | General use case                                                                                   |

---

**Disclaimer:** The text provided here comes exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly complies with applicable content policies and contains no illegal, offensive, or protected content. All processed data is legal and publicly accessible.