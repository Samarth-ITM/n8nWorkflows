Automatically Create YouTube Metadata with AI

https://n8nworkflows.xyz/workflows/automatically-create-youtube-metadata-with-ai-2976


# Automatically Create YouTube Metadata with AI

### 1. Workflow Overview

This workflow automates the generation and updating of YouTube video metadata using AI, specifically targeting content creators, marketers, and affiliate promoters. It accepts user input including a YouTube video link, transcript, and optional focus keywords, then processes this data to produce optimized video titles, descriptions, tags, hashtags, and call-to-action elements. The workflow integrates affiliate and promotional links retrieved from a Google Docs source and directly updates the video metadata on YouTube via the YouTube API.

The workflow is logically divided into the following blocks:

- **1.1 User Input Reception:** Captures video link, transcript, and optional keywords via a web form.
- **1.2 Video ID Extraction:** Parses the YouTube URL to extract the video ID.
- **1.3 Affiliate & Promotional Link Retrieval:** Fetches relevant links from a Google Docs document.
- **1.4 AI-Powered Metadata Generation:** Uses OpenAI GPT-4 via a LangChain agent to generate structured metadata.
- **1.5 Metadata Parsing and Formatting:** Parses AI output and formats tags for YouTube.
- **1.6 YouTube Metadata Update:** Updates the video metadata on YouTube using the YouTube API.
- **1.7 Confirmation Display:** Shows a success message with updated video details.
- **1.8 Documentation and Credits:** Provides workflow credits and support information via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 User Input Reception

- **Overview:**  
  Captures user inputs through a web form, including the YouTube video link, transcript, and optional focus keywords.

- **Nodes Involved:**  
  - On form submission (Form Trigger)

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point capturing user inputs via a web form titled "Syncbricks Youtube" with fields for YouTube Video Link (required), Video Transcript (required), and Focus Keywords (optional).  
    - Key expressions: Captures raw input JSON for downstream processing.  
    - Input: HTTP webhook trigger on form submission.  
    - Output: JSON containing user inputs.  
    - Edge cases: Missing required fields; invalid URL format not explicitly validated here.  
    - Version: 2.2

#### 2.2 Video ID Extraction

- **Overview:**  
  Extracts the YouTube video ID from the provided URL to facilitate API calls.

- **Nodes Involved:**  
  - Extract Video ID (Set node)

- **Node Details:**  
  - **Extract Video ID**  
    - Type: Set  
    - Role: Parses the YouTube URL string to isolate the video ID by removing the "https://youtu.be/" prefix.  
    - Configuration: Uses an expression to replace the URL prefix with an empty string, storing the result in `videoID`.  
    - Input: Output from "Youtube Meta Generator" node (which is triggered after form submission).  
    - Output: JSON with `videoID` string.  
    - Edge cases: URLs in formats other than "https://youtu.be/..." may not parse correctly; no validation for malformed URLs.  
    - Version: 3.4

#### 2.3 Affiliate & Promotional Link Retrieval

- **Overview:**  
  Retrieves affiliate, course, social media, and other promotional links from a Google Docs document to enrich video metadata.

- **Nodes Involved:**  
  - syncbricks information (Google Docs Tool)

- **Node Details:**  
  - **syncbricks information**  
    - Type: Google Docs Tool  
    - Role: Fetches content from a specific Google Docs document identified by its document ID.  
    - Configuration: Operation set to "get" with a manual description indicating it contains affiliate and promotional links.  
    - Input: None (runs independently, data used by AI agent).  
    - Output: Text content of the document for use as a tool by the AI agent.  
    - Edge cases: Google Docs API authentication errors, document access permissions, or empty document content.  
    - Version: 2

#### 2.4 AI-Powered Metadata Generation

- **Overview:**  
  Uses a LangChain AI agent powered by OpenAI GPT-4 to generate structured YouTube metadata based on the transcript, keywords, and affiliate/promotional links.

- **Nodes Involved:**  
  - OpenAI Chat Model (Language Model)  
  - Youtube Meta Generator (LangChain Agent)  
  - Output Parser (Structured Output Parser)

- **Node Details:**  
  - **OpenAI Chat Model**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o-mini model for natural language generation.  
    - Configuration: Model set to "gpt-4o-mini" with default options.  
    - Input: Prompt from "Youtube Meta Generator".  
    - Output: Raw AI-generated text.  
    - Edge cases: API rate limits, authentication errors, model unavailability.  
    - Version: 1.2

  - **Youtube Meta Generator**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI prompt execution, uses "syncbricks information" as a tool to incorporate affiliate/promotional links, and generates a JSON-formatted metadata output.  
    - Configuration:  
      - Prompt instructs the AI to generate a structured JSON with title, description, keywords, hashtags, affiliate links, call to action, and additional promotional links.  
      - Emphasizes SEO optimization, natural language, and inclusion of relevant links.  
      - Uses input variables: Video Transcript, Focus Keywords, and syncbricks information content.  
      - Limits iterations to 10 for prompt refinement.  
    - Input: Form submission data and syncbricks information tool output.  
    - Output: AI-generated structured JSON metadata.  
    - Edge cases: AI hallucination, incomplete or malformed JSON output, missing or irrelevant affiliate links.  
    - Version: 1.7

  - **Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Validates and parses the AI-generated JSON output against a defined JSON schema to extract metadata fields.  
    - Configuration: Manual schema defining expected fields: title, description, tags (array), hashtags (array), call_to_action (string).  
    - Input: Raw AI output from "Youtube Meta Generator".  
    - Output: Parsed JSON object with structured metadata.  
    - Edge cases: Parsing failures if AI output deviates from schema, missing fields.  
    - Version: 1.2

#### 2.5 Metadata Formatting and Update

- **Overview:**  
  Formats tags into a comma-separated string and updates the YouTube video metadata using the YouTube API.

- **Nodes Involved:**  
  - Format Tags (Set)  
  - YouTube (YouTube API node)

- **Node Details:**  
  - **Format Tags**  
    - Type: Set  
    - Role: Converts the array of tags from AI output into a single comma-separated string for YouTube API compatibility.  
    - Configuration: Uses expression to join tags array with commas.  
    - Input: Parsed metadata from "Youtube Meta Generator".  
    - Output: JSON with `formatted_tags` string.  
    - Edge cases: Empty or null tags array results in empty string.  
    - Version: 3.4

  - **YouTube**  
    - Type: YouTube API  
    - Role: Updates video metadata fields including title, description, and tags on YouTube.  
    - Configuration:  
      - Operation: Update video resource.  
      - Uses video ID from "Extract Video ID".  
      - Title, description, and tags populated from AI-generated metadata.  
      - Description appended with fixed social media and contact links plus AI-generated call to action and hashtags.  
      - Category ID set to 28 (Science & Technology).  
      - Region code set to "OM" (Oman).  
    - Input: Formatted tags and AI metadata.  
    - Output: API response confirming update.  
    - Edge cases: API authentication failure, invalid video ID, quota limits, network errors.  
    - Version: 1

#### 2.6 Confirmation Display

- **Overview:**  
  Displays a confirmation message to the user indicating successful video metadata update.

- **Nodes Involved:**  
  - Form (Completion)

- **Node Details:**  
  - **Form**  
    - Type: Form (Completion)  
    - Role: Sends a completion message back to the user with the updated video title and video link.  
    - Configuration:  
      - Completion title and message dynamically populated with video title and original YouTube link.  
    - Input: Output from "YouTube" node.  
    - Output: HTTP response to user.  
    - Edge cases: Failure to receive update confirmation from YouTube node.  
    - Version: 1

#### 2.7 Documentation and Credits

- **Overview:**  
  Provides workflow credits, support information, and external resource links for users.

- **Nodes Involved:**  
  - Sticky Note11  
  - Sticky Note

- **Node Details:**  
  - **Sticky Note11**  
    - Type: Sticky Note  
    - Role: Contains detailed credits by Amjid Ali, PayPal donation link, course links, contact info, and social media profiles.  
    - Position: Visual documentation only, no data flow.  
    - Content includes:  
      - PayPal donation URL  
      - SyncBricks LMS and Udemy course links  
      - Contact email and LinkedIn  
      - YouTube channel link  
    - Version: 1

  - **Sticky Note**  
    - Type: Sticky Note  
    - Role: Notes about customizing the YouTube Meta Generator for personal channels.  
    - Version: 1

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                          | Input Node(s)              | Output Node(s)           | Sticky Note                                                                                      |
|---------------------|--------------------------------------|----------------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                         | Captures user input (video link, transcript, keywords) | -                         | Youtube Meta Generator    |                                                                                                |
| Youtube Meta Generator | LangChain Agent                    | Generates structured YouTube metadata using AI and affiliate links | On form submission, syncbricks information, OpenAI Chat Model, Output Parser | Extract Video ID          | Sticky Note: Customize it for your own YouTube channel                                         |
| syncbricks information | Google Docs Tool                   | Retrieves affiliate and promotional links from Google Docs | -                         | Youtube Meta Generator    |                                                                                                |
| OpenAI Chat Model    | LangChain OpenAI Chat Model          | Provides GPT-4o-mini model for AI text generation | Youtube Meta Generator     | Youtube Meta Generator    |                                                                                                |
| Output Parser       | LangChain Structured Output Parser    | Parses AI JSON output into structured metadata | Youtube Meta Generator     | Youtube Meta Generator    |                                                                                                |
| Extract Video ID     | Set                                  | Extracts video ID from YouTube URL     | Youtube Meta Generator     | Format Tags               |                                                                                                |
| Format Tags         | Set                                  | Formats tags array into comma-separated string | Extract Video ID           | YouTube                  |                                                                                                |
| YouTube             | YouTube API                         | Updates video metadata on YouTube      | Format Tags                | Form                     |                                                                                                |
| Form                | Form Completion                      | Displays success confirmation message  | YouTube                   | -                        |                                                                                                |
| Sticky Note11       | Sticky Note                         | Workflow credits, support, and resource links | -                         | -                        | Developed by Amjid Ali. Support via PayPal: http://paypal.me/pmptraining. See SyncBricks LMS and courses. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node ("On form submission")**  
   - Type: Form Trigger  
   - Configure webhook with ID `syncbricks-youtube-meta-automation`  
   - Form Title: "Syncbricks Youtube"  
   - Fields:  
     - "Youtube Video Link" (required)  
     - "Video Transcript" (required)  
     - "Focus Keywords" (optional, placeholder text)  
   - Button Label: "Update Youtube Video"  
   - Form Description: "Generate Youtube Video Title, Description, Tags and Hashtags"

2. **Create a Google Docs Tool Node ("syncbricks information")**  
   - Type: Google Docs Tool  
   - Operation: Get  
   - Document URL: Use document ID `15lN3FJ3iXABf_bd061-F7j-gGx2WBH8Jr6fjBLa3tis`  
   - Description: "affiliate links, course links, social media links and other relevant links related to syncbricks"  
   - Configure Google Docs credentials with appropriate OAuth2 access.

3. **Create an OpenAI Chat Model Node ("OpenAI Chat Model")**  
   - Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Configure OpenAI credentials with API key.

4. **Create a LangChain Agent Node ("Youtube Meta Generator")**  
   - Type: LangChain Agent  
   - Text prompt: Use the detailed prompt instructing AI to generate structured JSON metadata including title, description, keywords, hashtags, affiliate links, call to action, and additional promotional links.  
   - Tools: Add "syncbricks information" as a tool for affiliate/promotional link retrieval.  
   - Max iterations: 10  
   - Input variables: Map from form submission fields for transcript and focus keywords.  
   - Connect "On form submission" main output to this node's main input.  
   - Connect "syncbricks information" output to this node's AI tool input.  
   - Connect "OpenAI Chat Model" output to this node's AI language model input.

5. **Create an Output Parser Node ("Output Parser")**  
   - Type: LangChain Structured Output Parser  
   - Schema: Define JSON schema expecting fields: video_title (string), video_description (string), youtube_metadata (object with title, description, tags array, hashtags array, call_to_action string), additional_notes (string).  
   - Connect "Youtube Meta Generator" AI output parser output to this node's input.

6. **Create a Set Node ("Extract Video ID")**  
   - Type: Set  
   - Assignment: Create string variable `videoID` by extracting from the YouTube link:  
     Expression: `={{ $('On form submission').item.json['Youtube Video Link'].replace("https://youtu.be/","") }}`  
   - Connect "Youtube Meta Generator" main output to this node.

7. **Create a Set Node ("Format Tags")**  
   - Type: Set  
   - Assignment: Create string variable `formatted_tags` by joining tags array with commas:  
     Expression: `={{ $('Youtube Meta Generator').item.json.output.youtube_metadata.tags.join() }}`  
   - Connect "Extract Video ID" main output to this node.

8. **Create a YouTube Node ("YouTube")**  
   - Type: YouTube API  
   - Operation: Update video resource  
   - Video ID: Use `={{ $('Extract Video ID').item.json.videoID }}`  
   - Title: Use `={{ $('Youtube Meta Generator').item.json.output.youtube_metadata.title }}`  
   - Description: Use AI-generated description appended with fixed social media links and call to action:  
     ```
     {{ $('Youtube Meta Generator').item.json.output.youtube_metadata.description }}

     Connect with us : 
     Facebook: https://www.facebook.com/syncbricks
     LinkedIn : https://linkedin.com/company/syncbricks
     Instagram : https://instagram.com/syncbricks_com

     Subscribe to youtube Channel : https://www.youtube.com/channel/UC1ORA3oNGYuQ8yQHrC7MzBg?sub_confirmation=1

     Website : 
     Sync Bricks: https://syncbricks.com/

     Contact : info@syncbricks.com

     {{ $('Youtube Meta Generator').item.json.output.youtube_metadata.call_to_action }}

     {{ $('Youtube Meta Generator').item.json.output.youtube_metadata.hashtags }}
     ```  
   - Tags: Use `={{ $json.formatted_tags }}`  
   - Category ID: 28 (Science & Technology)  
   - Region Code: OM (Oman)  
   - Configure YouTube API credentials with OAuth2.

9. **Create a Form Completion Node ("Form")**  
   - Type: Form Completion  
   - Completion Title: `=Video is updated with Title : {{ $json.snippet.title }}`  
   - Completion Message: `=Video is updated with Title : {{ $json.snippet.title }} and below is the video link\n{{ $('On form submission').item.json['Youtube Video Link'] }}`  
   - Connect "YouTube" main output to this node.

10. **Add Sticky Notes for Documentation**  
    - Add a sticky note with workflow credits and support information (content from Sticky Note11).  
    - Add a sticky note about customizing the YouTube Meta Generator (content from Sticky Note).

11. **Connect Nodes in the Following Order:**  
    - On form submission → Youtube Meta Generator  
    - syncbricks information → Youtube Meta Generator (AI tool input)  
    - OpenAI Chat Model → Youtube Meta Generator (AI language model input)  
    - Youtube Meta Generator → Output Parser (AI output parser input)  
    - Output Parser → Youtube Meta Generator (ai_outputParser output)  
    - Youtube Meta Generator → Extract Video ID  
    - Extract Video ID → Format Tags  
    - Format Tags → YouTube  
    - YouTube → Form (completion)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Developed by Amjid Ali. Support via PayPal: http://paypal.me/pmptraining. For full courses on ERPNext and AI automation, visit http://lms.syncbricks.com.                                                                                                                                                                                                       | Sticky Note11 content                                                                            |
| Customize the YouTube Meta Generator prompt for your own channel needs.                                                                                                                                                                                                                                                                                         | Sticky Note content                                                                             |
| Watch the tutorial video on this workflow: https://youtu.be/VNpr9T00Is0                                                                                                                                                                                                                                                                                         | Workflow description section                                                                    |
| More courses and resources available at SyncBricks LMS: https://lms.syncbricks.com/ and Udemy course: https://www.udemy.com/course/ai-automation-mastery-build-intelligent-agents-with-lowcode/?referralCode=0062E7C1D64784AB70CA                                                                                                                                  | Workflow description section                                                                    |
| Contact info@syncbricks.com for support. Follow on LinkedIn: http://linkedin.com/in/amjidali, YouTube: https://www.youtube.com/channel/UC1ORA3oNGYuQ8yQHrC7MzBg?sub_confirmation=1, and website https://syncbricks.com                                                                                                                                               | Workflow description section                                                                    |
| Subscribe for more AI & automation workflows: https://n8n.syncbricks.com                                                                                                                                                                                                                                                                                        | Workflow description section                                                                    |

---

This comprehensive documentation enables advanced users and AI agents to fully understand, reproduce, and customize the workflow for automated YouTube metadata generation with integrated affiliate marketing and AI optimization.