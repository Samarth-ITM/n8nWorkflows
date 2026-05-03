Generate Social Media Ads with Website Scraping using Gemini & Ideogram AI for FB, IG & Pinterest

https://n8nworkflows.xyz/workflows/generate-social-media-ads-with-website-scraping-using-gemini---ideogram-ai-for-fb--ig---pinterest-9388


# Generate Social Media Ads with Website Scraping using Gemini & Ideogram AI for FB, IG & Pinterest

### 1. Workflow Overview

This workflow automates the generation of social media advertisement images tailored for Facebook, Instagram, and Pinterest by scraping website content and leveraging AI (Google Gemini & Ideogram AI) for prompt generation and image creation. It is designed for marketing teams or agencies aiming to create multiple social media ad formats efficiently from user input and website data.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user inputs via a web form to start the ad generation process.
- **1.2 Website Scraping:** Retrieves website content through a custom scraping API to gather data for ad content.
- **1.3 AI Prompt Generation:** Uses Google Gemini AI to create ad copy or creative prompts based on scraped data.
- **1.4 Dimension Routing:** Routes the workflow into multiple branches based on requested ad image dimensions.
- **1.5 Image Generation Requests:** Sends requests to AI-driven image generation endpoints for each social media dimension.
- **1.6 Image Downloading:** Downloads the generated images from the image generation responses.
- **1.7 Distribution:** Sends the final images to Slack channels for review, sharing, or further processing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Captures user inputs through a configured form trigger node, which initiates the workflow when a user submits ad generation parameters.

**Nodes Involved:**  
- 📝 Form Trigger

**Node Details:**  
- **Type & Role:** Form Trigger node; serves as the entry point capturing user input fields such as website URL, desired ad dimensions, and other relevant parameters.  
- **Configuration:** Form fields are pre-configured; webhook URL is generated for external access.  
- **Expressions/Variables:** Captures dynamic user inputs to be passed downstream.  
- **Connections:** Outputs to the website scraping node.  
- **Edge Cases:** Missing or malformed inputs can cause failures; webhook must be publicly accessible and secured to prevent unauthorized use.  
- **Sub-workflow:** None.

---

#### 2.2 Website Scraping

**Overview:**  
Requests content from the user-provided website URL using a Firecrawl scraping API at the `/scrape` endpoint, extracting data for AI prompt creation.

**Nodes Involved:**  
- Scrape One Page via Firecrawl - user /scrape endpoint

**Node Details:**  
- **Type & Role:** HTTP Request node; performs an API call to scrape website content.  
- **Configuration:** Configured with the Firecrawl API endpoint, likely passing the user URL as a parameter.  
- **Expressions/Variables:** Uses form input URL from the previous node.  
- **Connections:** Outputs scraped content to the AI prompt generation node.  
- **Edge Cases:** API failures, network timeouts, invalid URLs, or scraping restrictions by target websites.  
- **Sub-workflow:** None.

---

#### 2.3 AI Prompt Generation

**Overview:**  
Generates text prompts for image creation and ad copy using Google Gemini AI based on the scraped website content.

**Nodes Involved:**  
- Prompt Generation With AI  
- Set AI Prompt

**Node Details:**  
- **Prompt Generation With AI**  
  - **Type & Role:** Google Gemini node; leverages language AI to produce creative prompts.  
  - **Configuration:** Uses scraped content as input, with model-specific parameters set for prompt generation.  
  - **Expressions/Variables:** Inputs use data from scraping node; outputs contain AI-generated text.  
  - **Connections:** Outputs to the Set AI Prompt node.  
  - **Edge Cases:** API quota limits, authentication errors, malformed input causing generation failures.

- **Set AI Prompt**  
  - **Type & Role:** Set node; formats or adjusts AI output for subsequent routing.  
  - **Configuration:** Likely sets variables or expressions to standardize prompt format.  
  - **Connections:** Outputs to the routing node by ad dimension.  
  - **Edge Cases:** Expression evaluation failures if AI output is missing or malformed.

---

#### 2.4 Dimension Routing

**Overview:**  
Distributes the workflow into seven parallel paths corresponding to different social media ad image dimensions (e.g., Facebook Feed, Instagram Story).

**Nodes Involved:**  
- 🔀 Route by Dimensions

**Node Details:**  
- **Type & Role:** Switch node; routes based on the requested ad dimension(s).  
- **Configuration:** Seven outputs configured, each matching a specific dimension (FB Feed, FB Story, IG Feed, IG Story, IG Reel, Pinterest Pin, Pinterest Story).  
- **Options:** "Output All Matching" enabled to allow multiple concurrent dimension processing.  
- **Connections:** Each output connects to a dedicated image generation HTTP Request node.  
- **Edge Cases:** Input dimension values outside configured options will lead to no routing; multiple selections handled correctly.

---

#### 2.5 Image Generation Requests

**Overview:**  
Sends HTTP requests to AI image generation endpoints (likely Ideogram AI or similar) for each selected social media dimension, passing the AI prompt and dimension parameters.

**Nodes Involved:**  
- 🎨 FB Feed (1200x630)  
- 🎨 FB Story (1080x1920)  
- 🎨 IG Feed (1080x1080)  
- 🎨 IG Story (1080x1920)  
- 🎨 IG Reel (1080x1920)  
- 🎨 Pinterest Pin (1000x1500)  
- 🎨 Pinterest Story (1080x1920)

**Node Details:**  
- **Type & Role:** HTTP Request nodes; responsible for requesting image generation per dimension.  
- **Configuration:** Each node is configured with dimension-specific parameters and the AI-generated prompt; likely calls external image generation API endpoints.  
- **Connections:** Each node outputs to a corresponding image download node.  
- **Edge Cases:** API failures, timeouts, invalid prompt format, or dimension mismatch errors.

---

#### 2.6 Image Downloading

**Overview:**  
Downloads the generated images from the URLs or binary data returned by the image generation API.

**Nodes Involved:**  
- Download Image - FB Feed  
- Download Image - FB Story  
- Download Image - IG Feed  
- Download Image - IG Story  
- Download Image - IG Reel  
- Download Image - Pinterest Pin  
- Download Image - Pinterest Story

**Node Details:**  
- **Type & Role:** HTTP Request nodes; fetch the actual image files from URLs provided by previous nodes.  
- **Configuration:** Configured to handle binary data download; uses URLs from the image generation responses.  
- **Connections:** Outputs feed directly into Slack send nodes.  
- **Edge Cases:** Broken URLs, download timeouts, corrupted files, or unsupported content types.

---

#### 2.7 Distribution

**Overview:**  
Sends the final images to dedicated Slack channels (or webhooks) for each social media dimension, enabling team visibility or further actions.

**Nodes Involved:**  
- 📤 Send FB Feed  
- 📤 Send FB Story  
- 📤 Send IG Feed  
- 📤 Send IG Story  
- 📤 Send IG Reel  
- 📤 Send Pinterest Pin  
- 📤 Send Pinterest Story

**Node Details:**  
- **Type & Role:** Slack node; posts messages/files to Slack channels or webhooks.  
- **Configuration:** Each node has a webhook URL or Slack credential configured, targeting specific channels or threads.  
- **Connections:** Final node in each branch.  
- **Edge Cases:** Slack API limits, authentication errors, file size restrictions, or channel permissions issues.

---

### 3. Summary Table

| Node Name                               | Node Type                | Functional Role                   | Input Node(s)                            | Output Node(s)                             | Sticky Note                        |
|----------------------------------------|--------------------------|---------------------------------|----------------------------------------|--------------------------------------------|----------------------------------|
| 📝 Form Trigger                        | Form Trigger             | Capture user input to start flow| None                                   | Scrape One Page via Firecrawl               | SETUP: Configure form fields, share webhook URL |
| Scrape One Page via Firecrawl - user /scrape endpoint | HTTP Request             | Scrape website content           | 📝 Form Trigger                        | Prompt Generation With AI                    |                                  |
| Prompt Generation With AI              | Google Gemini AI          | Generate AI text prompts         | Scrape One Page via Firecrawl          | Set AI Prompt                               |                                  |
| Set AI Prompt                         | Set                      | Format AI output for routing     | Prompt Generation With AI               | 🔀 Route by Dimensions                       |                                  |
| 🔀 Route by Dimensions                 | Switch                   | Route by requested image sizes   | Set AI Prompt                         | 🎨 FB Feed, 🎨 FB Story, 🎨 IG Feed, 🎨 IG Story, 🎨 IG Reel, 🎨 Pinterest Pin, 🎨 Pinterest Story | SETUP: 7 routes, output all matching enabled |
| 🎨 FB Feed (1200x630)                  | HTTP Request             | Request FB Feed image generation | 🔀 Route by Dimensions (FB Feed path)  | Download Image - FB Feed                     |                                  |
| Download Image - FB Feed               | HTTP Request             | Download FB Feed image           | 🎨 FB Feed                           | 📤 Send FB Feed                             |                                  |
| 📤 Send FB Feed                       | Slack                    | Post FB Feed image to Slack      | Download Image - FB Feed               | None                                       |                                  |
| 🎨 FB Story (1080x1920)                | HTTP Request             | Request FB Story image generation| 🔀 Route by Dimensions (FB Story path) | Download Image - FB Story                    |                                  |
| Download Image - FB Story              | HTTP Request             | Download FB Story image          | 🎨 FB Story                         | 📤 Send FB Story                            |                                  |
| 📤 Send FB Story                     | Slack                    | Post FB Story image to Slack     | Download Image - FB Story              | None                                       |                                  |
| 🎨 IG Feed (1080x1080)                 | HTTP Request             | Request IG Feed image generation | 🔀 Route by Dimensions (IG Feed path)  | Download Image - IG Feed                     |                                  |
| Download Image - IG Feed               | HTTP Request             | Download IG Feed image           | 🎨 IG Feed                         | 📤 Send IG Feed                             |                                  |
| 📤 Send IG Feed                      | Slack                    | Post IG Feed image to Slack      | Download Image - IG Feed               | None                                       |                                  |
| 🎨 IG Story (1080x1920)                | HTTP Request             | Request IG Story image generation| 🔀 Route by Dimensions (IG Story path) | Download Image - IG Story                    |                                  |
| Download Image - IG Story              | HTTP Request             | Download IG Story image          | 🎨 IG Story                        | 📤 Send IG Story                            |                                  |
| 📤 Send IG Story                    | Slack                    | Post IG Story image to Slack     | Download Image - IG Story              | None                                       |                                  |
| 🎨 IG Reel (1080x1920)                 | HTTP Request             | Request IG Reel image generation | 🔀 Route by Dimensions (IG Reel path)  | Download Image - IG Reel                     |                                  |
| Download Image - IG Reel               | HTTP Request             | Download IG Reel image           | 🎨 IG Reel                        | 📤 Send IG Reel                             |                                  |
| 📤 Send IG Reel                    | Slack                    | Post IG Reel image to Slack      | Download Image - IG Reel               | None                                       |                                  |
| 🎨 Pinterest Pin (1000x1500)           | HTTP Request             | Request Pinterest Pin image      | 🔀 Route by Dimensions (Pinterest Pin path) | Download Image - Pinterest Pin               |                                  |
| Download Image - Pinterest Pin        | HTTP Request             | Download Pinterest Pin image     | 🎨 Pinterest Pin                  | 📤 Send Pinterest Pin                       |                                  |
| 📤 Send Pinterest Pin              | Slack                    | Post Pinterest Pin image to Slack| Download Image - Pinterest Pin        | None                                       |                                  |
| 🎨 Pinterest Story (1080x1920)         | HTTP Request             | Request Pinterest Story image    | 🔀 Route by Dimensions (Pinterest Story path) | Download Image - Pinterest Story             |                                  |
| Download Image - Pinterest Story      | HTTP Request             | Download Pinterest Story image   | 🎨 Pinterest Story                | 📤 Send Pinterest Story                     |                                  |
| 📤 Send Pinterest Story            | Slack                    | Post Pinterest Story to Slack    | Download Image - Pinterest Story      | None                                       |                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger Node:**  
   - Type: Form Trigger  
   - Configure form fields to accept user inputs such as website URL and desired ad dimensions.  
   - Save and copy the webhook URL for external form submission.

2. **Create HTTP Request Node for Scraping:**  
   - Name: Scrape One Page via Firecrawl - user /scrape endpoint  
   - Configure to call the Firecrawl scraping API `/scrape` endpoint.  
   - Pass the website URL from the form trigger as a parameter.  
   - Connect output from Form Trigger to this node.

3. **Add Google Gemini AI Node for Prompt Generation:**  
   - Name: Prompt Generation With AI  
   - Configure credentials for Google Gemini.  
   - Use the scraped content as input.  
   - Set parameters appropriate for prompt generation.  
   - Connect output from scraping node to this node.

4. **Add Set Node to Format AI Prompt:**  
   - Name: Set AI Prompt  
   - Configure to set or transform the AI-generated text into a standardized prompt variable.  
   - Connect output from AI node to this node.

5. **Add Switch Node to Route by Dimensions:**  
   - Name: 🔀 Route by Dimensions  
   - Configure 7 outputs for the following dimensions: FB Feed, FB Story, IG Feed, IG Story, IG Reel, Pinterest Pin, Pinterest Story.  
   - Enable "Output All Matching" to allow multiple routes.  
   - Use the ad dimension input from the form or AI prompt for routing logic.  
   - Connect output from Set AI Prompt node to this switch.

6. **Create HTTP Request Nodes for Image Generation (7 total):**  
   For each dimension:  
   - Name accordingly (e.g., 🎨 FB Feed (1200x630))  
   - Configure endpoint to call the AI image generation API (Ideogram AI or similar).  
   - Pass the standardized AI prompt and dimension parameters.  
   - Connect each output of the routing node to the corresponding image generation node.

7. **Create HTTP Request Nodes to Download Images (7 total):**  
   For each dimension:  
   - Name accordingly (e.g., Download Image - FB Feed)  
   - Configure to download the image binary from the URL returned by the previous node.  
   - Connect output from each image generation node to its corresponding download node.

8. **Create Slack Nodes to Send Images (7 total):**  
   For each dimension:  
   - Name accordingly (e.g., 📤 Send FB Feed)  
   - Configure Slack credentials or webhook URL.  
   - Set to send the downloaded image as a file or message attachment.  
   - Connect output from each download node to its corresponding Slack node.

9. **Test the whole workflow:**  
   - Submit the form with test inputs.  
   - Verify that the image generation, downloading, and Slack posting work as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The form trigger node setup requires sharing the webhook URL externally for user input collection.                                                       | Workflow Setup Instructions                      |
| The routing node is configured to support multiple dimensions concurrently via "Output All Matching."                                                    | Routing logic for multi-size image generation    |
| Slack nodes require proper OAuth2 or webhook credentials for posting images to channels.                                                                  | Slack API Integration                            |
| Firecrawl scraping depends on the target website's accessibility and permissions; consider rate limits and robots.txt restrictions.                       | Website Scraping API Usage                        |
| AI services (Google Gemini, Ideogram AI) must have valid API keys and quota for smooth operation; handle API errors gracefully in production.             | AI Integration and Quotas                         |

---

**Disclaimer:**  
The provided content stems exclusively from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.