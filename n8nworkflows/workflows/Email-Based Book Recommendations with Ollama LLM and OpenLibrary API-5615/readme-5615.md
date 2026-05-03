Email-Based Book Recommendations with Ollama LLM and OpenLibrary API

https://n8nworkflows.xyz/workflows/email-based-book-recommendations-with-ollama-llm-and-openlibrary-api-5615


# Email-Based Book Recommendations with Ollama LLM and OpenLibrary API

### 1. Workflow Overview

This workflow automates the process of providing personalized book recommendations via email. It is triggered by incoming emails requesting book suggestions, uses a large language model (Ollama LLM) to extract the types or genres of books requested, and queries the OpenLibrary API to find relevant books. The workflow handles cases where no matching books are found, retrieves detailed book information, formats this data into a user-friendly email, and sends the recommendation back to the requester.

**Target Use Cases:**
- Automated book recommendation services via email
- Reading clubs or newsletters providing curated book suggestions
- Educational or library bots assisting users in finding books based on genre or interest

**Logical Blocks:**

- **1.1 Email Input Processing:** Receiving book request emails and extracting requested book types using the Ollama language model.
- **1.2 Book Search Query and Validation:** Creating search queries, querying the OpenLibrary API, and validating responses.
- **1.3 Book Data Retrieval and Preparation:** Selecting a book, retrieving summaries and detailed information, and preparing the data.
- **1.4 Output Generation and Email Dispatch:** Formatting the book information, enhancing it for readability, generating email content, and sending the recommendation.
- **1.5 Error Handling:** Managing cases where no books are found and notifying the user accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Input Processing

**Overview:**  
This block listens for incoming emails requesting book recommendations, extracts the plain text content, and uses the Ollama LLM to identify the types or genres of books mentioned in the message.

**Nodes Involved:**  
- Email Trigger – Book Request  
- Ollama Model  
- Analyze Email with Ollama

**Node Details:**  

- **Email Trigger – Book Request**  
  - Type: IMAP Email Trigger  
  - Role: Watches a configured email inbox for new messages.  
  - Configuration: Uses IMAP credentials to connect to the email server; no additional filters specified.  
  - Inputs: None (trigger node)  
  - Outputs: Email data including plain text content.  
  - Potential Failures: Authentication errors, connection timeouts, empty or malformed emails.

- **Ollama Model**  
  - Type: Language Model (LLM) node using Ollama API  
  - Role: Provides the underlying LLM for the agent node.  
  - Configuration: Uses llama3.2-16000:latest model with default options; authenticated via Ollama API credentials.  
  - Inputs: None directly; connected to the Agent node.  
  - Outputs: Model inference results.  
  - Potential Failures: API authentication issues, model unavailability, rate limiting.

- **Analyze Email with Ollama**  
  - Type: Langchain Agent Node  
  - Role: Uses the Ollama LLM to parse the email plain text and extract book genres/types requested.  
  - Configuration:  
    - Input text expression: `{{$json.textPlain}}` (email plain text)  
    - System message instructs the LLM to list only the types of books mentioned, lowercase, one per line, no headers or symbols.  
  - Inputs: Email content from trigger  
  - Outputs: Extracted list of book types for search query  
  - Edge Cases: LLM misinterpretation, empty or ambiguous email content, expression errors in text extraction.

---

#### 1.2 Book Search Query and Validation

**Overview:**  
This block creates the search query based on extracted book types, calls the OpenLibrary API to find books in the specified subject, and checks if any results are returned.

**Nodes Involved:**  
- Create Book Search Query  
- Call Book Search API  
- Check API Response  
- Handle No Book Found (error path)

**Node Details:**  

- **Create Book Search Query**  
  - Type: Set Node  
  - Role: Prepares the subject-name parameter for the API call by assigning the extracted book types.  
  - Configuration: Sets `subject-name` to the output of the previous node (`$json.output`).  
  - Inputs: Output of Analyze Email with Ollama  
  - Outputs: JSON with `subject-name` for API usage.  
  - Edge Cases: Empty or incorrect subject names.

- **Call Book Search API**  
  - Type: HTTP Request  
  - Role: Queries OpenLibrary subjects API for books in the specified subject.  
  - Configuration:  
    - URL: `https://openlibrary.org/subjects/{{ $json.subject-name }}`  
    - Query parameter: `limit=0` (returns metadata including work_count)  
  - Inputs: Search query from Set node  
  - Outputs: API response containing metadata about works available under the subject.  
  - Edge Cases: Network errors, invalid subjects, API downtime.

- **Check API Response**  
  - Type: If Node  
  - Role: Validates whether any books (work_count) were found for the subject.  
  - Configuration: Checks if `work_count` > 0 from the API response.  
  - Inputs: API response  
  - Outputs:  
    - True path: Proceed to book selection  
    - False path: Trigger no-book handling  
  - Edge Cases: Missing or malformed work_count field.

- **Handle No Book Found**  
  - Type: Email Send  
  - Role: Sends an apology email notifying no books were found for the requested subject.  
  - Configuration:  
    - Recipient and sender emails are statically set (e.g., john.doe@example.com)  
    - Email body includes a link to OpenLibrary subjects for user reference.  
  - Inputs: False path from Check API Response  
  - Outputs: None  
  - Edge Cases: SMTP authentication errors, email delivery failures.

---

#### 1.3 Book Data Retrieval and Preparation

**Overview:**  
Selects a random book from the search results, retrieves its summary and detailed information via the OpenLibrary API, and prepares the data for formatting.

**Nodes Involved:**  
- Check Book Name  
- Extract Book Summary  
- Wait for Summary Response  
- Retrieve Book Details  
- Format Book Data

**Node Details:**  

- **Check Book Name**  
  - Type: Code Node (JavaScript)  
  - Role: Randomly selects one book index from the total `work_count` to retrieve.  
  - Configuration:  
    - Retrieves `work_count` from input JSON  
    - Generates a random integer between 1 and `work_count` inclusive  
    - Stores result in `retrieve_book` field for downstream use.  
  - Inputs: Output of Check API Response (true path)  
  - Outputs: JSON with `retrieve_book` index  
  - Edge Cases: Zero or undefined work_count, random number generation errors.

- **Extract Book Summary**  
  - Type: HTTP Request  
  - Role: Fetches a list of works in the subject with limit=1 and offset=`retrieve_book` to get a single book summary.  
  - Configuration:  
    - URL: `http://openlibrary.org/subjects/{{$json["name"]}}.json`  
    - Query params: `limit=1`, `offset={{$json["retrieve_book"]}}`, `detail=true`  
  - Inputs: Output of Check Book Name  
  - Outputs: JSON containing book summary and metadata.  
  - Edge Cases: Invalid offsets, API errors, missing fields.

- **Wait for Summary Response**  
  - Type: Wait Node  
  - Role: Ensures asynchronous API response is ready before next step.  
  - Configuration: Standard wait; webhook ID assigned for response synchronization.  
  - Inputs: Output of Extract Book Summary  
  - Outputs: Passes data forward without modification.  
  - Edge Cases: Timeout, webhook failures.

- **Retrieve Book Details**  
  - Type: HTTP Request  
  - Role: Fetches detailed information about the selected book via its unique key.  
  - Configuration:  
    - URL built dynamically: `http://openlibrary.org{{$node["Extract Book Summary"].json["works"][0]["key"]}}.json`  
    - Query param `limit=1` (though likely not needed here)  
  - Inputs: Output of Wait for Summary Response  
  - Outputs: Detailed book JSON including description.  
  - Edge Cases: Missing keys, network errors.

- **Format Book Data**  
  - Type: Set Node  
  - Role: Extracts and sets key book fields for email generation.  
  - Configuration: Sets `authors`, `title`, `description`, and `URL` fields from previous nodes.  
  - Inputs: Output of Retrieve Book Details  
  - Outputs: Clean JSON with book data ready for formatting.  
  - Edge Cases: Missing or malformed description or author data.

---

#### 1.4 Output Generation and Email Dispatch

**Overview:**  
Enhances author data with hyperlinks, generates personalized email content with book details, and sends the recommendation email to the user.

**Nodes Involved:**  
- Enhance Data with Code  
- Generate Email Content  
- Send Recommendation Email

**Node Details:**  

- **Enhance Data with Code**  
  - Type: Code Node (JavaScript)  
  - Role: Converts author objects into HTML hyperlinks to their OpenLibrary pages and joins them as a comma-separated string.  
  - Configuration:  
    - Maps authors array to `<a href="...">Author Name</a>`  
    - Joins with commas  
    - Replaces authors field with this HTML string.  
  - Inputs: Output of Format Book Data  
  - Outputs: JSON with enhanced `authors` field.  
  - Edge Cases: Empty author list, unexpected data structures.

- **Generate Email Content**  
  - Type: Set Node  
  - Role: Creates email subject and HTML body content using the formatted book data.  
  - Configuration:  
    - Subject: `"Book Recommendation: [Book Title]"`  
    - Body: HTML with clickable title, author links, and description.  
  - Inputs: Output of Enhance Data with Code  
  - Outputs: JSON containing `msgSubject` and `msgBody` for email node.  
  - Edge Cases: Missing titles or descriptions.

- **Send Recommendation Email**  
  - Type: Email Send Node  
  - Role: Sends the recommendation email to the recipient.  
  - Configuration:  
    - Subject and HTML body from previous node  
    - Recipient and sender emails statically set (e.g., abc@gmail.com and xyz@gmail.com)  
    - Uses SMTP credentials.  
  - Inputs: Output of Generate Email Content  
  - Outputs: None  
  - Edge Cases: SMTP authentication errors, email delivery failures.

---

#### 1.5 Error Handling

**Overview:**  
Manages scenarios when no matching books are found for the requested subject by sending a notification email with guidance.

**Nodes Involved:**  
- Handle No Book Found (also documented above in 1.2)

**Node Details:**  
Described in block 1.2.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                        | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                         |
|----------------------------|---------------------------------|-------------------------------------|------------------------------|------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Email Trigger – Book Request| IMAP Email Trigger              | Receives incoming book request emails| None                         | Analyze Email with Ollama     | ## Email Input Processing: Triggers on new book request emails and uses Ollama LLM to extract user intent.          |
| Ollama Model               | Langchain LLM Ollama Model      | Provides LLM for intent extraction  | None                         | Analyze Email with Ollama     |                                                                                                                     |
| Analyze Email with Ollama  | Langchain Agent Node            | Extracts book types from email text | Email Trigger, Ollama Model  | Create Book Search Query      |                                                                                                                     |
| Create Book Search Query   | Set Node                       | Prepares subject name for API call  | Analyze Email with Ollama     | Call Book Search API          | ## Book Data Retrieval: Generates a search query, fetches book data (title, summary, details) via API, validates.   |
| Call Book Search API       | HTTP Request                   | Queries OpenLibrary for books by subject | Create Book Search Query      | Check API Response            |                                                                                                                     |
| Check API Response         | If Node                       | Checks if books found (work_count>0)| Call Book Search API          | Check Book Name, Handle No Book Found |                                                                                                                     |
| Handle No Book Found       | Email Send                    | Sends apology when no books found   | Check API Response (false)    | None                         |                                                                                                                     |
| Check Book Name            | Code Node (JS)                | Randomly selects book index          | Check API Response (true)     | Extract Book Summary          |                                                                                                                     |
| Extract Book Summary       | HTTP Request                  | Retrieves book summary data          | Check Book Name               | Wait for Summary Response     | ## Data Preparation: Extracts summary, ensures data readiness, gathers additional book details.                     |
| Wait for Summary Response  | Wait Node                    | Synchronizes async summary response | Extract Book Summary          | Retrieve Book Details         |                                                                                                                     |
| Retrieve Book Details      | HTTP Request                  | Fetches detailed book info           | Wait for Summary Response     | Format Book Data              |                                                                                                                     |
| Format Book Data           | Set Node                     | Extracts and formats key book fields | Retrieve Book Details         | Enhance Data with Code        |                                                                                                                     |
| Enhance Data with Code     | Code Node (JS)                | Converts authors to HTML hyperlinks  | Format Book Data              | Generate Email Content        | ## Output Generation: Formats data, refines it with code, creates personalized email, sends to user.                |
| Generate Email Content     | Set Node                     | Generates email subject and body    | Enhance Data with Code        | Send Recommendation Email     |                                                                                                                     |
| Send Recommendation Email | Email Send                   | Sends personalized book recommendation email | Generate Email Content        | None                         |                                                                                                                     |
| Sticky Note                | Sticky Note                  | Workflow overview note               | None                         | None                         | ## Overview: This workflow triggers on book request emails, uses Ollama LLM to extract intent, queries API, handles errors, formats data, sends email. Ideal for automated book suggestions. Great for newsletter, reading clubs, educational bots. |
| Sticky Note1               | Sticky Note                  | How to run instructions              | None                         | None                         | ## How to Run: 1. Valid inbox, 2. Ollama model setup, 3. Book API credentials, 4. Execute or activate, 5. Test email, 6. Check sent mailbox. Tip: customize parser or API call. |
| Sticky Note2               | Sticky Note                  | Email input processing summary       | None                         | None                         | ## Email Input Processing: Triggers on new book request emails and uses Ollama LLM to extract user intent.          |
| Sticky Note3               | Sticky Note                  | Output generation summary            | None                         | None                         | ## Output Generation: Formats data, refines it with code, creates personalized email, and sends it to the user.     |
| Sticky Note4               | Sticky Note                  | Book data retrieval summary          | None                         | None                         | ## Book Data Retrieval: Generates a search query, fetches book data (title, summary, details) via API, validates.   |
| Sticky Note5               | Sticky Note                  | Data preparation summary             | None                         | None                         | ## Data Preparation: Extracts summary, ensures data readiness, gathers additional book details.                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Email Trigger Node**  
   - Type: Email Read IMAP  
   - Credentials: Configure valid IMAP credentials for the monitored inbox  
   - Parameters: Default options, no filters  
   - Position: Start of workflow

2. **Add Ollama Model Node**  
   - Type: Langchain LLM - Ollama  
   - Credentials: Provide Ollama API credentials  
   - Parameters: Model set to `llama3.2-16000:latest`, default options

3. **Add Analyze Email with Ollama Node**  
   - Type: Langchain Agent Node  
   - Input: Email plain text (`{{$json.textPlain}}`) from Email Trigger  
   - Parameters:  
     - Prompt Type: Define  
     - System Message:  
       ```
       You are a helper that identifies and lists only the types of books mentioned in the message.

       Which types of books are being asked for in this message?

       Reply with the types in lowercase, one on each line. Don’t include anything else — no headers, labels, or symbols.

       Input:
       {{ $json.textPlain }}
       ```
   - Connect Ollama Model node as the language model provider

4. **Add Create Book Search Query Node**  
   - Type: Set Node  
   - Parameters: Set string field `subject-name` to `{{$json.output}}` (from Analyze Email with Ollama)  
   - Connect from Analyze Email node

5. **Add Call Book Search API Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `https://openlibrary.org/subjects/{{$json.subject-name}}`  
     - Query Parameter: `limit=0` (default metadata only)  
   - Connect from Create Book Search Query node

6. **Add Check API Response Node**  
   - Type: If Node  
   - Parameters: Check if `{{$node["Call Book Search API"].json["work_count"]}} > 0`  
   - Connect from Call Book Search API node

7. **Add Handle No Book Found Node**  
   - Type: Email Send  
   - Credentials: Configure SMTP credentials for sending email  
   - Parameters:  
     - To: e.g., `john.doe@example.com` (replace with dynamic email if needed)  
     - From: e.g., `john.doe@example.com`  
     - Subject: `Book not found in {{$node["Check API Response"].json["name"]}}`  
     - HTML Body: Apology message with link to OpenLibrary subjects  
   - Connect from Check API Response false output

8. **Add Check Book Name Node**  
   - Type: Code Node (JavaScript)  
   - Parameters:  
     ```js
     var book_count = items[0].json.work_count;
     var retrieve_book = Math.floor(Math.random() * book_count) + 1;
     items[0].json.retrieve_book = retrieve_book;
     return items;
     ```  
   - Connect from Check API Response true output

9. **Add Extract Book Summary Node**  
   - Type: HTTP Request  
   - Parameters:  
     - URL: `http://openlibrary.org/subjects/{{$json["name"]}}.json`  
     - Query Parameters: `limit=1`, `offset={{$json["retrieve_book"]}}`, `detail=true`  
   - Connect from Check Book Name node

10. **Add Wait for Summary Response Node**  
    - Type: Wait Node  
    - Parameters: Default; webhook ID auto-generated  
    - Connect from Extract Book Summary node

11. **Add Retrieve Book Details Node**  
    - Type: HTTP Request  
    - Parameters:  
      - URL: `http://openlibrary.org{{$node["Extract Book Summary"].json["works"][0]["key"]}}.json`  
      - Query Parameter: `limit=1`  
    - Connect from Wait for Summary Response node

12. **Add Format Book Data Node**  
    - Type: Set Node  
    - Parameters: Set fields:  
      - `authors`: `{{$node["Extract Book Summary"].json["works"][0]["authors"]}}`  
      - `title`: `{{$node["Extract Book Summary"].json["works"][0]["title"]}}`  
      - `description`: `{{$node["Retrieve Book Details"].json["description"]["value"]}}`  
      - `URL`: `https://openlibrary.org{{$node["Extract Book Summary"].json["works"][0]["key"]}}`  
    - Connect from Retrieve Book Details node

13. **Add Enhance Data with Code Node**  
    - Type: Code Node (JavaScript)  
    - Parameters:  
      ```js
      var arrAuthors = items[0].json.authors;
      var arrNames = arrAuthors.map(function(author) {
        return "<a href=\"https://openlibrary.org" + author['key'] + "\">" + author['name'] + "</a>";
      });
      items[0].json.authors = arrNames.join(", ");
      return items;
      ```  
    - Connect from Format Book Data node

14. **Add Generate Email Content Node**  
    - Type: Set Node  
    - Parameters:  
      - `msgSubject`: `Book Recommendation: {{$node["Enhance Data with Code"].json["title"]}}`  
      - `msgBody`:  
        ```html
        <H2><a href="{{$node["Enhance Data with Code"].json["URL"]}}">{{$node["Enhance Data with Code"].json["title"]}}</a></H2>
        <p><em>By {{$node["Enhance Data with Code"].json["authors"]}}</em><br>
        {{$node["Enhance Data with Code"].json["description"]}}</p>
        ```  
    - Connect from Enhance Data with Code node

15. **Add Send Recommendation Email Node**  
    - Type: Email Send  
    - Credentials: Configure SMTP credentials  
    - Parameters:  
      - To: e.g., `abc@gmail.com` (replace with dynamic email if needed)  
      - From: e.g., `xyz@gmail.com`  
      - Subject: `{{$node["Generate Email Content"].json["msgSubject"]}}`  
      - HTML Body: `{{$node["Generate Email Content"].json["msgBody"]}}`  
    - Connect from Generate Email Content node

16. **(Optional) Add Sticky Notes**  
    - To document workflow overview, running instructions, and block explanations as per user needs.

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Workflow triggered by email requests uses Ollama LLM for intent extraction and OpenLibrary API for book data retrieval.   | Workflow overview sticky note in the original workflow.                                                  |
| How to Run: Connect valid inbox, configure Ollama and SMTP credentials, activate workflow, test with sample emails.       | Sticky note with stepwise run instructions.                                                              |
| OpenLibrary Subjects available at https://openlibrary.org/subjects — useful for customizing or testing subject queries.   | Link provided in apology email node and sticky notes.                                                   |
| Ollama LLM model used: llama3.2-16000:latest — ensure API availability and credential correctness for smooth operation.    | Node configuration details.                                                                              |
| SMTP credentials must be valid for email sending; consider dynamic recipient fields for production use.                    | Send email nodes in error handling and recommendation sending.                                          |
| Random book selection mitigates bias but could be improved with user preferences or advanced filtering.                    | Code node logic selecting a random book from API results.                                                |

---

This structured documentation fully describes the workflow design, logic, and configuration, enabling reproduction, modification, and troubleshooting for advanced users and automation agents alike.