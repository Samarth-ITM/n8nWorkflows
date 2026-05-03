Generate LinkedIn Activity Reports via Slack Commands with GPT-4.1 and Email

https://n8nworkflows.xyz/workflows/generate-linkedin-activity-reports-via-slack-commands-with-gpt-4-1-and-email-11052


# Generate LinkedIn Activity Reports via Slack Commands with GPT-4.1 and Email

### 1. Workflow Overview

This n8n workflow enables users to generate detailed LinkedIn activity reports by invoking a Slack slash command. When a user types `/check-linkedin [firstName lastName]` in Slack, the workflow extracts the name, finds the corresponding LinkedIn profile using Apify actors, scrapes recent posts, and uses GPT-4.1 AI models to analyze posting frequency, topics, and highlights. Finally, it composes a formatted HTML report and sends it via email to a designated recipient.

The workflow is logically divided into these blocks:

- **1.1 Slack Command Intake & Validation:** Receive Slack slash command input, validate if it contains a full name.
- **1.2 Name Extraction:** Use GPT to extract first and last names from the input.
- **1.3 LinkedIn Profile Lookup:** Use Apify actors to find the LinkedIn profile URL for the extracted name.
- **1.4 LinkedIn Posts Scraping:** Scrape recent LinkedIn posts of the profile.
- **1.5 Posts Aggregation:** Aggregate and structure the scraped posts for analysis.
- **1.6 AI Analysis & Summarization:** Use GPT-4.1 and a Langchain structured output parser to analyze posts and generate a detailed summary.
- **1.7 HTML Report Generation & Email Delivery:** Format the AI summary into an HTML report and email it to the requester.
- **1.8 Error Handling & Slack Feedback:** Notify Slack if the input name is invalid.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Slack Command Intake & Validation

**Overview:**  
This block receives the Slack slash command POST request, checks if the input text is a full name, branching to error messaging if not.

**Nodes Involved:**  
- Webhook  
- GPT 4.1-mini for classification  
- Do we have a full name ? (text classifier)  
- Tell on slack that no full name was found

**Node Details:**  
- **Webhook**  
  - Type: HTTP Webhook (POST)  
  - Configuration: Receives Slack slash command at a defined path (placeholder to be replaced)  
  - Input: Slack slash command payload (JSON)  
  - Output: Passes the raw Slack command data downstream  
  - Edge cases: Invalid HTTP method or malformed requests  

- **GPT 4.1-mini for classification**  
  - Type: OpenAI GPT chat model (GPT 4.1-mini)  
  - Role: Classify if input text is a full name or not  
  - Input: `{{ $json.body.text }}` (text entered in Slack command)  
  - Output: Prediction used for branching  
  - Edge cases: API errors, rate limiting, classification uncertainty  

- **Do we have a full name ?**  
  - Type: Text Classifier node (Langchain)  
  - Role: Categorize input as "full-name" or "not-full-name"  
  - InputText: `{{ $json.body.text }}`  
  - Categories: "full-name", "not-full-name"  
  - Output: Branches workflow based on category  
  - Edge cases: Misclassification, expression evaluation failures  

- **Tell on slack that no full name was found**  
  - Type: Slack node (chat message)  
  - Role: Sends a message back to the Slack channel notifying the user that the input was not a full name  
  - Configuration: Uses OAuth2 Slack credential, posts to the originating channel `{{ $json.body.channel_id }}`  
  - Input: Triggered if name classification is negative  
  - Edge cases: Slack API auth failures, invalid channel IDs  

---

#### 1.2 Name Extraction

**Overview:**  
Extract first name and last name explicitly from the validated input text using GPT.

**Nodes Involved:**  
- GPT 4.1-mini to extract firstName + lastName  
- Extract firstName and lastName (Information Extractor node)

**Node Details:**  
- **GPT 4.1-mini to extract firstName + lastName**  
  - Type: OpenAI GPT chat model (GPT 4.1-mini)  
  - Role: Refine and normalize the name extraction process  
  - Input: Raw Slack command text  
  - Output: Passes text to the information extractor node  
  - Edge cases: API errors, ambiguous names  

- **Extract firstName and lastName**  
  - Type: Langchain Information Extractor  
  - Role: Identifies and extracts structured attributes: `firstName` and `lastName`  
  - Input: Text from previous GPT node  
  - Output: JSON with `firstName` and `lastName` fields (both required)  
  - Edge cases: Missing attributes, parsing failures  

---

#### 1.3 LinkedIn Profile Lookup

**Overview:**  
Find the LinkedIn profile URL for the extracted person using Apify actor.

**Nodes Involved:**  
- Find Linkedin Profile (Apify actor)  
- Get Linkedin Profile URL (Apify dataset query)

**Node Details:**  
- **Find Linkedin Profile**  
  - Type: Apify actor call  
  - Role: Runs an Apify actor (configured via URL) that searches LinkedIn based on the first and last name  
  - Input: JSON body with `firstName` and `lastName`  
  - Output: Returns datasetId for collected profile data  
  - Edge cases: Actor timeout, invalid input, no profile found  

- **Get Linkedin Profile URL**  
  - Type: Apify dataset query  
  - Role: Retrieves profile URL from Apify dataset using the datasetId from previous node  
  - Input: Uses `defaultDatasetId` from previous node  
  - Output: Extracted LinkedIn profile URL (`linkedinProfileUrl`)  
  - Edge cases: Dataset empty, API errors  

---

#### 1.4 LinkedIn Posts Scraping

**Overview:**  
Scrape recent posts of the found LinkedIn profile via another Apify actor.

**Nodes Involved:**  
- Scrap what this person posted recently (Apify actor)  
- Structure recent posts (Apify dataset query)  
- Aggregate (Aggregate node)

**Node Details:**  
- **Scrap what this person posted recently**  
  - Type: Apify actor call  
  - Role: Runs Apify actor to scrape LinkedIn posts for the given profile URL  
  - Input: JSON body with `usernames` array containing the LinkedIn profile URL  
  - Output: Dataset id for scraped posts  
  - Edge cases: Actor failure, rate limits, empty posts  

- **Structure recent posts**  
  - Type: Apify dataset query  
  - Role: Retrieves up to 20 recent posts from Apify dataset  
  - Input: Dataset ID from scraping node  
  - Output: List of post objects with text and posted_at.date properties  
  - Edge cases: Empty dataset, partial data  

- **Aggregate**  
  - Type: Aggregate node (standard n8n)  
  - Role: Collects and merges fields `posted_at.date` and `text` from the posts into a single object  
  - Input: List of recent posts  
  - Output: Aggregated JSON data for analysis  
  - Edge cases: Empty input array, aggregation errors  

---

#### 1.5 AI Analysis & Summarization

**Overview:**  
Analyze aggregated LinkedIn posts data with GPT 4.1 to generate a detailed analytical summary following a strict JSON schema.

**Nodes Involved:**  
- Summer McBriefing (Langchain Agent)  
- Structured Output Parser (Langchain Output Parser)  
- GPT 4.1 (OpenAI chat model)

**Node Details:**  
- **Summer McBriefing**  
  - Type: Langchain Agent (AI language model with custom prompt)  
  - Role: Receives aggregated post dates and text, generates a JSON report including posting frequency, topics, highlights, and data quality stats  
  - Prompt: Detailed instructions to produce exactly the required JSON schema, no extra fields or prose  
  - Input: Aggregated dates and texts from previous nodes  
  - Output: JSON object matching LinkedInPostsAnalysis schema  
  - Edge cases: Parsing ambiguity, incomplete input data, GPT API errors  

- **Structured Output Parser**  
  - Type: Langchain Output Parser  
  - Role: Validates and enforces JSON structure according to the provided JSON schema  
  - Input: Raw JSON from Summer McBriefing  
  - Output: Structured, validated JSON for further processing  
  - Edge cases: Schema validation failures, parser errors  

- **GPT 4.1**  
  - Type: OpenAI GPT chat model (GPT 4.1)  
  - Role: Connected as AI languageModel input to Summer McBriefing (used internally)  
  - Edge cases: API failures, rate limits  

---

#### 1.6 HTML Report Generation & Email Delivery

**Overview:**  
Generate an HTML report from the structured AI output and send it by email to a configured recipient.

**Nodes Involved:**  
- HTML (HTML generation)  
- Send report via Email (Gmail node)  
- Sticky Note12 (Contact info)

**Node Details:**  
- **HTML**  
  - Type: HTML node (n8n built-in)  
  - Role: Formats the JSON report into a styled, professional HTML email  
  - Uses Mustache-style expressions to inject AI output and extracted names into the template  
  - Includes summary, posting frequency, topics, and highlights in a visually clear layout  
  - Edge cases: Missing JSON fields, malformed HTML  

- **Send report via Email**  
  - Type: Gmail node (OAuth2)  
  - Role: Sends the generated HTML email to a fixed recipient (`admin@example.com`)  
  - Subject includes person’s name dynamically: `Recent LinkedIn Activity about [personName]`  
  - Requires Gmail OAuth2 credentials  
  - Edge cases: Email sending failures, auth errors, invalid recipient  

- **Sticky Note12**  
  - Type: Sticky Note  
  - Role: Contains contact information for questions or feedback (email and LinkedIn)  
  - No direct functional role in automation  

---

#### 1.7 Error Handling & Slack Feedback

**Overview:**  
If the input text is not a full name, this block provides user feedback on Slack.

**Nodes Involved:**  
- Tell on slack that no full name was found  
- Sticky Note1 (Error handling explanation)

**Node Details:**  
- **Tell on slack that no full name was found**  
  - As described in 1.1  
- **Sticky Note1**  
  - Provides a note reminding the user about error handling for no full name found input  

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                              | Input Node(s)                  | Output Node(s)                  | Sticky Note                                                   |
|-----------------------------------|--------------------------------------|----------------------------------------------|-------------------------------|--------------------------------|---------------------------------------------------------------|
| Webhook                           | Webhook                              | Receive Slack slash command request          | —                             | GPT 4.1-mini for classification |                                                               |
| GPT 4.1-mini for classification   | OpenAI GPT chat model                 | Classify if input is a full name              | Webhook                       | Do we have a full name ?        |                                                               |
| Do we have a full name ?           | Text classifier (Langchain)           | Branch workflow on full name check           | GPT 4.1-mini for classification | Extract firstName and lastName / Tell on slack no full name | Sticky Note1: No full name found error handling               |
| Tell on slack that no full name was found | Slack message node                   | Notify user on Slack of invalid input         | Do we have a full name ?       | —                              | Sticky Note1: No full name found error handling               |
| GPT 4.1-mini to extract firstName + lastName | OpenAI GPT chat model           | Extract first and last name from input       | Do we have a full name ?       | Extract firstName and lastName  |                                                               |
| Extract firstName and lastName    | Langchain Information Extractor       | Extract structured firstName and lastName    | GPT 4.1-mini to extract names  | Find Linkedin Profile           |                                                               |
| Find Linkedin Profile             | Apify actor                          | Find LinkedIn profile URL from name          | Extract firstName and lastName | Get Linkedin Profile URL        | Sticky Note3: Data gathering via Apify                        |
| Get Linkedin Profile URL          | Apify dataset query                   | Retrieve LinkedIn profile URL                 | Find Linkedin Profile          | Scrap what this person posted recently | Sticky Note3: Data gathering via Apify                    |
| Scrap what this person posted recently | Apify actor                      | Scrape recent LinkedIn posts                  | Get Linkedin Profile URL       | Structure recent posts          | Sticky Note3: Data gathering via Apify                        |
| Structure recent posts            | Apify dataset query                   | Retrieve structured posts (limit 20)          | Scrap what this person posted recently | Aggregate                   | Sticky Note3: Data gathering via Apify                        |
| Aggregate                       | Aggregate node                        | Aggregate post dates and texts                | Structure recent posts         | Summer McBriefing              | Sticky Note4: Summarization & Intelligence Layer             |
| Summer McBriefing                | Langchain Agent                      | Analyze posts and generate JSON report       | Aggregate                     | HTML                          | Sticky Note4: Summarization & Intelligence Layer             |
| Structured Output Parser         | Langchain Output Parser               | Validate and structure AI output JSON         | Summer McBriefing              | Summer McBriefing (loop back)  |                                                               |
| HTML                            | HTML node                            | Format JSON summary into styled HTML email   | Summer McBriefing              | Send report via Email          | Sticky Note5: HTML Output & Email                             |
| Send report via Email            | Gmail node                          | Email the report to recipient                  | HTML                         | —                             |                                                               |
| Sticky Note12                   | Sticky Note                         | Contact info for questions and feedback        | —                             | —                             |                                                               |
| Sticky Note                      | Sticky Note                         | Workflow overview and instructions             | —                             | —                             |                                                               |
| Sticky Note2                     | Sticky Note                         | Slack intake and parsing explanation           | —                             | —                             |                                                               |
| Sticky Note3                     | Sticky Note                         | Notes on Apify data gathering                   | —                             | —                             | Sticky Note3: Data gathering via Apify                        |
| Sticky Note4                     | Sticky Note                         | Summarization and AI layer explanation         | —                             | —                             | Sticky Note4: Summarization & Intelligence Layer             |
| Sticky Note5                     | Sticky Note                         | HTML output and email explanation               | —                             | —                             | Sticky Note5: HTML Output & Response via Email               |
| Sticky Note9                     | Sticky Note                         | Video walkthrough link                           | —                             | —                             | # 🎙️ Video Walkthrough @[youtube](-4VwNYsOIPc)               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/path-placeholder-to-be-replaced-by-your-webhook` (replace with your actual path)  
   - Purpose: Receive Slack slash command payloads  

2. **Add GPT 4.1-mini for Classification Node**  
   - Type: OpenAI GPT chat model  
   - Model: `gpt-4.1-mini`  
   - Input: Text from `Webhook` node — `{{$json.body.text}}`  
   - Purpose: Classify if input is a full name or not  

3. **Add Text Classifier Node "Do we have a full name ?"**  
   - Type: Langchain Text Classifier  
   - Input Text: `{{$json.body.text}}`  
   - Categories:  
     - `full-name` (input is a full name)  
     - `not-full-name` (otherwise)  
   - Connect from GPT classification node  
   - Purpose: Branch the workflow  

4. **Add Slack Node for Invalid Name Feedback**  
   - Type: Slack  
   - Action: Send Message to Channel  
   - Channel: `{{$json.body.channel_id}}` (from Webhook)  
   - Text: `"{{$json.body.text}} is not a full name. Please send a full name to get information."`  
   - Authentication: OAuth2 (Slack credentials)  
   - Connect from classifier node on `not-full-name` branch  

5. **Add GPT 4.1-mini to Extract First and Last Name**  
   - Model: `gpt-4.1-mini`  
   - Input: Slack command text `{{$json.body.text}}`  
   - Connect from classifier node on `full-name` branch  

6. **Add Information Extractor Node**  
   - Type: Langchain Information Extractor  
   - Input Text: Output text from previous GPT node  
   - Attributes:  
     - `firstName` (required)  
     - `lastName` (required)  
   - Connect from GPT extraction node  

7. **Add Apify Actor Node "Find Linkedin Profile"**  
   - Actor URL: `https://console.apify.com/actors/FbqC9BRstFBddhUqj/input`  
   - Input JSON Body:  
     ```json
     {
       "firstName": "{{ $json.output.firstName }}",
       "lastName": "{{ $json.output.lastName }}"
     }
     ```  
   - Connect from Information Extractor node  
   - Credentials: Apify API key  

8. **Add Apify Dataset Query Node "Get Linkedin Profile URL"**  
   - Resource: Datasets  
   - Dataset ID: `={{ $json.defaultDatasetId }}` (from previous node)  
   - Connect from "Find Linkedin Profile"  

9. **Add Apify Actor Node "Scrap what this person posted recently"**  
   - Actor URL: `https://console.apify.com/actors/r4oNX7IHlW4RQAjKP`  
   - Input JSON Body:  
     ```json
     {
       "usernames": ["{{ $json.linkedinProfileUrl }}"]
     }
     ```  
   - Connect from "Get Linkedin Profile URL"  
   - Credentials: Apify API key  

10. **Add Apify Dataset Query Node "Structure recent posts"**  
    - Resource: Datasets  
    - Dataset ID: `={{ $json.defaultDatasetId }}` (from previous node)  
    - Limit: 20  
    - Connect from "Scrap what this person posted recently"  

11. **Add Aggregate Node**  
    - Fields to Aggregate:  
      - `posted_at.date`  
      - `text`  
    - Connect from "Structure recent posts"  

12. **Add Langchain Agent Node "Summer McBriefing"**  
    - Text Input Template:  
      ```
      Dates:
      {{ $json.date }}

      Posts: 
      {{ $json.text }}
      ```
    - System Message: Detailed analytical instructions to produce JSON summary with posting frequency, topics, highlights, and data quality  
    - Input: Connect from Aggregate node  
    - Built-in Tools: GPT 4.1 model (configured)  
    - Credentials: OpenAI API key  

13. **Add Langchain Output Parser Node "Structured Output Parser"**  
    - Schema: Use the detailed LinkedInPostsAnalysis JSON schema embedded in the workflow  
    - Connect as AI output parser to "Summer McBriefing" node  

14. **Add HTML Node**  
    - Paste the provided HTML template from the workflow, including placeholders for dynamic data injection using n8n expressions, e.g., `{{ $json.output.latest_post_date }}`  
    - Connect from "Summer McBriefing" node output  

15. **Add Gmail Node "Send report via Email"**  
    - Send To: `admin@example.com` (replace with desired recipient)  
    - Subject: `Recent LinkedIn Activity about {{ $('Get Linkedin Profile URL').item.json.personName }}`  
    - Message: `={{ $json.html }}` (HTML output from previous node)  
    - Credentials: Gmail OAuth2 credentials configured in n8n  
    - Connect from HTML node  

16. **Set up Slack App and Slash Command**  
    - Create Slack App with `/check-linkedin` slash command  
    - Set Request URL to your n8n webhook URL path  
    - Ensure Slack OAuth2 credentials have `chat:write` and `commands` scopes  

17. **Configure Credentials**  
    - Apify API key with access to required actors  
    - OpenAI API key with GPT 4.1 and GPT 4.1-mini  
    - Slack OAuth2 token with required scopes  
    - Gmail OAuth2 account for sending emails  

18. **Testing & Validation**  
    - Test slash command with valid full names to verify end-to-end functionality  
    - Test with invalid inputs to verify error handling messages in Slack  
    - Monitor Apify actor usage and costs  
    - Monitor OpenAI API usage and costs  

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| Slack Slash Command: Run `/check-linkedin [firstName lastName]` in Slack to get LinkedIn activity. | Sticky Note overview at the start of the workflow               |
| Workflow requires two Apify actors for LinkedIn profile lookup and post scraping.                   | Sticky Note3: Data gathering via Apify                           |
| Summarization and intelligence layer uses GPT 4.1 with a detailed JSON schema for output format.   | Sticky Note4: Summarization & Intelligence Layer                |
| HTML email format includes profile name, activity stats, topics, and highlights.                    | Sticky Note5: HTML Output & Response via Email                   |
| Video walkthrough available on YouTube: `@[youtube](-4VwNYsOIPc)`                                  | Sticky Note9: Video Walkthrough                                  |
| Contact for feedback: Emir Belkahia, email: emir.belkahia@gmail.com, LinkedIn: linkedin.com/in/emirbelkahia | Sticky Note12                                                  |
| Cost considerations: Apify actors and GPT calls incur usage costs; optimize parameters accordingly. | Sticky Note3 and Sticky Note4                                    |

---

This document comprehensively covers the workflow logic, node configurations, integration points, and reproduction steps to facilitate thorough understanding, modification, and deployment by advanced users or automation agents.