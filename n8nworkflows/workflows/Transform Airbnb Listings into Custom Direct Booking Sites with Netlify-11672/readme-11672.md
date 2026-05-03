Transform Airbnb Listings into Custom Direct Booking Sites with Netlify

https://n8nworkflows.xyz/workflows/transform-airbnb-listings-into-custom-direct-booking-sites-with-netlify-11672


# Transform Airbnb Listings into Custom Direct Booking Sites with Netlify

### 1. Workflow Overview

This workflow, titled **"Direct Booking Site Generator"**, transforms any Airbnb listing into a professionally designed static direct booking website and deploys it automatically to Netlify's free hosting. It targets Airbnb hosts or property managers who want to offer direct bookings, thereby avoiding platform fees and increasing control over their listings.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Accepts an Airbnb listing ID manually to start the process.
- **1.2 Data Scraping:** Uses an Airbnb Scraper API to fetch comprehensive listing data.
- **1.3 Site Generation:** Generates a responsive, visually attractive HTML site from the scraped data.
- **1.4 Binary Preparation & Packaging:** Converts the HTML into a binary file and compresses it as a ZIP archive.
- **1.5 Netlify Deployment:** Creates a new Netlify site and deploys the ZIP archive, making the direct booking site live.
- **1.6 Output & Reporting:** Outputs the deployment details including URLs and status messages.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow by receiving the Airbnb listing ID, which is essential to fetch the correct listing data.

**Nodes Involved:**  
- Manual Trigger  
- Set Listing ID  

**Node Details:**

- **Manual Trigger**  
  - *Type:* Trigger node  
  - *Role:* Starts the workflow manually by user action.  
  - *Configuration:* Default, no parameters.  
  - *Connections:* Output connected to "Set Listing ID".  
  - *Edge Cases:* None specific; manual activation required.

- **Set Listing ID**  
  - *Type:* Set node  
  - *Role:* Defines the Airbnb listing ID to be processed.  
  - *Configuration:* Hardcoded string assignment with key `listingId`. Default example value `"1501733424018064312"`.  
  - *Expressions:* None dynamic; static assignment.  
  - *Connections:* Input from Manual Trigger; output to "Airbnb Scraper".  
  - *Edge Cases:* Requires valid Airbnb listing ID format; incorrect IDs may cause scraping failure.

---

#### 2.2 Data Scraping

**Overview:**  
Fetches detailed data about the Airbnb listing, including title, photos, amenities, pricing, host info, and reviews.

**Nodes Involved:**  
- Airbnb Scraper  

**Node Details:**

- **Airbnb Scraper**  
  - *Type:* Airbnb Scraper API node  
  - *Role:* Scrapes all relevant listing data from Airbnb using the provided listing ID.  
  - *Configuration:*  
    - `operation`: `scrapeListing` to fetch all listing details.  
    - `listingId`: Expression using `{{$json.listingId}}` from previous node.  
    - Timeout set to 120 seconds to allow for slow responses.  
  - *Credentials:* Requires an Airbnb Scraper API token (from shortrentals.ai).  
  - *Connections:* Input from Set Listing ID; output to Generate HTML Site.  
  - *Edge Cases:*  
    - API rate limits or token expiration can cause errors.  
    - Listing ID invalid or listing removed results in empty or error responses.  
    - Network timeouts possible due to 120s timeout setting.

---

#### 2.3 Site Generation

**Overview:**  
Transforms the scraped JSON data into a complete, styled HTML static site representing the Airbnb listing as a direct booking page.

**Nodes Involved:**  
- Generate HTML Site  

**Node Details:**

- **Generate HTML Site**  
  - *Type:* Code node (JavaScript)  
  - *Role:* Runs custom JS code once for all input items to produce a full HTML page string and metadata such as site name and title.  
  - *Key Logic:*  
    - Cleans and formats text fields safely.  
    - Builds photo gallery with up to 6 images or placeholder.  
    - Generates amenities list and house rules with fallbacks.  
    - Composes location string from city, state, country.  
    - Extracts pricing and currency with defaults.  
    - Constructs a responsive HTML page including header, gallery, details, host info, booking form, and footer with embedded CSS styles.  
    - Generates a URL-friendly slug and a unique site name using the title and timestamp.  
  - *Expressions:* Uses `$input.first().json` to access the scraped data.  
  - *Connections:* Input from Airbnb Scraper; output to Prepare Binary.  
  - *Version:* Requires n8n version supporting `runOnceForAllItems` mode in Code node v2.  
  - *Edge Cases:*  
    - Missing or malformed data fields handled gracefully with defaults.  
    - Large image arrays trimmed to 6.  
    - Possible errors if input JSON structure differs from expected schema.  
    - HTML encoding and sanitization is minimal, assume trusted data source.

---

#### 2.4 Binary Preparation & Packaging

**Overview:**  
Converts the generated HTML string into a binary file named `index.html` and compresses it into a ZIP archive for deployment.

**Nodes Involved:**  
- Prepare Binary  
- Create ZIP  

**Node Details:**

- **Prepare Binary**  
  - *Type:* Code node  
  - *Role:* Converts the HTML text into a binary file buffer suitable for HTTP upload.  
  - *Key Logic:*  
    - Uses n8n helper method `prepareBinaryData` to create a binary property `data` with MIME type `text/html`.  
    - Passes along site metadata (`siteName`, `title`) in JSON.  
  - *Connections:* Input from Generate HTML Site; output to Create ZIP.  
  - *Edge Cases:* Handling of buffer creation errors (rare).  

- **Create ZIP**  
  - *Type:* Compression node  
  - *Role:* Compresses the binary HTML file into a ZIP archive named `site.zip`.  
  - *Configuration:*  
    - Operation: compress  
    - Output format: zip  
    - Input binary property: `data` (from Prepare Binary)  
  - *Connections:* Input from Prepare Binary; output to Create Netlify Site.  
  - *Edge Cases:* ZIP compression failures if binary data corrupted.

---

#### 2.5 Netlify Deployment

**Overview:**  
Creates a new site on Netlify using the generated site name and deploys the ZIP archive as the site content.

**Nodes Involved:**  
- Create Netlify Site  
- Deploy ZIP  

**Node Details:**

- **Create Netlify Site**  
  - *Type:* HTTP Request node  
  - *Role:* Calls Netlify API to create a new site with a unique name.  
  - *Configuration:*  
    - Method: POST to `https://api.netlify.com/api/v1/sites`  
    - Body: JSON with `{ name: siteName }` from Prepare Binary output.  
    - Auth: Bearer token using Netlify API Token credentials.  
    - Headers: `Content-Type: application/json`  
  - *Connections:* Input from Create ZIP; output to Deploy ZIP.  
  - *Edge Cases:*  
    - Name conflicts or invalid names rejected by Netlify API.  
    - Invalid or expired API token causes auth failure.  
    - Network errors or rate limits.

- **Deploy ZIP**  
  - *Type:* HTTP Request node  
  - *Role:* Uploads the ZIP archive to the created Netlify site to trigger deployment.  
  - *Configuration:*  
    - Method: POST to `https://api.netlify.com/api/v1/sites/{site_id}/builds` where `site_id` is from Create Netlify Site response.  
    - Body: Binary ZIP data from Create ZIP node.  
    - Auth: Bearer token same as above.  
    - Headers: `Content-Type: application/zip`  
  - *Connections:* Input from Create Netlify Site; output to Output Results.  
  - *Edge Cases:*  
    - Upload failures due to connection or file corruption.  
    - Invalid site ID or unauthorized token.

---

#### 2.6 Output & Reporting

**Overview:**  
Aggregates deployment details and outputs a success message with site URLs and other metadata.

**Nodes Involved:**  
- Output Results  

**Node Details:**

- **Output Results**  
  - *Type:* Code node  
  - *Role:* Collects JSON responses from Netlify site creation and deployment, along with site metadata, and returns a unified success object.  
  - *Outputs:*  
    - `success`: true  
    - `message`: Deployment success notification  
    - `title`: Listing title  
    - `siteUrl`: Public URL of the deployed site (SSL preferred)  
    - `adminUrl`: Netlify admin dashboard URL for the site  
    - `siteId`: Netlify site identifier  
    - `deployId`: Deployment ID  
    - `deployState`: Deployment status  
    - `note`: User guidance message  
  - *Connections:* Input from Deploy ZIP; terminal node.  
  - *Edge Cases:* If previous nodes fail, no output or partial data may be present.

---

### 3. Summary Table

| Node Name           | Node Type               | Functional Role                | Input Node(s)      | Output Node(s)       | Sticky Note                                                                                                                             |
|---------------------|-------------------------|-------------------------------|--------------------|----------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger      | manualTrigger           | Start workflow manually         | -                  | Set Listing ID       | ## ⚙️ Configuration - Edit the listingId field below with your Airbnb listing ID.                                                       |
| Set Listing ID      | set                     | Define Airbnb listing ID        | Manual Trigger     | Airbnb Scraper       | ## ⚙️ Configuration - Edit the listingId field below with your Airbnb listing ID.                                                       |
| Airbnb Scraper      | airbnbScraper           | Scrape Airbnb listing data      | Set Listing ID     | Generate HTML Site   | ## 🔍 Scrape Data - Fetches comprehensive listing data including title, photos, amenities, pricing, host info, reviews                |
| Generate HTML Site  | code                    | Generate responsive HTML site   | Airbnb Scraper     | Prepare Binary       | ## 🎨 Generate Site and Zip File - Creates a beautiful, responsive HTML page with photo gallery, property details, booking form, host info |
| Prepare Binary      | code                    | Convert HTML to binary file     | Generate HTML Site | Create ZIP           | ## 🎨 Generate Site and Zip File - Creates a beautiful, responsive HTML page with photo gallery, property details, booking form, host info |
| Create ZIP          | compression             | Compress HTML file as ZIP       | Prepare Binary     | Create Netlify Site  | ## 🎨 Generate Site and Zip File - Creates a beautiful, responsive HTML page with photo gallery, property details, booking form, host info |
| Create Netlify Site | httpRequest             | Create new Netlify site         | Create ZIP         | Deploy ZIP           | ## 🚀 Deploy to Netlify - Publishes the site to Netlify's free hosting. Site URL returned in output. Save site ID for updates.          |
| Deploy ZIP          | httpRequest             | Upload ZIP to Netlify deployment| Create Netlify Site | Output Results       | ## 🚀 Deploy to Netlify - Publishes the site to Netlify's free hosting. Site URL returned in output. Save site ID for updates.          |
| Output Results      | code                    | Output deployment results       | Deploy ZIP         | -                    | ## 🚀 Deploy to Netlify - Publishes the site to Netlify's free hosting. Site URL returned in output. Save site ID for updates.          |
| Sticky Note         | stickyNote              | Workflow overview               | -                  | -                    | ## 🏠 Direct Booking Site Generator - Workflow purpose, how it works, required credentials                                               |
| Sticky Note1        | stickyNote              | Configuration instructions      | -                  | -                    | ## ⚙️ Configuration - Edit the listingId field below with your Airbnb listing ID                                                        |
| Sticky Note2        | stickyNote              | Scrape data description         | -                  | -                    | ## 🔍 Scrape Data - Fetches comprehensive listing data including title, photos, amenities, pricing, host info, reviews                 |
| Sticky Note3        | stickyNote              | Site generation description     | -                  | -                    | ## 🎨 Generate Site and Zip File - Creates a beautiful, responsive HTML page with photo gallery, property details, booking form, host info |
| Sticky Note4        | stickyNote              | Deployment description          | -                  | -                    | ## 🚀 Deploy to Netlify - Publishes the site to Netlify's free hosting. Site URL returned in output. Save site ID for updates.          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - No configuration needed. This will start the workflow manually.

2. **Add a Set node named "Set Listing ID"**  
   - Add a string field named `listingId`.  
   - Set its value to the Airbnb listing ID you want to process (e.g., `"1501733424018064312"`).  
   - Connect Manual Trigger’s output to this node.

3. **Add an Airbnb Scraper node**  
   - Set operation to `scrapeListing`.  
   - Set `listingId` parameter to expression: `{{$json["listingId"]}}`.  
   - Configure credentials for Airbnb Scraper API (token from https://scraper.shortrentals.ai).  
   - Increase timeout to 120 seconds.  
   - Connect output of Set Listing ID to this node.

4. **Add a Code node named "Generate HTML Site"**  
   - Set to mode `runOnceForAllItems`.  
   - Paste the JavaScript code provided (which builds the full HTML page from scraped JSON).  
   - Connect Airbnb Scraper output here.

5. **Add a Code node named "Prepare Binary"**  
   - Set to mode `runOnceForAllItems`.  
   - Paste JavaScript code that converts the generated HTML string into a binary file named `index.html` with MIME type `text/html`.  
   - Connect from Generate HTML Site.

6. **Add a Compression node named "Create ZIP"**  
   - Operation: Compress  
   - Output format: ZIP  
   - Binary property to compress: `data` (binary from Prepare Binary)  
   - File name: `site.zip`  
   - Connect from Prepare Binary.

7. **Add an HTTP Request node named "Create Netlify Site"**  
   - Method: POST  
   - URL: `https://api.netlify.com/api/v1/sites`  
   - Body content type: JSON  
   - Body: `{"name": "={{$json["siteName"]}}"}` (expression to use siteName from Prepare Binary)  
   - Authentication: HTTP Bearer Auth with Netlify API key credential (generate token at https://app.netlify.com/user/applications#personal-access-tokens)  
   - Header: Content-Type: application/json  
   - Connect from Create ZIP node.

8. **Add an HTTP Request node named "Deploy ZIP"**  
   - Method: POST  
   - URL: `https://api.netlify.com/api/v1/sites/{{$json["site_id"]}}/builds` (dynamic site_id from previous node)  
   - Content-Type header: application/zip  
   - Authentication: HTTP Bearer Auth with same Netlify API credential  
   - Body: Binary data from Create ZIP node (`{{$binary.data}}`)  
   - Connect from Create Netlify Site.

9. **Add a Code node named "Output Results"**  
   - Mode: runOnceForAllItems  
   - Paste JS code to aggregate deployment data and output a success object with URLs and messages.  
   - Connect from Deploy ZIP.

10. **Optional: Add Sticky Note nodes** to document the workflow steps inside n8n visually, using the content provided in the Sticky Note nodes for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow creates a direct booking site from Airbnb listings and deploys to Netlify automatically. | Overview from the Sticky Note titled "Direct Booking Site Generator".                           |
| Airbnb Scraper API token required, available at [shortrentals.ai](https://scraper.shortrentals.ai) | Credential setup for Airbnb data scraping.                                                     |
| Netlify Personal Access Token required for API access, create at [Netlify Tokens](https://app.netlify.com/user/applications#personal-access-tokens) | Credential setup for Netlify API authentication.                                               |
| Site URL and Netlify admin URL returned after deployment for easy access to live site and management. | Deployment info provided in Output Results node.                                               |
| The generated site includes a responsive photo gallery, booking form, host info, and house rules. | Design and UX details embedded in the Generate HTML Site code node.                            |
| The booking form on the site is static and does not process payments; it is for display only.     | Implied by generated static HTML without backend booking integration.                          |
| Site names are sanitized and suffixed with a timestamp-based slug for uniqueness on Netlify.      | Prevents name collisions during site creation.                                                |
| First deployment creates a new site; for updates, you must save and reuse the site ID (not included in this workflow). | Important for versioning or updating deployed sites.                                          |

---

**Disclaimer:** The provided text is exclusively generated from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.