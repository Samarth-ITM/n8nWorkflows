Transforming Scripts into Engaging Instagram Reels with Blotato: Featuring Unique Human Review

https://n8nworkflows.xyz/workflows/transforming-scripts-into-engaging-instagram-reels-with-blotato--featuring-unique-human-review-9858


# Transforming Scripts into Engaging Instagram Reels with Blotato: Featuring Unique Human Review

### 1. Workflow Overview

This workflow automates the transformation of a user-submitted video script into an engaging Instagram Reel using Blotato's AI-powered video creation platform, with a unique human approval step via Gmail before posting. It targets content creators and social media managers aiming to streamline video content production from textual scripts, ensuring quality control through manual review.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Script Processing:** Collects the script and caption text from a form, then breaks down the script into structured parts (narration and visual description) using AI.
- **1.2 Video Creation via Blotato:** Aggregates script parts and sends them to Blotato to generate a realistic-style AI selfie video with a consistent character.
- **1.3 Video Generation Monitoring:** Periodically checks if the video has been generated successfully.
- **1.4 Human Approval via Gmail:** Sends the generated video to an approver’s Gmail inbox and waits for an approval response.
- **1.5 Posting to Instagram:** Upon approval, posts the video to Instagram via Blotato with the provided caption.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Script Processing

**Overview:**  
This block receives user input through a web form and uses AI to break down the raw video script into manageable narrative segments with corresponding visual scene descriptions.

**Nodes Involved:**  
- On form submission2  
- Information Extractor2  
- Aggregate2

**Node Details:**

- **On form submission2**  
  - *Type:* Form Trigger  
  - *Role:* Receives script and caption text from a user web form titled "Script" with two fields: "Script" (textarea) and "Caption Text".  
  - *Configuration:* Webhook ID set to receive submissions; appends no attribution; no default values.  
  - *Inputs:* External web form submissions.  
  - *Outputs:* JSON containing user script and caption text.  
  - *Potential Failures:* Webhook downtime, invalid form data submission.

- **Information Extractor2**  
  - *Type:* Langchain Information Extractor node  
  - *Role:* Uses AI to parse and break down the input script into an array of objects, each with "narration" and "description" fields.  
  - *Configuration:* The prompt explicitly instructs to break down the script into parts, outputting an array with a schema requiring narration and description strings. The script input is dynamically injected from the form submission.  
  - *Key Expressions:* `{{ $json.Script }}` for script text injection.  
  - *Inputs:* Script text from "On form submission2".  
  - *Outputs:* JSON array of structured script parts.  
  - *Potential Failures:* AI model downtime, malformed input script, schema mismatch.

- **Aggregate2**  
  - *Type:* Aggregate node  
  - *Role:* Aggregates the array output from the Information Extractor to a single data structure to prepare for video creation.  
  - *Configuration:* Aggregates the "output" field from incoming data.  
  - *Inputs:* Output array from Information Extractor2.  
  - *Outputs:* Aggregated JSON array for video generation.  
  - *Potential Failures:* Empty or malformed input array.

---

#### 1.2 Video Creation via Blotato

**Overview:**  
Sends the structured script parts as scene descriptions to Blotato’s video generation API to create an AI selfie video with a consistent character in a realistic style.

**Nodes Involved:**  
- Create video3  
- Get video2

**Node Details:**

- **Create video3**  
  - *Type:* Blotato node (video resource)  
  - *Role:* Initiates video creation using Blotato with a predefined AI Selfie Talking Video template.  
  - *Configuration:*  
    - Template ID fixed to "AI Selfie Talking Video with Consistent Character".  
    - Template inputs include:  
      - `scenes`: JSON stringified output from Aggregate2.  
      - `style`: "realistic".  
      - `aspectRatio`: "9:16" (vertical video for Instagram Reels).  
      - `characterDescription`: "A man".  
  - *Credentials:* Blotato API key required.  
  - *Inputs:* Aggregated script scene data.  
  - *Outputs:* Video creation job response including video ID.  
  - *Potential Failures:* API authentication errors, invalid scene data, template mismatch, quota limits.

- **Get video2**  
  - *Type:* Blotato node (video resource)  
  - *Role:* Retrieves the generated video status and metadata by video ID.  
  - *Configuration:* Uses video ID from "Create video3" or from the wait loop.  
  - *Credentials:* Blotato API key required.  
  - *Inputs:* Video ID.  
  - *Outputs:* Video metadata including mediaUrl if ready.  
  - *Potential Failures:* Video not ready, API call failure, network timeouts.

---

#### 1.3 Video Generation Monitoring

**Overview:**  
Periodically checks if the video has been successfully generated by querying the video status every 40 seconds.

**Nodes Involved:**  
- If2  
- Wait2

**Node Details:**

- **If2**  
  - *Type:* If node  
  - *Role:* Checks if the video metadata contains a non-empty mediaUrl indicating the video is ready.  
  - *Configuration:* Condition tests if `item.mediaUrl` is not empty.  
  - *Inputs:* Output from "Get video2".  
  - *Outputs:*  
    - True branch: proceeds to send email for approval.  
    - False branch: goes to wait timer.  
  - *Potential Failures:* Incorrect field references, empty responses.

- **Wait2**  
  - *Type:* Wait node  
  - *Role:* Delays the workflow for 40 seconds before re-checking video status.  
  - *Configuration:* Fixed 40 second delay.  
  - *Inputs:* False branch from If2.  
  - *Outputs:* Triggers "Get video2" to retry.  
  - *Potential Failures:* Workflow timeout if video generation is excessively delayed.

---

#### 1.4 Human Approval via Gmail

**Overview:**  
Sends the generated video via email to a human reviewer and waits for a double confirmation approval before proceeding.

**Nodes Involved:**  
- Send message and wait for response1  
- If3

**Node Details:**

- **Send message and wait for response1**  
  - *Type:* Gmail node (send and wait)  
  - *Role:* Sends an HTML email containing an embedded video player for the generated video and waits for approval response.  
  - *Configuration:*  
    - Recipient: configured email (placeholder "your_email").  
    - Subject: "Video Approval Email".  
    - Email body: HTML table embedding the video with a video tag and fallback link.  
    - Approval type: double approval required (implying two-step human confirmation).  
  - *Credentials:* Gmail OAuth2 credentials required.  
  - *Inputs:* Video mediaUrl from If2’s true branch.  
  - *Outputs:* Approval response data.  
  - *Potential Failures:* Email delivery failures, OAuth token expiry, no response timeout.

- **If3**  
  - *Type:* If node  
  - *Role:* Checks if the approval response data has `approved` set to true.  
  - *Configuration:* Condition tests if `data.approved` is true.  
  - *Inputs:* Approval response from Gmail node.  
  - *Outputs:*  
    - True branch: proceeds to post video.  
    - False branch: ends or halts posting.  
  - *Potential Failures:* Incorrect approval data parsing, false negatives.

---

#### 1.5 Posting to Instagram

**Overview:**  
Posts the approved video to Instagram through Blotato, attaching the user-provided caption.

**Nodes Involved:**  
- Create post2

**Node Details:**

- **Create post2**  
  - *Type:* Blotato node (post resource)  
  - *Role:* Creates a social media post on Instagram with video and caption text.  
  - *Configuration:*  
    - Account ID statically set to "17066" (represents the Instagram account).  
    - Post content text dynamically taken from form field "Caption Text".  
    - Media URL dynamically taken from the approved video mediaUrl.  
  - *Credentials:* Blotato API key required.  
  - *Inputs:* Approval passed from If3 node.  
  - *Outputs:* Post creation confirmation.  
  - *Potential Failures:* API authentication failure, invalid media URL, posting quota exceeded.

---

### 3. Summary Table

| Node Name                     | Node Type                             | Functional Role                            | Input Node(s)            | Output Node(s)                      | Sticky Note                                                                                                   |
|-------------------------------|-------------------------------------|--------------------------------------------|--------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------|
| On form submission2            | Form Trigger                        | Receives script and caption from user      | External form submission  | Information Extractor2             |                                                                                                              |
| Information Extractor2         | Langchain Information Extractor     | Parses script into narration and description | On form submission2       | Aggregate2                       |                                                                                                              |
| Aggregate2                    | Aggregate                          | Aggregates script parts into single array  | Information Extractor2    | Create video3                    |                                                                                                              |
| Create video3                 | Blotato (video resource)           | Initiates AI video generation on Blotato   | Aggregate2                | Get video2                      |                                                                                                              |
| Get video2                   | Blotato (video resource)           | Retrieves video status and metadata         | Create video3 / Wait2     | If2                             |                                                                                                              |
| If2                          | If                                | Checks if video mediaUrl is ready           | Get video2                | Send message and wait for response1 / Wait2 |                                                                                                              |
| Wait2                        | Wait                              | Waits 40 seconds before re-checking video   | If2 (false branch)        | Get video2                      |                                                                                                              |
| Send message and wait for response1 | Gmail                          | Sends video for human approval and waits   | If2 (true branch)         | If3                            |                                                                                                              |
| If3                          | If                                | Checks if human approved video               | Send message and wait for response1 | Create post2 / End       |                                                                                                              |
| Create post2                 | Blotato (post resource)            | Posts approved video to Instagram            | If3 (true branch)         |                                   |                                                                                                              |
| Sticky Note                  | Sticky Note                       | Link collection and documentation hints     | N/A                      | N/A                             | # [Click here and Sign up for Blotato](https://blotato.com/?ref=karne)                                       |
| Sticky Note1                 | Sticky Note                       | Video script collection and formatting using AI | N/A                      | N/A                             | ## Video script collection and formatting using AI                                                          |
| Sticky Note2                 | Sticky Note                       | Create Video using Blotato                   | N/A                      | N/A                             | ## Create Video using Blotato                                                                                 |
| Sticky Note3                 | Sticky Note                       | Check if Video is generated every 40 seconds | N/A                      | N/A                             | ## Check if Video is generated every 40 seconds                                                              |
| Sticky Note4                 | Sticky Note                       | Human Approval using Gmail                    | N/A                      | N/A                             | ## Human Approval using Gmail                                                                                  |
| Sticky Note5                 | Sticky Note                       | Post Video to Instagram if approved           | N/A                      | N/A                             | ## Post Video to Instagram if approved                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Name: "On form submission2"  
   - Type: Form Trigger (n8n-nodes-base.formTrigger)  
   - Webhook ID: generate or reuse existing  
   - Configure form with title "Script" and two fields:  
     - "Script" (textarea)  
     - "Caption Text" (text)  
   - Disable append attribution.

2. **Add Langchain Information Extractor node:**  
   - Name: "Information Extractor2"  
   - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
   - Prompt: "Break down the script and write it in parts." plus the user script inserted dynamically:  
     ```
     Script:
     ''
     {{ $json.Script }}
     ''
     ```  
   - Set schema type to manual with item schema requiring "narration" and "description" strings.  
   - Connect output of "On form submission2" to input.

3. **Add Aggregate node:**  
   - Name: "Aggregate2"  
   - Type: n8n-nodes-base.aggregate  
   - Configure to aggregate the field "output" from "Information Extractor2".  
   - Connect output of "Information Extractor2" to input.

4. **Add Blotato node to create video:**  
   - Name: "Create video3"  
   - Type: `@blotato/n8n-nodes-blotato.blotato`  
   - Configure resource as "video" and select template "AI Selfie Talking Video with Consistent Character" (template id `/base/v2/ai-selfie-video/57f5a565-fd17-458b-be43-4a2d8ccaca75/v1`).  
   - Set template inputs:  
     - `scenes`: set to JSON stringified data from "Aggregate2" output.  
     - `style`: "realistic"  
     - `aspectRatio`: "9:16"  
     - `characterDescription`: "A man"  
   - Set Blotato API credentials.  
   - Connect output of "Aggregate2" to input.

5. **Add Blotato node to get video status:**  
   - Name: "Get video2"  
   - Type: Blotato video resource  
   - Configure operation: "get" by videoId taken from previous "Create video3" output `.item.id`.  
   - Use same Blotato credentials.  
   - Connect output of "Create video3" to input.

6. **Add If node to check video readiness:**  
   - Name: "If2"  
   - Condition: Check if `item.mediaUrl` is not empty or null (string not empty).  
   - Connect output of "Get video2" to input.

7. **Add Wait node:**  
   - Name: "Wait2"  
   - Set wait duration to 40 seconds.  
   - Connect false branch of "If2" to this node.

8. **Loop back:**  
   - Connect output of "Wait2" to "Get video2" to recheck video status until ready.

9. **Add Gmail node to send video for approval and wait:**  
   - Name: "Send message and wait for response1"  
   - Operation: sendAndWait  
   - Recipient: set to approver’s email address.  
   - Subject: "Video Approval Email"  
   - Message body: HTML with embedded `<video>` tag showing video from `{{ $json.item.mediaUrl }}` and fallback link.  
   - Approval type: double approval required.  
   - Use Gmail OAuth2 credentials.  
   - Connect true branch of "If2" to this node.

10. **Add If node to check approval:**  
    - Name: "If3"  
    - Condition: Check if `data.approved` is true (boolean).  
    - Connect output of Gmail node to input.

11. **Add Blotato node to create Instagram post:**  
    - Name: "Create post2"  
    - Resource: post  
    - Parameters:  
      - Account ID: "17066" (replace with actual Instagram account ID)  
      - Post content text: from form field "Caption Text" (`$('On form submission2').item.json['Caption Text']`)  
      - Post media URLs: from approved video mediaUrl (`$('If2').item.json.item.mediaUrl`)  
    - Use Blotato API credentials.  
    - Connect true branch of "If3" to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                          |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------|
| Click here and Sign up for Blotato                                                                       | https://blotato.com/?ref=karne          |
| Google Cloud console                                                                                       | https://console.cloud.google.com/       |
| YouTube tutorial on Blotato AI Selfie Video creation                                                     | https://youtu.be/4UL1GrW09O0             |
| Video script collection and formatting using AI                                                          | Sticky Note1 content in the workflow    |
| Create Video using Blotato                                                                                 | Sticky Note2 content                     |
| Check if Video is generated every 40 seconds                                                             | Sticky Note3 content                     |
| Human Approval using Gmail                                                                                 | Sticky Note4 content                     |
| Post Video to Instagram if approved                                                                        | Sticky Note5 content                     |

---

This documentation enables thorough understanding, reproduction, and modification of the workflow, highlighting key nodes, configurations, and potential failure points for robust operation.