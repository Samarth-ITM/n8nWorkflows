Respond to Inbound Leads with Personalized Videos using Heygen AI and GPT-4

https://n8nworkflows.xyz/workflows/respond-to-inbound-leads-with-personalized-videos-using-heygen-ai-and-gpt-4-6404


# Respond to Inbound Leads with Personalized Videos using Heygen AI and GPT-4

### 1. Workflow Overview

This workflow automates personalized video and email responses to inbound leads captured via a web form. It is designed primarily for marketing agencies, sales teams, and founders who want to increase engagement and conversion by sending hyper-personalized video messages followed by contextual emails automatically.

**Logical Blocks:**

- **1.1 Input Reception:** Captures lead data from a web form submission.
- **1.2 AI Video Script Generation:** Uses an AI prompt agent (LangChain + GPT-4) to create a personalized video script based on lead inputs.
- **1.3 Video Generation & Monitoring:** Sends the script to Heygen API to generate a branded avatar video, waits and polls for completion.
- **1.4 Email Content Generation:** Uses GPT-4 to compose a personalized outreach email embedding the generated video thumbnail and link.
- **1.5 Email Delivery:** Sends the composed email to the lead using Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Initiates the workflow by capturing inbound lead details via a web form.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - Type: Form Trigger node  
    - Configuration:  
      - Listens to submissions on a form titled "Inbound Form" with required fields: Name, Email Address, optional Phone Number, Business URL, and a dropdown for marketing pain points.  
      - Form description guides users to fill it for personalized marketing services.  
    - Inputs: External web form submission  
    - Outputs: JSON object with all submitted fields  
    - Failure cases: Missing required fields, malformed data, or webhook connectivity issues.

#### 2.2 AI Video Script Generation

- **Overview:**  
  Creates a short, personalized video script tailored to the lead’s inputs, instructing the avatar what to say.

- **Nodes Involved:**  
  - Selfie Video Prompt Agent (LangChain Agent)  
  - OpenAI Chat Model1 (GPT-4 based model used by the Agent)

- **Node Details:**  
  - **Selfie Video Prompt Agent**  
    - Type: LangChain Agent (text prompt agent)  
    - Configuration:  
      - Constructs a prompt incorporating lead’s name, email, phone number, business URL, and biggest marketing pain point.  
      - System message defines the agent’s role: To create a 10-second video prompt starting with “Hey this is Jason from Purple Unicorn Marketing...” aiming to get leads to book a Calendly call.  
      - Uses GPT-4 mini model for language understanding and generation.  
    - Inputs: JSON data from form submission  
    - Outputs: Generated text script for video input  
    - Failure modes: Expression parsing errors, API timeout, or incomplete prompt data.

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Backend LLM used by Selfie Video Prompt Agent  
    - Configuration: Model set to "gpt-4.1-mini" for efficient prompt generation.  
    - Inputs: Forwarded from the Agent node  
    - Outputs: Text prompt used by Heygen API  
    - Failure cases: API authentication, rate limits, or network errors.

#### 2.3 Video Generation & Monitoring

- **Overview:**  
  Sends the personalized script to Heygen API to generate a branded avatar video, waits for the video to be processed, then polls for the video’s completion status.

- **Nodes Involved:**  
  - HeyGen Post (HTTP Request)  
  - Wait (Pause node)  
  - Get Video (HTTP Request)  
  - If1 (Conditional check)  
  - Wait1 (Additional waiting if video not ready)

- **Node Details:**  
  - **HeyGen Post**  
    - Type: HTTP Request node  
    - Configuration:  
      - POSTs to Heygen API endpoint `/v2/video/generate`  
      - Sends JSON body defining video dimension (1280x720), avatar ID, voice ID, speech speed, and the AI-generated input text script.  
      - Uses HTTP header authentication with Heygen API key stored in generic credentials.  
      - Sets Accept header to application/json.  
    - Inputs: Generated video script text from Agent  
    - Outputs: Video generation request response including video ID  
    - Failure modes: API key invalid, malformed request, or API downtime.

  - **Wait**  
    - Type: Wait node  
    - Configuration: Defaults to pause workflow for a short duration (not specified, but typically seconds to minutes) to allow video processing.  
    - Inputs: Response from HeyGen Post  
    - Outputs: Passes to Get Video node

  - **Get Video**  
    - Type: HTTP Request node  
    - Configuration:  
      - GET request to Heygen’s `/v1/video_status.get` endpoint with query parameter video_id from previous response.  
      - Authenticated via HTTP header with Heygen API key.  
      - Accepts JSON response containing video status, thumbnail URL, and video URL.  
    - Inputs: Video ID from HeyGen Post, triggered after Wait  
    - Outputs: Video status and metadata  
    - Failure modes: Invalid video ID, network errors, or API limits.

  - **If1**  
    - Type: If conditional node  
    - Configuration: Checks if `data.status` from Get Video response equals "completed".  
    - Inputs: Output from Get Video  
    - Outputs:  
      - If true: proceeds to OpenAI email generation  
      - If false: loops back to Wait1 to pause again before next status check.  
    - Failure modes: Missing or unexpected status values.

  - **Wait1**  
    - Type: Secondary Wait node  
    - Configuration: Similar to Wait, used for repeated polling delays.  
    - Inputs: From If1 false branch  
    - Outputs: Back to Get Video for next status check.

#### 2.4 Email Content Generation

- **Overview:**  
  Uses GPT-4 to craft a personalized HTML email embedding the video thumbnail linked to the video URL, encouraging the lead to book a Calendly call.

- **Nodes Involved:**  
  - OpenAI (LangChain OpenAI node)

- **Node Details:**  
  - **OpenAI**  
    - Type: LangChain OpenAI node  
    - Configuration:  
      - Model: GPT-4.1 (full version)  
      - Messages include:  
        - User message with prospect info (name, email, video link, thumbnail URL)  
        - System message positioning the node as an email agent for Paradise Homes, emphasizing embedding the video thumbnail with link and a CTA to book on Calendly (calendly.com/automatewithmarc).  
      - Output is the HTML body of the email (only the body, excluding headers) intended for direct input to Gmail node.  
    - Inputs: Video metadata from Get Video and form data from submission node  
    - Outputs: Structured HTML email content  
    - Failure modes: API limits, malformed responses, or missing video data.

#### 2.5 Email Delivery

- **Overview:**  
  Sends the personalized email with embedded video thumbnail to the lead’s email address.

- **Nodes Involved:**  
  - Send Email & Video (Gmail node)

- **Node Details:**  
  - **Send Email & Video**  
    - Type: Gmail node  
    - Configuration:  
      - Sends email to address captured in form submission  
      - Uses HTML content from OpenAI node as email body  
      - Subject line set to "Hey from Paradise Homes"  
      - Uses OAuth2 credentials for Gmail account authentication  
    - Inputs: Email address from form and HTML message content from OpenAI node  
    - Outputs: Email delivery status  
    - Failure modes: OAuth token expiration, quota limits, invalid email addresses, or SMTP errors.

---

### 3. Summary Table

| Node Name               | Node Type                     | Functional Role                                | Input Node(s)               | Output Node(s)           | Sticky Note                                                                                      |
|-------------------------|-------------------------------|------------------------------------------------|-----------------------------|---------------------------|------------------------------------------------------------------------------------------------|
| On form submission       | Form Trigger                  | Captures inbound lead data through form         | —                           | Selfie Video Prompt Agent  | Form Submission Trigger                                                                        |
| Selfie Video Prompt Agent| LangChain Agent               | Generates personalized video script prompt      | On form submission           | HeyGen Post                | Script Agent                                                                                   |
| OpenAI Chat Model1       | LangChain OpenAI Chat Model  | Backend LLM for video prompt generation          | Selfie Video Prompt Agent    | Selfie Video Prompt Agent  | Script Agent                                                                                   |
| HeyGen Post              | HTTP Request                 | Sends prompt to Heygen API to generate video    | Selfie Video Prompt Agent    | Wait                      | Post & Wait                                                                                   |
| Wait                    | Wait Node                    | Pauses for video processing                      | HeyGen Post                 | Get Video                 | Post & Wait                                                                                   |
| Get Video               | HTTP Request                 | Polls Heygen API for video status and URLs      | Wait, Wait1                 | If1                       | Get Video                                                                                     |
| If1                     | If Conditional               | Checks if video generation is complete          | Get Video                   | OpenAI / Wait1            | Get Video                                                                                     |
| Wait1                   | Wait Node                    | Additional wait for video processing             | If1 (false branch)           | Get Video                 | Get Video                                                                                     |
| OpenAI                  | LangChain OpenAI Node        | Generates personalized HTML email content       | If1 (true branch)            | Send Email & Video        | Send Outreach Email with Video                                                                |
| Send Email & Video      | Gmail Node                  | Sends the personalized email with video          | OpenAI                      | —                         | Send Outreach Email with Video                                                                |
| Sticky Note (multiple)  | Sticky Note                  | Documentation and grouping of workflow sections | —                           | —                         | Detailed notes on each functional block (Form Submission Trigger, Script Agent, Post & Wait, Get Video, Send Outreach Email with Video) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure form with title "Inbound Form"  
   - Add fields:  
     - Name (text, required)  
     - Email Address (email, required)  
     - Phone Number (number, optional)  
     - Business URL (text, optional)  
     - Dropdown "What's your biggest pain point (marketing wise)?" with options: Performance Marketing, Content Marketing, Product Marketing, I don't know  
   - Description: "Fill Up This Form to Get Personalized Marketing Services"  

2. **Add "Selfie Video Prompt Agent" node**  
   - Type: LangChain Agent  
   - Connect input from "On form submission"  
   - Set prompt text with expressions to insert form fields:  
     ```
     Name: {{ $('On form submission').item.json.Name }}
     Email Address: {{ $json['Email Address'] }}
     Phone Number: {{ $json['Phone Number'] }}
     Business URL: {{ $json['Business URL'] }}
     Biggest Pain Points: {{ $json['What\'s your biggest pain point (marketing wise)?'] }}
     ```  
   - Configure System Message:  
     ```
     You are an effective video prompt agent for Purple Unicorn Marketing Agency...
     Start the video with "Hey this is Jason from Purple Unicorn Marketing...".
     Aim for a short 10 second video to get them to book a Calendly call.
     ```  
   - Model: Use GPT-4 mini model (e.g., "gpt-4.1-mini")  

3. **Add "HeyGen Post" HTTP Request node**  
   - Connect input from "Selfie Video Prompt Agent"  
   - Method: POST  
   - URL: `https://api.heygen.com/v2/video/generate`  
   - Body (JSON, raw):  
     ```json
     {
       "caption": false,
       "dimension": { "width": 1280, "height": 720 },
       "video_inputs": [{
         "character": {
           "type": "avatar",
           "avatar_id": "5ecf0af57a104efe98029940d33381de",
           "scale": 1,
           "avatar_style": "normal"
         },
         "voice": {
           "type": "text",
           "voice_id": "63dff8b32aeb43a39620c74345ce0652",
           "input_text": "{{ $json.output }}",
           "speed": 1.2
         }
       }]
     }
     ```  
   - Headers:  
     - Accept: application/json  
   - Authentication: HTTP Header Auth with Heygen API key credential  

4. **Add "Wait" node**  
   - Connect from "HeyGen Post"  
   - Default wait duration (e.g., 1-2 minutes) to allow video processing  

5. **Add "Get Video" HTTP Request node**  
   - Connect from "Wait"  
   - Method: GET  
   - URL: `https://api.heygen.com/v1/video_status.get`  
   - Query parameter: `video_id` set from `{{$json.data.video_id}}` from HeyGen Post response  
   - Headers:  
     - Accept: application/json  
   - Authentication: HTTP Header Auth with Heygen API key  

6. **Add "If1" conditional node**  
   - Connect input from "Get Video"  
   - Condition:  
     - Check if `{{$json.data.status}} == "completed"`  
   - True branch: proceed to OpenAI email generation  
   - False branch: connect to Wait1 node for further polling  

7. **Add "Wait1" node**  
   - Connect from If1 false branch  
   - Same wait duration as the first wait node  

8. **Loop "Wait1" output back to "Get Video"**  
   - To keep polling until video is ready  

9. **Add "OpenAI" LangChain OpenAI node**  
   - Connect from If1 true branch  
   - Model: GPT-4.1 (full)  
   - Messages:  
     - User content: includes prospect info (name, email, video URL, thumbnail URL) with expressions referencing form and Get Video outputs  
     - System message: instructs to craft impactful HTML email embedding the thumbnail linked to the video URL, with CTA to book on Calendly (calendly.com/automatewithmarc)  
   - Output: HTML email body only  

10. **Add "Send Email & Video" Gmail node**  
    - Connect from OpenAI node  
    - Recipient: `{{$('On form submission').item.json['Email Address']}}`  
    - Subject: "Hey from Paradise Homes"  
    - Message: `{{$json.message.content}}` (HTML from OpenAI)  
    - Credentials: Gmail OAuth2 setup  

11. **(Optional) Add Sticky Notes**  
    - To document each logical block for clarity during workflow maintenance  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| 🎥 AI-Powered Inbound Video Agent – Auto-Respond to Leads with Personalized Videos. This workflow instantly responds to inbound leads with hyper-personalized videos and emails using n8n, Heygen API, and OpenAI. Ideal for marketing agencies, sales teams, and freelancers aiming to boost conversions with video touchpoints. Setup requires Heygen API key, OpenAI API key, and Gmail OAuth2 credentials. Customize prompts and Calendly links to suit your brand.                                                                                                                                         | https://www.youtube.com/@Automatewithmarc                                                        |
| The video prompt agent uses LangChain with GPT-4 mini to tailor a concise 10-second script for Heygen’s avatar to say, using lead data from the form submission.                                                                                                                                                                                                                                                                                                                                                                                             | Workflow block 2.2                                                                                |
| The email generation node uses full GPT-4 to compose an HTML email embedding the video thumbnail as a clickable link, encouraging prospects to book a call via Calendly.                                                                                                                                                                                                                                                                                                                                                                                  | Workflow block 2.4                                                                                |
| Heygen avatar is configured with a specific avatar ID and voice ID; modify these IDs in the HeyGen Post node to change avatar appearance or voice style.                                                                                                                                                                                                                                                                                                                                                                                                | Workflow block 2.3                                                                                |
| Gmail node requires OAuth2 credentials; ensure the Gmail account has API access enabled and quota available for sending emails.                                                                                                                                                                                                                                                                                                                                                                                                                        | Workflow block 2.5                                                                                |

---

**Disclaimer:** The provided text is sourced exclusively from an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal, offensive, or protected material. All handled data is legal and public.