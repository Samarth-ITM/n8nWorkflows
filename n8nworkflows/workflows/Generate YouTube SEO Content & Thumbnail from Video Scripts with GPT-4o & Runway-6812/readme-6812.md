Generate YouTube SEO Content & Thumbnail from Video Scripts with GPT-4o & Runway

https://n8nworkflows.xyz/workflows/generate-youtube-seo-content---thumbnail-from-video-scripts-with-gpt-4o---runway-6812


# Generate YouTube SEO Content & Thumbnail from Video Scripts with GPT-4o & Runway

### 1. Workflow Overview

This n8n workflow automates the generation of YouTube SEO content and thumbnail images based on video scripts provided in a Google Sheet. It is designed for content creators and marketers who want to streamline the process of optimizing YouTube videos for search visibility and engagement, including automated generation of titles, tags, keywords, descriptions, and thumbnails using AI models.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Trigger:** Watches a Google Sheet for new video scripts.
- **1.2 English Script Generation:** Converts Roman Hindi scripts to English if an English version is missing.
- **1.3 SEO Content Generation:** Uses GPT-4o to create YouTube titles, tags, keywords, descriptions, and thumbnail prompts.
- **1.4 Content Parsing and Sheet Update:** Parses the AI-generated content and updates the Google Sheet.
- **1.5 Thumbnail Generation:** Crafts a dynamic prompt and invokes Runway API to generate a thumbnail image.
- **1.6 Thumbnail Image Update:** Inserts the generated thumbnail URL back into the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Trigger

- **Overview:**  
  Listens for new rows added to a specified Google Sheet to start the workflow.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - If English script is available or not  
  - No Operation, do nothing

- **Node Details:**  
  - *Google Sheets Trigger*  
    - Type: Trigger node for Google Sheets  
    - Configured to trigger every minute upon new row addition in the sheet named "Sheet1" within a specific Google Sheets document.  
    - Inputs: None (trigger)  
    - Outputs: To "If English script is available or not" node  
    - Potential Failures: Google API connection or credential errors; polling rate limits.

  - *If English script is available or not*  
    - Type: Conditional node  
    - Checks if the "Roman Hindi Script" field is not empty and "English script" field is empty.  
    - If true, proceeds to generate English script; if false, goes to No Operation node.  
    - Inputs: From Google Sheets Trigger  
    - Outputs: Two outputs — true branch to English script generation, false branch to no-op  
    - Edge Cases: If input fields are malformed or missing, condition may fail or cause workflow to branch incorrectly.

  - *No Operation, do nothing*  
    - Type: No operation placeholder  
    - Purpose: Ends workflow branch when no English script generation is needed.  
    - Inputs: False branch from If node  
    - Outputs: None

#### 2.2 English Script Generation

- **Overview:**  
  Converts Roman Hindi script into an English script using OpenAI GPT-4o model.

- **Nodes Involved:**  
  - Generating English Script (OpenAI node)  
  - Creating field for english script (Set node)  
  - Adding English script to GSheet (Google Sheets node)

- **Node Details:**  
  - *Generating English Script*  
    - Type: OpenAI (LangChain) node  
    - Uses model "gpt-4o" to convert Roman Hindi script to English by sending prompt: "Convert this hindi roman script into english script."  
    - Inputs: True branch from If node with Roman Hindi script  
    - Outputs: JSON containing generated English script in message.content  
    - Failures: API key limits, request timeouts, malformed input.

  - *Creating field for english script*  
    - Type: Set node  
    - Transfers the generated English script from OpenAI response to a field named message.content for easier downstream access.  
    - Inputs: Output from Generating English Script  
    - Outputs: To Adding English script to GSheet node

  - *Adding English script to GSheet*  
    - Type: Google Sheets  
    - Updates the existing Google Sheet row by matching the "Number" column, inserting the generated English script alongside the original Roman Hindi script and number.  
    - Inputs: From Creating field for english script  
    - Outputs: To "Generating Youtube Tags, Keywords and more." node  
    - Failures: Google Sheets API errors, mismatched row numbers, permission issues.

#### 2.3 SEO Content Generation

- **Overview:**  
  Uses GPT-4o latest to generate YouTube SEO content including titles, tags, keywords, descriptions, and thumbnail prompts based on the English script.

- **Nodes Involved:**  
  - Generating Youtube Tags, Keywords and more. (OpenAI node)

- **Node Details:**  
  - *Generating Youtube Tags, Keywords and more.*  
    - Type: OpenAI (LangChain) node  
    - Model: "chatgpt-4o-latest"  
    - Prompt includes instructions to generate:  
      1. 10 engaging YouTube titles (≤ 70 chars each)  
      2. Up to 20 tags (comma-separated)  
      3. Up to 20 keywords (comma-separated)  
      4. SEO-friendly description optimized for 2025 algorithm with hooks, timestamps, hashtags, etc.  
      5. Thumbnail AI prompt (for image generation)  
      6. Thumbnail tagline (catchy 3–5 words)  
    - Inputs: Updated Google Sheet data including English script  
    - Outputs: To JS Code for separating different topics node  
    - Failures: API quota, malformed prompt response, long response truncation.

#### 2.4 Content Parsing and Sheet Update

- **Overview:**  
  Parses the AI-generated text containing multiple sections into structured fields and updates the Google Sheet with these SEO contents.

- **Nodes Involved:**  
  - JS Code for separating different topics (Code node)  
  - Updating row in GSheet with Tags, Description so on (Google Sheets node)

- **Node Details:**  
  - *JS Code for separating different topics*  
    - Type: Code (JavaScript)  
    - Function: Uses regex to extract 6 defined sections (titles, tags, keywords, description, thumbnail prompt, tagline) from the AI output string.  
    - Splits multiple titles by newline or dash, cleans numbering prefixes, and returns structured JSON fields.  
    - Inputs: Response from the SEO content generation node  
    - Outputs: To Updating row in GSheet node  
    - Edge Cases: Unexpected formatting in AI text could break regex extraction.

  - *Updating row in GSheet with Tags, Description so on*  
    - Type: Google Sheets (Update operation)  
    - Updates the row by matching "Number" field, sets columns: Youtube Tags, Youtube Title, Youtube Keywords, Youtube Description, Youtube Thumbnail Brief, Youtube Thumbnail Tagline.  
    - Inputs: Parsed structured data from JS Code node  
    - Outputs: To Python code for creating Dynamic Prompt node  
    - Failures: Google Sheets API errors, mismatched or missing row IDs.

#### 2.5 Thumbnail Generation

- **Overview:**  
  Constructs a detailed prompt to instruct the Runway API for generating a professional YouTube thumbnail image.

- **Nodes Involved:**  
  - Python code for creating Dynamic Prompt (Code node)  
  - HTTP Request calling Runware API for generating Thumbnail image (HTTP Request node)

- **Node Details:**  
  - *Python code for creating Dynamic Prompt*  
    - Type: Code (Python)  
    - Reads YouTube Thumbnail Brief and Tagline from JSON input  
    - Builds a detailed positive prompt specifying style, lighting, resolution, and text emphasis for thumbnail generation  
    - Creates a payload object with unique UUID, model specification ("bytedance:3@1"), dimensions 1280x720, and other parameters for Runway API  
    - Outputs JSON body for the HTTP request  
    - Inputs: Updated Google Sheet data with thumbnail brief and tagline  
    - Outputs: To HTTP Request node

  - *HTTP Request calling Runware API for generating Thumbnail image*  
    - Type: HTTP Request  
    - Method: POST to https://api.runware.ai/v1/image  
    - Sends JSON body constructed by Python node, including authorization header (API key expected to be configured in credentials)  
    - Receives JSON response with generated image URL  
    - Inputs: Payload from Python code  
    - Outputs: To Updating Gsheet with Thumbnail Image node  
    - Failures: HTTP errors, API authentication failure, rate limits, malformed payload.

#### 2.6 Thumbnail Image Update

- **Overview:**  
  Updates the Google Sheet row with the URL of the generated thumbnail image.

- **Nodes Involved:**  
  - Updating Gsheet with Thumbnail Image (Google Sheets node)

- **Node Details:**  
  - *Updating Gsheet with Thumbnail Image*  
    - Type: Google Sheets (Update operation)  
    - Matches the row by "Number" column and updates "Youtube Generated Thumbnail" with the image URL received from Runway API  
    - Inputs: HTTP response from Runway API  
    - Outputs: None (workflow end)  
    - Failures: Google Sheets API errors, connectivity issues.

---

### 3. Summary Table

| Node Name                                   | Node Type                 | Functional Role                                       | Input Node(s)                      | Output Node(s)                              | Sticky Note                                                                                                                     |
|---------------------------------------------|---------------------------|------------------------------------------------------|----------------------------------|---------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger                        | Google Sheets Trigger     | Entry trigger on new row in Google Sheet             | None                             | If English script is available or not       | ## Connect with your Google Sheet                                                                                              |
| If English script is available or not       | If                        | Checks if English script exists or needs generation  | Google Sheets Trigger            | Generating English Script, No Operation      |                                                                                                                                |
| No Operation, do nothing                     | NoOp                      | Ends workflow branch if no English script needed     | If English script is available or not | None                                    |                                                                                                                                |
| Generating English Script                    | OpenAI (LangChain)        | Converts Roman Hindi to English script                | If English script is available or not | Creating field for english script          | ## Add your OpenAI Keys                                                                                                        |
| Creating field for english script            | Set                       | Prepares English script field for downstream nodes   | Generating English Script        | Adding English script to GSheet              |                                                                                                                                |
| Adding English script to GSheet              | Google Sheets             | Updates Google Sheet with generated English script   | Creating field for english script | Generating Youtube Tags, Keywords and more |                                                                                                                                |
| Generating Youtube Tags, Keywords and more. | OpenAI (LangChain)        | Generates YouTube SEO content and thumbnail prompts  | Adding English script to GSheet  | JS Code for separating different topics     | ## Prompt for Generating YouTue Tags, Description so on..                                                                      |
| JS Code for separating different topics     | Code (JavaScript)         | Parses AI text into structured fields                 | Generating Youtube Tags, Keywords and more. | Updating row in GSheet with Tags, Description so on | ## Javascript code for separating the data                                                                                      |
| Updating row in GSheet with Tags, Description so on | Google Sheets             | Updates sheet with parsed SEO fields                   | JS Code for separating different topics | Python code for creating Dynamic Prompt    |                                                                                                                                |
| Python code for creating Dynamic Prompt      | Code (Python)             | Creates dynamic prompt for thumbnail generation       | Updating row in GSheet with Tags, Description so on | HTTP Request calling  Runware API for generating Thumbnail image | ## Python code for generating dynamic prompt                                                                                   |
| HTTP Request calling  Runware API for generating Thumbnail image | HTTP Request              | Calls Runway API to generate thumbnail image          | Python code for creating Dynamic Prompt | Updating Gsheet with Thumbnail Image         | ## Add your RUNWARE API or any other 3rd party                                                                                 |
| Updating Gsheet with Thumbnail Image         | Google Sheets             | Updates sheet with returned thumbnail image URL       | HTTP Request calling  Runware API for generating Thumbnail image | None                                      |                                                                                                                                |
| Sticky Note                                  | Sticky Note               | Tutorial video link                                   | None                             | None                                        | ## Step-by-Step Tutorial YouTube Video  \n\n## [Follow Video- Click here](https://youtu.be/R426d84slWY?si=Ecozf55ypxg6PnQz)     |
| Sticky Note1                                 | Sticky Note               | Prompt content for SEO generation                      | None                             | None                                        | ## Prompt for Generating YouTue Tags, Description so on..                                                                      |
| Sticky Note2                                 | Sticky Note               | JavaScript code explanation                            | None                             | None                                        | ## Javascript code for separating the data                                                                                      |
| Sticky Note3                                 | Sticky Note               | Python code explanation                                | None                             | None                                        | ## Python code for generating dynamic prompt                                                                                   |
| Sticky Note4                                 | Sticky Note               | Reminder for OpenAI key setup                          | None                             | None                                        | ## Add your OpenAI Keys                                                                                                        |
| Sticky Note5                                 | Sticky Note               | Reminder for Google Sheets connection                  | None                             | None                                        | ## Connect with your Google Sheet                                                                                              |
| Sticky Note6                                 | Sticky Note               | Reminder for Runway API key setup                       | None                             | None                                        | ## Add your RUNWARE API or any other 3rd party                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Google Sheets Trigger Node**  
   - Set it to trigger on "rowAdded" event.  
   - Configure it to poll every minute.  
   - Connect to the Google Sheet document with ID: `1BMtXN3Wl_pqyHJQW9KfRaErjPmg5wrm9kfIYcnuUVAM`, sheet named "Sheet1" (gid=0).  
   - Requires Google Sheets OAuth2 credentials.

2. **Add an If Node "If English script is available or not"**  
   - Condition:  
     - Check if `Roman Hindi Script` is not empty  
     - AND `English script` is empty  
   - True output: Proceed to English script generation  
   - False output: Connect to a No Operation node.

3. **Add a No Operation Node "No Operation, do nothing"**  
   - Connect false output of If node here (ends workflow branch).

4. **Add an OpenAI Node "Generating English Script"**  
   - Use OpenAI API with GPT-4o model.  
   - Prompt: `{{ $json['Roman Hindi Script'] }}\n\nConvert this hindi roman script in to english script.`  
   - Set appropriate OpenAI API credentials.

5. **Add a Set Node "Creating field for english script"**  
   - Copy `message.content` from OpenAI response to same field for ease of use downstream.

6. **Add a Google Sheets Node "Adding English script to GSheet"**  
   - Operation: Update row based on "Number" column.  
   - Update columns: "Number", "Roman Hindi Script", and "English script" (with generated English script).  
   - Use same Google Sheet document and sheet as trigger.

7. **Add an OpenAI Node "Generating Youtube Tags, Keywords and more."**  
   - Use "chatgpt-4o-latest" model.  
   - Prompt template: Provide English script and instruct generation of:  
     - 10 YouTube titles (≤ 70 chars)  
     - Up to 20 tags, keywords  
     - SEO optimized description for 2025  
     - Thumbnail AI prompt and tagline  
   - Use your OpenAI API credentials.

8. **Add a Code Node (JavaScript) "JS Code for separating different topics"**  
   - Paste the provided JavaScript code to parse the OpenAI response into structured fields: titles array, tags, keywords, description, thumbnail prompt, tagline.

9. **Add a Google Sheets Node "Updating row in GSheet with Tags, Description so on"**  
   - Operation: Update row by "Number"  
   - Columns to update: Youtube Titles, Youtube Tags, Youtube Keywords, Youtube Description, Youtube Thumbnail Brief, Youtube Thumbnail Tagline.

10. **Add a Code Node (Python) "Python code for creating Dynamic Prompt"**  
    - Use the provided Python code to create a JSON payload for Runway API, including a unique UUID and thumbnail generation parameters.  
    - Inputs: Thumbnail Brief and Tagline fields from previous node.

11. **Add an HTTP Request Node "HTTP Request calling Runware API for generating Thumbnail image"**  
    - Method: POST  
    - URL: `https://api.runware.ai/v1/image`  
    - Headers: Authorization (set your Runway API key), Content-Type: application/json  
    - Body: JSON from previous Python node  
    - Send as JSON body.

12. **Add a Google Sheets Node "Updating Gsheet with Thumbnail Image"**  
    - Operation: Update row by "Number"  
    - Update "Youtube Generated Thumbnail" column with the image URL from Runway API response.

13. **Connect all nodes in sequence:**  
    - Google Sheets Trigger → If English script → (True) Generating English Script → Creating field for english script → Adding English script to GSheet → Generating Youtube Tags → JS Code for separating topics → Updating row in GSheet with SEO info → Python code for prompt → HTTP Request to Runway → Updating Gsheet with thumbnail  
    - (False branch from If English script) → No Operation node.

14. **Credentials Setup:**  
    - Google Sheets OAuth2 for all Google Sheets nodes.  
    - OpenAI API key for OpenAI nodes.  
    - Runway API key for HTTP Request node.

15. **Set default values and constraints where applicable:**  
    - Polling interval for Google Sheets trigger set to every minute.  
    - YouTube titles limited to ≤ 70 characters.  
    - Thumbnail resolution fixed at 1280x720.  
    - Number of tags/keywords capped at 20.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Step-by-step tutorial video available: [YouTube Video](https://youtu.be/R426d84slWY?si=Ecozf55ypxg6PnQz)      | Sticky Note in workflow provides direct video link for detailed usage instructions.                         |
| Prompt for generating YouTube SEO content includes titles, tags, keywords, description, and thumbnail prompts | Found in Sticky Note1 attached to SEO content generation node.                                              |
| JavaScript code used to parse multi-section AI responses into structured fields                               | Provided in Sticky Note2 for reference and debugging.                                                       |
| Python code for dynamically creating thumbnail generation prompt for Runway API                               | Included in Sticky Note3, helpful for understanding image generation customization.                         |
| Reminders for API keys setup for OpenAI and Runway API are included as sticky notes for user guidance        | Sticky Note4 (OpenAI), Sticky Note6 (Runway), and Sticky Note5 (Google Sheets) remind credential setup.     |

---

This documentation provides the detailed structural, functional, and technical breakdown needed to understand, reproduce, and modify the YouTube SEO Automation workflow in n8n. It highlights conditional logic, AI model usage, data parsing, and external API integration with emphasis on potential failure points and configuration details.