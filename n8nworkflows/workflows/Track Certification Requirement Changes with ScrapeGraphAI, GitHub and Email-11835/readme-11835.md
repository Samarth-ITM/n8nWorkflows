Track Certification Requirement Changes with ScrapeGraphAI, GitHub and Email

https://n8nworkflows.xyz/workflows/track-certification-requirement-changes-with-scrapegraphai--github-and-email-11835


# Track Certification Requirement Changes with ScrapeGraphAI, GitHub and Email

### 1. Workflow Overview

This workflow automates the monitoring of certification or license requirement changes published on multiple industry-body websites. It is designed for professionals who want to track updates like renewal criteria, continuing education (CE) credits, costs, and effective dates without manually visiting each website.

The workflow is logically divided into these blocks:

- **1.1 Input & Batching:** Receives a list of certification URLs and processes them one by one to avoid rate-limiting and simplify debugging.
- **1.2 Scraping & Normalising:** Fetches live webpage data using ScrapeGraphAI, extracts structured certification details via AI-driven natural language prompts, and normalizes the data into a consistent JSON format.
- **1.3 Change Detection:** Retrieves the last saved snapshot of the certification’s data from a GitHub repository and compares it with the newly scraped data to detect any meaningful changes.
- **1.4 Store & Notify:** If changes are detected, updates the GitHub repository snapshot and sends a concise email summary of the differences to the user. If no changes are found, the workflow ends quietly.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Batching

**Overview:**  
This block initializes the workflow and serializes the list of certifications to be monitored. It uses a manual trigger for ad hoc runs (easily replaceable with a schedule trigger) and splits the certification URLs into single batches to process sequentially.

**Nodes Involved:**  
- Start Manual (Manual Trigger)  
- Define Certification URLs (Code)  
- Split In Batches (SplitInBatches)

**Node Details:**

- **Start Manual**  
  - Type: Manual Trigger  
  - Role: Starts workflow execution on demand.  
  - Configuration: Default manual trigger, no special parameters.  
  - Inputs: None  
  - Outputs: To "Define Certification URLs"  
  - Edge Cases: None  

- **Define Certification URLs**  
  - Type: Code  
  - Role: Returns an array of certification objects, each with a `url` and `name`.  
  - Configuration: Hardcoded JavaScript array of certifications. User must edit this array to add or modify URLs.  
  - Key Expression: Returns array with structure `[{ json: { url, name } }, ...]`  
  - Input: From Manual Trigger  
  - Output: To Split In Batches  
  - Edge Cases: Empty or malformed URL list will cause downstream failures.  

- **Split In Batches**  
  - Type: SplitInBatches  
  - Role: Processes each certification URL individually to prevent rate-limiting and simplify debugging.  
  - Configuration: Batch size set to 1 (default).  
  - Input: From Define Certification URLs  
  - Output: To Scrape Certification Page  
  - Edge Cases: Large URL lists will loop correctly but increase total run time.

---

#### 2.2 Scraping & Normalising

**Overview:**  
Loads each certification webpage and extracts structured data using ScrapeGraphAI, then normalizes and enriches the data for further processing.

**Nodes Involved:**  
- Scrape Certification Page (ScrapeGraphAI)  
- Format Scraped Data (Code)

**Node Details:**

- **Scrape Certification Page**  
  - Type: ScrapeGraphAI  
  - Role: Performs AI-driven scraping of certification pages, running JavaScript and returning a JSON with certification details.  
  - Configuration:  
    - `userPrompt` instructs the AI to extract fields: certification (string), requirements (string), credits (number), cost (string), effective_date (YYYY-MM-DD).  
    - `websiteUrl` dynamically set from batch item `$json.url`.  
  - Input: From Split In Batches  
  - Output: To Format Scraped Data  
  - Edge Cases:  
    - Website structure changes may cause inaccurate data extraction.  
    - API limits or connectivity issues with ScrapeGraphAI could cause errors.  
    - The prompt must be kept accurate for consistent results.  

- **Format Scraped Data**  
  - Type: Code  
  - Role: Normalizes the scraped JSON data; converts fields, handles missing values, creates a slugified filename key, and adds a `scraped_at` timestamp.  
  - Configuration:  
    - Builds `slug` from certification name or fallback.  
    - Ensures numeric conversion for credits.  
    - Adds fields with default fallbacks if missing.  
    - Returns normalized object for GitHub storage.  
  - Input: From Scrape Certification Page  
  - Output: To GitHub – Get Previous Snapshot  
  - Edge Cases:  
    - Missing or malformed scraped data could cause null fields.  
    - Slug generation may fail if certification name contains unusual characters.

---

#### 2.3 Change Detection

**Overview:**  
Fetches the last saved snapshot from GitHub, merges it with the new data, and performs a deep comparison to detect changes.

**Nodes Involved:**  
- GitHub – Get Previous Snapshot (GitHub)  
- Merge Current & Previous (Merge)  
- Detect Changes (Code)  
- Changes Detected? (IF)

**Node Details:**

- **GitHub – Get Previous Snapshot**  
  - Type: GitHub  
  - Role: Retrieves the last JSON snapshot file for the certification from the GitHub repository.  
  - Configuration:  
    - Owner, repository (`certification-requirements-tracker`), file path dynamically built as `data/{slug}.json`.  
    - Operation: get file content.  
    - Requires GitHub personal access token with repo access.  
  - Input: From Format Scraped Data  
  - Output: To Merge Current & Previous (second input)  
  - Edge Cases:  
    - File not found returns error or null (handled downstream).  
    - Auth errors if credentials missing or invalid.  

- **Merge Current & Previous**  
  - Type: Merge  
  - Role: Combines newly scraped data and previous snapshot into a single item for comparison.  
  - Configuration: Mode set to `combine` (merges input items).  
  - Inputs:  
    - Primary: new normalized data  
    - Secondary: previous GitHub snapshot  
  - Output: To Detect Changes  
  - Edge Cases: None  

- **Detect Changes**  
  - Type: Code  
  - Role: Parses previous snapshot (base64 encoded), compares it with new snapshot ignoring volatile fields, sets boolean `hasChanges`.  
  - Configuration:  
    - JSON parsing with error handling.  
    - Deep comparison using `JSON.stringify`.  
    - Outputs `hasChanges` flag for routing.  
  - Input: From Merge Current & Previous  
  - Output: To Changes Detected?  
  - Edge Cases:  
    - Malformed previous data may cause parsing errors (caught and defaults to null).  
    - False positives if volatile fields not excluded correctly.

- **Changes Detected?**  
  - Type: IF  
  - Role: Routes items based on whether meaningful changes were detected.  
  - Configuration: Checks if `hasChanges` is `true`.  
  - Input: From Detect Changes  
  - Output:  
    - True: To GitHub – Upsert Snapshot  
    - False: Ends workflow silently  
  - Edge Cases: None  

---

#### 2.4 Store & Notify

**Overview:**  
If changes are detected, this block updates the GitHub snapshot file and sends an email notification summarizing the changes.

**Nodes Involved:**  
- GitHub – Upsert Snapshot (GitHub)  
- Prepare Summary Email (Code)  
- Email Send Notification (EmailSend)

**Node Details:**

- **GitHub – Upsert Snapshot**  
  - Type: GitHub  
  - Role: Commits the new normalized snapshot JSON to the GitHub repo, overwriting or creating the file `data/{slug}.json`.  
  - Configuration:  
    - Owner, repo, and file path dynamically set.  
    - Content encoded in base64.  
    - Commit message includes certification name and timestamp.  
    - Requires GitHub personal access token with write permissions.  
  - Input: From Changes Detected? (true branch)  
  - Output: To Prepare Summary Email  
  - Edge Cases:  
    - Commit failures due to permission or network issues.  

- **Prepare Summary Email**  
  - Type: Code  
  - Role: Builds a plain-text email summarizing the changes between previous and new requirements, including credits, cost, and effective dates.  
  - Configuration:  
    - Conditional formatting if no previous record exists.  
    - Sets email subject and body fields.  
  - Input: From GitHub – Upsert Snapshot  
  - Output: To Email Send Notification  
  - Edge Cases: None  

- **Email Send Notification**  
  - Type: EmailSend  
  - Role: Sends the summary email to the configured recipient.  
  - Configuration:  
    - `toEmail` and `fromEmail` must be set by user.  
    - Subject and body dynamically set from previous node output.  
    - Requires configured email credentials (SMTP or service).  
  - Input: From Prepare Summary Email  
  - Output: Workflow ends  
  - Edge Cases:  
    - Email delivery failures if credentials or network misconfigured.

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                          | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
|-----------------------------|-------------------------|----------------------------------------|---------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Overview           | StickyNote              | Describes workflow purpose and setup   | None                            | None                            | ## How it works  This workflow lets professionals keep an eye on evolving certification or licence requirements without manually visiting every industry-body website. Start the workflow with the Manual Trigger and a Code node immediately produces a list of certification-issuer URLs. Each URL moves through a batch splitter so every page is handled one-by-one. ScrapeGraphAI then analyses the live web page and returns a clean JSON object that contains the certification name, renewal criteria, CE credit totals, cost and the current effective-date. The data is normalised in a Code node and compared with the last saved snapshot that lives in a GitHub repository. If a difference is detected—say the number of CE credits has changed—the new JSON snapshot overwrites the old file on GitHub and a concise email summary lands in your inbox. No change means no email, keeping noise to a minimum.  ## Setup steps  1. Add your ScrapeGraphAI API credential in **Credentials / ScrapeGraphAI** 2. Fill the URL list in the **Define Certification URLs** Code node 3. Add a GitHub personal-access token and set owner/repo names in both GitHub nodes 4. Configure **from** and **to** addresses in the Email Send node 5. Commit and enable the workflow, then run the Manual Trigger to test 6. (Optional) Replace the Manual Trigger with a Schedule Trigger for production use |
| Section – Input & Batching  | StickyNote              | Explains input and batching strategy   | None                            | None                            | ## Input & Batching  The first three functional nodes work together to generate and serialise the list of certification URLs that will be monitored. The **Manual Trigger** gives you instant control so you can test the workflow on demand or wire it into a Schedule Trigger later for hands-free operation. Immediately after the trigger fires, the **Define Certification URLs** Code node returns an array of JSON items—each item holds a unique `url` and a human-readable `name`. These individual items travel to the **Split In Batches** node. Setting the batch-size to 1 forces n8n to process each certification page sequentially, preventing accidental rate-limit hits on ScrapeGraphAI and making downstream debugging easier because you see one certification at a time. If you add more URLs later, n8n will automatically loop through them without additional configuration, ensuring scalability while maintaining a straightforward linear flow. |
| Section – Scraping & Normalising | StickyNote          | Describes scraping and data formatting | None                            | None                            | ## Scraping & Normalising  Once a single URL enters the Scrape stage the **ScrapeGraphAI** node takes centre stage. It loads the live web page of the certification body, executes any required JavaScript, and then—guided by a natural-language prompt—extracts structured details like renewal cycles, CE credit counts and associated fees. The AI returns data in a predictable JSON payload that feeds directly into the **Format Scraped Data** Code node. Here the workflow cleans null values, converts numbers, and generates a slugified filename that later becomes the unique key in GitHub (`data/slug.json`). The node also stamps the record with a `scraped_at` ISO timestamp for auditing. Normalising the data right away ensures downstream steps can rely on consistent field names and types, simplifying change-detection logic and keeping historical snapshots tidy inside the repository. |
| Section – Change Detection  | StickyNote              | Explains how changes are detected      | None                            | None                            | ## Change Detection  Detecting meaningful differences is the heart of the workflow. First, the **GitHub – Get Previous Snapshot** node fetches the prior JSON file that matches the slugified certification name. The content arrives Base64 encoded so the retrieval is lightweight. Both the fresh ScrapeGraphAI payload and the historic GitHub snapshot converge in a **Merge** node using the *combine* mode, effectively stitching both datasets into a single item. Inside **Detect Changes (Code)** the workflow decodes the stored Base64, converts it into a JavaScript object and performs a deep string comparison against the freshly scraped fields. The node then sets a simple boolean `hasChanges`. This flag drives the subsequent **IF** node that cleanly routes only modified certifications to the GitHub Update and email-alert path, thereby eliminating noise and keeping commit history meaningful. |
| Section – Store & Notify    | StickyNote              | Describes storing updates and notification | None                            | None                            | ## Store & Notify  When a change is confirmed, two final responsibilities kick in: persisting the new truth and alerting stakeholders. The **GitHub – Upsert Snapshot** node writes the freshly normalised JSON back to the repo, replacing or creating the corresponding file under `data/`. A concise commit message includes the certification name and timestamp, making it easy to audit the version history from GitHub’s web interface. Immediately after the commit, **Prepare Summary (Code)** crafts a plain-text diff highlighting what has changed—ideal for quick human scanning. Finally, **Email Send** delivers this summary straight to your inbox. Using email keeps the notification method universally accessible while GitHub provides a durable, version-controlled storage layer. If the IF node determines no updates occurred, execution simply ends, which means zero unnecessary commits and a silent inbox. |
| Start Manual               | ManualTrigger           | Starts workflow execution manually     | None                            | Define Certification URLs       |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Define Certification URLs | Code                    | Outputs list of certification URLs     | Start Manual                   | Split In Batches                |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Split In Batches           | SplitInBatches          | Serializes URLs for sequential processing | Define Certification URLs      | Scrape Certification Page      |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Scrape Certification Page | ScrapeGraphAI           | Extracts structured data with AI       | Split In Batches               | Format Scraped Data             |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Format Scraped Data        | Code                    | Normalizes scraped data and adds metadata | Scrape Certification Page      | GitHub – Get Previous Snapshot |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| GitHub – Get Previous Snapshot | GitHub              | Retrieves previous snapshot JSON file  | Format Scraped Data            | Merge Current & Previous        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Merge Current & Previous   | Merge                   | Combines new and old snapshots          | GitHub – Get Previous Snapshot, Format Scraped Data | Detect Changes               |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Detect Changes             | Code                    | Compares snapshots and sets change flag | Merge Current & Previous       | Changes Detected?               |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Changes Detected?          | IF                      | Routes based on change detection boolean | Detect Changes                | GitHub – Upsert Snapshot (True) |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| GitHub – Upsert Snapshot   | GitHub                  | Commits updated snapshot to GitHub repo | Changes Detected? (True)       | Prepare Summary Email          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Prepare Summary Email      | Code                    | Creates text email summarizing changes | GitHub – Upsert Snapshot       | Email Send Notification        |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| Email Send Notification    | EmailSend               | Sends notification email                | Prepare Summary Email          | None                          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Start Manual" node**  
   - Type: Manual Trigger  
   - Purpose: To manually start the workflow for testing or on-demand runs.  

2. **Create "Define Certification URLs" node**  
   - Type: Code  
   - Parameters: Use JavaScript code to return an array of certifications, each with `url` and `name`.  
   - Example code:  
     ```javascript
     return [
       { json: { url: 'https://example.com/certification-a', name: 'Certification A' } },
       { json: { url: 'https://example.com/certification-b', name: 'Certification B' } }
     ];
     ```  
   - Connect output of Manual Trigger to this node.

3. **Create "Split In Batches" node**  
   - Type: SplitInBatches  
   - Parameters: Set batch size to 1 for sequential processing.  
   - Connect input from "Define Certification URLs".  

4. **Create "Scrape Certification Page" node**  
   - Type: ScrapeGraphAI  
   - Parameters:  
     - `userPrompt`:  
       ```
       You are extracting certification renewal details for professionals. Return JSON: {"certification":"string","requirements":"string","credits":"number","cost":"string","effective_date":"YYYY-MM-DD"}. Pull only publicly available data.
       ```  
     - `websiteUrl`: Expression `={{ $json.url }}` to dynamically set URL.  
   - Connect input from "Split In Batches".  
   - Credential: Add and select your ScrapeGraphAI API credential in n8n credentials.

5. **Create "Format Scraped Data" node**  
   - Type: Code  
   - Parameters: JavaScript code to normalize data, create slug, and add timestamp:  
     ```javascript
     const item = items[0].json;
     const slugSource = item.certification || item.name || 'unknown';
     const slug = slugSource.toLowerCase().replace(/[^a-z0-9]+/g, '-').replace(/(^-|-$)/g, '');
     return [{
       json: {
         slug,
         certification: item.certification || item.name,
         requirements: item.requirements || '',
         credits: Number(item.credits || 0),
         cost: item.cost || 'N/A',
         effective_date: item.effective_date || null,
         scraped_at: new Date().toISOString()
       }
     }];
     ```  
   - Connect input from "Scrape Certification Page".

6. **Create "GitHub – Get Previous Snapshot" node**  
   - Type: GitHub  
   - Parameters:  
     - Owner: Your GitHub username  
     - Repository: `certification-requirements-tracker` (or your chosen repo name)  
     - File path: Expression `={{ 'data/' + $json.slug + '.json' }}`  
     - Resource: File  
     - Operation: Get  
   - Connect input from "Format Scraped Data".  
   - Credential: Set GitHub personal access token with repo read access.

7. **Create "Merge Current & Previous" node**  
   - Type: Merge  
   - Parameters: Mode set to `combine`.  
   - Inputs:  
     - Primary input from "Format Scraped Data"  
     - Secondary input from "GitHub – Get Previous Snapshot" (connect second input port)  

8. **Create "Detect Changes" node**  
   - Type: Code  
   - Parameters: JavaScript code for deep comparison:  
     ```javascript
     const data = items[0].json;
     let previousData = null;
     if (data.content) {
       try {
         previousData = JSON.parse(Buffer.from(data.content, 'base64').toString('utf8'));
       } catch (e) {
         previousData = null;
       }
     }
     const snapshot = {
       certification: data.certification,
       requirements: data.requirements,
       credits: data.credits,
       cost: data.cost,
       effective_date: data.effective_date
     };
     const hasChanges = !previousData || JSON.stringify(previousData) !== JSON.stringify(snapshot);
     return [{
       json: {
         ...data,
         previousData,
         snapshot,
         hasChanges
       }
     }];
     ```  
   - Connect input from "Merge Current & Previous".

9. **Create "Changes Detected?" node**  
   - Type: IF  
   - Parameters: Check boolean condition where `$json.hasChanges` equals `true`.  
   - Connect input from "Detect Changes".  

10. **Create "GitHub – Upsert Snapshot" node**  
    - Type: GitHub  
    - Parameters:  
      - Owner: Your GitHub username  
      - Repository: `certification-requirements-tracker`  
      - File path: Expression `={{ 'data/' + $json.slug + '.json' }}`  
      - Resource: File  
      - Operation: Create or update (upsert) file  
      - File content: Expression `={{ Buffer.from(JSON.stringify($json.snapshot, null, 2)).toString('base64') }}`  
      - Commit message: Expression `={{ 'Update requirements for ' + $json.certification + ' on ' + new Date().toISOString() }}`  
    - Connect input from "Changes Detected?" true branch.  
    - Credential: GitHub personal access token with write access.

11. **Create "Prepare Summary Email" node**  
    - Type: Code  
    - Parameters: JavaScript code to build email text content:  
      ```javascript
      const d = items[0].json;
      let body = `Certification: ${d.certification}\n`;
      if (d.previousData) {
        body += `Previous Requirements: ${d.previousData.requirements}\n`;
        body += `New Requirements: ${d.requirements}\n`;
        body += `Credits: ${d.previousData.credits} ➜ ${d.credits}\n`;
        body += `Cost: ${d.previousData.cost} ➜ ${d.cost}\n`;
        body += `Effective Date: ${d.previousData.effective_date} ➜ ${d.effective_date}\n`;
      } else {
        body += 'No previous record found. Initial snapshot created.';
      }
      return [{
        json: {
          emailSubject: `Certification Update – ${d.certification}`,
          emailBody: body
        }
      }];
      ```  
    - Connect input from "GitHub – Upsert Snapshot".

12. **Create "Email Send Notification" node**  
    - Type: EmailSend  
    - Parameters:  
      - To Email: Your email address (e.g. `your.email@example.com`)  
      - From Email: Sender address (e.g. `no-reply@example.com`)  
      - Subject: Expression `={{ $json.emailSubject }}`  
      - Body: Expression `={{ $json.emailBody }}` (default body field)  
    - Connect input from "Prepare Summary Email".  
    - Credential: Configure SMTP or other email credentials.

13. **Set workflow active and test**  
    - Run the manual trigger to verify operation.  
    - Confirm emails are sent only when changes occur.  

14. **Optional: Replace "Start Manual" with Schedule Trigger**  
    - Configure desired schedule for periodic monitoring runs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to minimize noise by sending email alerts only if meaningful changes are detected in certification requirements, avoiding unnecessary emails and GitHub commits. It uses GitHub as a durable version-controlled storage for snapshots and email as a universal notification channel.                                                                                                                                                                                                                                                    | Workflow Design Philosophy                                                                                                                                                           |
| For more information on ScrapeGraphAI and how to write effective prompts for structured data extraction, visit [ScrapeGraphAI homepage](https://scrapegraph.ai).                                                                                                                                                                                                                                                                                                                                                                                            | ScrapeGraphAI Official Website                                                                                                                                                       |
| GitHub personal access tokens need appropriate scopes: `repo` for full repository read/write access if private, or `public_repo` scope if the repository is public. Ensure tokens are stored securely in n8n credentials.                                                                                                                                                                                                                                                                                                                                    | GitHub Token Configuration                                                                                                                                                           |
| EmailSend node requires an SMTP or email service credential (e.g., Gmail OAuth2, SMTP server). Make sure to test email sending independently before deploying the workflow.                                                                                                                                                                                                                                                                                                                                                                                  | Email Credential Setup                                                                                                                                                               |
| The slug generation logic assumes certification names contain only alphanumeric or hyphen characters after normalization. If certification names are unusual or non-English, consider adjusting the slug logic to avoid filename issues in GitHub.                                                                                                                                                                                                                                                                                                          | Data Normalization Consideration                                                                                                                                                     |
| The workflow can be extended by adding more URLs to the "Define Certification URLs" node or replacing it with a dynamic source such as a database or API call.                                                                                                                                                                                                                                                                                                                                                                                             | Extensibility Tip                                                                                                                                                                    |

---

**Disclaimer:** The text provided arises solely from an automated workflow created using n8n, a tool for integration and automation. The processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.