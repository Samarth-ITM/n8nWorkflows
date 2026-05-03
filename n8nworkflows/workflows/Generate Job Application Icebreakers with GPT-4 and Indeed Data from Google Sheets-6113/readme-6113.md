Generate Job Application Icebreakers with GPT-4 and Indeed Data from Google Sheets

https://n8nworkflows.xyz/workflows/generate-job-application-icebreakers-with-gpt-4-and-indeed-data-from-google-sheets-6113


# Generate Job Application Icebreakers with GPT-4 and Indeed Data from Google Sheets

### 1. Workflow Overview

This workflow automates the generation of personalized email icebreakers for job applications, leveraging job listing data scraped and stored in a Google Sheet. The primary use case is to create concise, customized five-line introductory messages that subtly imply familiarity with the job and company, enhancing outreach effectiveness.

The workflow consists of three main logical blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and retrieval of job listing data from Google Sheets.
- **1.2 AI Processing:** Sending extracted job and company information to GPT-4 via LangChain to generate a personalized icebreaker message.
- **1.3 Output Update:** Updating the original Google Sheet row with the generated icebreaker.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow manually and fetches relevant rows from a Google Sheet containing job and company data.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Get row(s) in sheet  
  - Sticky Note (AI Personal Icebreaker generation)

- **Node Details:**

  1. **When clicking ‘Execute workflow’**  
     - *Type:* Manual Trigger  
     - *Role:* Entry point for manual execution of the workflow.  
     - *Configuration:* Default manual trigger with no parameters.  
     - *Input/Output:* No input; output triggers the next node.  
     - *Edge Cases:* None significant; failure unlikely unless system downtime occurs.

  2. **Get row(s) in sheet**  
     - *Type:* Google Sheets node  
     - *Role:* Reads rows from a specified Google Sheet tab that contains job listings and related data.  
     - *Configuration:*  
       - Document ID and Sheet Name reference a Google Sheet titled "Copy of Indeed JSHI".  
       - Filters applied on the "icebreaker" column to fetch rows (likely those missing icebreakers).  
       - Uses authenticated Google Sheets OAuth2 credentials.  
       - Version 4.6 of the node.  
     - *Key Expressions:* None complex; standard filter on "icebreaker" column.  
     - *Input/Output:* Input from manual trigger; outputs JSON data representing job listing rows.  
     - *Edge Cases:* Possible issues with Google API limits, authentication errors, or no rows matching filter.

  3. **Sticky Note (AI Personal Icebreaker generation)**  
     - *Type:* Sticky Note (informational)  
     - *Role:* Visual block label — no execution impact.  
     - *Content:* "## AI Personal Icebreaker generation" — marks the AI processing section.

#### 1.2 AI Processing

- **Overview:** Uses OpenAI GPT-4 (via LangChain) to generate a customized short email icebreaker based on job and company data.
- **Nodes Involved:**  
  - Personalization  
  - Sticky Note1 (Informational note about workflow context)

- **Node Details:**

  1. **Personalization**  
     - *Type:* LangChain OpenAI node  
     - *Role:* Sends structured prompts to GPT-4 to generate an icebreaker message.  
     - *Configuration:*  
       - Model: GPT-4.1-MINI (a GPT-4 variant)  
       - Temperature: 0.8 (moderate randomness for creativity)  
       - System prompt: Sets the assistant role as a helpful writing assistant.  
       - User prompt: Detailed instructions to generate a five-line personalized icebreaker using specific formula rules (e.g., Spartan tone, use of acronyms, subtle familiarity).  
       - Input message content dynamically constructed using expressions referencing fields from the Google Sheet row, such as CEO name, job title, description, company information.  
       - Output format: JSON with a single field "icebreaker".  
       - Json Output enabled to parse the response automatically.  
     - *Input/Output:* Receives job data from the sheet; outputs JSON with generated icebreaker.  
     - *Edge Cases:* Potential failures include OpenAI API rate limits, malformed input data causing prompt errors, or unexpected response formats.  
     - *Version Requirements:* Requires n8n version supporting LangChain OpenAI node v1.8 or higher.  
     - *Notes:* The node's prompt carefully instructs GPT-4 to maintain a consistent tone and format.

  2. **Sticky Note1**  
     - *Type:* Sticky Note (informational)  
     - *Role:* Provides context and explanation for the workflow segment.  
     - *Content:* Describes this workflow as part two of a larger Indeed Job Scraper and Enrichment process, emphasizing the generation of personalized icebreakers from stored job data.

#### 1.3 Output Update

- **Overview:** Updates the original Google Sheet row with the newly generated icebreaker text, preserving all other original job data fields.
- **Nodes Involved:**  
  - Update row in sheet

- **Node Details:**

  1. **Update row in sheet**  
     - *Type:* Google Sheets node  
     - *Role:* Updates a row in the Google Sheet, adding the generated icebreaker to the "icebreaker" column.  
     - *Configuration:*  
       - Document ID and Sheet Name same as the "Get row(s) in sheet" node.  
       - Operation: "update" mode, matching rows by the unique "row_number" column.  
       - Columns mapped include all original data fields plus the new "icebreaker" generated from GPT-4 response.  
       - Uses the same Google Sheets OAuth2 credentials.  
       - Version 4.6 of the node.  
     - *Key Expressions:* Uses expressions to map fields from the original row and the icebreaker from the Personalization node output (`$json.message.content.icebreaker`).  
     - *Input/Output:* Input from the Personalization node; no further output in this workflow.  
     - *Edge Cases:* Possible update failures due to Google Sheets API limits, incorrect row_number references, or credential expiration.

---

### 3. Summary Table

| Node Name                    | Node Type                     | Functional Role                         | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                                         |
|------------------------------|-------------------------------|---------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                | Starts the workflow manually           |                              | Get row(s) in sheet          |                                                                                                                                    |
| Get row(s) in sheet           | Google Sheets                 | Reads job listing rows from Google Sheets | When clicking ‘Execute workflow’ | Personalization               |                                                                                                                                    |
| Sticky Note                  | Sticky Note                   | Visual label for AI Personal Icebreaker generation |                              |                              | ## AI Personal Icebreaker generation                                                                                               |
| Personalization              | LangChain OpenAI (GPT-4)     | Generates personalized icebreaker text | Get row(s) in sheet           | Update row in sheet           | ## Note  Part two of the Indeed Job Scraper, Filter and Enrichment workflow, this workflow takes information about the scraped and filtered job listings on Indeed via Apify, which is stored in Google Sheets to generate a customized, five-line email icebreaker to imply that the rest of the icebreaker is personalized. Personalized IJSFE (Indeed Job Scraper For Enrichment). |
| Sticky Note1                 | Sticky Note                   | Informational context about workflow  |                              |                              | ## Note  Part two of the Indeed Job Scraper, Filter and Enrichment workflow, this workflow takes information about the scraped and filtered job listings on Indeed via Apify, which is stored in Google Sheets to generate a customized, five-line email icebreaker to imply that the rest of the icebreaker is personalized. Personalized IJSFE (Indeed Job Scraper For Enrichment). |
| Update row in sheet          | Google Sheets                 | Updates the Google Sheet row with icebreaker | Personalization              |                              |                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a Manual Trigger node named "When clicking ‘Execute workflow’".  
   - No parameters needed.

2. **Add Google Sheets Node to Read Rows**  
   - Add a Google Sheets node named "Get row(s) in sheet".  
   - Set operation to "Read Rows" or equivalent.  
   - Configure document ID to your Google Sheet containing job listings.  
   - Set the sheet name to the specific tab with job data (e.g., "Copy of Indeed JSHI").  
   - Apply a filter on the "icebreaker" column to select rows needing icebreaker generation.  
   - Connect Google Sheets OAuth2 credentials with read access.  
   - Connect "When clicking ‘Execute workflow’" output to this node input.

3. **Add Sticky Note (Optional)**  
   - Place a Sticky Note named "AI Personal Icebreaker generation" for clarity.

4. **Add LangChain OpenAI Node for Personalization**  
   - Add an OpenAI node via LangChain named "Personalization".  
   - Select the GPT-4.1-MINI model or equivalent GPT-4 variant.  
   - Set temperature to 0.8.  
   - Enable JSON output parsing.  
   - Configure system message as: "You're a helpful, intelligent writing assistant."  
   - Configure user message with instructions to generate a five-line personalized icebreaker with the provided formula and tone rules.  
   - Use expressions to dynamically pass job and company data from the Google Sheets node output, including:  
     - CEO first name (`companyCeo/name`)  
     - Job title (`title`)  
     - Job description (`descriptionText`)  
     - Company name (`companyName`)  
     - Company description (`companyDescription`)  
   - Connect "Get row(s) in sheet" output to this node input.

5. **Add Sticky Note1 (Optional)**  
   - Add a Sticky Note named "Note" explaining the workflow context as part two of Indeed Job Scraper and Enrichment.

6. **Add Google Sheets Node to Update Rows**  
   - Add another Google Sheets node named "Update row in sheet".  
   - Set operation to "Update Row".  
   - Use the same document ID and sheet name as the read node.  
   - Map all original columns from the "Get row(s) in sheet" node output to preserve data.  
   - Map "icebreaker" column to the JSON output `message.content.icebreaker` from the "Personalization" node.  
   - Use "row_number" as the matching column to identify the row to update.  
   - Connect "Personalization" node output to this node input.  
   - Use the same Google Sheets OAuth2 credentials with write access.

7. **Verify Connections and Save**  
   - Ensure connections flow:  
     Manual Trigger -> Get row(s) in sheet -> Personalization -> Update row in sheet.  
   - Save the workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Part two of the Indeed Job Scraper, Filter and Enrichment workflow, this workflow takes information about the scraped and filtered job listings on Indeed via Apify, which is stored in Google Sheets to generate a customized, five-line email icebreaker to imply that the rest of the icebreaker is personalized. Personalized IJSFE (Indeed Job Scraper For Enrichment). | Sticky Note1 content in the workflow; describes overall project scope and integration approach.          |

---

**Disclaimer:** The text and data processed in this workflow are handled exclusively within n8n automation respecting all applicable content policies. No illegal, offensive, or protected information is involved. All data processed is legal and public.