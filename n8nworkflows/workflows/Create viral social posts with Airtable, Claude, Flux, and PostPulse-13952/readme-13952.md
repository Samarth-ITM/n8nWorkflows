Create viral social posts with Airtable, Claude, Flux, and PostPulse

https://n8nworkflows.xyz/workflows/create-viral-social-posts-with-airtable--claude--flux--and-postpulse-13952


# Create viral social posts with Airtable, Claude, Flux, and PostPulse

# 1. Workflow Overview

This workflow automates the creation and scheduling of AI-generated social media content from Airtable records. When a new record is created in Airtable, it uses the record’s text fields to generate an image with Flux via DumplingAI, generate a short post with Claude, upload the image to PostPulse, schedule the post for the next day, and finally mark the Airtable record as posted.

Typical use cases include:
- turning content ideas stored in Airtable into ready-to-schedule social posts,
- automating lightweight social publishing pipelines,
- creating a repeatable AI-assisted content production process for X/Twitter.

## 1.1 Input Reception from Airtable
The workflow starts when Airtable detects a newly created record. This record provides the two key inputs:
- `Post Idea` for text generation,
- `Prompt` for image generation.

## 1.2 AI Content Generation
Two AI tasks are performed from the Airtable data:
- image generation through DumplingAI using the FLUX.1-schnell model,
- post generation through Anthropic Claude.

Although the nodes run in sequence in this workflow, conceptually they form the AI generation layer.

## 1.3 Media Retrieval and PostPulse Preparation
After the image is generated, the workflow downloads the generated file and uploads it to PostPulse so that PostPulse returns a usable media path/identifier for scheduling.

## 1.4 Scheduling and Source Record Update
The generated text and uploaded image path are combined into a scheduled social post in PostPulse. Once scheduling succeeds, the Airtable record is updated so the `Posted?` field becomes `Yes`.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception from Airtable

### Overview
This block listens for new records in Airtable and starts the automation whenever a record is created. It is the workflow entry point and supplies the source content used downstream.

### Nodes Involved
- Airtable Trigger

### Node Details

#### Airtable Trigger
- **Type and technical role:** `n8n-nodes-base.airtableTrigger`  
  Trigger node that polls an Airtable table and emits a workflow item when a new record appears.
- **Configuration choices:**
  - Uses a specific Airtable base and table.
  - Trigger field is `Created`.
  - Authentication mode is Airtable token API.
  - Polling is enabled with default poll settings.
- **Key expressions or variables used:**
  - Downstream nodes reference:
    - `{{$json.fields.Prompt}}`
    - `{{$json.fields['Post Idea']}}`
    - record `id`
- **Input and output connections:**
  - No input; this is an entry point.
  - Output goes to **Generate AI Image**.
- **Version-specific requirements:**
  - Type version `1`.
  - Requires an Airtable credential compatible with trigger polling.
- **Edge cases or potential failure types:**
  - Invalid Airtable token or revoked access.
  - Wrong base/table selection.
  - Missing `Created` field or schema changes.
  - Polling delay means records are not picked up instantly.
  - If required fields such as `Prompt` or `Post Idea` are empty, downstream nodes may fail or generate poor output.
- **Sub-workflow reference:** None.

---

## 2.2 AI Content Generation

### Overview
This block transforms Airtable fields into publishable assets. It generates an image from the `Prompt` field using DumplingAI/Flux and creates a short social post from the `Post Idea` field using Claude.

### Nodes Involved
- Generate AI Image
- Create Social Media Post

### Node Details

#### Generate AI Image
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to DumplingAI’s image generation API.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`
  - Request body is JSON.
  - Headers include:
    - `Content-Type: application/json`
    - `Authorization: Bearer YOUR_TOKEN_HERE`
  - Model selected: `FLUX.1-schnell`
  - Aspect ratio set to `16:9`
  - Prompt taken from Airtable.
- **Key expressions or variables used:**
  - `{{ $json.fields.Prompt }}`
- **Input and output connections:**
  - Input from **Airtable Trigger**
  - Output to **Create Social Media Post**
- **Version-specific requirements:**
  - Type version `4.4`
  - HTTP Request node must support JSON body expressions as configured here.
- **Edge cases or potential failure types:**
  - Hardcoded bearer token placeholder must be replaced; otherwise authentication fails.
  - API rate limits or usage quota failures.
  - Empty or malformed prompt.
  - API response may not contain `images[0].url`, breaking the later download step.
  - The configured URL includes a trailing space in the JSON; in practice this should be removed to avoid request issues.
- **Sub-workflow reference:** None.

#### Create Social Media Post
- **Type and technical role:** `@n8n/n8n-nodes-langchain.anthropic`  
  Calls Anthropic Claude to generate short social copy.
- **Configuration choices:**
  - Anthropic model: `claude-haiku-4-5-20251001`
  - Uses a single message instructing the model to create a short social media post from the Airtable idea.
  - The prompt tells Claude to return only the generated post.
- **Key expressions or variables used:**
  - `{{ $('Airtable Trigger').item.json.fields['Post Idea'] }}`
  - Later, another node reads:
    - `{{ $('Create Social Media Post').item.json.content[0].text }}`
- **Input and output connections:**
  - Input from **Generate AI Image**
  - Output to **Download File**
- **Version-specific requirements:**
  - Type version `1`
  - Requires Anthropic credentials configured in n8n.
  - The selected model identifier may depend on node version and Anthropic availability.
- **Edge cases or potential failure types:**
  - Invalid or missing Anthropic credentials.
  - Model not available in the workspace or node version.
  - Empty `Post Idea` resulting in weak or generic output.
  - Response shape may vary by version; the scheduling node assumes output exists at `content[0].text`.
- **Sub-workflow reference:** None.

---

## 2.3 Media Retrieval and PostPulse Preparation

### Overview
This block takes the generated image URL, downloads the actual file, then uploads it to PostPulse. The purpose is to convert a generated remote image into a PostPulse media asset that can be attached to a scheduled post.

### Nodes Involved
- Download File
- Upload media

### Node Details

#### Download File
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Fetches the generated image from the URL returned by DumplingAI.
- **Configuration choices:**
  - Performs an HTTP request to the first image URL in the image-generation response.
  - No custom request options are defined.
  - In practice, this node is expected to download binary data for later upload.
- **Key expressions or variables used:**
  - `{{ $('Generate AI Image').item.json.images[0].url }}`
- **Input and output connections:**
  - Input from **Create Social Media Post**
  - Output to **Upload media**
- **Version-specific requirements:**
  - Type version `4.4`
  - To work reliably as a binary downloader, the node may need explicit file/binary response settings depending on n8n version and PostPulse node expectations.
- **Edge cases or potential failure types:**
  - Missing `images[0].url`
  - Expired or inaccessible image URL
  - Response not treated as binary if node defaults are incorrect
  - Large file downloads or timeout issues
- **Sub-workflow reference:** None.

#### Upload media
- **Type and technical role:** `@postpulse/n8n-nodes-postpulse.postPulse`  
  Uploads a media file to PostPulse and returns a media path or identifier for future attachment.
- **Configuration choices:**
  - Resource selected: `media`
  - No explicit operation is shown beyond media handling.
- **Key expressions or variables used:**
  - Downstream node expects returned field:
    - `{{ $json.path }}`
- **Input and output connections:**
  - Input from **Download File**
  - Output to **Schedule a light post**
- **Version-specific requirements:**
  - Type version `1`
  - Requires the PostPulse community/custom node to be installed in n8n.
  - Requires PostPulse credentials.
- **Edge cases or potential failure types:**
  - Missing binary payload from previous node.
  - Invalid PostPulse credentials.
  - PostPulse API upload rejection or unsupported file format.
  - Output schema differences if the node returns a different field than `path`.
- **Sub-workflow reference:** None.

---

## 2.4 Scheduling and Source Record Update

### Overview
This block schedules the generated content in PostPulse and then updates the originating Airtable record so it is marked as already posted. It finalizes the workflow and closes the source-of-truth loop.

### Nodes Involved
- Schedule a light post
- Create or update a record

### Node Details

#### Schedule a light post
- **Type and technical role:** `@postpulse/n8n-nodes-postpulse.postPulse`  
  Creates a scheduled lightweight social post in PostPulse.
- **Configuration choices:**
  - Operation: `scheduleLight`
  - Social media account: `X_TWITTER|1154`
  - Post content comes from Claude output.
  - Attachment path comes from the uploaded media response.
  - Scheduled time is set dynamically to one day from current execution time.
- **Key expressions or variables used:**
  - Content:
    - `{{ $('Create Social Media Post').item.json.content[0].text }}`
  - Scheduled time:
    - `{{ $now.plus({ days: 1 }) }}`
  - Attachment paths:
    - `{{ $json.path }}`
- **Input and output connections:**
  - Input from **Upload media**
  - Output to **Create or update a record**
- **Version-specific requirements:**
  - Type version `1`
  - Requires installed PostPulse node and valid credentials.
  - The target account identifier must exist in the connected PostPulse workspace.
- **Edge cases or potential failure types:**
  - Invalid account ID or unauthorized account access.
  - Incorrect datetime format depending on PostPulse API expectations.
  - Missing or invalid media `path`.
  - Claude response missing at `content[0].text`.
  - Scheduling may fail if content violates platform rules or exceeds length limits.
- **Sub-workflow reference:** None.

#### Create or update a record
- **Type and technical role:** `n8n-nodes-base.airtable`  
  Upserts the Airtable record and marks it as posted.
- **Configuration choices:**
  - Operation: `upsert`
  - Base: `appohoaAJ51qH5A72`
  - Table: `tblNdd8e8BP4CbgkB`
  - Matching column: `id`
  - Sets field `Posted?` to `Yes`
  - Uses defined column mapping rather than auto-map
- **Key expressions or variables used:**
  - Uses the record `id` from incoming item for matching.
  - Sets:
    - `Posted? = Yes`
- **Input and output connections:**
  - Input from **Schedule a light post**
  - No downstream node.
- **Version-specific requirements:**
  - Type version `2.1`
  - Requires Airtable credentials with write access.
- **Edge cases or potential failure types:**
  - If the incoming item no longer contains the original Airtable `id`, upsert may fail or create unintended records.
  - Schema drift in Airtable fields.
  - Option field `Posted?` must include `Yes`.
  - Permission issues on Airtable writes.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Airtable Trigger | `n8n-nodes-base.airtableTrigger` | Entry point; watches Airtable for new records |  | Generate AI Image | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Kick-Off workflow<br>Searches Airtable and uses AI to generate an image and a sociale media post with Flux and Claude respectively. |
| Generate AI Image | `n8n-nodes-base.httpRequest` | Calls DumplingAI to generate an AI image from the Airtable prompt | Airtable Trigger | Create Social Media Post | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Kick-Off workflow<br>Searches Airtable and uses AI to generate an image and a sociale media post with Flux and Claude respectively. |
| Create Social Media Post | `@n8n/n8n-nodes-langchain.anthropic` | Generates short social post text with Claude | Generate AI Image | Download File | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Kick-Off workflow<br>Searches Airtable and uses AI to generate an image and a sociale media post with Flux and Claude respectively. |
| Download File | `n8n-nodes-base.httpRequest` | Downloads the generated image file from its returned URL | Create Social Media Post | Upload media | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Upload Generated AI Image<br>Downloads the generated AI image and uploads it for later scheduling. |
| Upload media | `@postpulse/n8n-nodes-postpulse.postPulse` | Uploads media into PostPulse and returns attachment path | Download File | Schedule a light post | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Upload Generated AI Image<br>Downloads the generated AI image and uploads it for later scheduling. |
| Schedule a light post | `@postpulse/n8n-nodes-postpulse.postPulse` | Schedules the generated text and image on X/Twitter via PostPulse | Upload media | Create or update a record | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Schedule Post and Updates Airtable<br>Schedules generated social media post and generated image in PostPulse and updates a record in Airtable. |
| Create or update a record | `n8n-nodes-base.airtable` | Marks the Airtable record as posted | Schedule a light post |  | **Workflow Purpose**<br>A social media content creation workflow that generates AI image, create social media content with Claude, and schedule post in PostPulse - using data from Airtable.<br>## Workflow Steps<br>**1. Airtable Trigger**<br>* Triggers the worflow whenever a new table is created in Airtable.<br>**2. Generate AI Image**<br>* Creates an AI Image using Flux and the image prompt from Airtable.<br>**3. Create Social Media Post**<br>* Creates a social media post using Claude and the post idea prompt from Airtable.<br>**4. Download File**<br>* Download the generated AI Image.<br>**5. Upload Media**<br>* Upload the AI Image to PostPulse for use in step 6. Output will generate a path ID.<br>**6. Schedule a Light Post**<br>* Schedule post to go out daily.<br>**7. Create or Update a record**<br>* Update status of Airtable record to "yes".<br>## Notes<br>A json expression is used in step 6 that schedules the post a day in advance. Right now it schedules one day in advance but can be changed to schedule two days in advance or more based on specification.<br>## Schedule Post and Updates Airtable<br>Schedules generated social media post and generated image in PostPulse and updates a record in Airtable. |
| Sticky Note2 | `n8n-nodes-base.stickyNote` | Canvas documentation for the full workflow |  |  |  |
| Sticky Note1 | `n8n-nodes-base.stickyNote` | Canvas note for the kickoff/generation area |  |  |  |
| Sticky Note | `n8n-nodes-base.stickyNote` | Canvas note for media download/upload area |  |  |  |
| Sticky Note3 | `n8n-nodes-base.stickyNote` | Canvas note for scheduling/update area |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Recommended workflow name: **Create social media content with AI and schedule for daily posting**.

2. **Prepare the Airtable table**
   - Create or use an Airtable base.
   - Add a table containing at least these fields:
     - `Post Idea` (text)
     - `Prompt` (text)
     - `Created` (created time field recommended)
     - `Posted?` (single select or equivalent, with value `Yes` available)
   - Ensure records have unique Airtable record IDs, which Airtable always provides internally.

3. **Add the Airtable Trigger node**
   - Node type: **Airtable Trigger**
   - Configure:
     - Authentication: Airtable token API
     - Base: select your base
     - Table: select your table
     - Trigger field: `Created`
     - Polling: leave default unless you need a specific interval
   - This node starts the workflow whenever a new record is created.

4. **Configure Airtable credentials**
   - Create Airtable credentials in n8n.
   - Use an API token with access to the selected base and table.
   - Confirm it has read access for the trigger and write access for the later update node.

5. **Add the Generate AI Image node**
   - Node type: **HTTP Request**
   - Name it: **Generate AI Image**
   - Configure:
     - Method: `POST`
     - URL: `https://app.dumplingai.com/api/v1/generate-ai-image`
     - Send Headers: enabled
     - Send Body: enabled
     - Body Content Type: JSON
   - Add headers:
     - `Content-Type` = `application/json`
     - `Authorization` = `Bearer <your_dumplingai_token>`
   - Set JSON body to:
     ```json
     {
       "model": "FLUX.1-schnell",
       "input": {
         "prompt": "{{ $json.fields.Prompt }}",
         "aspect_ratio": "16:9"
       }
     }
     ```
   - In n8n expression mode, the `prompt` value should resolve from the Airtable Trigger item.

6. **Connect Airtable Trigger → Generate AI Image**

7. **Add the Create Social Media Post node**
   - Node type: **Anthropic** (LangChain Anthropic node)
   - Name it: **Create Social Media Post**
   - Configure:
     - Model: `claude-haiku-4-5-20251001` if available in your environment
     - Add one message
       - Role: `assistant` as in the original workflow, though in many setups `user` is more conventional
       - Content:
         ```
         Create a short social media post using the provided idea below. Generate only the output:

         {{ $('Airtable Trigger').item.json.fields['Post Idea'] }}
         ```
   - This node should output generated content in a structure accessible as `content[0].text`.

8. **Configure Anthropic credentials**
   - Add Anthropic API credentials in n8n.
   - Make sure the selected model is available for your account and node version.

9. **Connect Generate AI Image → Create Social Media Post**

10. **Add the Download File node**
    - Node type: **HTTP Request**
    - Name it: **Download File**
    - Configure:
      - Method: `GET` or default request mode
      - URL:
        ```
        {{ $('Generate AI Image').item.json.images[0].url }}
        ```
    - For a robust rebuild, enable file download/binary response handling if your n8n version requires it.
    - The goal is to pass the generated image as binary data to the next node.

11. **Connect Create Social Media Post → Download File**
    - Note: this means the workflow waits for text generation before downloading the image, even though the image URL already exists from a prior node.

12. **Install the PostPulse node package if needed**
    - The workflow uses `@postpulse/n8n-nodes-postpulse.postPulse`.
    - If not already available, install the PostPulse community/custom node in your n8n instance.
    - Restart n8n if required.

13. **Add the Upload media node**
    - Node type: **PostPulse**
    - Name it: **Upload media**
    - Configure:
      - Resource: `media`
    - Ensure the node is set up to receive the binary file from the previous step and upload it to PostPulse.
    - The output should expose a media path, expected later as `path`.

14. **Configure PostPulse credentials**
    - Add PostPulse credentials in n8n.
    - Confirm they allow media upload and post scheduling.

15. **Connect Download File → Upload media**

16. **Add the Schedule a light post node**
    - Node type: **PostPulse**
    - Name it: **Schedule a light post**
    - Configure:
      - Operation: `scheduleLight`
      - Content:
        ```
        {{ $('Create Social Media Post').item.json.content[0].text }}
        ```
      - Scheduled Time:
        ```
        {{ $now.plus({ days: 1 }) }}
        ```
      - Attachment Paths:
        ```
        {{ $json.path }}
        ```
      - Social Media Account:
        ```
        X_TWITTER|1154
        ```
    - Replace `X_TWITTER|1154` with your actual PostPulse-connected account identifier if different.

17. **Connect Upload media → Schedule a light post**

18. **Add the Create or update a record node**
    - Node type: **Airtable**
    - Name it: **Create or update a record**
    - Configure:
      - Authentication: same Airtable credentials
      - Operation: `upsert`
      - Base: same Airtable base
      - Table: same Airtable table
      - Matching columns: `id`
      - Mapping mode: define below
      - Field mapping:
        - `Posted?` = `Yes`
    - Keep the Airtable `id` available in the item so the node updates the original record instead of creating a new unrelated one.

19. **Connect Schedule a light post → Create or update a record**

20. **Optionally add sticky notes for documentation**
    - Add canvas notes similar to the original:
      - one for overall workflow purpose,
      - one for Airtable/AI generation,
      - one for image download/upload,
      - one for scheduling/Airtable update.

21. **Test the workflow with a sample Airtable record**
    - Insert a new record in Airtable with:
      - a valid `Post Idea`
      - a valid `Prompt`
    - Confirm:
      - Airtable Trigger fires
      - DumplingAI returns an image URL
      - Claude returns text
      - file downloads correctly
      - PostPulse upload returns `path`
      - scheduled post is created
      - Airtable `Posted?` becomes `Yes`

22. **Harden the workflow before production**
    - Replace the hardcoded DumplingAI bearer token with a secure credential or variable.
    - Remove any trailing spaces from API URLs.
    - Consider adding validation or IF nodes for:
      - missing `Prompt`
      - missing `Post Idea`
      - missing `images[0].url`
      - missing `content[0].text`
      - PostPulse upload/scheduling errors
    - Consider adding retry logic around external API calls.

### Sub-workflow setup
This workflow does **not** invoke any sub-workflows and has only one entry point:
- **Airtable Trigger**

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Workflow Purpose: A social media content creation workflow that generates AI image, creates social media content with Claude, and schedules posts in PostPulse using data from Airtable. | Overall workflow intent |
| The schedule time is controlled by a JSON expression and currently posts one day in advance. This can be changed to two days or more. | Scheduling logic in PostPulse node |
| Kick-Off workflow: Searches Airtable and uses AI to generate an image and a social media post with Flux and Claude respectively. | Input and AI generation area |
| Upload Generated AI Image: Downloads the generated AI image and uploads it for later scheduling. | Media handling area |
| Schedule Post and Updates Airtable: Schedules generated social media post and generated image in PostPulse and updates a record in Airtable. | Final scheduling and record update area |