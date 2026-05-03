Upload Podcast Episodes to Spotify via RSS & Google Drive

https://n8nworkflows.xyz/workflows/upload-podcast-episodes-to-spotify-via-rss---google-drive-7319


# Upload Podcast Episodes to Spotify via RSS & Google Drive

### 1. Workflow Overview

This workflow automates the process of uploading new podcast episodes to Spotify by managing the episode audio file and updating the podcast RSS feed stored on GitHub. It is designed to streamline the final publishing steps by integrating various services—GitHub for RSS feed storage, Google Drive for audio file hosting, and Spotify via RSS feed distribution.

The workflow is logically divided into the following blocks:

- **1.1 Trigger & Input Setup:** Manual initiation and setting episode-specific data.
- **1.2 Audio File Handling:** Reading the local audio file and uploading it to Google Drive with public sharing.
- **1.3 RSS Feed Management:** Fetching the existing RSS XML from GitHub, converting it, appending new episode info, and updating the RSS feed in the repository.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Input Setup

**Overview:**  
This block initiates the workflow manually and sets all parameters necessary for the podcast episode upload, including audio file path, episode title, description, and GitHub repository details.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Input Data (Set node)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow on user command.  
  - *Configuration:* Default manual trigger with no parameters.  
  - *Inputs:* None  
  - *Outputs:* Triggers the next node (Input Data)  
  - *Edge cases:* None typical; user must manually execute.

- **Input Data**  
  - *Type:* Set node  
  - *Role:* Defines static parameters for the episode upload such as audio file path, episode title, description, and GitHub repo info.  
  - *Configuration:*  
    - `audio_path`: Path to local audio file (e.g., `/newsletter2podcast/tmp/final_merged.mp3`)  
    - `title`: Title of the episode  
    - `description`: Detailed episode description with optional contact and promotional links  
    - `github_user_name`: GitHub username (e.g., `acosta1991`)  
    - `GitHub_repo_name`: Repository name (e.g., `newsletter2podcast`)  
  - *Key expressions:* None, static values are set.  
  - *Input:* Manual Trigger node output  
  - *Output:* Passes parameters to "Read audio file"  
  - *Edge cases:* Audio file path must be valid; description length should be within platform limits to avoid truncation.

---

#### 2.2 Audio File Handling

**Overview:**  
This block reads the podcast audio file from the local filesystem, uploads it to a designated Google Drive folder, and then sets the file to be publicly accessible to generate a shareable download link.

**Nodes Involved:**  
- Read audio file (Read/Write File)  
- Upload to Google Drive (Google Drive Node)  
- Set public url (Google Drive Node)

**Node Details:**

- **Read audio file**  
  - *Type:* Read/Write File node  
  - *Role:* Reads the audio file from the specified local path to prepare for upload.  
  - *Configuration:*  
    - File selector expression: `={{ $json.audio_path }}` dynamically uses the path from Input Data.  
    - Data property name: `data` (stores file binary content).  
  - *Input:* Output from Input Data node  
  - *Output:* Binary audio data to "Upload to Google Drive"  
  - *Edge cases:* File not found or inaccessible path will cause failure; error handling recommended.

- **Upload to Google Drive**  
  - *Type:* Google Drive node  
  - *Role:* Uploads the audio file to a specific folder in Google Drive.  
  - *Configuration:*  
    - File name: `={{ $json.fileName }}` (dynamic, expects fileName property to be set, else defaults)  
    - Drive ID: "My Drive"  
    - Folder ID: `1Z_5DEzJYg2DYQAQ2jA6phkCOIlqVzMBx` (Podcast folder)  
  - *Credentials:* Google Drive OAuth2 with appropriate permissions.  
  - *Input:* Binary audio data from "Read audio file"  
  - *Output:* Metadata of uploaded file, including file ID, to "Set public url"  
  - *Edge cases:* OAuth token expiration, folder ID invalid, upload failures.

- **Set public url**  
  - *Type:* Google Drive node  
  - *Role:* Sets the uploaded file’s sharing permissions to public read (anyone with link can view).  
  - *Configuration:*  
    - File ID: `={{ $json.id }}` from previous node.  
    - Permissions: role = reader, type = anyone.  
  - *Credentials:* Google Drive OAuth2 same as above.  
  - *Input:* Uploaded file metadata from previous node  
  - *Output:* Confirmation of sharing and public URL (though URL not explicitly extracted here)  
  - *Edge cases:* Permission errors, sharing restrictions on Google Drive accounts.

---

#### 2.3 RSS Feed Management

**Overview:**  
This block handles fetching the existing podcast RSS feed XML file from GitHub, converting it from binary to JSON for processing, appending the new episode entry, encoding the updated feed, and pushing the changes back to GitHub.

**Nodes Involved:**  
- Get RSS File (GitHub)  
- Covert to Json (Code node)  
- Add new episode (Code node)  
- Edit a file (GitHub)

**Node Details:**

- **Get RSS File**  
  - *Type:* GitHub node (file get operation)  
  - *Role:* Downloads the current `rss.xml` from GitHub repository.  
  - *Configuration:*  
    - Owner: `Acosta1991`  
    - Repository: `newsletter2podcast`  
    - File Path: `n8n/rss.xml`  
  - *Credentials:* GitHub API credentials with read access.  
  - *Input:* From "Set public url" (triggered by public URL setting)  
  - *Output:* Binary file content to "Covert to Json"  
  - *Edge cases:* Repository/file missing, permission denied, rate limits.

- **Covert to Json**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Converts binary XML data (RSS feed) to JSON format for manipulation.  
  - *Configuration:*  
    - Reads binary property named `data`  
    - Converts base64 encoded XML binary to UTF-8 string and outputs as JSON key `xml`.  
  - *Input:* Binary RSS file from "Get RSS File"  
  - *Output:* JSON with RSS XML string to "Add new episode"  
  - *Edge cases:* Binary property not found, decoding errors.

- **Add new episode**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Creates a new RSS `<item>` entry for the new podcast episode and inserts it into the existing RSS XML feed.  
  - *Configuration:*  
    - Reads existing RSS XML string (`xml`)  
    - Builds new `<item>` with episode title, description, current UTC pubDate, and enclosure URL referencing the Google Drive public audio link (`https://drive.google.com/uc?export=download&id={fileId}`)  
    - Inserts new item before closing `</channel></rss>` tags  
    - Encodes updated XML to base64 string `updatedRssXml` for GitHub upload.  
  - *Key expressions:*  
    - Extracts `fileId` from `"Upload to Google Drive"` node output  
    - Uses `title` and `description` from "Input Data" node  
  - *Input:* JSON RSS XML from "Covert to Json"  
  - *Output:* Base64 encoded updated RSS XML to "Edit a file"  
  - *Edge cases:* Missing RSS XML input, missing Google Drive file ID, malformed XML.

- **Edit a file**  
  - *Type:* GitHub node (file edit operation)  
  - *Role:* Commits the updated RSS feed XML back to the GitHub repository.  
  - *Configuration:*  
    - Owner: `Acosta1991`  
    - Repository: `newsletter2podcast`  
    - File Path: `n8n/rss.xml`  
    - File Content: `={{ $json.updatedRssXml }}` (base64 encoded XML)  
    - Commit Message: `Update RSS feed with new episode`  
  - *Credentials:* GitHub API with write permissions.  
  - *Input:* Updated base64 RSS XML from "Add new episode"  
  - *Output:* Confirmation of commit  
  - *Edge cases:* Insufficient write permissions, commit conflicts, invalid file content format.

---

### 3. Summary Table

| Node Name                  | Node Type            | Functional Role                                          | Input Node(s)               | Output Node(s)          | Sticky Note                                                                                                      |
|----------------------------|----------------------|----------------------------------------------------------|-----------------------------|-------------------------|------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger       | Manual start of the workflow                              | -                           | Input Data               | 🎤 Podcast Episode Upload to Spotify Workflow (Overview and contact info)                                        |
| Input Data                 | Set                  | Defines episode metadata and file paths                   | When clicking ‘Execute workflow’ | Read audio file          | 📡 Input Data parameters setup and validation notes                                                              |
| Read audio file            | Read/Write File       | Reads local audio file binary content                      | Input Data                  | Upload to Google Drive   | 🎧 Read Audio File role and validation                                                                           |
| Upload to Google Drive     | Google Drive         | Uploads audio file to Google Drive                         | Read audio file             | Set public url           | ☁️ Upload to Google Drive purpose and configuration tips                                                        |
| Set public url             | Google Drive         | Sets uploaded audio file as publicly accessible           | Upload to Google Drive      | Get RSS File             | 🌐 Set Public URL permission settings and validation                                                             |
| Get RSS File               | GitHub               | Retrieves current RSS feed XML file                        | Set public url              | Covert to Json           | 📄 Get RSS File retrieval and validation                                                                         |
| Covert to Json             | Code                 | Converts binary XML RSS feed into JSON for processing     | Get RSS File                | Add new episode          | 🗂️ Convert to Json role and configuration                                                                        |
| Add new episode            | Code                 | Constructs and inserts new episode into RSS feed XML      | Covert to Json              | Edit a file              | 🎙️ Add new episode to RSS feed, error handling                                                                   |
| Edit a file                | GitHub               | Updates RSS feed file in GitHub repository                 | Add new episode             | -                       | 📝 Edit a File: GitHub commit with updated RSS feed, permissions, and formatting notes                            |
| Sticky Note - Overview     | Sticky Note          | Provides workflow overview and contact info                | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Input Data   | Sticky Note          | Explains Input Data node configuration                      | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Read audio file | Sticky Note       | Details Read Audio File node purpose                        | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Upload to Google Drive | Sticky Note | Details Upload to Google Drive node                         | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Set public url | Sticky Note        | Details setting public URL on Google Drive                  | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Get RSS File | Sticky Note          | Details getting RSS file from GitHub                        | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Covert to Json | Sticky Note        | Details converting binary XML to JSON                       | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Add new episode | Sticky Note       | Details adding new episode to RSS feed                       | -                           | -                       | See content in overview                                                                                            |
| Sticky Note - Edit a file  | Sticky Note          | Details editing file in GitHub repository                   | -                           | -                       | See content in overview                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named "When clicking ‘Execute workflow’".  
   - No parameters needed.

2. **Create Input Data Node**  
   - Add a Set node named "Input Data".  
   - Configure fields:  
     - `audio_path` (String): e.g., `/newsletter2podcast/tmp/final_merged.mp3`  
     - `title` (String): episode title text  
     - `description` (String): detailed episode description with contact info and promo links  
     - `github_user_name` (String): GitHub username (e.g., `acosta1991`)  
     - `GitHub_repo_name` (String): GitHub repo name (e.g., `newsletter2podcast`)  
   - Connect output of Manual Trigger to this node.

3. **Create Read Audio File Node**  
   - Add a Read/Write File node named "Read audio file".  
   - Set File Selector expression to `={{ $json.audio_path }}` to dynamically load the audio file path.  
   - Set Data Property Name to "data".  
   - Connect output of Input Data node to this node.

4. **Create Upload to Google Drive Node**  
   - Add a Google Drive node named "Upload to Google Drive".  
   - Configure operation: Upload file.  
   - Set file name to `={{ $json.fileName }}` or provide a static name if unavailable.  
   - Set Drive ID to "My Drive" (default).  
   - Set Folder ID to your Google Drive folder for podcasts (e.g., `1Z_5DEzJYg2DYQAQ2jA6phkCOIlqVzMBx`).  
   - Use Google Drive OAuth2 credentials authorized for your account.  
   - Connect output of "Read audio file" node to this node.

5. **Create Set Public URL Node**  
   - Add a Google Drive node named "Set public url".  
   - Set operation to "Share".  
   - File ID: set expression `={{ $json.id }}` from previous node output.  
   - Permissions: role = reader, type = anyone (public read access).  
   - Use the same Google Drive OAuth2 credentials.  
   - Connect output of "Upload to Google Drive" node to this node.

6. **Create Get RSS File Node**  
   - Add a GitHub node named "Get RSS File".  
   - Operation: Get file.  
   - Owner: set to your GitHub username (e.g., `Acosta1991`).  
   - Repository: your podcast repo (e.g., `newsletter2podcast`).  
   - File Path: path to RSS feed file (e.g., `n8n/rss.xml`).  
   - Use GitHub API credentials with read access.  
   - Connect output of "Set public url" to this node.

7. **Create Convert to JSON Node**  
   - Add a Code node named "Covert to Json".  
   - Paste the following JavaScript code (adjust `binName` if needed):  
     ```javascript
     const binName = 'data';
     return items.map(item => {
       const b = item.binary[binName];
       const xml = Buffer.from(b.data, b.encoding || 'base64').toString('utf8');
       return { json: { xml } };
     });
     ```  
   - Connect output of "Get RSS File" node to this node.

8. **Create Add New Episode Node**  
   - Add a Code node named "Add new episode".  
   - Paste this JavaScript code (ensure it matches your nodes' names exactly):  
     ```javascript
     const rssText = $input.first().json.xml;
     if (!rssText) {
       throw new Error("No RSS text found in $input.first().json.data");
     }
     const fileId = $('Upload to Google Drive').first().json.id;
     const mp3Link = `https://drive.google.com/uc?export=download&id=${fileId}`;
     const titulo = $('Input Data').first().json.title;
     const description = $('Input Data').first().json.description;
     const pubDate = new Date().toUTCString();
     const newItem = `
     <item>
       <title>${titulo}</title>
       <description>${description}</description>
       <pubDate>${pubDate}</pubDate>
       <enclosure url="${mp3Link}" length="1234567" type="audio/mpeg" />
       <guid isPermaLink="false">${Date.now()}</guid>
     </item>
     `;
     const updatedRss = rssText.replace(
       "</channel>\n</rss>",
       `${newItem}\n</channel>\n</rss>`
     );
     return {
       updatedRssXml: Buffer.from(updatedRss, "utf-8").toString("base64")
     };
     ```  
   - Connect output of "Covert to Json" node to this node.

9. **Create Edit a file Node**  
   - Add a GitHub node named "Edit a file".  
   - Operation: Edit file.  
   - Owner: GitHub username (e.g., `Acosta1991`).  
   - Repository: your podcast repo (e.g., `newsletter2podcast`).  
   - File Path: `n8n/rss.xml`.  
   - File Content: expression `={{ $json.updatedRssXml }}` (base64 encoded updated RSS XML).  
   - Commit Message: `Update RSS feed with new episode`.  
   - Use GitHub API credentials with write permissions.  
   - Connect output of "Add new episode" node to this node.

10. **Verify and Test**  
    - Confirm all credentials (Google Drive OAuth2 and GitHub API) are properly configured and authorized.  
    - Test with a sample audio file and ensure the RSS feed updates correctly in GitHub.  
    - Validate the public Google Drive URL is accessible and audio plays.  
    - Confirm the updated RSS feed is valid XML and recognized by Spotify.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                                                             |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow automates podcast episode upload to Spotify via RSS feed manipulation and Google Drive hosting. Manual trigger allows control over execution. Ensure audio file meets Spotify's format requirements. Google Drive folder ID and GitHub repo details must be tailored to your environment.                                                                                                                                                                                               | Workflow overview and operational context.                                                                 |
| Contact for help or collaboration: Luis Acosta (Luis.acosta@news2podcast.com), or Twitter [@guanchehacker](http://www.x.com/GuancheHacker).                                                                                                                                                                                                                                                                                                                                                     | Support and collaboration contact details.                                                                 |
| Google Drive Sharing: After upload, files are set to public read access ensuring Spotify or any podcast platform can access the media file through the RSS enclosure URL. Validate sharing permissions if issues arise.                                                                                                                                                                                                                                                                          | Google Drive file sharing best practices.                                                                   |
| GitHub commit messages should be meaningful for version control and tracking podcast feed changes. Ensure credentials have write permission to avoid commit errors.                                                                                                                                                                                                                                                                                                                            | GitHub repository management best practices.                                                                |
| The code nodes assume node names are exact; renaming nodes requires updating expressions accordingly. XML manipulation is simple string replacement; ensure existing RSS XML is well-formed to prevent invalid RSS feeds.                                                                                                                                                                                                                                                                       | Code node usage notes and potential XML formatting pitfalls.                                                 |
| To debug failures, check node execution logs for error messages related to file access, permissions, or data format. Use n8n's error workflow or node-specific error handling to trap and manage exceptions gracefully.                                                                                                                                                                                                                                                                          | Troubleshooting and error handling advice.                                                                   |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, a workflow automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.