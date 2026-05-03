Extract YouTube Channel Metadata to Google Docs using RapidAPI

https://n8nworkflows.xyz/workflows/extract-youtube-channel-metadata-to-google-docs-using-rapidapi-7629


# Extract YouTube Channel Metadata to Google Docs using RapidAPI

---

### 1. Workflow Overview

This workflow automates the extraction of detailed metadata from a YouTube channel URL submitted via a web form and appends the formatted information into a Google Docs document. It is designed for users who want to quickly gather and store YouTube channel statistics and descriptive data in a readable format for reporting, analysis, or archival purposes.

**Logical Blocks:**

- **1.1 Input Reception:** Captures the YouTube channel URL input through a web form.
- **1.2 Metadata Retrieval:** Calls a RapidAPI YouTube Metadata API to fetch detailed channel data.
- **1.3 Data Transformation:** Reformats the raw API response into a structured, human-readable text with emojis and markdown styling.
- **1.4 Data Storage:** Appends the formatted metadata into a specified Google Docs document using Google Docs API.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow when a user submits a YouTube channel URL via a web form, effectively initiating the metadata extraction process.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **Node Name:** On form submission  
    - **Type:** Form Trigger  
    - **Role:** Starts the workflow upon user input submission.  
    - **Configuration:**  
      - Form titled "YouTube Channel Metadata" with a single required field labeled "url" for the YouTube channel URL input.  
      - No additional options or filters.  
    - **Expressions/Variables:** None directly; user input is captured as `$json.url`.  
    - **Connections:** Output connected to the "YouTube Channel Metadata" HTTP Request node.  
    - **Potential Failures:**  
      - Invalid or malformed URL input by user (not validated here).  
      - Form submission errors or webhook connectivity issues.  
    - **Version:** 2.2

---

#### 1.2 Metadata Retrieval

- **Overview:**  
  This block sends the submitted YouTube channel URL to the RapidAPI YouTube Channel Metadata API via an HTTP POST request and receives detailed JSON metadata about the channel.

- **Nodes Involved:**  
  - YouTube Channel Metadata

- **Node Details:**

  - **Node Name:** YouTube Channel Metadata  
    - **Type:** HTTP Request  
    - **Role:** Fetches YouTube channel metadata from an external API.  
    - **Configuration:**  
      - Method: POST  
      - URL: https://youtube-metadata1.p.rapidapi.com/channel_metadata.php  
      - Content-Type: multipart/form-data  
      - Body parameters: single "url" parameter set dynamically from the form input (`={{ $json.url }}`).  
      - Headers:  
        - `x-rapidapi-host`: youtube-metadata1.p.rapidapi.com  
        - `x-rapidapi-key`: Requires user to input their RapidAPI key ("your key" placeholder).  
    - **Expressions/Variables:** Uses expression to dynamically set the channel URL in the request body.  
    - **Connections:** Output connected to the "Reformat" code node.  
    - **Potential Failures:**  
      - Invalid API key or RapidAPI authentication failure (401 errors).  
      - API rate limiting or quota exhaustion.  
      - Network connectivity or timeout errors.  
      - Invalid URL leading to API returning errors or empty results.  
      - Unexpected API response structure changes.  
    - **Version:** 4.2

---

#### 1.3 Data Transformation

- **Overview:**  
  This block takes the raw JSON response from the API and extracts relevant channel details, formatting them into a nicely structured, emoji-enhanced markdown string to be used downstream.

- **Nodes Involved:**  
  - Reformat

- **Node Details:**

  - **Node Name:** Reformat  
    - **Type:** Code (JavaScript)  
    - **Role:** Parses and formats API response data into a human-readable string.  
    - **Configuration:**  
      - Extracts the first item in the `items` array from the API response JSON.  
      - Pulls fields from `snippet`, `statistics`, and `brandingSettings`.  
      - Applies fallback defaults for missing fields (e.g., "No title", "0" subscribers).  
      - Formats date and constructs a multiline string with emojis and markdown-style labels.  
      - Outputs an object with property `docContent` containing the formatted string.  
    - **Expressions/Variables:**  
      - `$input.first().json.items[0]` to access channel data.  
      - Uses standard JavaScript for conditional property access and formatting.  
    - **Connections:** Output connected to "Add Data in Google Docs".  
    - **Potential Failures:**  
      - If API response structure changes causing missing `items` or nested properties, code may throw errors.  
      - Null or undefined fields if the channel has limited or private data.  
      - Date parsing errors if `publishedAt` is malformed.  
    - **Version:** 2

---

#### 1.4 Data Storage

- **Overview:**  
  This block receives the formatted text and appends it to a Google Docs document, providing a persistent, shareable record of the YouTube channel metadata.

- **Nodes Involved:**  
  - Add Data in Google Docs

- **Node Details:**

  - **Node Name:** Add Data in Google Docs  
    - **Type:** Google Docs  
    - **Role:** Inserts the formatted metadata text into a Google Docs document.  
    - **Configuration:**  
      - Operation: Update  
      - Action: Insert text using the `docContent` field from the previous node.  
      - Document URL: **Must be specified by user** (currently empty in the workflow JSON).  
      - Authentication: Service Account with Google API credentials configured.  
    - **Expressions/Variables:** Uses expression `={{ $json.docContent }}` to insert formatted content.  
    - **Connections:** Terminal node (no output).  
    - **Potential Failures:**  
      - Missing or incorrect Google Docs document URL.  
      - Google API authentication failure or permission issues.  
      - Service account lacking write permissions to the document.  
      - API rate limits or quota exceeded.  
    - **Version:** 2

---

### 3. Summary Table

| Node Name               | Node Type         | Functional Role                               | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                      |
|-------------------------|-------------------|-----------------------------------------------|-------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| On form submission      | Form Trigger      | Starts workflow by capturing YouTube channel URL input | None                    | YouTube Channel Metadata  | **On form submission** (Form Trigger)  Triggers the workflow when a user submits a YouTube Channel URL via the web form. Collects user input (YouTube channel URL) to start the workflow. |
| YouTube Channel Metadata| HTTP Request      | Fetches YouTube channel metadata from RapidAPI | On form submission      | Reformat                  | **YouTube Channel Metadata** (HTTP Request)  Calls the RapidAPI YouTube Channel Metadata API with the submitted URL. Sends a POST request with the channel URL to fetch channel data like title, subscribers, description, etc. |
| Reformat               | Code              | Formats raw API JSON into readable markdown string | YouTube Channel Metadata| Add Data in Google Docs   | **Reformat** (Code)  Transforms the raw API JSON response into a clean, human-readable formatted string. Extracts relevant fields (title, subscribers, keywords, etc.) Formats data with emojis and markdown-style layout for easy reading Outputs the formatted string as `docContent` |
| Add Data in Google Docs | Google Docs       | Appends formatted metadata text into Google Docs document | Reformat                | None                      | **Add Data in Google Docs** (Google Docs)  Inserts the formatted channel metadata text into a specified Google Docs document. Uses Google Docs API with service account authentication Appends the formatted channel data into the doc for record-keeping or sharing |
| Sticky Note            | Sticky Note       | Documentation notes                            | None                    | None                      | # **YouTube Channel Metadata to Google Docs** Workflow description and nodes breakdown.         |
| Sticky Note1           | Sticky Note       | Documentation note                            | None                    | None                      | **On form submission** (Form Trigger)  Triggers the workflow when a user submits a YouTube Channel URL via the web form. Collects user input (YouTube channel URL) to start the workflow. |
| Sticky Note2           | Sticky Note       | Documentation note                            | None                    | None                      | **YouTube Channel Metadata** (HTTP Request)  Calls the RapidAPI YouTube Channel Metadata API with the submitted URL. Sends a POST request with the channel URL to fetch channel data like title, subscribers, description, etc. |
| Sticky Note3           | Sticky Note       | Documentation note                            | None                    | None                      | **Reformat** (Code)  Transforms the raw API JSON response into a clean, human-readable formatted string. Extracts relevant fields (title, subscribers, keywords, etc.) Formats data with emojis and markdown-style layout for easy reading Outputs the formatted string as `docContent` |
| Sticky Note4           | Sticky Note       | Documentation note                            | None                    | None                      | **Add Data in Google Docs** (Google Docs)  Inserts the formatted channel metadata text into a specified Google Docs document. Uses Google Docs API with service account authentication Appends the formatted channel data into the doc for record-keeping or sharing |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node**
   - Add a **Form Trigger** node named "On form submission".
   - Configure:
     - Form Title: "YouTube Channel Metadata"
     - Add a single form field:
       - Label: "url"
       - Placeholder: "Channel url"
       - Set as required.
   - This node triggers the workflow on user submission of the YouTube channel URL.

2. **Add HTTP Request Node to Fetch Metadata**
   - Add an **HTTP Request** node named "YouTube Channel Metadata".
   - Connect "On form submission" → "YouTube Channel Metadata".
   - Configure:
     - HTTP Method: POST
     - URL: https://youtube-metadata1.p.rapidapi.com/channel_metadata.php
     - Content-Type: multipart/form-data
     - Send Body: enabled
     - Body Parameters: Add parameter
       - Name: url
       - Value: `={{ $json.url }}`
     - Header Parameters: Add headers
       - `x-rapidapi-host`: youtube-metadata1.p.rapidapi.com
       - `x-rapidapi-key`: Your RapidAPI key (replace `"your key"` placeholder)
   - Ensure your RapidAPI key is valid and has access to the YouTube Metadata API.

3. **Add a Code Node to Reformat Data**
   - Add a **Code** node named "Reformat".
   - Connect "YouTube Channel Metadata" → "Reformat".
   - Paste the following JavaScript code (adjust if needed):

```javascript
const channel = $input.first().json.items[0];

const {
  snippet,
  statistics,
  brandingSettings,
} = channel;

const title = snippet.title || 'No title';
const description = snippet.description || 'No description available';
const customUrl = snippet.customUrl || 'No custom URL';
const publishedDate = new Date(snippet.publishedAt).toLocaleDateString();
const country = snippet.country || 'Not specified';

const subscriberCount = statistics.subscriberCount || '0';
const viewCount = statistics.viewCount || '0';
const videoCount = statistics.videoCount || '0';

const bannerUrl = brandingSettings.image?.bannerExternalUrl || 'No banner URL';
const keywords = brandingSettings.channel?.keywords || 'No keywords';

const formatted = `
📺 **Channel:** ${title}  
🔗 **Custom URL:** https://youtube.com/${customUrl}  
🗓️ **Published On:** ${publishedDate}  
🌍 **Country:** ${country}

👥 **Subscribers:** ${subscriberCount}  
👁️ **Total Views:** ${viewCount}  
🎥 **Total Videos:** ${videoCount}

📝 **Description:**  
${description}

🏷️ **Keywords:** ${keywords}

🖼️ **Banner Image:**  
${bannerUrl}
`;

return [
  {
    json: {
      docContent: formatted.trim(),
    },
  },
];
```

4. **Add Google Docs Node to Append Data**
   - Add a **Google Docs** node named "Add Data in Google Docs".
   - Connect "Reformat" → "Add Data in Google Docs".
   - Configure:
     - Operation: Update
     - Actions: Insert Text (add an action)
       - Text: `={{ $json.docContent }}`
     - Document URL: Enter the Google Docs document URL where you want to append the content.
     - Authentication: Select your configured Google API Service Account credential.
   - Ensure the service account has write permissions on the specified Google Docs document.

5. **Credentials Setup**
   - **RapidAPI Key:**  
     - Obtain a key from RapidAPI for the YouTube Metadata API.  
     - Replace the placeholder `"your key"` in the HTTP Request node headers.  
   - **Google Docs Service Account:**  
     - Create a Google Cloud service account with Google Docs API enabled.  
     - Share the target Google Docs document with the service account email with Editor rights.  
     - Add the service account credentials in n8n under Google API credentials.

6. **Test the Workflow**
   - Deploy the workflow.  
   - Access the form URL generated by the "On form submission" node.  
   - Submit a valid YouTube channel URL.  
   - Verify the Google Docs document is updated with the formatted metadata.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                       | Context or Link                                          |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------|
| This workflow automates YouTube channel metadata extraction and storage for easy reporting or archiving.                                                        | Main workflow purpose                                    |
| Ensure to replace `"your key"` in the HTTP Request node with your actual RapidAPI key for authentication.                                                       | RapidAPI key usage                                       |
| Google Docs document URL must be specified in the Google Docs node; otherwise, the insertion will fail.                                                         | Google Docs node configuration                           |
| Service account must have Editor access to the target Google Docs document to update content.                                                                    | Google Docs API permissions                              |
| The formatting code uses emojis and markdown-style syntax for readability but can be customized to suit other output formats.                                   | Customization of output                                  |
| RapidAPI YouTube Metadata API documentation: https://rapidapi.com/                                                                                              | External API reference                                   |
| n8n Google Docs node documentation: https://docs.n8n.io/nodes/n8n-nodes-base.googledocs/                                                                        | Node reference                                          |
| Form Trigger node URL can be accessed via the webhook URL generated after deployment to submit channel URLs manually or integrated into other portals.          | Form trigger usage                                      |

---

**Disclaimer:**  
The text provided is generated exclusively from an automated workflow built with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.