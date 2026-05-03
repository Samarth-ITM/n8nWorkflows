Automatic Email Notifications for n8n Version Releases with Gmail

https://n8nworkflows.xyz/workflows/automatic-email-notifications-for-n8n-version-releases-with-gmail-11005


# Automatic Email Notifications for n8n Version Releases with Gmail

### 1. Workflow Overview

This workflow automates monitoring of new n8n software releases (latest and beta versions) and sends email notifications with release notes. It is designed for n8n instance administrators or users who want to keep up to date with n8n releases without manual checks.

The workflow consists of the following logical blocks:

- **1.1 Input & Scheduling:** Periodically triggers the workflow and sets configuration variables such as n8n instance URL and release tags.
- **1.2 Fetch Versions:** Retrieves the current running n8n instance version, and the latest and beta release versions from npm registry.
- **1.3 Deduplication:** Removes versions that have already been processed in previous runs to avoid duplicate notifications.
- **1.4 Tag Info Setting and Merging:** Sets version-related variables for further processing and merges them into a consolidated dataset.
- **1.5 Release Notes Retrieval and Formatting:** Fetches release notes from GitHub for new versions and converts them from Markdown to HTML.
- **1.6 Email Notification:** Sends a styled email via Gmail with the release information and formatted release notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input & Scheduling

- **Overview:** Triggers the workflow every hour and sets initial configuration variables including the user’s n8n instance URL and the npm tags to monitor ("latest" and "beta").

- **Nodes Involved:**  
  - Schedule Trigger  
  - n8n URL & Version References  
  - Get Instance Version

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs every 1 hour (interval set to hours)  
    - Inputs: None (start node)  
    - Outputs: Connects to *n8n URL & Version References*  
    - Failure Modes: None expected, but scheduling misconfiguration is possible if interval changed improperly.

  - **n8n URL & Version References**  
    - Type: Set  
    - Configuration: Sets three string variables:  
      - `my_n8n_url` (user-configurable, e.g., "https://YOUR_n8n_INSTANCE_LINK")  
      - `n8n_tag_1` = "latest"  
      - `n8n_tag_2` = "beta"  
    - Inputs: From Schedule Trigger  
    - Outputs: Three parallel outputs leading to:  
      - Get Instance Version  
      - Get Latest n8n Release  
      - Get Beta n8n Release  
    - Edge Cases: If `my_n8n_url` is empty or incorrect, subsequent instance version fetch will fail.

  - **Get Instance Version**  
    - Type: HTTP Request  
    - Configuration: GET request to `<my_n8n_url>/rest/settings` to retrieve current instance version info  
    - Inputs: From *n8n URL & Version References*  
    - Outputs: Connects to *Combine Versions*  
    - Failure Types: Network errors, invalid URL, authentication issues if instance requires auth.

#### 2.2 Fetch Versions

- **Overview:** Retrieves version information for the latest and beta releases of n8n from the npm registry.

- **Nodes Involved:**  
  - Get Latest n8n Release  
  - Get Beta n8n Release

- **Node Details:**

  - **Get Latest n8n Release**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://registry.npmjs.org/n8n/{{ $json.n8n_tag_1 }}` (resolves to `.../n8n/latest`)  
    - Inputs: From *n8n URL & Version References*  
    - Outputs: Connects to *Combine Versions*  
    - Failure Types: Network errors, npm registry downtime.

  - **Get Beta n8n Release**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://registry.npmjs.org/n8n/{{ $json.n8n_tag_2 }}` (resolves to `.../n8n/beta`)  
    - Inputs: From *n8n URL & Version References*  
    - Outputs: Connects to *Combine Versions*  
    - Failure Types: Same as above.

#### 2.3 Deduplication

- **Overview:** Deduplicates newly fetched versions against previous executions to ensure notifications only for new releases.

- **Nodes Involved:**  
  - Deduplicate Latest Releases  
  - Deduplicate Beta Releases

- **Node Details:**

  - **Deduplicate Latest Releases**  
    - Type: Remove Duplicates  
    - Configuration: Removes items whose `n8n_tag1_version` was seen in the last 10 runs  
    - Inputs: From *Version Variables*  
    - Outputs: Connects to *Set Latest Tag Info*  
    - Edge Cases: If version info is missing or malformed, deduplication may fail.

  - **Deduplicate Beta Releases**  
    - Type: Remove Duplicates  
    - Configuration: Removes items whose `n8n_tag2_version` was seen in the last 10 runs  
    - Inputs: From *Version Variables*  
    - Outputs: Connects to *Set Beta Tag Info*  
    - Edge Cases: Same as above.

#### 2.4 Tag Info Setting and Merging

- **Overview:** Sets detailed tag information for each release and merges the latest and beta release information into a single data object.

- **Nodes Involved:**  
  - Version Variables  
  - Set Latest Tag Info  
  - Set Beta Tag Info  
  - Merge Releases  
  - Combine Versions

- **Node Details:**

  - **Combine Versions**  
    - Type: Merge  
    - Configuration: Combines three inputs by position (instance version, latest release, beta release) into one output array  
    - Inputs: From *Get Instance Version*, *Get Latest n8n Release*, *Get Beta n8n Release*  
    - Outputs: Connects to *Version Variables*  
    - Edge Cases: Requires all three inputs to be present; missing data can cause incomplete merges.

  - **Version Variables**  
    - Type: Set  
    - Configuration: Extracts and sets variables:  
      - `my_n8n_version` from instance version response  
      - `n8n_tag1_version` from latest release response  
      - `n8n_tag2_version` from beta release response  
    - Inputs: From *Combine Versions*  
    - Outputs: Connects to both *Deduplicate Beta Releases* and *Deduplicate Latest Releases*  
    - Edge Cases: If expected fields are missing, variables may be undefined.

  - **Set Latest Tag Info**  
    - Type: Set  
    - Configuration: Sets:  
      - `my_n8n_version` (passed through)  
      - `n8n_tag` = "latest" (from reference node)  
      - `n8n_version` = deduplicated latest version  
    - Inputs: From *Deduplicate Latest Releases*  
    - Outputs: Connects to *Merge Releases* (main output 0)  
    - Edge Cases: Version may be empty if no new latest release.

  - **Set Beta Tag Info**  
    - Type: Set  
    - Configuration: Sets:  
      - `my_n8n_version` (passed through)  
      - `n8n_tag` = "beta" (from reference node)  
      - `n8n_version` = deduplicated beta version  
    - Inputs: From *Deduplicate Beta Releases*  
    - Outputs: Connects to *Merge Releases* (main output 1)  
    - Edge Cases: Version may be empty if no new beta release.

  - **Merge Releases**  
    - Type: Merge  
    - Configuration: Default merge mode (merge inputs into one output)  
    - Inputs: From *Set Latest Tag Info* (output 0) and *Set Beta Tag Info* (output 1)  
    - Outputs: Connects to *Get GitHub Release Notes*  
    - Edge Cases: If both inputs are empty, no releases to process.

#### 2.5 Release Notes Retrieval and Formatting

- **Overview:** Fetches GitHub release notes for new n8n versions and converts Markdown notes to HTML suitable for email content.

- **Nodes Involved:**  
  - Get GitHub Release Notes  
  - Convert Notes To HTML

- **Node Details:**

  - **Get GitHub Release Notes**  
    - Type: HTTP Request  
    - Configuration: GET request to GitHub API for release info at `https://api.github.com/repos/n8n-io/n8n/releases/tags/n8n@{{ $json.n8n_version }}`  
    - Inputs: From *Merge Releases*  
    - Outputs: Connects to *Convert Notes To HTML*  
    - Edge Cases:  
      - GitHub API rate limiting or downtime.  
      - Release tag might not exist or be misspelled.  
      - Requires no authentication or a GitHub token if rate limits become an issue.

  - **Convert Notes To HTML**  
    - Type: Markdown  
    - Configuration: Converts Markdown text from release notes (`$json.body`) to HTML using extended parsing options (e.g., emoji, tables, task lists, GitHub mentions)  
    - Inputs: From *Get GitHub Release Notes*  
    - Outputs: Connects to *Send Email*  
    - Edge Cases: Improperly formatted Markdown could lead to conversion errors or unexpected HTML output.

#### 2.6 Email Notification

- **Overview:** Sends an email notification via Gmail to a configured recipient with the release info and formatted release notes.

- **Nodes Involved:**  
  - Send Email

- **Node Details:**

  - **Send Email**  
    - Type: Gmail  
    - Configuration:  
      - Recipient email configured in the node (replace `"Your_Email"` with actual recipient)  
      - Subject includes release tag and release name dynamically  
      - HTML email body includes:  
        - n8n logo, release name, new release tag, current instance version  
        - Release notes formatted in HTML  
        - Link to GitHub release page  
        - Styled with dark theme and branding colors  
      - Gmail OAuth2 credentials required for sending  
    - Inputs: From *Convert Notes To HTML*  
    - Outputs: None (end of workflow)  
    - Edge Cases:  
      - Gmail OAuth2 token expiry or revocation  
      - Email sending rate limits or quota exceeded  
      - Invalid recipient email address

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                                | Input Node(s)                             | Output Node(s)                          | Sticky Note                                                                                              |
|-----------------------------|---------------------|-----------------------------------------------|------------------------------------------|---------------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger             | Schedule Trigger    | Triggers workflow hourly                       | None                                     | n8n URL & Version References           |                                                                                                         |
| n8n URL & Version References | Set                 | Sets instance URL and npm tags ("latest", "beta") | Schedule Trigger                         | Get Instance Version, Get Latest n8n Release, Get Beta n8n Release |                                                                                                         |
| Get Instance Version         | HTTP Request        | Fetches current n8n instance version           | n8n URL & Version References              | Combine Versions                      |                                                                                                         |
| Get Latest n8n Release       | HTTP Request        | Gets latest release info from npm              | n8n URL & Version References              | Combine Versions                      |                                                                                                         |
| Get Beta n8n Release         | HTTP Request        | Gets beta release info from npm                 | n8n URL & Version References              | Combine Versions                      |                                                                                                         |
| Combine Versions            | Merge               | Combines instance, latest, and beta versions   | Get Instance Version, Get Latest n8n Release, Get Beta n8n Release | Version Variables                    |                                                                                                         |
| Version Variables            | Set                 | Extracts version variables                      | Combine Versions                         | Deduplicate Beta Releases, Deduplicate Latest Releases |                                                                                                         |
| Deduplicate Latest Releases  | Remove Duplicates   | Deduplicates latest version against history    | Version Variables                        | Set Latest Tag Info                   |                                                                                                         |
| Deduplicate Beta Releases    | Remove Duplicates   | Deduplicates beta version against history      | Version Variables                        | Set Beta Tag Info                    |                                                                                                         |
| Set Latest Tag Info          | Set                 | Sets latest release info variables              | Deduplicate Latest Releases              | Merge Releases (main output 0)       |                                                                                                         |
| Set Beta Tag Info            | Set                 | Sets beta release info variables                | Deduplicate Beta Releases                | Merge Releases (main output 1)        |                                                                                                         |
| Merge Releases               | Merge               | Merges latest and beta release info             | Set Latest Tag Info, Set Beta Tag Info   | Get GitHub Release Notes              |                                                                                                         |
| Get GitHub Release Notes     | HTTP Request        | Fetches release notes markdown from GitHub     | Merge Releases                           | Convert Notes To HTML                 |                                                                                                         |
| Convert Notes To HTML        | Markdown            | Converts release notes from Markdown to HTML   | Get GitHub Release Notes                 | Send Email                          |                                                                                                         |
| Send Email                  | Gmail               | Sends email notification with release info     | Convert Notes To HTML                    | None                                |                                                                                                         |
| Sticky Note                  | Sticky Note         | Notes: "Fetch & Combine Versions"               | None                                     | None                                | ## Fetch & Combine Versions: Runs hourly to get the current instance version, plus the latest and beta versions from npm. |
| Sticky Note1                 | Sticky Note         | Notes: "Check for New Releases"                  | None                                     | None                                | ## Check for New Releases: Deduplicates the fetched versions against past runs, Only new releases will continue from this point. |
| Sticky Note2                 | Sticky Note         | Notes: "Format & Send Email"                      | None                                     | None                                | ## Format & Send Email: For any new release, fetches notes from GitHub, converts Markdown to HTML, and sends a formatted email notification. |
| Sticky Note3                 | Sticky Note         | General workflow description                      | None                                     | None                                | Monitor n8n releases and get notifications for new versions. Designed for users managing their own instances to stay updated. |
| Sticky Note4                 | Sticky Note         | Email Sample visualization                        | None                                     | None                                | Email Sample screenshot from n8n community forum.                                                       |
| Sticky Note5                 | Sticky Note         | Setup and customization tips                      | None                                     | None                                | Setup instructions: configure instance URL, Gmail credentials, recipient email. Tips for customization and extension. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set interval to every 1 hour.

2. **Create Set Node: "n8n URL & Version References":**  
   - Add three string variables:  
     - `my_n8n_url` = `"https://YOUR_n8n_INSTANCE_LINK"` (replace with your instance URL or leave blank)  
     - `n8n_tag_1` = `"latest"`  
     - `n8n_tag_2` = `"beta"`  
   - Connect Schedule Trigger → this node.

3. **Create HTTP Request Node: "Get Instance Version":**  
   - Method: GET  
   - URL: `={{ $json.my_n8n_url }}/rest/settings`  
   - Response Format: JSON (full response)  
   - Connect "n8n URL & Version References" → this node.

4. **Create HTTP Request Node: "Get Latest n8n Release":**  
   - Method: GET  
   - URL: `=https://registry.npmjs.org/n8n/{{ $json.n8n_tag_1 }}`  
   - Response Format: JSON (full response)  
   - Connect "n8n URL & Version References" → this node.

5. **Create HTTP Request Node: "Get Beta n8n Release":**  
   - Method: GET  
   - URL: `=https://registry.npmjs.org/n8n/{{ $json.n8n_tag_2 }}`  
   - Response Format: JSON (full response)  
   - Connect "n8n URL & Version References" → this node.

6. **Create Merge Node: "Combine Versions":**  
   - Mode: Combine  
   - Combine By: Position  
   - Number Inputs: 3  
   - Connect outputs of:  
     - Get Instance Version → Input 1  
     - Get Latest n8n Release → Input 2  
     - Get Beta n8n Release → Input 3

7. **Create Set Node: "Version Variables":**  
   - Assign variables:  
     - `my_n8n_version` = `={{ $json.body_1.data.versionCli }}` (from instance version response)  
     - `n8n_tag1_version` = `={{ $json.body_2.version }}` (from latest release response)  
     - `n8n_tag2_version` = `={{ $json.body_3.version }}` (from beta release response)  
   - Connect "Combine Versions" → this node.

8. **Create Remove Duplicates Node: "Deduplicate Beta Releases":**  
   - Operation: Remove Items Seen In Previous Executions  
   - Dedupe Value: `={{ $json.n8n_tag2_version }}`  
   - History Size: 10  
   - Connect "Version Variables" → this node.

9. **Create Remove Duplicates Node: "Deduplicate Latest Releases":**  
   - Operation: Remove Items Seen In Previous Executions  
   - Dedupe Value: `={{ $json.n8n_tag1_version }}`  
   - History Size: 10  
   - Connect "Version Variables" → this node.

10. **Create Set Node: "Set Latest Tag Info":**  
    - Assign:  
      - `my_n8n_version` = `={{ $json.my_n8n_version }}` (pass-through)  
      - `n8n_tag` = `"latest"` (reference from a set node or hardcode)  
      - `n8n_version` = `={{ $json.n8n_tag1_version }}`  
    - Connect "Deduplicate Latest Releases" → this node.

11. **Create Set Node: "Set Beta Tag Info":**  
    - Assign:  
      - `my_n8n_version` = `={{ $json.my_n8n_version }}`  
      - `n8n_tag` = `"beta"`  
      - `n8n_version` = `={{ $json.n8n_tag2_version }}`  
    - Connect "Deduplicate Beta Releases" → this node.

12. **Create Merge Node: "Merge Releases":**  
    - Mode: Merge (default)  
    - Connect "Set Latest Tag Info" → Input 1  
    - Connect "Set Beta Tag Info" → Input 2

13. **Create HTTP Request Node: "Get GitHub Release Notes":**  
    - Method: GET  
    - URL: `=https://api.github.com/repos/n8n-io/n8n/releases/tags/n8n@{{ $json.n8n_version }}`  
    - Response Format: JSON  
    - Connect "Merge Releases" → this node.

14. **Create Markdown Node: "Convert Notes To HTML":**  
    - Mode: Markdown to HTML  
    - Input Markdown: `={{ $json.body }}` (GitHub release notes body)  
    - Enable options for emoji, tables, GitHub mentions, task lists, etc. (see workflow options)  
    - Connect "Get GitHub Release Notes" → this node.

15. **Create Gmail Node: "Send Email":**  
    - Credentials: Use Gmail OAuth2 (set up and authorize with a Gmail account that will send emails)  
    - Recipient: Set your email address in the `sendTo` parameter  
    - Subject: `=⚡ n8n New ({{ $('Merge Releases').item.json.n8n_tag }}) Version Released! [{{ $json.name }}]`  
    - Message: Use an HTML template embedding variables for the release name, tag, current version, release notes HTML, and GitHub URL  
    - Connect "Convert Notes To HTML" → this node.

16. **Configure Credentials:**  
    - Set up Gmail OAuth2 credentials in n8n with appropriate scopes for sending email.

17. **Test the workflow:**  
    - Trigger manually or wait for the hourly schedule to verify all data flows and email delivery.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------|
| This workflow is designed to automatically monitor n8n releases and notify users via email, easing update management and awareness of new features.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note3 content summarizing the workflow purpose.                                                                   |
| Email styling and template are customizable in the Gmail node; you can adapt colors, layout, and content to match your branding or preferences.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Sticky Note2 and Sticky Note4 showing email formatting and sample screenshot.                                            |
| To customize, update your n8n instance URL in the Set node `my_n8n_url`. If you do not want to track your current instance version, leave it blank.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note5 setup instructions.                                                                                          |
| Requires a Gmail account configured with OAuth2 credentials for sending emails. Update the recipient email in the Gmail node accordingly.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note5 setup instructions.                                                                                          |
| You can extend this workflow to include additional notification channels such as Slack or Discord or automate instance updates upon new releases.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Sticky Note5 customization tips.                                                                                          |
| Original workflow creator: Mohamed Anan, a top contributor in the n8n community. [n8n Workflows](https://n8n.io/creators/mohamed3nan/) | [LinkedIn](https://link.anan.dev/Linkedin) | Community profile and credits from Sticky Note6.                                                                         |

---

**Disclaimer:** The text and workflow described above originate exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.