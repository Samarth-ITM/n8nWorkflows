Personalized Lead Response with AI Videos using Scout, GPT-4, and HeyGen

https://n8nworkflows.xyz/workflows/personalized-lead-response-with-ai-videos-using-scout--gpt-4--and-heygen-9967


# Personalized Lead Response with AI Videos using Scout, GPT-4, and HeyGen

### 1. Workflow Overview

This workflow automates personalized lead engagement by responding to inbound form submissions with AI-generated outreach videos and follow-up emails. It is designed for real estate or lead generation agencies aiming to create hyper-personalized video responses to new leads, leveraging data from Scout, script generation via GPT-4 with LangChain, video creation via HeyGen API, and email delivery via Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures lead data from a web form submission.
- **1.2 Lead Data Enrichment:** Queries the Scout API to enrich lead information.
- **1.3 AI Script Generation:** Uses LangChain with GPT-4 to generate a personalized video script.
- **1.4 Video Generation & Monitoring:** Creates the personalized video via HeyGen API and waits until the video is ready.
- **1.5 Outreach Email Composition:** Generates a customized email with embedded video thumbnail and booking call-to-action.
- **1.6 Email Delivery:** Sends the composed email to the lead via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Initiates the workflow upon receiving new lead data via a form submission.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Captures inbound lead data from a web form titled "Inbound Form".  
    - Configuration: The form has fields such as first name, middle name (email type), last name, address, city, state, county, zip code, exact match (dropdown), minscore, maxscore, count, level (dropdown), phone, email, and count.  
    - Input/Output: Outputs the submitted form data as JSON for downstream processing.  
    - Edge Cases: Missing or malformed form input; delayed form submissions; duplicate submissions.  
    - Sticky Note: "Form Submission Trigger"

---

#### 2.2 Lead Data Enrichment

- **Overview:**  
  Enriches incoming lead data by querying the Scout API to retrieve detailed lead information.

- **Nodes Involved:**  
  - HTTP Request (Scout API lookup)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request  
    - Role: Posts the form-submitted lead data to Scout's people lookup API endpoint to fetch enriched data.  
    - Configuration: POST request with JSON body containing lead fields mapped from the form submission, including first name, middle name, last name, address, city, state, county, zip code, exactmatch (hardcoded false), minscore (0.0), maxscore (100.0), count (1), level, phone, and email.  
    - Headers: X-Api-Key for Scout API authentication (placeholder to be replaced with real key).  
    - Input/Output: Input from form submission node; outputs enriched JSON data about the lead.  
    - Edge Cases: API key errors, network timeouts, no matching records found, partial data returns.  
    - Sticky Note: None

---

#### 2.3 AI Script Generation

- **Overview:**  
  Generates a personalized, short video script based on enriched lead data for use in HeyGen video creation.

- **Nodes Involved:**  
  - Selfie Video Prompt Agent (LangChain Agent)  
  - OpenAI Chat Model1 (GPT-4 model, configured for script generation)

- **Node Details:**

  - **Selfie Video Prompt Agent**  
    - Type: LangChain Agent  
    - Role: Uses a detailed system message and prompt to craft a concise, personalized video script adhering to specific rules for tone, structure, and content.  
    - Configuration:  
      - System Message defines the agent as an expert video script generator for HeyGen, tasked with producing a short first-touch outreach script for a real estate agent.  
      - Includes dynamic variables (lead name, location, property details, financials, language, occupation, children) from the Scout API output.  
      - Script must be under 15 seconds (~55–70 words), personalized, and include three discussion paths for social proof.  
      - Output is only the final script text in the lead’s language or English by default.  
    - Input/Output: Receives enriched lead data from HTTP Request node output, outputs script text for video generation.  
    - Edge Cases: Missing dynamic data fields, language variables not provided, AI prompt or API errors, rate limits.  
    - Sticky Note: "Script Agent"

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model (GPT-4.1-mini)  
    - Role: Provides advanced language model support for the Selfie Video Prompt Agent node.  
    - Configuration: Model set to GPT-4.1-mini, connected as AI language model for the agent.  
    - Input/Output: Receives prompt from Selfie Video Prompt Agent, outputs generated script.  
    - Edge Cases: API key limits, network issues.  
    - Credentials: OpenAI API key configured.  
    - Sticky Note: None

---

#### 2.4 Video Generation & Monitoring

- **Overview:**  
  Creates a personalized video using HeyGen API with the generated script and continuously checks the video status until completion.

- **Nodes Involved:**  
  - Generate Video (HeyGen API POST)  
  - Wait  
  - Get Video (HeyGen API GET)  
  - If1 (Status check)  
  - Wait1 (Conditional wait loop)

- **Node Details:**

  - **Generate Video**  
    - Type: HTTP Request  
    - Role: Sends a POST request to HeyGen's video generation endpoint with the script text as input.  
    - Configuration:  
      - JSON body includes video input specifying avatar type, avatar ID ("Gala_standing_businesssofa_side_close"), avatar style, voice type and ID, and the script text (`input_text` from the script output).  
      - Video dimensions set to 1280x720.  
      - Authenticated via HeyGen HTTP Header Authentication using an API key credential.  
    - Input/Output: Input from Selfie Video Prompt Agent node (script text), output includes video generation ID for status tracking.  
    - Edge Cases: API errors, invalid avatar or voice ID, rate limits, JSON parsing errors.  
    - Sticky Note: "Post & Wait"

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow execution to allow video generation processing time before status check.  
    - Configuration: Default wait (no explicit duration specified; likely immediate pass-through or minimal wait).  
    - Input/Output: Input from Generate Video node; outputs to Get Video node.  
    - Edge Cases: Insufficient wait causing status check before completion.  
    - Sticky Note: "Post & Wait"

  - **Get Video**  
    - Type: HTTP Request  
    - Role: Queries HeyGen API for current video generation status and fetches video URL and thumbnail.  
    - Configuration:  
      - GET request to HeyGen's video_status.get endpoint with the `video_id` query parameter from Generate Video output.  
      - Authenticated with HeyGen API key.  
      - Headers accept JSON response.  
    - Input/Output: Takes video_id from Generate Video output; outputs video status and URLs.  
    - Edge Cases: API timeouts, invalid video_id, transient errors.  
    - Sticky Note: "Get Video"

  - **If1 (Status check)**  
    - Type: If  
    - Role: Checks if the video generation status is "completed".  
    - Configuration: Condition checks if `data.status` equals "completed".  
    - Input/Output:  
      - If true, passes to OpenAI node for email generation.  
      - If false, passes to Wait1 node to pause before retry.  
    - Edge Cases: Unexpected status values, missing status field.  
    - Sticky Note: None

  - **Wait1**  
    - Type: Wait  
    - Role: Waits 20 seconds before re-checking video status to avoid excessive polling.  
    - Configuration: Wait duration set to 20 seconds.  
    - Input/Output: On completion, loops back to Get Video node for status recheck.  
    - Edge Cases: Long video generation times delaying workflow completion.  
    - Sticky Note: None

---

#### 2.5 Outreach Email Composition

- **Overview:**  
  Generates a personalized outreach email in HTML format embedding the video thumbnail and link, and invites the lead to book a meeting.

- **Nodes Involved:**  
  - OpenAI

- **Node Details:**

  - **OpenAI**  
    - Type: LangChain OpenAI node  
    - Role: Generates a professional email body embedding the video thumbnail linked to the video URL with a call-to-action to book a meeting.  
    - Configuration:  
      - Model: GPT-4.1 (full)  
      - Messages:  
        - System message sets the role as an email agent for Sam at Best Real Estate, instructing to embed video thumbnail linking to video URL and include Calendly booking link.  
        - User message provides prospect information (name, email) and video details (thumbnail URL and video link).  
      - Output: Only the HTML body of the email is generated; subject and recipient addressed downstream.  
    - Input/Output:  
      - Input from If1 node on "completed" path with video and lead info.  
      - Output to the Gmail node.  
    - Edge Cases: Missing video URLs, malformed HTML, API errors.  
    - Credentials: OpenAI API key configured.  
    - Sticky Note: "Send Outreach Email with Video"

---

#### 2.6 Email Delivery

- **Overview:**  
  Sends the personalized outreach email with embedded video to the lead’s email address.

- **Nodes Involved:**  
  - Send Email & Video (Gmail node)

- **Node Details:**

  - **Send Email & Video**  
    - Type: Gmail Node  
    - Role: Sends the generated email body to the lead’s email address with subject "Hey from Scout".  
    - Configuration:  
      - Recipient email dynamically set from the original form submission field "Email Address".  
      - Message body set to the HTML content generated by the OpenAI node.  
      - No additional options configured.  
    - Input/Output: Input from OpenAI email generator node; outputs email send status.  
    - Edge Cases: OAuth token expiration, Gmail quota limits, invalid email addresses.  
    - Credentials: Gmail OAuth2 credentials configured.  
    - Sticky Note: "Send Outreach Email with Video"

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                          | Input Node(s)           | Output Node(s)         | Sticky Note                               |
|-------------------------|--------------------------------|----------------------------------------|------------------------|------------------------|-------------------------------------------|
| On form submission      | Form Trigger                   | Captures inbound lead data from form   | —                      | HTTP Request           | Form Submission Trigger                    |
| HTTP Request            | HTTP Request                   | Scout API lookup for enriched lead data| On form submission      | Selfie Video Prompt Agent|                                           |
| Selfie Video Prompt Agent| LangChain Agent                | Generates personalized video script    | HTTP Request           | Generate Video         | Script Agent                              |
| OpenAI Chat Model1      | LangChain OpenAI Chat Model   | Supports script generation agent       | Selfie Video Prompt Agent (AI model) | Selfie Video Prompt Agent |                                           |
| Generate Video          | HTTP Request                   | Sends script to HeyGen to create video | Selfie Video Prompt Agent | Wait                   | Post & Wait                              |
| Wait                    | Wait                          | Pause before checking video status      | Generate Video          | Get Video              | Post & Wait                              |
| Get Video               | HTTP Request                   | Retrieves video status and URLs         | Wait, Wait1             | If1                    | Get Video                                |
| If1                     | If                            | Checks if video generation is complete  | Get Video               | OpenAI, Wait1          |                                           |
| Wait1                   | Wait                          | Waits 20s before rechecking video status| If1 (false branch)      | Get Video              |                                           |
| OpenAI                  | LangChain OpenAI               | Generates outreach email with embedded video | If1 (true branch)    | Send Email & Video     | Send Outreach Email with Video            |
| Send Email & Video      | Gmail                         | Sends personalized email with video    | OpenAI                  | —                      | Send Outreach Email with Video            |
| Sticky Note             | Sticky Note                   | Informational notes                     | —                      | —                      | Form Submission Trigger                    |
| Sticky Note1            | Sticky Note                   | Informational notes                     | —                      | —                      | Script Agent                              |
| Sticky Note2            | Sticky Note                   | Informational notes                     | —                      | —                      | Post & Wait                              |
| Sticky Note3            | Sticky Note                   | Informational notes                     | —                      | —                      | Get Video                                |
| Sticky Note4            | Sticky Note                   | Informational notes                     | —                      | —                      | Send Outreach Email with Video            |
| Sticky Note5            | Sticky Note                   | Detailed workflow description and setup| —                      | —                      | See General Notes below                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure the form titled "Inbound Form" with fields: first_name, middle_name (email type), last_name, address, city, state, county, zip_code, exactmatch (dropdown: true/false), minscore (number), maxscore (number), count (number), level (dropdown: address, city, state, county, zip), phone, email, count (number).  
   - This node starts the workflow upon form submission.

2. **Add HTTP Request Node to Query Scout API**  
   - Type: HTTP Request  
   - Name: "HTTP Request"  
   - Method: POST  
   - URL: `https://api.trustscout.com/enterprise/people/lookup/`  
   - Headers: Add header "X-Api-Key" with your Scout API key.  
   - Body (JSON): Map form fields dynamically into JSON properties: first_name, middle_name, last_name, address, city, state, county, zip_code, exactmatch (false), minscore (0.0), maxscore (100.0), count (1), level, phone, email.  
   - Connect "On form submission" output to this node input.

3. **Set up LangChain Agent Node for Video Script Generation**  
   - Type: LangChain Agent  
   - Name: "Selfie Video Prompt Agent"  
   - Configure prompt text using enriched lead data from Scout API output, including variables such as matched_first_name, city, state, homeownercd, prop_type, year_built, hhi, wealth, language, occupation, children.  
   - System Message: Define agent instructions to generate a personalized outreach script under 15 seconds, including three client discussion paths and a call-to-action.  
   - Connect "HTTP Request" node output to this node input.  
   - Link this agent node to an OpenAI Chat Model node (GPT-4.1-mini) as its AI language model.

4. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Name: "OpenAI Chat Model1"  
   - Model: GPT-4.1-mini  
   - Set this node as the AI model for "Selfie Video Prompt Agent".

5. **Add HTTP Request Node to Generate Video via HeyGen API**  
   - Type: HTTP Request  
   - Name: "Generate Video"  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Authentication: HTTP Header Auth with HeyGen API key credential.  
   - Body (JSON): Include video_inputs array with character info (avatar ID: "Gala_standing_businesssofa_side_close", avatar style: normal), voice info (voice_id: "35b75145af9041b298c720f23375f578"), and `input_text` set to the script output from the agent node. Set video dimension 1280x720.  
   - Connect "Selfie Video Prompt Agent" output to this node input.

6. **Add Wait Node**  
   - Type: Wait  
   - Name: "Wait"  
   - No explicit duration needed (default or minimal wait).  
   - Connect "Generate Video" output to this node.

7. **Add HTTP Request Node to Get Video Status**  
   - Type: HTTP Request  
   - Name: "Get Video"  
   - Method: GET  
   - URL: `https://api.heygen.com/v1/video_status.get`  
   - Authentication: HTTP Header Auth with HeyGen API key credential.  
   - Query Parameter: video_id set dynamically from "Generate Video" output.  
   - Connect "Wait" output to this node.

8. **Add If Node to Check Video Status**  
   - Type: If  
   - Name: "If1"  
   - Condition: Check if `data.status` equals "completed" (strict string equality).  
   - Connect "Get Video" output to this node.

9. **Add Wait Node for Rechecking Video Status**  
   - Type: Wait  
   - Name: "Wait1"  
   - Duration: 20 seconds  
   - Connect "If1" false output (status not completed) to this node.

10. **Loop Wait1 Back to Get Video**  
    - Connect "Wait1" output back to "Get Video" node input for polling until completion.

11. **Add OpenAI Node to Generate Outreach Email**  
    - Type: LangChain OpenAI  
    - Name: "OpenAI"  
    - Model: GPT-4.1 (full)  
    - Messages:  
      - System message instructs to craft a professional HTML email embedding the thumbnail linked to the video URL and including a Calendly booking link (https://calendly.com/best-real-estate).  
      - User message provides prospect info (name, email) and video data (thumbnail URL, video URL) from previous nodes.  
    - Connect "If1" true output (video status completed) to this node.

12. **Add Gmail Node to Send Email**  
    - Type: Gmail  
    - Name: "Send Email & Video"  
    - Recipient email set dynamically from original form submission email field.  
    - Email subject: "Hey from Scout"  
    - Message body: The HTML output from the OpenAI email generation node.  
    - Credentials: Configure Gmail OAuth2 credentials.  
    - Connect "OpenAI" node output to this node.

13. **Add Sticky Note Nodes for Documentation** (Optional)  
    - Add sticky notes near groups of nodes to document purpose, e.g., "Form Submission Trigger", "Script Agent", "Post & Wait", "Get Video", "Send Outreach Email with Video".  
    - Add a large sticky note summarizing the workflow description and setup instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| 🎥 AI-Powered Inbound Video Agent – Auto-Respond to Leads with Personalized Videos using Scout, HeyGen API, OpenAI GPT-4, Gmail | Detailed workflow description and setup guidance (see Sticky Note5 for full content)              |
| Setup Instructions: Obtain HeyGen API Key and configure HTTP Header Auth credential. Set your Scout API key in HTTP Request.   | https://app.trustscout.com, https://heygen.com                                                      |
| Connect OpenAI API key and Gmail OAuth2 credentials for email sending.                                                         | https://platform.openai.com, Gmail OAuth2 setup instructions                                      |
| Customize system messages in LangChain Agent and email generation nodes to reflect brand voice and call-to-action links.      | Modify prompts in "Selfie Video Prompt Agent" and "OpenAI" nodes                                   |
| Calendly link placeholder is https://calendly.com/best-real-estate — update to your real booking URL in prompt system messages.|                                                                                                   |
| Use Cases: Automate welcome sequences, personalized video lead responses, agency outreach campaigns.                          |                                                                                                |

---

This comprehensive documentation enables both human developers and AI systems to understand, reproduce, troubleshoot, and customize the workflow fully without access to the original JSON export.