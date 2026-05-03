LinkedIn scraping, structuring, and messaging using PhantomBuster and GPT-4

https://n8nworkflows.xyz/workflows/linkedin-scraping--structuring--and-messaging-using-phantombuster-and-gpt-4-5070


# LinkedIn scraping, structuring, and messaging using PhantomBuster and GPT-4

---
### 1. Workflow Overview

This workflow automates LinkedIn profile data scraping, structured data extraction, and personalized messaging using PhantomBuster and GPT-4 via Azure OpenAI. It targets use cases such as lead generation, recruitment outreach, or sales prospecting by automating data retrieval from LinkedIn URLs stored in a Google Sheet, processing the data with AI models, and updating messaging results back to the spreadsheet.

The workflow is divided into the following logical blocks:

- **1.1 Trigger and Input Retrieval:** Scheduled initiation and fetching LinkedIn profile URLs from a Google Sheet.
- **1.2 PhantomBuster Scraping Process:** Storing URLs temporarily, invoking PhantomBuster API to scrape profiles, and polling for results.
- **1.3 AI-Based Data Extraction:** Using GPT-4 to parse PhantomBuster results, extract profile URLs, and retrieve detailed LinkedIn data.
- **1.4 Personalized Message Generation:** Creating tailored messages for each profile using a GPT-4 template and updating the spreadsheet.
- **1.5 Cleanup:** Removing temporary URL storage to maintain data hygiene.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule and retrieves LinkedIn profile URLs from a Google Sheet to process.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch URL from Spreadsheet  
  - Temp URL Storage

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow automatically based on a configured schedule.  
    - Configuration: Default scheduling parameters; triggers without manual input.  
    - Input: None  
    - Output: Triggers next node to fetch URLs.  
    - Failure Cases: Scheduling misconfiguration or n8n downtime can delay execution.

  - **Fetch URL from Spreadsheet**  
    - Type: Google Sheets  
    - Role: Reads LinkedIn profile URLs from a specified spreadsheet and worksheet.  
    - Configuration: Google Sheets credential, sheet ID, range or filters for URLs.  
    - Input: Trigger from Schedule Trigger.  
    - Output: Passes rows containing URLs to Temp URL Storage.  
    - Failure Cases: Authentication errors, incorrect sheet/range, API limits.

  - **Temp URL Storage**  
    - Type: Google Sheets  
    - Role: Temporarily stores the URLs fetched for processing. Acts as a staging area before passing to PhantomBuster.  
    - Configuration: Separate sheet/tab for temporary data with append or overwrite mode.  
    - Input: Output from Fetch URL from Spreadsheet.  
    - Output: Initiates PhantomBuster API POST request.  
    - Failure Cases: Sheet access errors, rate limits.

---

#### 2.2 PhantomBuster Scraping Process

- **Overview:**  
  This block sends the URLs to PhantomBuster via API to scrape LinkedIn profiles, then waits and polls until scraping results are ready.

- **Nodes Involved:**  
  - POST to Phantombuster API  
  - Wait  
  - GET from Phantombuster API

- **Node Details:**

  - **POST to Phantombuster API**  
    - Type: HTTP Request  
    - Role: Sends a POST request to PhantomBuster API to start scraping jobs with stored URLs.  
    - Configuration: API endpoint URL, authentication headers (API key), request body including URLs.  
    - Input: Output from Temp URL Storage.  
    - Output: Initiates Wait node.  
    - Failure Cases: API authentication failure, network errors, invalid request body.

  - **Wait**  
    - Type: Wait  
    - Role: Pauses workflow for a predefined duration allowing PhantomBuster to complete scraping.  
    - Configuration: Time duration configured to balance between speed and data availability.  
    - Input: Triggered after POST request.  
    - Output: Triggers GET from PhantomBuster API.  
    - Failure Cases: Timeout too short leading to incomplete data; too long causing delays.

  - **GET from Phantombuster API**  
    - Type: HTTP Request  
    - Role: Polls PhantomBuster API to retrieve scraping results.  
    - Configuration: API endpoint URL with job ID, authentication headers.  
    - Input: After Wait node.  
    - Output: Passes raw scraping data to AI extraction.  
    - Failure Cases: API errors, rate limits, incomplete or empty data.

---

#### 2.3 AI-Based Data Extraction

- **Overview:**  
  Processes raw PhantomBuster output with GPT-4 to extract clean LinkedIn profile URLs and then fetches detailed profile information.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model  
  - Structured Output Parser  
  - Extract URL from Output using GPT-4  
  - GET LinkedIn Data from URL

- **Node Details:**

  - **Azure OpenAI Chat Model**  
    - Type: Azure OpenAI Chat Model (GPT-4)  
    - Role: Chat-based AI model to interpret PhantomBuster output and assist in extraction.  
    - Configuration: Uses Azure credentials, GPT-4 model, prompt engineering to guide extraction.  
    - Input: Raw data from GET from PhantomBuster API.  
    - Output: Passes to Structured Output Parser.  
    - Failure Cases: API quota limits, prompt failures, malformed data.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI chat output into a structured JSON format for reliable downstream processing.  
    - Configuration: Schema-based parsing to extract URLs.  
    - Input: AI model response.  
    - Output: Clean URLs to Extract URL from Output using GPT-4 node.  
    - Failure Cases: Parsing errors if AI output deviates from schema.

  - **Extract URL from Output using GPT-4**  
    - Type: Langchain Agent Node (GPT-4)  
    - Role: Further refines URL extraction and validates LinkedIn URLs.  
    - Configuration: Uses AI agent with custom prompt for URL extraction logic.  
    - Input: Parsed structured output.  
    - Output: Provides LinkedIn profile URLs to the HTTP request for detailed data.  
    - Failure Cases: AI misinterpretation, empty or invalid URLs.

  - **GET LinkedIn Data from URL**  
    - Type: HTTP Request  
    - Role: Retrieves detailed LinkedIn profile data from extracted URLs via API or scraping endpoint.  
    - Configuration: URL parameterized from previous node, retry enabled for robustness.  
    - Input: Extracted LinkedIn URLs.  
    - Output: Passes detailed profile data to message generation.  
    - Failure Cases: Request failures, rate limits, 404 or private profiles.

---

#### 2.4 Personalized Message Generation

- **Overview:**  
  Uses GPT-4 to produce personalized outreach messages based on LinkedIn profile data and a messaging template, then updates results back to the spreadsheet.

- **Nodes Involved:**  
  - Azure OpenAI Chat Model1  
  - Structured Output Parser1  
  - Create Personalised Messages using a Template  
  - Update Personalised Messages to Spreadsheet

- **Node Details:**

  - **Azure OpenAI Chat Model1**  
    - Type: Azure OpenAI Chat Model (GPT-4)  
    - Role: Generates personalized messages using profile data and a predefined template.  
    - Configuration: Uses GPT-4, system prompt includes message template context.  
    - Input: Detailed LinkedIn data.  
    - Output: Sent to Structured Output Parser1.  
    - Failure Cases: API errors, template mismatches.

  - **Structured Output Parser1**  
    - Type: Langchain Structured Output Parser  
    - Role: Ensures the AI-generated messages adhere to expected structured format.  
    - Configuration: Output schema validation.  
    - Input: AI response from message generation.  
    - Output: Passes messages to update node.  
    - Failure Cases: Parsing errors.

  - **Create Personalised Messages using a Template**  
    - Type: Langchain Chain LLM  
    - Role: Constructs finalized personalized messages ready for outreach.  
    - Configuration: Chain of prompts with template variables.  
    - Input: Structured message data.  
    - Output: Provides message content to Google Sheets update.  
    - Failure Cases: Chain failures, template errors.

  - **Update Personalised Messages to Spreadsheet**  
    - Type: Google Sheets  
    - Role: Writes personalized messages back to a designated spreadsheet for tracking or further use.  
    - Configuration: Spreadsheet ID, range, and append/update mode with credentials.  
    - Input: Personalized messages.  
    - Output: Triggers cleanup node.  
    - Failure Cases: Write permission errors, API limits.

---

#### 2.5 Cleanup

- **Overview:**  
  Deletes temporary URL data from the spreadsheet to maintain cleanliness and avoid stale data.

- **Nodes Involved:**  
  - Delete Temp URL Storage

- **Node Details:**

  - **Delete Temp URL Storage**  
    - Type: Google Sheets  
    - Role: Clears temporary URL storage sheet/tab after processing is complete.  
    - Configuration: Uses Google Sheets credential and delete or clear range command.  
    - Input: Triggered after updating messages to the spreadsheet.  
    - Output: Workflow end.  
    - Failure Cases: Sheet access or permission errors, partial deletes.

---

### 3. Summary Table

| Node Name                           | Node Type                           | Functional Role                                  | Input Node(s)                   | Output Node(s)                      | Sticky Note                            |
|-----------------------------------|-----------------------------------|-------------------------------------------------|--------------------------------|-----------------------------------|--------------------------------------|
| Schedule Trigger                  | Schedule Trigger                  | Initiates workflow periodically                  | None                           | Fetch URL from Spreadsheet         |                                      |
| Fetch URL from Spreadsheet        | Google Sheets                    | Reads LinkedIn URLs from spreadsheet             | Schedule Trigger               | Temp URL Storage                  |                                      |
| Temp URL Storage                 | Google Sheets                    | Temporarily stores URLs for processing           | Fetch URL from Spreadsheet     | POST to Phantombuster API        |                                      |
| POST to Phantombuster API         | HTTP Request                    | Sends scraping job request to PhantomBuster API | Temp URL Storage               | Wait                            |                                      |
| Wait                             | Wait                            | Pauses to allow PhantomBuster to complete jobs  | POST to Phantombuster API      | GET from Phantombuster API       |                                      |
| GET from Phantombuster API        | HTTP Request                    | Polls for scraping results                        | Wait                          | Azure OpenAI Chat Model          |                                      |
| Azure OpenAI Chat Model           | Azure OpenAI Chat Model (GPT-4) | Parses raw PhantomBuster output                   | GET from Phantombuster API     | Structured Output Parser         |                                      |
| Structured Output Parser          | Langchain Output Parser          | Converts AI response to structured JSON          | Azure OpenAI Chat Model        | Extract URL from Output using GPT-4 |                                      |
| Extract URL from Output using GPT-4 | Langchain Agent (GPT-4)          | Extracts LinkedIn URLs from structured data      | Structured Output Parser       | GET LinkedIn Data from URL       |                                      |
| GET LinkedIn Data from URL        | HTTP Request                    | Retrieves detailed LinkedIn profile data          | Extract URL from Output using GPT-4 | Azure OpenAI Chat Model1        |                                      |
| Azure OpenAI Chat Model1          | Azure OpenAI Chat Model (GPT-4) | Generates personalized messages                   | GET LinkedIn Data from URL     | Structured Output Parser1        |                                      |
| Structured Output Parser1         | Langchain Output Parser          | Parses message generation AI output               | Azure OpenAI Chat Model1       | Create Personalised Messages using a Template | Should break line                   |
| Create Personalised Messages using a Template | Langchain Chain LLM             | Finalizes personalized messages                   | Structured Output Parser1      | Update Personalised Messages to Spreadsheet |                                      |
| Update Personalised Messages to Spreadsheet | Google Sheets                    | Updates spreadsheet with personalized messages    | Create Personalised Messages using a Template | Delete Temp URL Storage         |                                      |
| Delete Temp URL Storage           | Google Sheets                    | Cleans temporary URL storage                       | Update Personalised Messages to Spreadsheet | None                         |                                      |
| Sticky Note                      | Sticky Note                     | No content                                        | None                           | None                            |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Set a schedule (e.g., daily or hourly) to start the workflow automatically.

2. **Add Google Sheets node “Fetch URL from Spreadsheet”:**  
   - Configure with Google Sheets credentials.  
   - Select the spreadsheet and worksheet containing LinkedIn profile URLs.  
   - Read the relevant range or column with URLs.

3. **Add Google Sheets node “Temp URL Storage”:**  
   - Configure to write fetched URLs into a temporary sheet/tab for staging.  
   - Set append or overwrite mode as required.

4. **Connect “Fetch URL from Spreadsheet” output to “Temp URL Storage” input.**

5. **Add HTTP Request node “POST to Phantombuster API”:**  
   - Configure with PhantomBuster API endpoint for starting scraping.  
   - Add authentication header with PhantomBuster API key.  
   - Compose JSON body including URLs from “Temp URL Storage”.  
   - Connect “Temp URL Storage” output to this node.

6. **Add Wait node:**  
   - Set wait time (e.g., a few minutes) to allow PhantomBuster scraping to complete.  
   - Connect “POST to Phantombuster API” output to “Wait”.

7. **Add HTTP Request node “GET from Phantombuster API”:**  
   - Configure with endpoint to retrieve scraping results using job ID.  
   - Include proper authentication.  
   - Connect “Wait” output to this node.

8. **Add Azure OpenAI Chat Model node:**  
   - Configure with Azure OpenAI GPT-4 credentials and model.  
   - Design prompt to parse PhantomBuster output into structured data.  
   - Connect “GET from Phantombuster API” output to this node.

9. **Add Structured Output Parser node:**  
   - Define JSON schema to extract URLs cleanly from AI output.  
   - Connect Azure OpenAI Chat Model output to this node.

10. **Add Langchain Agent node “Extract URL from Output using GPT-4”:**  
    - Configure with GPT-4 credentials.  
    - Use prompt to validate and extract clean LinkedIn URLs from parsed data.  
    - Connect Structured Output Parser output to this node.

11. **Add HTTP Request node “GET LinkedIn Data from URL”:**  
    - Configure to retrieve detailed profile data from extracted URLs.  
    - Enable retry on failure for robustness.  
    - Connect Extract URL node output to this node.

12. **Add Azure OpenAI Chat Model1 node:**  
    - Setup GPT-4 model with a personalized message template prompt.  
    - Connect “GET LinkedIn Data from URL” output here.

13. **Add Structured Output Parser1 node:**  
    - Define schema to parse AI-generated messages.  
    - Connect Azure OpenAI Chat Model1 output to this node.

14. **Add Langchain Chain LLM node “Create Personalised Messages using a Template”:**  
    - Configure chain with message template and variables.  
    - Connect Structured Output Parser1 output to this node.

15. **Add Google Sheets node “Update Personalised Messages to Spreadsheet”:**  
    - Configure to update a spreadsheet with personalized messages.  
    - Connect message creation node output to this node.

16. **Add Google Sheets node “Delete Temp URL Storage”:**  
    - Configure to clear the temporary URL storage sheet after processing.  
    - Connect Update Messages node output to this node.

17. **Verify all connections flow as: Schedule Trigger → Fetch URL → Temp Storage → POST PhantomBuster → Wait → GET PhantomBuster → Azure OpenAI Chat Model → Structured Output Parser → Extract URL → GET LinkedIn Data → Azure OpenAI Chat Model1 → Structured Output Parser1 → Create Messages → Update Spreadsheet → Delete Temp Storage.**

18. **Set up all credentials properly:**  
    - Google Sheets OAuth2 credentials.  
    - PhantomBuster API key in HTTP Request nodes.  
    - Azure OpenAI GPT-4 credentials for all Azure OpenAI nodes.

19. **Test each node independently before running the full workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| The node "Structured Output Parser1" includes a note: "should break line" indicating formatting need. | Suggests that output parsing expects newline-separated content for clarity.                                        |
| Workflow leverages PhantomBuster API for LinkedIn scraping: https://phantombuster.com/api            | Official PhantomBuster API documentation for job management and scraping details.                                 |
| Uses Azure OpenAI GPT-4 model through Langchain nodes: https://learn.microsoft.com/en-us/azure/cognitive-services/openai/ | Azure OpenAI service info and best practices for prompt engineering and structured output parsing.                |
| Google Sheets nodes require OAuth2 with correct scopes for reading/writing sheets: https://developers.google.com/sheets/api/guides/authorizing | Google Sheets API authorization details for seamless integration.                                                 |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.