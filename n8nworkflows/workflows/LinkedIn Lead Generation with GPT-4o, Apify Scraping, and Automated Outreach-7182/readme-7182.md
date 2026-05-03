LinkedIn Lead Generation with GPT-4o, Apify Scraping, and Automated Outreach

https://n8nworkflows.xyz/workflows/linkedin-lead-generation-with-gpt-4o--apify-scraping--and-automated-outreach-7182


# LinkedIn Lead Generation with GPT-4o, Apify Scraping, and Automated Outreach

### 1. Workflow Overview

This workflow automates LinkedIn lead generation by integrating GPT-4o for intelligent URL construction and personalized message creation, Apify for web scraping prospects’ data, Google Sheets for data storage, and Phantombuster for sending connection requests on LinkedIn. It targets sales and marketing professionals seeking to streamline outreach by automatically generating prospect lists, crafting personalized icebreakers, and executing connection requests.

Logical blocks:

- **1.1 Input Reception:** Capture audience description via a web form.
- **1.2 AI-Driven URL Generation:** Use GPT-4o to translate audience description into an Apollo search URL.
- **1.3 Prospect Data Scraping:** Run an Apify actor to scrape LinkedIn prospect data using the generated URL.
- **1.4 Icebreaker Generation:** Use GPT-4o to create a concise, personalized icebreaker message from scraped data.
- **1.5 Data Aggregation and Storage:** Aggregate results and append/update Google Sheets with prospect info and icebreakers.
- **1.6 Automated Outreach:** Trigger Phantombuster to send personalized LinkedIn connection requests.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Collects a plain-text description of the target audience for lead generation via a web form.

- **Nodes Involved:**  
"Description of the audience you want to scrap"

- **Node Details:**

  - **Description of the audience you want to scrap**  
    - Type: Form Trigger (Webhook)  
    - Role: Entry point; receives user input describing company characteristics (location, size, industry).  
    - Configuration: Single required text field labeled “descritpion of the company” with a placeholder prompting details like company location, size, and industry. Form title: "Audience".  
    - Inputs: External HTTP request (form submission).  
    - Outputs: Passes form data to the URL generation node.  
    - Edge Cases: User submits incomplete or vague descriptions; no validation beyond required field. Network issues could block webhook reception.

#### 1.2 AI-Driven URL Generation

- **Overview:**  
Transforms the audience description into a structured Apollo search URL using GPT-4o, guiding the Apify scraper.

- **Nodes Involved:**  
"Genrating appolo Url for apify to scrap", "Sticky Note4"

- **Node Details:**

  - **Genrating appolo Url for apify to scrap**  
    - Type: OpenAI (Chat Completion) node using GPT-4o model  
    - Role: Converts natural language audience description into an Apollo.io people search URL in JSON format.  
    - Configuration: System prompt defines the task, provides example URL structure, and restricts editable fields to location, titles, and keywords. Input prompt dynamically uses the form field `descritpion of the company `.  
    - Key Expressions: `={{ $json['descritpion of the company '] }}` used to insert form input.  
    - Inputs: Receives audience description JSON from form trigger.  
    - Outputs: JSON with `SearchUrl` field, sent to the Apify scraper node.  
    - Edge Cases: GPT might generate malformed URLs or incomplete JSON; requires validation downstream.  
    - Sticky Note4 Content: Explains purpose of URL generation via GPT-4o.

#### 1.3 Prospect Data Scraping

- **Overview:**  
Executes an Apify actor that scrapes prospect data from Apollo.io using the generated search URL.

- **Nodes Involved:**  
"Run apify actor to scrap the proscpect", "Sticky Note3"

- **Node Details:**

  - **Run apify actor to scrap the proscpect**  
    - Type: HTTP Request (POST)  
    - Role: Calls Apify API to run a predefined actor synchronously and obtain prospect data.  
    - Configuration:  
      - URL: Apify actor run-sync endpoint  
      - Method: POST  
      - Headers: Accept JSON, Authorization with Apify API key ("Bearer api key" placeholder)  
      - Body: JSON containing `cleanOutput: true` and the URL from previous node (`{{ $json.message.content.SearchUrl }}`)  
      - Sends JSON body with search URL to Apify.  
    - Inputs: Receives JSON from GPT URL generation node.  
    - Outputs: Scraped prospect data passed to icebreaker generation node.  
    - Edge Cases: API key invalid or expired, Apify actor errors, malformed URL input, network failure.  
    - Sticky Note3 Content: Confirms API key required, node sends request to Apify actor.

#### 1.4 Icebreaker Generation

- **Overview:**  
Creates a concise, human-like icebreaker message for each prospect based on scraped LinkedIn profile information.

- **Nodes Involved:**  
"Genrate ice breaker by scraping linkedin data", "Sticky Note2"

- **Node Details:**

  - **Genrate ice breaker by scraping linkedin data**  
    - Type: OpenAI (Chat Completion) node with GPT-4o  
    - Role: Generates a short, punchy icebreaker using LinkedIn profile data fields.  
    - Configuration:  
      - System prompt instructs to produce a laconic message following a specific template and to avoid common clichés.  
      - Input example embedded in prompt includes LinkedIn fields such as name, location, position, and previous roles (hardcoded in the example but dynamically mapped in actual use).  
      - Output format: JSON with `Icebreaker` key.  
    - Inputs: Receives scraped prospect data from Apify node.  
    - Outputs: JSON icebreaker message sent to Google Sheets node.  
    - Edge Cases: GPT may misunderstand data or output invalid JSON; prompt tuning needed for quality.  
    - Sticky Note2 Content: Notes prompt is hardcoded and can be customized.

#### 1.5 Data Aggregation and Storage

- **Overview:**  
Aggregates generated icebreakers with prospect data, then appends or updates the Google Sheet to maintain the lead list.

- **Nodes Involved:**  
"Adding ice breaker to google sheets", "Aggregate", "Sticky Note"

- **Node Details:**

  - **Aggregate**  
    - Type: Aggregate node  
    - Role: Merges or collects all item data from previous node(s) into a single output batch.  
    - Configuration: Uses default aggregate all items settings.  
    - Inputs: Receives icebreaker data from Google Sheets append/update node.  
    - Outputs: Feeds aggregated data to the next node (Phantombuster trigger).  
    - Edge Cases: Large datasets might cause performance issues.

  - **Adding ice breaker to google sheets**  
    - Type: Google Sheets node  
    - Role: Appends or updates rows in a specified Google Sheet with prospect details and icebreaker texts.  
    - Configuration:  
      - Operation: Append or Update  
      - Sheet Name and Document ID must be set (placeholders present, user needs to specify).  
    - Inputs: Receives icebreaker + prospect data from GPT node.  
    - Outputs: Passes data to aggregation node.  
    - Edge Cases: Authentication failures, incorrect sheet/document IDs, quota limits.  
    - Sticky Note Content: Advises to create a Google Sheet with columns for name, LinkedIn URL, company website, description, and other scraped fields, mapping Apify output accordingly.

#### 1.6 Automated Outreach

- **Overview:**  
Triggers a Phantombuster agent via HTTP request to send personalized LinkedIn connection requests to scraped leads using generated icebreakers.

- **Nodes Involved:**  
"trigger phantom buster to send personlized connection request", "Aggregate", "Sticky Note1"

- **Node Details:**

  - **trigger phantom buster to send personlized connection request**  
    - Type: HTTP Request (POST)  
    - Role: Initiates a Phantombuster API call to launch an agent that sends connection requests.  
    - Configuration:  
      - URL contains placeholder for Phantombuster agent ID and requires API key in header (`X-Phantombuster-Key`).  
      - No body needed.  
    - Inputs: Receives aggregated data from Aggregate node.  
    - Outputs: None (end of workflow).  
    - Edge Cases: Incorrect API key, invalid agent ID, rate limits, Phantombuster service downtime.  
    - Sticky Note1 Content: Instructs user to replace agent ID and API key; mentions no request body needed.

---

### 3. Summary Table

| Node Name                                | Node Type                      | Functional Role                          | Input Node(s)                           | Output Node(s)                                   | Sticky Note                                                                                                      |
|-----------------------------------------|--------------------------------|----------------------------------------|---------------------------------------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Description of the audience you want to scrap | Form Trigger (Webhook)          | Capture audience description input     | (External HTTP request)                | Genrating appolo Url for apify to scrap          |                                                                                                                  |
| Genrating appolo Url for apify to scrap | OpenAI (Chat Completion)        | Generate Apollo search URL from input  | Description of the audience you want to scrap | Run apify actor to scrap the proscpect            | Generates an Apollo URL,using GPT-4o, which will be scraped by apify actor to get the leads data. the url is generated by the information you input in the form. |
| Run apify actor to scrap the proscpect | HTTP Request                   | Run Apify actor to scrape prospect data| Genrating appolo Url for apify to scrap | Genrate ice breaker by scraping linkedin data    | Send a request to run the apify actor that scrapes the leads; everything is configuered add your API key         |
| Genrate ice breaker by scraping linkedin data | OpenAI (Chat Completion)        | Generate personalized icebreaker message | Run apify actor to scrap the proscpect | Adding ice breaker to google sheets               | Hard-coded the prompt for GPT4o to generate the icebreaker; alter the prompt according to your desired need       |
| Adding ice breaker to google sheets     | Google Sheets                   | Append/update prospect data and icebreaker | Genrate ice breaker by scraping linkedin data | Aggregate                                           | Create a google sheet add relevant colums name,linkedin url,company website,description and oter coloums that are genrated by the apify node and map them accordingly |
| Aggregate                              | Aggregate                      | Aggregate all item data                 | Adding ice breaker to google sheets   | trigger phantom buster to send personlized connection request |                                                                                                                  |
| trigger phantom buster to send personlized connection request | HTTP Request                   | Trigger Phantombuster agent to send connection requests | Aggregate                             |                                                 | Triggers the phantom buster to send LinkedIn connection requests. Add your API key and agent ID in the request URL and headers. No body content needed.            |
| Sticky Note                            | Sticky Note                   | Instruction on Google Sheets setup     |                                       |                                                 | Create a google sheet add relevant colums name,linkedin url,company website,description and oter coloums that are genrated by the apify node and map them accordingly |
| Sticky Note1                           | Sticky Note                   | Instruction on Phantombuster setup     |                                       |                                                 | Triggers the phantom buster a platform that sends linkedin connection requests. Add your API key and agent ID in the header and URL.                                |
| Sticky Note2                           | Sticky Note                   | Instruction on Icebreaker prompt       |                                       |                                                 | Hard-coded the prompt for GPT4o to generate the icebreaker; alter the prompt according to your desired need       |
| Sticky Note3                           | Sticky Note                   | Instruction on Apify actor request     |                                       |                                                 | Send a request to run the apify actor that scrapes the leads; everything is configuered add your API key          |
| Sticky Note4                           | Sticky Note                   | Instruction on Apollo URL generation   |                                       |                                                 | Generates an Apollo URL,using GPT-4o, which will be scraped by apify actor to get the leads data. the url is generated by the information you input in the form. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Description of the audience you want to scrap"**  
   - Type: Form Trigger  
   - Configure webhook with accessible URL  
   - Form Title: "Audience"  
   - Add one text field:  
     - Label: "descritpion of the company"  
     - Placeholder: "company location,size,industry etc"  
     - Required: Yes

2. **Add OpenAI Node: "Genrating appolo Url for apify to scrap"**  
   - Type: OpenAI (Chat Completion)  
   - Model: chatgpt-4o-latest  
   - Credentials: Your OpenAI API credentials  
   - System prompt: Define task to convert audience description into Apollo.io search URL (use example URL and restrictions as per original prompt)  
   - User prompt: Insert expression `{{$json["descritpion of the company "]}}` to pass form input  
   - Enable JSON output parsing  
   - Connect input from Form Trigger node

3. **Add HTTP Request Node: "Run apify actor to scrap the proscpect"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apify.com/v2/acts/jljBwyyQakqrL1wae/run-sync-get-dataset-items`  
   - Headers:  
     - Accept: application/json  
     - Authorization: Bearer [Your Apify API Key]  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "cleanOutput": true,
       "url": "{{ $json.message.content.SearchUrl }}"
     }
     ```  
   - Connect input from OpenAI URL generation node

4. **Add OpenAI Node: "Genrate ice breaker by scraping linkedin data"**  
   - Type: OpenAI (Chat Completion)  
   - Model: chatgpt-4o-latest  
   - Credentials: Your OpenAI API credentials  
   - System prompt: Instructions to create short, laconic icebreaker messages using LinkedIn fields, with a JSON output format.  
   - Input message: Map prospect data fields (name, location, title, previous roles) dynamically from Apify output.  
   - Connect input from Apify HTTP Request node

5. **Add Google Sheets Node: "Adding ice breaker to google sheets"**  
   - Type: Google Sheets  
   - Operation: Append or Update  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: Specify your sheet name with columns for name, LinkedIn URL, company website, description, icebreaker, etc.  
   - Ensure proper mapping of fields from icebreaker and prospect data  
   - Connect input from Icebreaker OpenAI node

6. **Add Aggregate Node: "Aggregate"**  
   - Type: Aggregate  
   - Default settings (aggregate all items)  
   - Connect input from Google Sheets node

7. **Add HTTP Request Node: "trigger phantom buster to send personlized connection request"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.phantombuster.com/api/v2/[Your Agent ID]/launch`  
   - Headers:  
     - X-Phantombuster-Key: [Your Phantombuster API Key]  
   - No body content needed  
   - Connect input from Aggregate node

8. **Add Sticky Notes as needed to document the workflow steps and instructions for maintenance or handover.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                 | Context or Link                                                                                      |
|------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create a Google Sheet with relevant columns: name, LinkedIn URL, company website, description, and other Apify-generated fields. Map these correctly in the workflow. | Instruction for Google Sheets node and data management                                             |
| Phantombuster agent URL requires your specific agent ID; add your API key in request headers. No body content is necessary.   | Guidance for setting up LinkedIn connection automation                                            |
| The icebreaker prompt is hardcoded in GPT-4o node; customize it to suit your outreach tone and style.                        | Customization tip for improving message quality                                                  |
| The Apollo URL generation prompt restricts editable URL fields and provides an example for structure.                        | Important for URL generation reliability and ensuring correct scraping                            |

---

**Disclaimer:** This document is based exclusively on an automated workflow created with n8n, respecting all current content policies. It contains no illegal, offensive, or protected content. All data processed is legal and public.