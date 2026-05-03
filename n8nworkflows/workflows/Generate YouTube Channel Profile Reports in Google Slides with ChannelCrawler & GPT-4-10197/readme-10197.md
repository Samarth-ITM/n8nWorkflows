Generate YouTube Channel Profile Reports in Google Slides with ChannelCrawler & GPT-4

https://n8nworkflows.xyz/workflows/generate-youtube-channel-profile-reports-in-google-slides-with-channelcrawler---gpt-4-10197


# Generate YouTube Channel Profile Reports in Google Slides with ChannelCrawler & GPT-4

### 1. Workflow Overview

This workflow automates the creation of personalized YouTube channel profile reports in Google Slides by integrating data from the ChannelCrawler API and OpenAI’s GPT-4 model. Its primary use case is to generate marketing or influencer presentation slides that summarize and visually represent YouTube creators' profiles based on their channel URLs.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures YouTube channel URLs via a form submission.
- **1.2 Input Cleaning & Iteration:** Parses and cleans the submitted URLs, then loops over each URL for processing.
- **1.3 Channel Data Retrieval:** Calls the ChannelCrawler API to get detailed YouTube channel profile information.
- **1.4 AI Processing:** Uses GPT models to generate a profile title and a concise summary based on the retrieved channel data.
- **1.5 Google Slides Handling:** Retrieves a Google Slides presentation, duplicates a slide page, updates profile images and replaces text placeholders with the channel data and AI-generated content.
- **1.6 Final Looping:** After slide updates, the workflow loops to process the next channel URL until all are handled.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Receives user input of one or multiple YouTube channel URLs through a web form.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - *Type:* Form Trigger  
    - *Role:* Initiates workflow on user submitting the form.  
    - *Configuration:* Form titled "Youtube Persona Slide Creator" with a single textarea field labeled "Youtube URLs" where users input channel URLs.  
    - *Input:* User submission via webhook.  
    - *Output:* Raw form data containing one or multiple URLs in a single string.  
    - *Edge Cases:* Empty input, malformed URLs, or unexpected field names may cause downstream errors.

#### 2.2 Input Cleaning & Iteration

- **Overview:**  
  Cleans and structures the raw URL input into a list of individual URLs, then processes each URL one by one.

- **Nodes Involved:**  
  - Clean Form Submission  
  - Loop Over Items

- **Node Details:**  
  - **Clean Form Submission**  
    - *Type:* Code (Python)  
    - *Role:* Splits the textarea input by line breaks, trims whitespace, and converts each URL into an object with key `url`.  
    - *Key Expression:* Uses Python `split('\r\n')` and list comprehension to sanitize URLs.  
    - *Input:* Raw form submission JSON.  
    - *Output:* Array of JSON objects each containing a single `url`.  
    - *Edge Cases:* Handles empty or whitespace-only lines by filtering them out.  
  - **Loop Over Items**  
    - *Type:* SplitInBatches  
    - *Role:* Iterates over each cleaned URL object sequentially for further processing.  
    - *Input:* Array of URL objects from Clean Form Submission.  
    - *Output:* Single URL object per iteration to downstream nodes.  
    - *Edge Cases:* Empty arrays or large input batches may affect performance.

#### 2.3 Channel Data Retrieval

- **Overview:**  
  Queries the ChannelCrawler API with each YouTube channel URL to fetch comprehensive channel profile data.

- **Nodes Involved:**  
  - Channel Crawler

- **Node Details:**  
  - **Channel Crawler**  
    - *Type:* HTTP Request  
    - *Role:* Sends a POST request to ChannelCrawler API endpoint `/v1/priority/channel` with the current channel URL in JSON body.  
    - *Configuration:* Uses HTTP Bearer token authentication (credential required).  
    - *Key Expression:* Request body constructed as `{"channel": "{{ $json.url }}"}`.  
    - *Input:* Single URL object from Loop Over Items.  
    - *Output:* Detailed JSON profile of the YouTube channel including description, statistics, links, and classification.  
    - *Edge Cases:* API rate limits, invalid URLs, authentication failures, network timeouts, or empty/missing channel data.

#### 2.4 AI Processing

- **Overview:**  
  Generates a marketing title and a brief profile summary for the channel using OpenAI GPT models based on the ChannelCrawler data.

- **Nodes Involved:**  
  - Title the profile  
  - Profile Summary

- **Node Details:**  
  - **Title the profile**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Creates a short, catchy title derived from the creator’s description for use in presentations.  
    - *Configuration:* Uses `gpt-3.5-turbo` model; prompt includes description extracted as `{{ $json.json.result.core.description }}`.  
    - *Input:* Output from Channel Crawler node.  
    - *Output:* Text message with a suggested profile title.  
    - *Edge Cases:* Empty or poorly formatted descriptions might yield irrelevant titles. API quota and latency.  
  - **Profile Summary**  
    - *Type:* OpenAI (LangChain)  
    - *Role:* Produces a concise summary (max 4 sentences) focusing on name, industry, expertise, and content topics.  
    - *Configuration:* Uses `chatgpt-4o-latest` model; prompt uses two contexts from ChannelCrawler data: description and classification keywords.  
    - *Input:* Output from Title the profile node and Channel Crawler node (via expressions).  
    - *Output:* Text message with the summarized profile content.  
    - *Edge Cases:* Similar to above, empty or ambiguous data may cause low-quality summaries.

#### 2.5 Google Slides Handling

- **Overview:**  
  Manages Google Slides presentation by duplicating a template slide, updating images and replacing placeholder texts with profile data and AI output.

- **Nodes Involved:**  
  - Get Presentation  
  - Create a new page  
  - Get a new page  
  - Replace Profile image  
  - Replace text in Page

- **Node Details:**  
  - **Get Presentation**  
    - *Type:* Google Slides  
    - *Role:* Retrieves the slide presentation to prepare for modifications.  
    - *Configuration:* Resource set to “page” with pageObjectId “p” (template slide ID).  
    - *Input:* Output from Profile Summary node.  
    - *Output:* Presentation metadata including slide object IDs.  
    - *Edge Cases:* Authorization errors, invalid presentation ID.  
  - **Create a new page**  
    - *Type:* HTTP Request  
    - *Role:* Duplicates the slide object (template page) using Google Slides batchUpdate API.  
    - *Configuration:* POST request to `https://slides.googleapis.com/v1/presentations/{insert_Presentation_ID}:batchUpdate` with duplication request referencing the original objectId. OAuth2 Google Slides authentication required.  
    - *Input:* Presentation metadata from Get Presentation node.  
    - *Output:* Response with new revision ID and duplicated slide info.  
    - *Edge Cases:* API quota, invalid object IDs, auth errors.  
  - **Get a new page**  
    - *Type:* Google Slides  
    - *Role:* Fetches the duplicated slide data with its new object ID.  
    - *Configuration:* Resource “page”, pageObjectId “p”, presentationId from previous node output.  
    - *Input:* Response from Create a new page node.  
    - *Output:* Slide page elements including placeholders for image and texts.  
    - *Edge Cases:* Same as above.  
  - **Replace Profile image**  
    - *Type:* HTTP Request  
    - *Role:* Replaces the profile image placeholder on the slide with the channel’s avatar URL from ChannelCrawler data.  
    - *Configuration:* POST batchUpdate request to Google Slides API with replaceImage request specifying imageObjectId from slide elements and avatar URL from ChannelCrawler data. Uses OAuth2 credentials.  
    - *Input:* Slide page data from Get a new page node and ChannelCrawler data.  
    - *Output:* Confirmation of image replacement and new revision ID.  
    - *Edge Cases:* Image URL invalid/unreachable, auth errors, concurrent slide edits.  
  - **Replace text in Page**  
    - *Type:* Google Slides  
    - *Role:* Replaces multiple text placeholders on the duplicated slide with: profile title, country, subscriber count, GPT summary, external links, and AI-generated title.  
    - *Configuration:* Multiple text replace operations targeting pageObjectId “p” with expressions pulling data from ChannelCrawler and OpenAI nodes.  
    - *Input:* Outputs from Get a new page, ChannelCrawler, Profile Summary, and Title the profile nodes.  
    - *Output:* Slide updated with new textual content.  
    - *Edge Cases:* Text placeholders missing, expression resolution failures, Google API limits.

#### 2.6 Final Looping

- **Overview:**  
  After completing processing for one channel URL, the workflow loops back to process the next URL until all are done.

- **Nodes Involved:**  
  - Replace text in Page (connects back to) Loop Over Items

- **Node Details:**  
  - Loop connection ensures sequential processing of each URL.  
  - Supports multiple channels submitted simultaneously.  
  - Edge cases include partial failures in one iteration not stopping the entire loop due to error handling settings (e.g., `onError` configured to continue).

---

### 3. Summary Table

| Node Name            | Node Type                | Functional Role                         | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                                |
|----------------------|--------------------------|---------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger             | Captures user input of YouTube URLs   | -                      | Clean Form Submission   | ## ChannelCrawler API to Google Slides Template ... (full note describing overall workflow, usage, and prerequisites)      |
| Clean Form Submission | Code (Python)            | Cleans and splits URLs from input     | On form submission     | Loop Over Items         |                                                                                                                            |
| Loop Over Items       | SplitInBatches           | Iterates over each channel URL        | Clean Form Submission  | Channel Crawler (2nd output path) |                                                                                                                            |
| Channel Crawler       | HTTP Request             | Fetches detailed channel profile data | Loop Over Items        | Title the profile       |                                                                                                                            |
| Title the profile     | OpenAI (LangChain)       | Generates marketing title from desc. | Channel Crawler        | Profile Summary         | ## AI Titling and Summarisation ... (explains customizing OpenAI prompts for summaries)                                     |
| Profile Summary       | OpenAI (LangChain)       | Creates concise profile summary       | Title the profile      | Get Presentation        |                                                                                                                            |
| Get Presentation      | Google Slides            | Retrieves Google Slides presentation  | Profile Summary        | Create a new page       | ## Google Presentation Fetch and Duplication ... (links to Google OAuth2 credential setup and notes on Presentation ID)    |
| Create a new page     | HTTP Request             | Duplicates template slide page        | Get Presentation       | Get a new page          |                                                                                                                            |
| Get a new page        | Google Slides            | Gets duplicated slide page data       | Create a new page      | Replace Profile image   |                                                                                                                            |
| Replace Profile image | HTTP Request             | Replaces profile image placeholder    | Get a new page         | Replace text in Page    |                                                                                                                            |
| Replace text in Page  | Google Slides            | Replaces text placeholders in slide   | Replace Profile image  | Loop Over Items (loop)  |                                                                                                                            |
| Sticky Note           | Sticky Note              | Provides overall workflow explanation | -                      | -                      | See Node Name "On form submission" sticky note content                                                                     |
| Sticky Note1          | Sticky Note              | Notes on AI titling and summarisation | -                      | -                      | See Node Name "Title the profile" sticky note content                                                                       |
| Sticky Note2          | Sticky Note              | Notes on Google Slides setup and auth | -                      | -                      | See Node Name "Get Presentation" sticky note content                                                                        |
| Sticky Note3          | Sticky Note              | Sample ChannelCrawler API JSON response | -                      | -                      | Detailed sample API response to guide users on expected data                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**
   - Add a **Form Trigger** node named "On form submission".
   - Configure:
     - Form Title: "Youtube Persona Slide Creator"
     - Add a textarea form field labeled "Youtube URLs"
     - Save and activate webhook.

2. **Add Cleaning Node:**
   - Add a **Code** node named "Clean Form Submission".
   - Set language to Python.
   - Paste code to split input URLs by line breaks, trim whitespace, and output array of objects `{ "url": <url> }`.

3. **Add Looping Node:**
   - Add a **SplitInBatches** node named "Loop Over Items".
   - Connect it to "Clean Form Submission".
   - Configure default batch size (e.g., 1) to process URLs one at a time.

4. **Add ChannelCrawler API Call:**
   - Add an **HTTP Request** node named "Channel Crawler".
   - Method: POST
   - URL: `https://api.channelcrawler.com/v1/priority/channel`
   - Body Type: JSON
   - Body Content: `{"channel": "{{ $json.url }}"}`
   - Authentication: HTTP Bearer Auth with ChannelCrawler API token.
   - Connect input from "Loop Over Items".

5. **Add AI Title Generation:**
   - Add an **OpenAI (LangChain)** node named "Title the profile".
   - Model: `gpt-3.5-turbo`
   - Prompt: Use channel description from ChannelCrawler data to ask for a brief marketing title.
   - Connect to "Channel Crawler".

6. **Add AI Profile Summary Generation:**
   - Add another **OpenAI (LangChain)** node named "Profile Summary".
   - Model: `chatgpt-4o-latest`
   - Prompt: Use channel description and classification keywords to generate a 4-sentence summary starting with the creator’s name.
   - Connect to "Title the profile".

7. **Retrieve Google Slides Presentation:**
   - Add **Google Slides** node named "Get Presentation".
   - Resource: "page"
   - Page Object Id: "p" (template slide)
   - Connect to "Profile Summary".
   - Configure Google OAuth2 credentials with appropriate Google Slides API scopes.

8. **Duplicate Template Slide:**
   - Add **HTTP Request** node named "Create a new page".
   - Method: POST
   - URL: `https://slides.googleapis.com/v1/presentations/{insert_Presentation_ID}:batchUpdate`
   - Body: JSON containing duplicateObject request with objectId from "Get Presentation".
   - Authentication: Google OAuth2.
   - Connect to "Get Presentation".

9. **Retrieve the New Slide Page:**
   - Add **Google Slides** node named "Get a new page".
   - Resource: "page"
   - Page Object Id: "p"
   - Presentation Id: from "Create a new page" output.
   - Connect to "Create a new page".

10. **Replace Profile Image:**
    - Add **HTTP Request** node named "Replace Profile image".
    - Method: POST
    - URL: same Google Slides batchUpdate endpoint as above.
    - Body: JSON `replaceImage` request targeting image objectId from "Get a new page" and setting URL from ChannelCrawler profile avatar.
    - Authentication: Google OAuth2.
    - Connect to "Get a new page".

11. **Replace Text Placeholders:**
    - Add **Google Slides** node named "Replace text in Page".
    - Operation: replaceText
    - Configure multiple text replacements:
      - Profile title from ChannelCrawler core title.
      - Country name from classification.
      - Subscriber count formatted as "Youtube Subscribers: X".
      - Profile summary from AI output.
      - External links concatenated.
      - The AI-generated marketing title.
    - Target pageObjectId "p".
    - Connect to "Replace Profile image".
    - Configure error handling to continue on error.

12. **Connect Loop:**
    - Connect "Replace text in Page" back to "Loop Over Items" to continue processing the next URL.

13. **Credential Setup:**
    - Set up credentials for:
      - ChannelCrawler API (HTTP Bearer Token).
      - OpenAI (API key).
      - Google Slides OAuth2 with required scopes (`https://www.googleapis.com/auth/presentations`).

14. **Final Notes:**
    - Replace `{insert_Presentation_ID}` in HTTP requests with your Google Slides presentation ID.
    - The template slide ("p") must exist and include placeholders for image and text consistent with the replacements.
    - Test with a few URLs before bulk execution.
    - Enable error handling or notifications as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                        | Context or Link                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| This template demonstrates how to integrate the ChannelCrawler API with ChatGPT and Google Slides to create influencer profiles automatically. The workflow starts from user input and ends with slide creation including images and text.                                            | Sticky Note on "On form submission" node                                                                                |
| To customize AI-generated titles and summaries, adjust the OpenAI prompt messages accordingly to fit your reporting style and content needs.                                                                                                                                       | Sticky Note on "Title the profile" node                                                                                 |
| For Google Slides API authorization, follow n8n’s detailed [OAuth2 credential setup guide](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/?utm_source=n8n_app&utm_medium=credential_settings&utm_campaign=create_new_credentials_modal). You must provide the Presentation ID to the HTTP requests. | Sticky Note on "Get Presentation" node                                                                                  |
| ChannelCrawler API JSON response sample included in sticky note for reference on expected data structure, aiding in crafting expressions and debugging.                                                                                                                               | Sticky Note on area near "Loop Over Items" node                                                                         |

---

This documentation provides a complete, stepwise understanding of the workflow, enabling users and automation agents to reproduce, adapt, and troubleshoot the workflow effectively.