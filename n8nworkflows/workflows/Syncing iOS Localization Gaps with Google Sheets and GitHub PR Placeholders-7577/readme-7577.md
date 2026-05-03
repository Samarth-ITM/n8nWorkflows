Syncing iOS Localization Gaps with Google Sheets and GitHub PR Placeholders

https://n8nworkflows.xyz/workflows/syncing-ios-localization-gaps-with-google-sheets-and-github-pr-placeholders-7577


# Syncing iOS Localization Gaps with Google Sheets and GitHub PR Placeholders

### 1. Workflow Overview

This workflow automates the process of identifying missing iOS localization keys in `.strings` files for a target language (e.g., French) compared to a source language (e.g., English) in a GitHub repository. It integrates with Google Sheets to log these missing keys with placeholders and facilitates localization management by creating a clear overview of gaps requiring translation.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Receives an external POST request to trigger the workflow.
- **1.2 Configuration Setup:** Sets up environment variables and repository parameters.
- **1.3 GitHub Repository Tree Processing:** Retrieves and processes the file structure of the GitHub repo to identify relevant source and target localization files.
- **1.4 Localization Files Retrieval:** Downloads the actual `.strings` files from GitHub for both source and target languages.
- **1.5 Missing Keys Analysis:** Compares the localization files to find missing keys in the target language and prepares placeholder entries.
- **1.6 Logging to Google Sheets:** Appends the missing keys along with context to a specified Google Sheet for tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for an incoming HTTP POST request to start the workflow.

- **Nodes Involved:**  
  - Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: Webhook (Trigger node)  
    - Configuration: HTTP POST method on path `new-pathss`  
    - Inputs: External HTTP POST request  
    - Outputs: Triggers the next node (`Config`)  
    - Potential Failures: Incorrect HTTP method, invalid payload, network issues  
    - Version: 2  

#### 2.2 Configuration Setup

- **Overview:**  
  Defines static configuration data such as GitHub repo owner, repo name, branches, source and target languages, and placeholder string for missing translations.

- **Nodes Involved:**  
  - Config

- **Node Details:**  
  - **Config**  
    - Type: Set node  
    - Configuration: Hardcoded assignment of variables:
      - `GITHUB_OWNER`: GitHub username owning the repo  
      - `GITHUB_REPO`: Repository name  
      - `BASE_BRANCH`: Branch to work on (e.g., main)  
      - `SOURCE_LANG`: Source localization language (e.g., "en")  
      - `TARGET_LANG`: Target localization language (e.g., "fr")  
      - `PLACEHOLDER_VALUE`: Placeholder string `__TODO_TRANSLATE__` for missing translations  
    - Inputs: From Webhook  
    - Outputs: To `Get GitHub Tree`  
    - Edge cases: Misconfiguration may cause invalid URLs or API requests  
    - Version: 3.4  

#### 2.3 GitHub Repository Tree Processing

- **Overview:**  
  Retrieves the full file tree of the GitHub repo for the specified branch, then processes it to find `.strings` files for both source and target languages and pairs them by directory and filename.

- **Nodes Involved:**  
  - Get GitHub Tree  
  - Process File Tree

- **Node Details:**  
  - **Get GitHub Tree**  
    - Type: HTTP Request  
    - Configuration:  
      - Constructs URL dynamically using variables from `Config` to call GitHub API endpoint:  
        `https://api.github.com/repos/{GITHUB_OWNER}/{GITHUB_REPO}/git/trees/{BASE_BRANCH}?recursive=1`  
      - Authentication: GitHub OAuth via stored credentials  
    - Inputs: From `Config`  
    - Outputs: To `Process File Tree`  
    - Edge cases: Auth errors, rate-limiting, network failures, invalid branch or repo  
    - Version: 4.2  

  - **Process File Tree**  
    - Type: Code (JavaScript)  
    - Role: Parses the repo tree JSON response and extracts `.strings` files from source language folders (`Base.lproj` or `en.lproj`) and target language folder (`fr.lproj`). It pairs source and target files based on directory structure and filename, constructing expected target paths.  
    - Key Expressions:
      - Uses regex to detect language folders and extract directory paths  
      - Returns an array of objects each containing:  
        - `source`: source file info (path, sha, basename, directory)  
        - `target`: matching target file info or null  
        - `targetPath`: expected target file path  
    - Inputs: From `Get GitHub Tree`  
    - Outputs: To `Get Source File`  
    - Edge cases: No matching target files, unexpected directory structures, empty repo tree  
    - Version: 2  

#### 2.4 Localization Files Retrieval

- **Overview:**  
  Fetches the actual `.strings` files content from GitHub for both source and target files to enable comparison.

- **Nodes Involved:**  
  - Get Source File  
  - Edit Fields  
  - Code  
  - HTTP Request  
  - Merge

- **Node Details:**  
  - **Get Source File**  
    - Type: HTTP Request  
    - Configuration: Fetches content of each source file via GitHub API using the file path from `Process File Tree`.  
      - URL: `https://api.github.com/repos/github-user-name/n8n-iOS-Github-repo/contents/{source.path}?ref=main`  
      - Auth: GitHub OAuth  
    - Inputs: From `Process File Tree` (file pairs)  
    - Outputs: To `Edit Fields`  
    - Edge cases: File missing, API rate limits, auth failures  
    - Version: 4.2  

  - **Edit Fields**  
    - Type: Set node  
    - Role: Sets static URLs for `en.lproj` and `fr.lproj` Localizable.strings files for later requests.  
    - Configuration: Hardcoded URLs for fetching source and target string files from GitHub  
    - Inputs: From `Get Source File`  
    - Outputs: To `Code`  
    - Edge cases: Hardcoded URLs may cause issues if repo structure changes  
    - Version: 3.4  

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Transforms the URLs from `Edit Fields` into two JSON objects for further HTTP requests, representing English and French localization file URLs.  
    - Outputs: Two separate items, one for each language  
    - Inputs: From `Edit Fields`  
    - Outputs: To `HTTP Request` (two parallel requests)  
    - Version: 2  

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Fetches the actual localization `.strings` file content from the URLs provided by the previous node.  
    - Inputs: From `Code` (two items)  
    - Outputs: To `Merge` (merges both responses back)  
    - Edge cases: File not found, rate limits, network errors  
    - Version: 4.2  

  - **Merge**  
    - Type: Merge node  
    - Role: Combines the two HTTP responses (source and target `.strings` files) by position to prepare them for key comparison.  
    - Inputs: Two inputs from `HTTP Request`  
    - Outputs: To `Find Missing Keys`  
    - Version: 3.2  

#### 2.5 Missing Keys Analysis

- **Overview:**  
  Parses the content of the source and target `.strings` files, compares keys, and identifies which keys are missing in the target language. It prepares data entries including placeholders for missing translations.

- **Nodes Involved:**  
  - Find Missing Keys

- **Node Details:**  
  - **Find Missing Keys**  
    - Type: Code (JavaScript)  
    - Functions:  
      - Decodes base64 content of each localization file  
      - Parses `.strings` format lines to extract key-value pairs  
      - Compares source keys to target keys to find missing keys  
      - For missing keys, outputs JSON objects with file info, key, source value, and placeholder  
    - Uses the placeholder string `__TODO_TRANSLATE__` from the config  
    - Inputs: From `Merge` (combined source and target file contents)  
    - Outputs: To `Google Sheets`  
    - Edge cases: Malformed `.strings` files, empty files, no missing keys (outputs debug info)  
    - Version: 2  

#### 2.6 Logging to Google Sheets

- **Overview:**  
  Appends each missing localization key entry to a dedicated Google Sheet named "localize" for tracking and further translation work.

- **Nodes Involved:**  
  - Google Sheets

- **Node Details:**  
  - **Google Sheets**  
    - Type: Google Sheets node  
    - Operation: Append rows  
    - Sheet Name: `localize`  
    - Document ID: `1n_AIqOd10Q0ErQZSO4q4LBMekwgsR4cP7EW2q9nEzdk` (hardcoded)  
    - Columns mapped: `Key`, `File`, `Placeholder`, `Source Path`, `Target Path`, `Source Value`  
    - Inputs: From `Find Missing Keys` (missing keys data)  
    - Credentials: Google Sheets OAuth2  
    - Edge cases: API quota limits, credential expiration, sheet not found  
    - Version: 4.6  

---

### 3. Summary Table

| Node Name          | Node Type           | Functional Role                            | Input Node(s)          | Output Node(s)         | Sticky Note                      |
|--------------------|---------------------|------------------------------------------|-----------------------|------------------------|---------------------------------|
| Webhook            | Webhook             | Triggers workflow on POST request        | -                     | Config                 |                                 |
| Config             | Set                 | Defines repo, branch, languages, placeholder | Webhook               | Get GitHub Tree        |                                 |
| Get GitHub Tree    | HTTP Request        | Fetches GitHub repo file tree            | Config                | Process File Tree      |                                 |
| Process File Tree  | Code                | Extracts and pairs source/target `.strings` files | Get GitHub Tree       | Get Source File        |                                 |
| Get Source File    | HTTP Request        | Downloads source `.strings` file content | Process File Tree      | Edit Fields            |                                 |
| Edit Fields        | Set                 | Sets URLs for `.strings` files for both languages | Get Source File        | Code                   |                                 |
| Code               | Code                | Prepares URL objects for HTTP requests   | Edit Fields           | HTTP Request           |                                 |
| HTTP Request       | HTTP Request        | Fetches `.strings` file content for source and target | Code                  | Merge                  |                                 |
| Merge              | Merge               | Combines source and target file contents | HTTP Request          | Find Missing Keys      |                                 |
| Find Missing Keys  | Code                | Parses files, finds missing keys, outputs missing entries | Merge                  | Google Sheets           |                                 |
| Google Sheets      | Google Sheets       | Appends missing keys with placeholders to sheet | Find Missing Keys      | -                      |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `new-pathss`  
   - Purpose: To trigger the workflow externally.

2. **Add a Set node ("Config")**  
   - Define variables:  
     - `GITHUB_OWNER`: GitHub username (e.g., `github_user_name`)  
     - `GITHUB_REPO`: Repository name (e.g., `n8n-iOS-Github-repo`)  
     - `BASE_BRANCH`: Branch to monitor (e.g., `main`)  
     - `SOURCE_LANG`: Source language code (e.g., `en`)  
     - `TARGET_LANG`: Target language code (e.g., `fr`)  
     - `PLACEHOLDER_VALUE`: Placeholder string `__TODO_TRANSLATE__`  
   - Connect Webhook → Config.

3. **Add HTTP Request node ("Get GitHub Tree")**  
   - URL: `https://api.github.com/repos/{{$json.GITHUB_OWNER}}/{{$json.GITHUB_REPO}}/git/trees/{{$json.BASE_BRANCH}}?recursive=1`  
   - Authentication: GitHub OAuth with valid credentials  
   - Connect Config → Get GitHub Tree.

4. **Add Code node ("Process File Tree")**  
   - Paste JavaScript logic to parse tree, extract `.strings` files from source (Base.lproj or en.lproj) and target (fr.lproj), pair files by directory and filename, and output pairs with source and target info.  
   - Connect Get GitHub Tree → Process File Tree.

5. **Add HTTP Request node ("Get Source File")**  
   - URL: `https://api.github.com/repos/github-user-name/n8n-iOS-Github-repo/contents/{{$json.source.path}}?ref=main`  
   - Authentication: GitHub OAuth  
   - Connect Process File Tree → Get Source File.

6. **Add Set node ("Edit Fields")**  
   - Assign two fields:  
     - `en.lproj`: URL to English Localizable.strings file (hardcoded)  
     - `fr.lproj`: URL to French Localizable.strings file (hardcoded)  
   - Connect Get Source File → Edit Fields.

7. **Add Code node ("Code")**  
   - JavaScript to create two JSON objects for each language with `lang` and `url` properties for further fetching.  
   - Connect Edit Fields → Code.

8. **Add HTTP Request node ("HTTP Request")**  
   - URL: Use `{{$json.url}}` from input  
   - No authentication needed if URLs are public or use GitHub OAuth if private  
   - This node will be executed twice in parallel (once per language)  
   - Connect Code → HTTP Request.

9. **Add Merge node ("Merge")**  
   - Mode: Combine  
   - Combine by: Position (to merge the two HTTP request outputs)  
   - Connect HTTP Request → Merge (both outputs).

10. **Add Code node ("Find Missing Keys")**  
    - JavaScript to parse base64 content of `.strings` files, extract keys, compare source and target keys, find missing keys, and output JSON objects with fields: File, Source Path, Target Path, Key, Source Value, Placeholder.  
    - Uses `__TODO_TRANSLATE__` as placeholder value.  
    - Connect Merge → Find Missing Keys.

11. **Add Google Sheets node ("Google Sheets")**  
    - Operation: Append  
    - Document ID: Set your Google Sheets document ID  
    - Sheet name: `localize`  
    - Mapping columns: Key, File, Placeholder, Source Path, Target Path, Source Value  
    - Credentials: Google Sheets OAuth2  
    - Connect Find Missing Keys → Google Sheets.

12. **Credential Setup**  
    - Create and configure GitHub OAuth credentials with access to the repository and API.  
    - Create and configure Google Sheets OAuth2 credentials with access to the target spreadsheet.

13. **Activate the workflow**  
    - Test by sending an HTTP POST request to the webhook URL `https://<your-n8n-instance>/webhook/new-pathss`.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                |
|--------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow is designed specifically for iOS localization `.strings` files structured under `.lproj` folders, matching Apple conventions. | Workflow purpose                                              |
| Placeholder value `__TODO_TRANSLATE__` flags keys missing translations in the target language files.    | Placeholder convention                                        |
| GitHub API rate limits and authentication scopes must be respected to avoid failures.                   | GitHub API best practices                                     |
| Google Sheets document ID must correspond to an existing sheet with a tab named `localize`.             | Google Sheets integration                                     |
| Consider automating GitHub PR creation for missing keys after logging to Google Sheets for a full sync. | Potential enhancement                                         |
| GitHub API docs: https://docs.github.com/en/rest/reference/git#trees                                    | For understanding tree API                                    |
| Apple Localization `.strings` file format reference: https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPInternational/LocalizingYourApp/LocalizingYourApp.html | For `.strings` format parsing                                 |

---

This documentation enables a comprehensive understanding of the workflow’s structure and logic, facilitating reproduction, modification, and troubleshooting.