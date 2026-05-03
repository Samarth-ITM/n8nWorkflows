Clone & Customize Competitor Facebook Ads with Gemini AI and Apify

https://n8nworkflows.xyz/workflows/clone---customize-competitor-facebook-ads-with-gemini-ai-and-apify-8415


# Clone & Customize Competitor Facebook Ads with Gemini AI and Apify

### 1. Workflow Overview

This workflow automates the process of cloning and customizing competitor Facebook ads using AI and web scraping. It targets marketers and advertisers who want to quickly generate new ad creatives inspired by competitors’ Facebook ads but featuring their own product branding and packaging.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Collects user input via a form including a Facebook Ad Library URL and a product image.
- **1.2 Data Acquisition**: Scrapes competitor ads from the provided URL using Apify’s Facebook Ad Library Scraper.
- **1.3 Ad Iteration and Processing**: Iterates over each scraped ad, downloads the competitor ad image, converts it for AI processing, and uploads original ads for reference.
- **1.4 AI Prompt Generation**: Builds a detailed textual prompt for AI image generation, describing how to clone and customize the ad.
- **1.5 AI Image Generation and Content Filtering**: Uses Google Gemini AI to generate a new ad image based on the prompt and checks for prohibited content.
- **1.6 Output Handling**: Uploads generated images to Google Drive and loops back for next ad processing.
- **1.7 Aggregation and Finalization**: Aggregates results for further use or review.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Collects user input via a web form with fields for a Facebook Ad Library URL and a product image upload.

- **Nodes Involved:**  
  - form_trigger  
  - convert_product_image_to_base64

- **Node Details:**  
  - **form_trigger**  
    - Type: Form Trigger  
    - Role: Starts workflow by receiving user input.  
    - Config: Form titled "Facebook Ad Thief Form" with required URL input and a single product image file upload.  
    - Outputs: JSON containing URL and binary image file.  
    - Edge cases: Invalid URLs or missing file uploads could cause failures downstream.

  - **convert_product_image_to_base64**  
    - Type: Extract From File  
    - Role: Converts uploaded product image binary data to a base64 string for AI processing.  
    - Config: Converts binary property named "Your_Product_Image" to JSON property.  
    - Input: Binary image from form_trigger  
    - Output: Base64 encoded data in JSON.  
    - Edge cases: Corrupted or unsupported image formats may cause conversion errors.

---

#### 1.2 Data Acquisition

- **Overview:**  
  Scrapes up to 20 competitor ads from the Facebook Ad Library URL provided.

- **Nodes Involved:**  
  - scrape_ads  
  - iterate_ads

- **Node Details:**  
  - **scrape_ads**  
    - Type: Apify (Facebook Ad Library Scraper)  
    - Role: Scrapes ads data from the given URL with parameters to scrape 20 ads, all statuses, and all countries.  
    - Config: Uses Apify OAuth2 credentials, runs actor "XtaWFhbtfxyzqrFmd" with specified input JSON.  
    - Output: Dataset of ads with metadata including images and titles.  
    - Edge cases: Invalid URL, API quota limits, or scraper failures may disrupt data fetching.

  - **iterate_ads**  
    - Type: Split In Batches  
    - Role: Processes ads one by one in batches for sequential handling.  
    - Config: Default batch settings (likely 1 batch per iteration).  
    - Input: Output from scrape_ads  
    - Output: Single ad JSON per iteration.  
    - Edge cases: Empty dataset or batch errors may halt iteration.

---

#### 1.3 Ad Iteration and Processing

- **Overview:**  
  For each scraped ad, downloads the competitor ad image, converts it for AI input, and uploads the original ad image to Google Drive for reference.

- **Nodes Involved:**  
  - download_image  
  - convert_ad_image_to_base64  
  - upload_ad_reference  
  - merge

- **Node Details:**  
  - **download_image**  
    - Type: HTTP Request  
    - Role: Downloads the competitor ad image URL extracted from the current ad JSON batch.  
    - Config: URL set dynamically from ads snapshot data `snapshot.cards.first().original_image_url`.  
    - Output: Binary image data.  
    - Edge cases: Broken or inaccessible image URLs, network timeouts.

  - **convert_ad_image_to_base64**  
    - Type: Extract From File  
    - Role: Converts downloaded competitor ad image binary to base64 string.  
    - Config: Converts binary data to JSON property for AI ingestion.  
    - Input: Binary image from download_image node.  
    - Edge cases: Image conversion errors due to format or corruption.

  - **upload_ad_reference**  
    - Type: Google Drive  
    - Role: Uploads original competitor ad image to Google Drive for archival/reference.  
    - Config: Uses OAuth2 credentials for Google Drive; uploads to folder ID `1HoXBHX3gvz5TxhczPwI0d5gWI3mxJT7h`.  
    - Filename set dynamically from ad title `snapshot.cards.first().title`.  
    - Edge cases: Google Drive API rate limits or permission errors.

  - **merge**  
    - Type: Merge  
    - Role: Combines parallel streams of processing (original ad upload and image conversion) to synchronize for next steps.  
    - Edge cases: Missing inputs on one branch may cause merge failures.

---

#### 1.4 AI Prompt Generation

- **Overview:**  
  Uses Gemini AI (Google generative language models) to produce a detailed prompt that instructs how to clone the competitor ad while replacing branding with the user’s product.

- **Nodes Involved:**  
  - aggregate  
  - build_prompt  
  - generate_ad_image_prompt

- **Node Details:**  
  - **aggregate**  
    - Type: Aggregate  
    - Role: Aggregates data, though specific aggregation fields are empty (likely used to collect or normalize data streams).  
    - Edge cases: Misconfiguration could cause empty or incorrect aggregation.

  - **build_prompt**  
    - Type: Set  
    - Role: Builds a detailed textual prompt for AI describing the cloning instructions and edge cases.  
    - Config: Provides explicit instructions on how to replace competitor branding (“AG1”) with “ThriveMix”, analyzes partial text on packaging, and maintains original ad style with specific notes on text replacement.  
    - Output: JSON property `prompt` with the constructed string.  
    - Edge cases: Improper prompt formation may degrade AI output quality.

  - **generate_ad_image_prompt**  
    - Type: HTTP Request  
    - Role: Calls Gemini 2.5 Pro AI to generate content based on the prompt and two base64 images (product and competitor ad).  
    - Config: Sends JSON body with base64 images as inline data plus prompt text; uses Google Gemini HTTP header auth credentials.  
    - Retry enabled with 5s intervals.  
    - Output: AI response containing detailed content instructions.  
    - Edge cases: API failures, auth errors, or content flagged as prohibited.

---

#### 1.5 AI Image Generation and Content Filtering

- **Overview:**  
  Generates a new ad image using Gemini AI’s image preview model and filters output for prohibited content before proceeding.

- **Nodes Involved:**  
  - generate_ad_image  
  - check_if_prohibited  
  - set_result

- **Node Details:**  
  - **generate_ad_image**  
    - Type: HTTP Request  
    - Role: Sends AI-generated prompt content along with base64 images to Gemini 2.5 Flash Image Preview to generate the new ad image.  
    - Config: Uses HTTP header auth; retries on failure with 5s delay.  
    - Output: AI-generated image data or flagged content response.  
    - Edge cases: Content flagged as prohibited or API errors.

  - **check_if_prohibited**  
    - Type: If  
    - Role: Checks if AI generation was blocked due to prohibited content by inspecting `finishReason` field in AI response.  
    - Condition: `finishReason == "PROHIBITED_CONTENT"`  
    - True branch: Loops back to next ad iteration (skips current output).  
    - False branch: Proceeds to set the AI image result.  
    - Edge cases: Incorrect or unexpected AI response formats.

  - **set_result**  
    - Type: Set  
    - Role: Extracts base64 image data from AI response and assigns it to `image_result` property for downstream processing.  
    - Edge cases: Missing or malformed AI response data.

---

#### 1.6 Output Handling

- **Overview:**  
  Converts AI image base64 string back to binary file and uploads the cloned ad image to Google Drive.

- **Nodes Involved:**  
  - get_image  
  - upload_image

- **Node Details:**  
  - **get_image**  
    - Type: Convert To File  
    - Role: Converts base64 image data string in `image_result` back into binary file format.  
    - Config: Source property is `image_result`.  
    - Output: Binary image file.  
    - Edge cases: Conversion failures if base64 string is invalid.

  - **upload_image**  
    - Type: Google Drive  
    - Role: Uploads the newly generated ad image to a different Google Drive folder for cloned ads.  
    - Config: Uses OAuth2 credentials; folder URL `https://drive.google.com/drive/u/0/folders/1qY588PqswUfGTl3dZ_FbZhiDwK0Kv2Mz`; names files as "Cloned Ad #<index>".  
    - Edge cases: API permission errors, file naming conflicts.

---

#### 1.7 Aggregation and Finalization

- **Overview:**  
  Aggregates processed data and loops back to iterate over remaining ads.

- **Nodes Involved:**  
  - merge  
  - iterate_ads

- **Node Details:**  
  - **merge**  
    - Type: Merge  
    - Role: Joins previous upload and conversion results to continue the flow.  
    - Edge cases: Missing inputs can cause halt.

  - **iterate_ads**  
    - Loops back to process next ad until all ads processed or an exit condition occurs.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                          | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                                                                                                        |
|---------------------------|---------------------------|----------------------------------------|---------------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| form_trigger              | Form Trigger              | Input reception: collects URL & image | -                               | convert_product_image_to_base64  |                                                                                                                                                    |
| convert_product_image_to_base64 | Extract From File        | Converts uploaded product image to base64 | form_trigger                   | scrape_ads                      |                                                                                                                                                    |
| scrape_ads                | Apify (Facebook Ad Scraper) | Scrapes competitor ads from URL         | convert_product_image_to_base64 | iterate_ads                    |                                                                                                                                                    |
| iterate_ads               | Split In Batches          | Iterates over each scraped ad            | scrape_ads                     | download_image, (loop back)      |                                                                                                                                                    |
| download_image            | HTTP Request              | Downloads competitor ad image            | iterate_ads                   | upload_ad_reference, convert_ad_image_to_base64 |                                                                                                                                                    |
| convert_ad_image_to_base64 | Extract From File         | Converts competitor ad image to base64 | download_image                 | merge                          |                                                                                                                                                    |
| upload_ad_reference       | Google Drive              | Uploads original competitor ad image    | download_image                 | merge                          |                                                                                                                                                    |
| merge                     | Merge                     | Joins parallel streams                   | convert_ad_image_to_base64, upload_ad_reference | aggregate                    |                                                                                                                                                    |
| aggregate                 | Aggregate                 | Aggregates data                          | merge                         | build_prompt                   |                                                                                                                                                    |
| build_prompt              | Set                       | Builds AI prompt with detailed instructions | aggregate                     | generate_ad_image_prompt        |                                                                                                                                                    |
| generate_ad_image_prompt  | HTTP Request              | Calls Gemini AI to generate prompt content | build_prompt                 | generate_ad_image              |                                                                                                                                                    |
| generate_ad_image         | HTTP Request              | Generates new ad image from AI prompt   | generate_ad_image_prompt       | check_if_prohibited            |                                                                                                                                                    |
| check_if_prohibited       | If                        | Filters prohibited content from AI output | generate_ad_image             | iterate_ads (if prohibited), set_result (if allowed) |                                                                                                                                                    |
| set_result                | Set                       | Extracts generated image result          | check_if_prohibited            | get_image                     |                                                                                                                                                    |
| get_image                 | Convert To File           | Converts base64 image back to binary file | set_result                   | upload_image                  |                                                                                                                                                    |
| upload_image              | Google Drive              | Uploads cloned ad image                  | get_image                     | iterate_ads                   |                                                                                                                                                    |
| Sticky Note               | Sticky Note               | Nano Banana Facebook Ad Cloning System  | -                             | -                             | ## Nano Banana Facebook Ad Cloning System                                                                                                         |
| Sticky Note1              | Sticky Note               | Detailed workflow overview and instructions | -                           | -                             | ## Setup & Overview\n\nThis n8n template demonstrates how to automatically clone and adapt competitor Facebook ads for your own product using AI. Simply provide a Facebook Ad Library URL and your product image, and the workflow scrapes competitor ads, analyzes their design and messaging, then generates new versions featuring your product while maintaining the original ad's style and effectiveness.\n\n## Use cases\n* Adapt successful competitor ad creatives for your own products\n* Test proven ad formats without starting from scratch\n* Quickly produce multiple ad variations based on high-performing competitors\n* Scale ad creative production by leveraging competitor insights\n* A/B test different visual approaches inspired by market leaders\n\n## Good to know\n* The workflow processes up to 20 ads from the provided Facebook Ad Library URL\n* Gemini's image generation may occasionally flag content as prohibited (workflow handles this automatically)\n* Generated ads maintain the original style while swapping product branding and packaging\n* All competitor reference ads and generated clones are automatically saved to Google Drive\n* Image generation takes approximately 10-30 seconds per ad\n\n## How it works\n1. **Form Submission**: User submits a Facebook Ad Library URL and uploads their product image\n2. **Product Processing**: The product image is converted to base64 for AI processing\n3. **Ad Scraping**: Apify's Facebook Ad Library Scraper extracts up to 20 ads from the provided URL\n4. **Iteration Setup**: The workflow processes each scraped ad individually\n5. **Image Download**: Each competitor ad image is downloaded and converted to base64\n6. **Reference Storage**: Original competitor ads are uploaded to Google Drive for reference\n7. **Prompt Generation**: Gemini 2.5 Pro analyzes both images and creates detailed instructions for cloning the ad while replacing competitor branding with your product\n8. **Ad Generation**: Gemini 2.5 Flash Image Preview generates the new ad image based on the instructions\n9. **Content Filter**: Checks if generation was blocked for prohibited content\n10. **Upload & Loop**: Successfully generated ads are uploaded to Google Drive, then the workflow moves to the next ad\n\n## How to use\n1. Click the form trigger URL to access the submission form\n2. Enter a Facebook Ad Library URL (e.g., from a competitor's page showing active ads)\n3. Upload your product image with clear branding and packaging\n4. Submit the form and wait for processing to complete\n5. Find your cloned ads and reference images organized in Google Drive folders\n6. Review generated ads and select the best performers for your campaigns\n\n## Requirements\n* **Apify** account for Facebook Ad Library scraping\n* **Google Gemini API** account for ad analysis and image generation\n* **Google Drive** account for storing reference ads and generated clones\n* Valid Facebook Ad Library URL with accessible ads\n\n## Customizing this workflow\n* Adjust the **number of ads scraped** in the scrape_ads node (currently set to 20 per source)\n* Modify the **prompt instructions** in build_prompt node to emphasize different aspects (e.g., color schemes, layouts, text placement)\n* Change the **Google Drive folders** in upload_ad_reference and upload_image nodes to organize by campaign or product line\n* Add **text overlay generation** to include custom headlines or CTAs on generated images\n* Implement **quality scoring** to automatically filter and rank generated ad variations\n* Add **Slack/email notifications** when ad generation completes or fails\n* Include **metadata extraction** to capture ad copy and targeting insights from scraped adsRetryClaude does not have the ability to run the code it generates yet." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure form title: "Facebook Ad Thief Form"  
   - Add two fields:  
     - Text input (required): Label "Facebook Ad Library Url", placeholder "https://www.facebook.com/"  
     - File upload (required): Label "Your Product Image", single file only  
   - Save.

2. **Add Extract From File Node to Convert Product Image**  
   - Type: Extract From File  
   - Binary Property Name: "Your_Product_Image"  
   - Operation: binaryToProperty (convert to base64 string)  
   - Connect output of form_trigger to this node.

3. **Add Apify Node to Scrape Facebook Ads**  
   - Type: Apify (Facebook Ad Library Scraper actor)  
   - Use OAuth2 credential for Apify  
   - Configure actor ID to "XtaWFhbtfxyzqrFmd" (Facebook Ad Library Scraper)  
   - Set input JSON:  
     ```json
     {
       "limitPerSource": 20,
       "scrapeAdDetails": false,
       "scrapePageAds.activeStatus": "all",
       "scrapePageAds.countryCode": "ALL",
       "urls": [
         {
           "url": "={{ $('form_trigger').item.json['Facebook Ad Library Url'] }}",
           "method": "GET"
         }
       ]
     }
     ```  
   - Connect output of convert_product_image_to_base64 to this node.

4. **Add Split In Batches Node to Iterate Ads**  
   - Type: Split In Batches  
   - Default batch size 1 (process ads one at a time)  
   - Connect output of scrape_ads to this node.

5. **Add HTTP Request Node to Download Competitor Ad Image**  
   - Type: HTTP Request  
   - URL: `={{ $node['iterate_ads'].json.snapshot.cards.first().original_image_url }}`  
   - Method: GET  
   - Connect output of iterate_ads (first output) to this node.

6. **Add Extract From File Node to Convert Downloaded Ad Image**  
   - Type: Extract From File  
   - Operation: binaryToProperty  
   - Connect output of download_image to this node.

7. **Add Google Drive Node to Upload Original Ad Reference**  
   - Type: Google Drive  
   - Credential: Google Drive OAuth2  
   - Folder ID: `1HoXBHX3gvz5TxhczPwI0d5gWI3mxJT7h` (replace with your folder)  
   - File name: `={{ $node['iterate_ads'].json.snapshot.cards.first().title }}`  
   - Connect output of download_image to this node.

8. **Add Merge Node to Join Converted Image and Upload Streams**  
   - Type: Merge  
   - Connect outputs of convert_ad_image_to_base64 (index 1) and upload_ad_reference (index 0) to this merge node.

9. **Add Aggregate Node**  
   - Type: Aggregate  
   - Default configuration (used to collect results)  
   - Connect output of merge node.

10. **Add Set Node to Build AI Prompt**  
    - Type: Set  
    - Add field "prompt" with the detailed instructions text (as per the build_prompt node content)  
    - Connect output of aggregate node.

11. **Add HTTP Request Node to Call Gemini 2.5 Pro for Prompt Generation**  
    - Type: HTTP Request  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-pro:generateContent`  
    - Method: POST  
    - Authentication: HTTP Header Auth (Google Gemini credentials)  
    - JSON body includes:  
      - The "prompt" text from build_prompt  
      - Inline base64 images from product and competitor ad  
    - Enable retry on fail with 5s delay  
    - Connect output of build_prompt node.

12. **Add HTTP Request Node to Generate Ad Image with Gemini 2.5 Flash Image Preview**  
    - Type: HTTP Request  
    - URL: `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent`  
    - Method: POST  
    - Authentication: HTTP Header Auth (Google Gemini credentials)  
    - JSON body includes:  
      - Text from AI prompt result  
      - Inline base64 images as above  
    - Enable retry on fail with 5s delay  
    - Connect output of generate_ad_image_prompt node.

13. **Add If Node to Check for Prohibited Content**  
    - Type: If  
    - Condition: `{{$json.candidates.first().finishReason}} == "PROHIBITED_CONTENT"`  
    - Connect true branch back to iterate_ads node to skip and process next ad  
    - Connect false branch to set_result node.

14. **Add Set Node to Extract Generated Image Result**  
    - Type: Set  
    - Create field "image_result" with value:  
      `={{ $json.candidates[0].content.parts.filter(item => item.inlineData).first().inlineData.data }}`  
    - Connect output of check_if_prohibited (false branch).

15. **Add Convert To File Node to Convert Base64 Image to Binary**  
    - Type: Convert To File  
    - Source Property: "image_result"  
    - Connect output of set_result node.

16. **Add Google Drive Node to Upload Cloned Ad Image**  
    - Type: Google Drive  
    - Credential: Google Drive OAuth2  
    - Folder URL: `https://drive.google.com/drive/u/0/folders/1qY588PqswUfGTl3dZ_FbZhiDwK0Kv2Mz` (replace with your folder)  
    - File name: `=Cloned Ad #{{ $runIndex + 1 }}`  
    - Connect output of get_image node.  
    - Connect output back to iterate_ads node to continue processing remaining ads.

17. **Add Sticky Notes**  
    - Add descriptive sticky notes with workflow overview, setup instructions, and use cases for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| This workflow is branded "Nano Banana Facebook Ad Cloning System." It demonstrates an advanced use-case of combining Apify scraping, Google Gemini generative AI, and Google Drive storage to automate competitive ad cloning. It is suitable for digital marketers looking to scale ad creative production.                                                                                                                                                                                                             | Branding and project context                                                                                                       |
| The sticky note contains a comprehensive overview, use cases, good-to-know points, detailed workflow steps, requirements (Apify account, Google Gemini API, Google Drive), and customization pointers (e.g., adjusting ad count, prompt text, folders, notifications). It also highlights the approximate image generation time (10-30s per ad) and the handling of prohibited content flagged by Gemini AI.                                                                                                        | Sticky Note content within the workflow                                                                                            |
| Google Gemini API endpoints used are `gemini-2.5-pro:generateContent` for text prompt generation and `gemini-2.5-flash-image-preview:generateContent` for image generation. Retry logic with 5 seconds wait is implemented to improve stability.                                                                                                                                                                                                                                                                        | Google Gemini API specification                                                                                                   |
| The Apify actor used is "Facebook Ad Library Scraper" (ID: XtaWFhbtfxyzqrFmd) with parameters to scrape up to 20 ads per URL, all active statuses and countries, without detailed ad metadata scraping to optimize speed.                                                                                                                                                                                                                                                                                              | Apify actor details                                                                                                                |
| Google Drive folders are hardcoded by folder ID or URL for storing original competitor ads and generated cloned ads separately. Ensure correct permissions and credentials are configured for seamless uploads.                                                                                                                                                                                                                                                                                                       | Google Drive integration details                                                                                                  |
| The workflow handles edge cases such as partial competitor branding on images by instructing AI in the prompt to detect and replace all occurrences carefully, preserving original ad copy except for product naming. This is critical to avoid legal and branding issues.                                                                                                                                                                                                                                         | AI prompt instructions and risk mitigation                                                                                       |

---

*Disclaimer: The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and does not contain any illegal, offensive, or protected elements. All manipulated data are legal and publicly available.*