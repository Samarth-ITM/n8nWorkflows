Securely Call Google Cloud Run APIs with Service Account Authentication

https://n8nworkflows.xyz/workflows/securely-call-google-cloud-run-apis-with-service-account-authentication-7568


# Securely Call Google Cloud Run APIs with Service Account Authentication

### 1. Workflow Overview

This workflow demonstrates how to securely call Google Cloud Run APIs using service account authentication, specifically by obtaining and using Google ID tokens to authenticate requests. It is designed for use cases where a Google Cloud Run service is configured to require authentication, and calls must be made with a valid bearer token representing a service account.

The workflow is logically divided into the following blocks:

- **1.1 Initialization and Variable Setup**: Define base URLs and paths for the Cloud Run service and prepare example context data.
- **1.2 Authentication Token Acquisition**: Invoke a sub-workflow (“Service Auth”) to obtain a valid Google ID token using a service account.
- **1.3 Context Aggregation and Data Preparation**: Merge authentication data with example context and prepare arrays for batch processing.
- **1.4 Batch Processing and API Calls**: Split array data into individual items, loop over them, and for each item, invoke the Cloud Run API with the acquired bearer token.
- **1.5 Result Handling and Completion**: Collect responses, handle looping, and finalize workflow execution.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Variable Setup

**Overview:**  
Defines service URLs and example context variables. Prepares static data that will be used later for making authenticated requests.

**Nodes Involved:**  
- Execute (Manual Trigger)  
- Set Example Context Fields  
- Vars  

**Node Details:**

- **Execute**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow execution manually.  
  - Configuration: No special parameters.  
  - Inputs: None  
  - Outputs: Triggers next nodes.

- **Set Example Context Fields**  
  - Type: Set  
  - Role: Defines example context variables (`example_context` string and `array_of_soups` array) to illustrate processing with multiple items.  
  - Configuration:  
    - `example_context`: "a very interesting value"  
    - `array_of_soups`: ["ramen", "udon", "chicken noodle", "pho"]  
  - Inputs: Trigger from Execute node  
  - Outputs: Passes context data downstream.

- **Vars**  
  - Type: Set  
  - Role: Holds configuration variables for the Cloud Run service URL and an optional path.  
  - Configuration:  
    - `service_url`: (empty string, to be filled by the user)  
    - `service_path`: (empty string, to be filled by the user)  
  - Inputs: Triggered after Set Example Context Fields  
  - Outputs: Passes URL variables downstream.

**Edge Cases & Potential Failures:**  
- Empty `service_url` or `service_path` will cause failures in HTTP requests later.  
- Manual trigger requires user action to start workflow.

---

#### 2.2 Authentication Token Acquisition

**Overview:**  
Calls a sub-workflow named “Service Auth” to securely obtain or refresh a Google ID token using service account credentials.

**Nodes Involved:**  
- Get Auth (Execute Workflow)  

**Node Details:**

- **Get Auth**  
  - Type: Execute Workflow  
  - Role: Calls the “Service Auth” sub-workflow responsible for generating a Google ID token using a service account.  
  - Configuration:  
    - Inputs passed: `service_url` and `service_path` (from Vars node)  
    - Optional: `id_token` if already available (not used here)  
  - Inputs: From Vars node  
  - Outputs: Returns JSON including `id_token` for authentication  
  - Version: 1.2 (uses defined workflow inputs)  
  - Potential Failures:  
    - Failure in sub-workflow due to invalid credentials or network issues  
    - Missing or invalid service account keys  
    - Token expiration or refresh errors  
  - Sub-workflow Reference: “Templates — Service Auth (sub-workflow)”, workflow ID `3lkAlfsxT2cnJztz`  

---

#### 2.3 Context Aggregation and Data Preparation

**Overview:**  
Merges the obtained ID token with example context data and prepares the array of soups for iteration.

**Nodes Involved:**  
- Collect Context Example (Merge)  
- Split Out  

**Node Details:**

- **Collect Context Example**  
  - Type: Merge  
  - Role: Combines the authentication context (`id_token`) with example context items for subsequent processing.  
  - Configuration:  
    - Mode: Combine  
    - Combine By: All Possible Combinations  
  - Inputs: From Get Auth and Set Example Context Fields  
  - Outputs: Combined items containing `id_token`, `example_context`, and `array_of_soups`  
  - Edge Cases: Expression evaluation errors if keys missing.

- **Split Out**  
  - Type: Split Out Items  
  - Role: Splits the `array_of_soups` array into individual items while preserving all other fields (`id_token`, `service_url`, etc.).  
  - Configuration:  
    - Field to split: `array_of_soups`  
    - Include: All other fields  
    - Destination Field Name: `soup`  
  - Inputs: From Collect Context Example  
  - Outputs: One item per soup string for batch processing  
  - Edge Cases: Empty arrays result in no items passed forward.

---

#### 2.4 Batch Processing and API Calls

**Overview:**  
Processes each item (soup) in batches, looping through each to call the Cloud Run API with the bearer token for authentication.

**Nodes Involved:**  
- Loop Over Items (Split In Batches)  
- Check Auth (Execute Workflow)  
- Cloud Run Request  

**Node Details:**

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Iterates over each split item (each soup) individually for separate API calls.  
  - Configuration: Default batch size (1 item per batch)  
  - Inputs: From Split Out  
  - Outputs: Looped items one-by-one to next nodes  
  - Edge Cases: Empty input results in no iterations.

- **Check Auth**  
  - Type: Execute Workflow  
  - Role: Calls the same “Service Auth” sub-workflow to ensure the ID token is valid before each API call (can reuse token or refresh if needed).  
  - Configuration:  
    - Inputs passed: `id_token`, `service_url`, `service_path` (all extracted from current item)  
  - Inputs: From Loop Over Items  
  - Outputs: Validated `id_token` forwarded to Cloud Run Request  
  - Edge Cases: Token refresh failures; service unavailability.

- **Cloud Run Request**  
  - Type: HTTP Request  
  - Role: Performs the authenticated HTTP request to the Google Cloud Run endpoint using the bearer token for authorization.  
  - Configuration:  
    - URL: Constructed by concatenating `service_url` and `service_path` from the current item  
    - Authentication: HTTP Bearer Auth using the `id_token` (credential named “Bearer Auth account”)  
  - Inputs: From Check Auth  
  - Outputs: Response from the Cloud Run service  
  - Edge Cases: HTTP 401 Unauthorized if token invalid; network timeouts; URL misconfiguration.

---

#### 2.5 Result Handling and Completion

**Overview:**  
Handles workflow continuation and indicates completion after processing all batch items.

**Nodes Involved:**  
- Done (No Operation)  

**Node Details:**

- **Done**  
  - Type: NoOp  
  - Role: Marks the end of the batch processing loop and workflow execution.  
  - Inputs: From Loop Over Items after all batches processed  
  - Outputs: None  
  - Edge Cases: None

---

### 3. Summary Table

| Node Name                | Node Type               | Functional Role                                  | Input Node(s)                    | Output Node(s)               | Sticky Note                                                                                                                           |
|--------------------------|-------------------------|-------------------------------------------------|---------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note1             | Sticky Note             | Setup instructions for Google Cloud Run & Service Account | None                            | None                        | ## Required Setup — Google Cloud Run & Service Account  (Includes Medium article link)                                                |
| Execute                  | Manual Trigger          | Starts the workflow                              | None                            | Set Example Context Fields, Vars |                                                                                                                                    |
| Set Example Context Fields| Set                     | Defines example context variables                | Execute                         | Collect Context Example      |                                                                                                                                    |
| Vars                     | Set                     | Holds service_url and service_path variables     | Execute, Set Example Context Fields | Get Auth                   | #### Var Config - `service_url` and `service_path` base URLs                                                                         |
| Get Auth                 | Execute Workflow        | Calls sub-workflow to obtain ID token            | Vars                            | Collect Context Example      | #### Point To Service Auth (sub-workflow)                                                                                           |
| Collect Context Example   | Merge                   | Combines auth token and example context          | Get Auth, Set Example Context Fields | Split Out                 | ### Merge node — gathers all context. Use Combine mode to add auth context to items                                                  |
| Split Out                | Split Out Items         | Splits array into individual items while keeping context | Collect Context Example          | Loop Over Items              | ### Split Out node — iterate array, keep context                                                                                     |
| Loop Over Items          | Split In Batches        | Iterates over items for batch processing          | Split Out                      | Done, Check Auth             |                                                                                                                                    |
| Check Auth               | Execute Workflow        | Validates or refreshes ID token before API call  | Loop Over Items                 | Cloud Run Request            | #### Point to Service Auth (sub-workflow)                                                                                            |
| Cloud Run Request        | HTTP Request            | Calls Cloud Run API with bearer token             | Check Auth                     | Loop Over Items              | # Running the Workflow Instructions (main sticky note)                                                                               |
| Done                     | NoOp                    | Marks end of workflow processing                   | Loop Over Items                  | None                        |                                                                                                                                    |
| Sticky Note2             | Sticky Note             | Running instructions and troubleshooting          | None                            | None                        | # Running the Workflow (Detailed instructions and what to check after running)                                                      |
| Sticky Note4             | Sticky Note             | Variable configuration explanation                 | None                            | None                        | #### Var Config - Explains service_url and service_path                                                                              |
| Sticky Note5             | Sticky Note             | Explanation for Merge node                          | None                            | None                        | ### Merge node — gathers all context. Use Combine mode                                                                              |
| Sticky Note6             | Sticky Note             | Reference to Service Auth sub-workflow             | None                            | None                        | #### Point to Service Auth (sub-workflow). Medium article link referenced in main sticky note                                         |
| Sticky Note8             | Sticky Note             | Credential setup instructions for Google Service Account | None                            | None                        | ## Extra Configurations for Workflow (How to create Google credentials with service account)                                        |
| Sticky Note9             | Sticky Note             | Explanation for Split Out node                      | None                            | None                        | ### Split Out node — iterate array, keep context                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow.**

2. **Add a Manual Trigger node named “Execute”.**  
   - No special parameters.

3. **Add a Set node named “Set Example Context Fields”.**  
   - Connect “Execute” → “Set Example Context Fields”.  
   - Add fields:  
     - `example_context` (string): `"a very interesting value"`  
     - `array_of_soups` (array): `["ramen", "udon", "chicken noodle", "pho"]`.

4. **Add another Set node named “Vars”.**  
   - Connect “Set Example Context Fields” → “Vars”.  
   - Add fields:  
     - `service_url` (string): leave empty for user to fill with Cloud Run base URL.  
     - `service_path` (string): leave empty or optional path.

5. **Add an Execute Workflow node named “Get Auth”.**  
   - Connect “Vars” → “Get Auth”.  
   - Configure it to call the “Service Auth (sub-workflow)”.  
   - Pass inputs: `service_url` and `service_path` from Vars.  
   - Ensure the sub-workflow is imported and available with workflow ID `3lkAlfsxT2cnJztz`.

6. **Add a Merge node named “Collect Context Example”.**  
   - Connect “Get Auth” → “Collect Context Example” (main input).  
   - Connect “Set Example Context Fields” → “Collect Context Example” (second input).  
   - Set mode to “Combine” and combine by “All Possible Combinations”.

7. **Add a Split Out node named “Split Out”.**  
   - Connect “Collect Context Example” → “Split Out”.  
   - Set operation to “Split Out Items”.  
   - Field to split: `array_of_soups`.  
   - Include all other fields (to keep `id_token`, `service_url`, etc.)  
   - Destination field name: `soup`.

8. **Add a Split In Batches node named “Loop Over Items”.**  
   - Connect “Split Out” → “Loop Over Items”.  
   - Default batch size (1).

9. **Add an Execute Workflow node named “Check Auth”.**  
   - Connect “Loop Over Items” → “Check Auth”.  
   - Call the same “Service Auth (sub-workflow)” as in step 5.  
   - Pass inputs: `id_token`, `service_url`, `service_path` from the current item.

10. **Add an HTTP Request node named “Cloud Run Request”.**  
    - Connect “Check Auth” → “Cloud Run Request”.  
    - Configure URL as expression:  
      `{{$json["service_url"]}}{{$json["service_path"]}}`  
    - Authentication: HTTP Bearer Auth.  
    - Credential: Create a new HTTP Bearer Auth credential named e.g., “Bearer Auth account”.  
      - Token value to be set dynamically from `id_token` passed by “Check Auth”.  
      - In n8n, this can be done by setting the HTTP header manually with expression:  
        Header `Authorization: Bearer {{$json["id_token"]}}` (if dynamic header is needed instead of credential).  
      - Alternatively, configure the credential with a placeholder token and override header in the node parameters.

11. **Connect “Cloud Run Request” → “Loop Over Items”** (to continue batch processing).

12. **Add a NoOp node named “Done”.**  
    - Connect “Loop Over Items” → “Done” (this is the completion path after all batches are processed).

13. **Add Sticky Notes** to provide explanations and instructions as per the original workflow (optional but recommended for clarity).

14. **Create and configure Google credentials** for the “Service Auth (sub-workflow)” with:  
    - Authentication method: Service Account  
    - Service account email: from `.json private key`  
    - Private key: full private key block with BEGIN/END  
    - Enable relevant Google APIs in Google Cloud Console.  
    - Share resources with the service account email for access.

15. **Test the workflow** by filling in `service_url` and optionally `service_path`, then execute manually.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Configure a Google Cloud Run service with "Require authentication" enabled and a service account with Cloud Run Invoker role. | Sticky Note1 content and project prerequisites.                                                                 |
| Detailed walkthrough available: "Build a Secure Google Cloud Run API, Then Call It from n8n (Free Tier)" article by Marco Codes. | https://medium.com/@marcocodes/build-a-secure-google-cloud-run-api-then-call-it-from-n8n-88c03291a95f             |
| Credential setup requires copying `client_email` and `private_key` from your Google service account JSON key.  | Sticky Note8 content.                                                                                            |
| The “Service Auth (sub-workflow)” handles token generation and refresh logic; refer to it for authentication internals. | Sub-workflow ID referenced: `3lkAlfsxT2cnJztz`.                                                                 |
| The workflow uses batch splitting and merging techniques to maintain context across asynchronous calls.         | Sticky Notes 5 and 9 give explanations for Merge and Split Out nodes respectively.                              |
| If workflow execution fails, check outputs of “Bearer Token Request” and Cloud Run service logs for diagnostics.| Sticky Note2 instructions on troubleshooting.                                                                   |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with current content policies and contains no illegal or protected material. All handled data is legal and public.