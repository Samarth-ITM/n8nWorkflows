Competitive Ad Research & Image Generator with Apify, GPT-4o & Facebook Ad Library

https://n8nworkflows.xyz/workflows/competitive-ad-research---image-generator-with-apify--gpt-4o---facebook-ad-library-6172


# Competitive Ad Research & Image Generator with Apify, GPT-4o & Facebook Ad Library

### 1. Workflow Overview

This workflow automates competitive ad research and AI-powered image ad generation using Facebook's Ad Library, Apify scraper, OpenAI GPT-4o for image analysis and prompt spinning, and Google Drive & Google Sheets for asset management and logging. It is designed for marketers and advertisers seeking to streamline the process of collecting competitor ads, analyzing their content, generating creative variations, and organizing them for easy campaign deployment.

The workflow is logically divided into four main blocks:

- **1.1 Facebook Ad Library Scraping:** Uses Apify to scrape active competitor ads from Facebook’s Ad Library, filters to retain only image-based ads, and limits the batch size for manageability.
- **1.2 Google Drive Organization:** Creates a structured folder hierarchy in Google Drive to store original ads and AI-generated variations.
- **1.3 AI Analysis & Prompt Generation:** Uses OpenAI’s vision model to analyze ad images and GPT-4 to spin three stylistic variation prompts per ad.
- **1.4 AI Image Generation & Storage:** Generates new ad images based on spun prompts, uploads them to Google Drive, and logs metadata and links to Google Sheets for campaign tracking.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Facebook Ad Library Scraping

**Overview:**  
This block scrapes active Facebook ads using Apify's scraper API, filters out ads without images, and limits the number of ads for processing.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Google Drive (to initialize folder and database creation)  
- Google Sheets1 (creates Google Sheet titled “PPC Thievery” with a “scraped_ads” sheet)  
- Edit Fields1 (initializes empty variables for sheet columns)  
- Google Sheets2 (appends data to the Google Sheet)  
- Set Variables (sets Google Drive folder ID, spreadsheet ID, and prompt change request text)  
- Run Ad Library Scraper (HTTP POST request to Apify API)  
- Filter (keeps only ads with images)  
- Limit (restricts batch size for testing)  

**Node Details:**  
- **When clicking ‘Execute workflow’**: Manual trigger to start the process.  
- **Google Drive**: Creates or references a parent folder “PPC Thievery” in “My Drive” root; acts as base folder for assets.  
- **Google Sheets1**: Creates a new Google Sheet titled “PPC Thievery” with a “scraped_ads” tab.  
- **Edit Fields1**: Initializes all expected sheet columns with empty strings; prepares for data append.  
- **Google Sheets2**: Appends rows to the “scraped_ads” sheet using data from the workflow (ads metadata, folder URLs, etc.).  
- **Set Variables**: Holds critical IDs and the “changeRequest” prompt template. Must be updated manually after initial folder/sheet creation.  
- **Run Ad Library Scraper**: Sends a POST request to Apify’s Facebook Ad Library scraper with parameters: count=20 ads, period=last 7 days, active status=active, country=US, keyword query “agency”. Requires an Apify API key in the Authorization header.  
- **Filter**: Removes ads lacking an original image URL (video ads or image-less ads).  
- **Limit**: Limits output to 2 ads for testing; adjustable for production.

**Potential Failures & Edge Cases:**  
- Apify API key missing or invalid → auth error.  
- API rate limits or timeouts.  
- Ads without images filtered out; if all ads are video-only, output is empty.  
- Folder or sheet creation failures due to permission issues.  
- Manual update needed for folder and spreadsheet IDs in “Set Variables.”

---

#### 1.2 Google Drive Organization

**Overview:**  
Creates a hierarchical folder structure in Google Drive to organize original competitor ads and AI-generated spun assets.

**Nodes Involved:**  
- Create Asset Parent Folder  
- Create Child Source Folder  
- Create Child Spun Folder  
- Download Static Image Ad  

**Node Details:**  
- **Create Asset Parent Folder:** Creates a folder named after the ad archive ID inside the main Google Drive folder (ID from Set Variables). This serves as the container for all assets related to a particular ad scrape batch.  
- **Create Child Source Folder:** Creates a subfolder named “1. Source Assets” inside the parent folder to store original ads.  
- **Create Child Spun Folder:** Creates a sibling subfolder named “2. Spun Assets” inside the parent folder to store AI-generated variations.  
- **Download Static Image Ad:** Downloads the original ad image from the URL provided in the ad data into the Source Assets folder.

**Potential Failures & Edge Cases:**  
- Folder creation may fail due to invalid Drive IDs or permission issues.  
- Download may fail if the original image URL is invalid or inaccessible.  
- Naming collisions in folders unlikely due to unique ad archive IDs but should be monitored.

---

#### 1.3 AI Analysis & Prompt Generation

**Overview:**  
Analyzes each ad image with OpenAI’s vision model to generate a comprehensive description, then uses GPT-4 to create three stylistic variation “spin” prompts based on a predefined change request.

**Nodes Involved:**  
- Google Drive2 (shares the downloaded image file to make it accessible for OpenAI)  
- OpenAI (Vision model for image description)  
- Spin Prompts (GPT-4 prompt rewriting for variation generation)  
- Split Out (splits JSON array of variants into individual items)  
- Edit Fields (adds variant text and image URL to data for looping)  

**Node Details:**  
- **Google Drive2:** Shares the image file with “writer” permission to “anyone” to ensure OpenAI can access the image URL.  
- **OpenAI (Vision):** Uses “chatgpt-4o-latest” model with an “analyze” operation on the shared image URL to generate a detailed description of the ad image.  
- **Spin Prompts:** Uses GPT-4.1 with temperature 0.7 to rewrite the image description into three different stylistic change requests, incorporating the company name “LeftClick” and focusing on AI automation messaging. Outputs JSON with three prompt variants.  
- **Split Out:** Separates the JSON array “variants” from Spin Prompts into individual messages for downstream processing.  
- **Edit Fields:** Assigns each variant prompt and the original ad image URL to properties for the loop processing.

**Potential Failures & Edge Cases:**  
- OpenAI image analysis may fail due to inaccessible image URLs or API limits.  
- GPT prompt rewriting could produce empty or malformed JSON—expression parsing must be robust.  
- Sharing permissions must be correctly set to avoid 403 errors.  
- Variants must always include exactly three entries; deviations may break downstream logic.

---

#### 1.4 AI Image Generation & Storage

**Overview:**  
Generates AI-driven ad image variations based on spun prompts, uploads new images to Google Drive spun assets folder, and logs each variation with metadata and links in Google Sheets.

**Nodes Involved:**  
- Loop Over Items (processes each spun prompt individually)  
- Download Static Image Ad1 (downloads the original ad image for editing)  
- Generate Image Using GPT Image 1 (OpenAI image edit API)  
- Convert to File (converts base64 image response to a binary file)  
- Google Drive3 (uploads generated image to “Spun Assets” folder)  
- Google Sheets (logs ad variation metadata and Drive links)  
- Wait (rate-limits cycle to avoid API throttling)

**Node Details:**  
- **Loop Over Items:** Iterates over each spun prompt variant to process images one at a time, preventing concurrency issues.  
- **Download Static Image Ad1:** Downloads the original source image again for use as the base image in the OpenAI image edit request.  
- **Generate Image Using GPT Image 1:** Uses OpenAI’s “gpt-image-1” model to generate an edited image based on the spun prompt. Multipart form-data request includes the base image and prompt text, fixed output size 1024x1024. Requires OpenAI API key credential.  
- **Convert to File:** Converts the returned base64 JSON image data to a binary file for upload.  
- **Google Drive3:** Uploads the generated image file to the “2. Spun Assets” folder created earlier.  
- **Google Sheets:** Appends a row to the “scraped_ads” sheet logging ad metadata, spun prompt, folder URLs, and the direct link to the spun image.  
- **Wait:** Pauses 1 second between iterations to respect API rate limits.

**Potential Failures & Edge Cases:**  
- Image generation may fail due to invalid prompts or API quota exhaustion.  
- Download of original image must succeed for edit to work.  
- Google Drive upload may fail if folder permissions or IDs are invalid.  
- Google Sheets logging requires correct spreadsheet and sheet names.  
- Rate limiting might need adjustment for large batches or different API limits.

---

### 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                              | Input Node(s)                     | Output Node(s)                   | Sticky Note                                                                                                          |
|-----------------------------|--------------------------------|----------------------------------------------|----------------------------------|----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Entry point to start workflow                  |                                  | Google Drive                    |                                                                                                                      |
| Google Drive                | Google Drive Folder             | Creates main asset folder                       | When clicking ‘Execute workflow’ | Google Sheets1                  |                                                                                                                      |
| Google Sheets1              | Google Sheets                  | Creates Google Sheet “PPC Thievery” with tab  | Google Drive                    | Edit Fields1                   |                                                                                                                      |
| Edit Fields1                | Set                           | Initializes empty sheet columns                 | Google Sheets1                  | Google Sheets2                 |                                                                                                                      |
| Google Sheets2              | Google Sheets                  | Appends data rows to Google Sheet               | Edit Fields1                   |                                  |                                                                                                                      |
| Set Variables               | Set                           | Stores folder and sheet IDs, change request text| Google Sheets2                 | Run Ad Library Scraper          |                                                                                                                      |
| Run Ad Library Scraper      | HTTP Request                  | Scrapes Facebook ads via Apify API              | Set Variables                  | Filter                        | ## 📱 STEP 1: Facebook Ad Library Scraping\nScrapes Facebook's Ad Library for competitor analysis.                    |
| Filter                     | Filter                        | Filters ads with images only                     | Run Ad Library Scraper          | Limit                         |                                                                                                                      |
| Limit                      | Limit                         | Limits number of ads processed                    | Filter                        | Create Asset Parent Folder      |                                                                                                                      |
| Create Asset Parent Folder  | Google Drive Folder             | Creates parent folder for ad batch                | Limit                         | Create Child Source Folder      | ## 🏗️ STEP 2: Google Drive Organization\nCreates structured folder system for asset management.                     |
| Create Child Source Folder  | Google Drive Folder             | Creates “Source Assets” subfolder                 | Create Asset Parent Folder      | Create Child Spun Folder        |                                                                                                                      |
| Create Child Spun Folder    | Google Drive Folder             | Creates “Spun Assets” subfolder                   | Create Child Source Folder      | Download Static Image Ad        |                                                                                                                      |
| Download Static Image Ad    | HTTP Request                  | Downloads original ad image                        | Create Child Spun Folder        | Google Drive1                  |                                                                                                                      |
| Google Drive1              | Google Drive File Upload       | Uploads original ad image to source folder         | Download Static Image Ad        | Google Drive2                  |                                                                                                                      |
| Google Drive2              | Google Drive Share             | Shares uploaded image publicly for OpenAI access | Google Drive1                  | OpenAI                        |                                                                                                                      |
| OpenAI                    | OpenAI Vision Model            | Analyzes image and generates detailed description | Google Drive2                  | Spin Prompts                  | ## 🤖 STEP 3: AI Analysis & Prompt Generation\nAnalyzes competitor ads and generates variation prompts.               |
| Spin Prompts               | OpenAI GPT-4                  | Generates 3 stylistic prompt variants              | OpenAI                        | Split Out                    |                                                                                                                      |
| Split Out                  | Split Out                     | Splits prompt variants array for individual processing | Spin Prompts                 | Edit Fields                   |                                                                                                                      |
| Edit Fields                | Set                           | Assigns variant prompts and image URL for looping  | Split Out                    | Loop Over Items               |                                                                                                                      |
| Loop Over Items            | Split In Batches              | Processes each spun prompt one by one              | Edit Fields                   | Download Static Image Ad1       |                                                                                                                      |
| Download Static Image Ad1  | HTTP Request                  | Downloads original ad image again for editing      | Loop Over Items               | Generate Image Using GPT Image 1 |                                                                                                                      |
| Generate Image Using GPT Image 1 | HTTP Request              | Generates edited image from spun prompt             | Download Static Image Ad1      | Convert to File               | ## 🎨 STEP 4: AI Image Generation & Storage\nGenerates new ad variations and organizes results.                     |
| Convert to File            | Convert To File               | Converts base64 image to binary file                 | Generate Image Using GPT Image 1 | Google Drive3                |                                                                                                                      |
| Google Drive3              | Google Drive File Upload       | Uploads generated image to spun assets folder        | Convert to File               | Google Sheets                 |                                                                                                                      |
| Google Sheets              | Google Sheets                  | Logs ad variation metadata & links in spreadsheet   | Google Drive3                 | Wait                         |                                                                                                                      |
| Wait                      | Wait                          | Rate-limits processing loop                           | Google Sheets                 | Loop Over Items (second output) |                                                                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node:**  
   - Name: When clicking ‘Execute workflow’  
   - No parameters.

2. **Create Google Drive folder node:**  
   - Name: Google Drive  
   - Operation: Create folder  
   - Folder name: "PPC Thievery"  
   - Drive: My Drive (root folder)  

3. **Create Google Sheets node (Google Sheets1):**  
   - Name: Google Sheets1  
   - Operation: Create spreadsheet  
   - Title: "PPC Thievery"  
   - Add sheet named: "scraped_ads"  

4. **Create Set node (Edit Fields1):**  
   - Name: Edit Fields1  
   - Initialize all columns expected in the sheet (e.g., timestamp, ad_archive_id, page_id, original_image_url, etc.) with empty strings.

5. **Create Google Sheets append node (Google Sheets2):**  
   - Name: Google Sheets2  
   - Operation: Append data  
   - Document ID: from Google Sheets1 output  
   - Sheet name: "scraped_ads"  
   - Map columns matching initialized fields.

6. **Create Set Variables node:**  
   - Name: Set Variables  
   - Define variables:  
     - googleDriveFolderId (string): ID of "PPC Thievery" folder (from Google Drive node)  
     - spreadsheetId (string): ID of Google Sheet (from Google Sheets1)  
     - changeRequest (string): Prompt template for AI spinning (e.g., "Spin this ad so that...")  
   - This node is to be updated manually after initial folder and sheet creation.

7. **Create HTTP Request node for Apify scraper:**  
   - Name: Run Ad Library Scraper  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/XtaWFhbtfxyzqrFmd/run-sync-get-dataset-items`  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer `<your-apify-api-key-here>` (replace with actual key)  
   - Body (JSON):  
     ```json
     {
       "count": 20,
       "period": "last7d",
       "scrapeAdDetails": true,
       "scrapePageAds.activeStatus": "active",
       "urls": [
         {
           "url": "https://www.facebook.com/ads/library/?active_status=active&ad_type=all&country=US&is_targeted_country=false&media_type=all&q=agency&search_type=keyword_unordered",
           "method": "GET"
         }
       ]
     }
     ```  
   - Send body as JSON.

8. **Add Filter node:**  
   - Name: Filter  
   - Condition: Keep only items where `$json.snapshot.images[0].original_image_url` exists (non-empty).

9. **Add Limit node:**  
   - Name: Limit  
   - Max items: 2 (for testing; increase for production).

10. **Create Google Drive folder hierarchy nodes:**  
    - Create Asset Parent Folder:  
      - Name: Create Asset Parent Folder  
      - Folder name: `={{ $json.ad_archive_id }}`  
      - Parent folder ID: `={{ $json.googleDriveFolderId }}` (from Set Variables)  
    - Create Child Source Folder:  
      - Name: Create Child Source Folder  
      - Folder name: "1. Source Assets"  
      - Parent folder ID: output of Create Asset Parent Folder  
    - Create Child Spun Folder:  
      - Name: Create Child Spun Folder  
      - Folder name: "2. Spun Assets"  
      - Parent folder ID: output of Create Asset Parent Folder  

11. **Download original ad image:**  
    - Name: Download Static Image Ad  
    - URL: `={{ $json.snapshot.images[0].original_image_url }}` (from Run Ad Library Scraper output)

12. **Upload original image to Google Drive source folder:**  
    - Name: Google Drive1  
    - File name: `={{ $binary.data.fileName }}`  
    - Folder ID: output of Create Child Source Folder  

13. **Share the uploaded image:**  
    - Name: Google Drive2  
    - Operation: Share file  
    - File ID: `={{ $json.id }}` (from Google Drive1)  
    - Permissions: Role = writer, Type = anyone  

14. **Use OpenAI Vision model to analyze image:**  
    - Name: OpenAI  
    - Resource: Image  
    - Operation: Analyze  
    - Model: chatgpt-4o-latest  
    - Image URL: `=https://drive.google.com/uc?export=download&id={{ $('Google Drive1').item.json.id }}`  
    - Prompt: "What's in this image? Describe it extremely comprehensively. Leave nothing out."

15. **Spin prompts with GPT-4:**  
    - Name: Spin Prompts  
    - Model: gpt-4.1  
    - Temperature: 0.7  
    - System prompt: Explains task to generate 3 stylistic prompt variants in JSON format.  
    - User prompt: Includes OpenAI vision output and the changeRequest variable.

16. **Split out prompt variants:**  
    - Name: Split Out  
    - Field to split: `message.content.variants` (from Spin Prompts output)

17. **Edit fields to assign variant and image URL:**  
    - Name: Edit Fields  
    - Assign fields:  
      - variant = `={{ $json["message.content.variants"] }}`  
      - imageAdUrl = `={{ $('Filter').item.json.snapshot.images[0].original_image_url }}`

18. **Loop over variants:**  
    - Name: Loop Over Items  
    - Split items one by one for sequential processing.

19. **Download original image again:**  
    - Name: Download Static Image Ad1  
    - URL: `={{ $json.imageAdUrl }}` (from Edit Fields)

20. **Generate image using OpenAI image edit API:**  
    - Name: Generate Image Using GPT Image 1  
    - Method: POST  
    - URL: `https://api.openai.com/v1/images/edits`  
    - Authentication: OpenAI API key credential  
    - Body (multipart/form-data):  
      - model: "gpt-image-1"  
      - image[]: binary data from Download Static Image Ad1  
      - prompt: `={{ $json.variant }}`  
      - size: "1024x1024"

21. **Convert returned base64 image to binary file:**  
    - Name: Convert to File  
    - Operation: toBinary  
    - Property: `data[0].b64_json` (from Generate Image Using GPT Image 1)

22. **Upload generated image to spun assets folder:**  
    - Name: Google Drive3  
    - File name: `={{ $('Google Drive1').item.json.name }}` (or appropriate naming convention)  
    - Folder ID: output of Create Child Spun Folder

23. **Log ad variation in Google Sheets:**  
    - Name: Google Sheets  
    - Operation: Append row  
    - Document ID: from Set Variables or Google Sheets1  
    - Sheet name: "scraped_ads"  
    - Columns: Map metadata including ad_body, page_id, spun_prompts, folder URLs, direct spun image link, timestamps, etc.

24. **Add Wait node for rate limiting:**  
    - Name: Wait  
    - Duration: 1 second  

25. **Connect Wait node output back to Loop Over Items to process next variant.**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| **Replace `<your-apify-api-key-here>` with your Apify API key in the HTTP request header for scraping.**        | Apify API authentication                                                                         |
| **OpenAI credentials required for GPT-4 and GPT vision models, plus image edit API.**                            | OpenAI API key setup                                                                             |
| **Google Drive and Google Sheets require OAuth2 credentials with write permissions to create folders, upload files, and edit spreadsheets.** | n8n credential configuration                                                                     |
| **The workflow's “Set Variables” node must be manually updated with actual folder and spreadsheet IDs after initial run.** | Workflow configuration step                                                                      |
| Video and course material referenced in sticky notes relate to n8n course for advanced automation techniques.    | https://n8n.io/learning (example resource)                                                      |
| The change request in the “Set Variables” node uses brand name “LeftClick” and specifies detailed style instructions for image prompt spinning. | Adapt this text to your brand and target audience                                               |
| The workflow uses rate limiting (Wait node) to prevent API throttling during image generation loop.              | Adjust wait duration based on API quota                                                        |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It strictly adheres to applicable content policies and contains no illegal, offensive, or protected content. All data processed is legal and publicly available.