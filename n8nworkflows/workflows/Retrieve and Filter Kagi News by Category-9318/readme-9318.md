Retrieve and Filter Kagi News by Category

https://n8nworkflows.xyz/workflows/retrieve-and-filter-kagi-news-by-category-9318


# Retrieve and Filter Kagi News by Category

### 1. Workflow Overview

This workflow, titled **"Retrieve and Filter Kagi News by Category"**, is designed to fetch the latest news stories from the Kagi News API filtered by a user-specified category. It is intended primarily as a reusable sub-workflow that can be triggered with an input category name, and returns news stories from that category, extracting only relevant details (such as titles) for further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives the category name as input from a calling workflow or manual trigger.
- **1.2 Retrieve Available Categories:** Calls the Kagi API to get the list of all news categories.
- **1.3 Filter Category:** Splits and filters the retrieved categories to find the one matching the input.
- **1.4 Retrieve Stories:** Requests the latest news stories for the matched category.
- **1.5 Extract Story Details:** Splits the stories and extracts only essential fields (title) for output.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block is the entry point for the workflow. It accepts a single input parameter — the news category to filter by.

- **Nodes Involved:**  
  - Start (Execute Workflow Trigger)

- **Node Details:**  
  - **Start**  
    - Type: `executeWorkflowTrigger` (trigger node to start workflow execution)  
    - Configuration: Accepts an input parameter named `category` (string)  
    - Input: External trigger or manual start with parameter `category` (e.g., "World")  
    - Output: Passes the input JSON object downstream  
    - Edge Cases: Missing or invalid category input may lead to no matching category found downstream  
    - No special version requirements  

---

#### 1.2 Retrieve Available Categories

- **Overview:**  
  This block calls the Kagi News API endpoint to retrieve the latest news categories available.

- **Nodes Involved:**  
  - Get Latest Categories (HTTP Request)  
  - Split Categories (Split Out)

- **Node Details:**  
  - **Get Latest Categories**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://kite.kagi.com/api/batches/latest/categories`  
    - Input: Triggered after Start node  
    - Output: JSON response containing a `categories` array  
    - Edge Cases: HTTP failures (timeouts, 4xx/5xx errors), API changes, empty category list  
    - Version: 4.2  
  - **Split Categories**  
    - Type: Split Out  
    - Configuration: Splits the `categories` array from the previous response into individual items  
    - Input: Output of HTTP Request node  
    - Output: Individual category objects downstream  
    - Edge Cases: Empty input array, malformed data  

---

#### 1.3 Filter Category

- **Overview:**  
  Filters the list of categories to find the one matching the input category name.

- **Nodes Involved:**  
  - Filter by wanted category (Filter)  
  - Limit to 1 category (Limit)

- **Node Details:**  
  - **Filter by wanted category**  
    - Type: Filter node  
    - Configuration: Checks if the category name from split category (`categoryName` field) equals the input `category` from Start node  
    - Key Expression:  
      `{{$json.categoryName}} === {{$('Start').item.json.category}}`  
    - Input: Individual category items from Split Categories  
    - Output: Only the matching category passes  
    - Edge Cases: No match found (results in empty output), case sensitivity (configured as case sensitive)  
  - **Limit to 1 category**  
    - Type: Limit node  
    - Configuration: Limits output to only 1 item to avoid multiple matches  
    - Input: Filter output  
    - Output: Single category object downstream  
    - Edge Cases: No output if filter yields none  

---

#### 1.4 Retrieve Stories

- **Overview:**  
  Fetches the news stories from the matched category by calling the Kagi API with the category ID.

- **Nodes Involved:**  
  - Get Stories in Category (HTTP Request)  
  - Split out stories (Split Out)

- **Node Details:**  
  - **Get Stories in Category**  
    - Type: HTTP Request  
    - Configuration: GET request to `https://kite.kagi.com/api/batches/latest/categories/{{ $json.id }}/stories?lang=en`  
    - Input: Single category object with `id` from Limit node  
    - Output: JSON response containing `stories` array in the selected category  
    - Edge Cases: HTTP errors, empty stories array, malformed data  
    - Version: 4.2  
  - **Split out stories**  
    - Type: Split Out  
    - Configuration: Splits the `stories` array into individual story objects  
    - Input: Output of Get Stories node  
    - Output: Individual story objects downstream  
    - Edge Cases: Empty stories array  

---

#### 1.5 Extract Story Details

- **Overview:**  
  Processes each story object to extract and output only selected fields, such as the title.

- **Nodes Involved:**  
  - Pick only title (Set)

- **Node Details:**  
  - **Pick only title**  
    - Type: Set node  
    - Configuration: Keeps only the `title` field from each story object, discarding other fields  
    - Expression: `title = {{$json.title}}`  
    - Input: Individual story objects from Split out stories  
    - Output: Objects with only the `title` property  
    - Edge Cases: Missing title field in some stories (results in empty or undefined titles)  
    - Version: 3.4  

---

### 3. Summary Table

| Node Name               | Node Type                  | Functional Role                     | Input Node(s)            | Output Node(s)           | Sticky Note                                                                                   |
|-------------------------|----------------------------|-----------------------------------|--------------------------|--------------------------|-----------------------------------------------------------------------------------------------|
| Start                   | executeWorkflowTrigger     | Entry point, accepts input category | -                        | Get Latest Categories     | This is meant to be used in other workflows, but you can manually trigger it by setting a `category` in the `Start` node. |
| Get Latest Categories   | HTTP Request               | Fetch all latest news categories  | Start                    | Split Categories          | ## Get available categories                                                                   |
| Split Categories        | Split Out                  | Split categories array into items | Get Latest Categories    | Filter by wanted category | ## Find the category you want                                                                 |
| Filter by wanted category | Filter                    | Filter category matching input    | Split Categories         | Limit to 1 category       |                                                                                               |
| Limit to 1 category     | Limit                      | Limit to single matching category | Filter by wanted category | Get Stories in Category   |                                                                                               |
| Get Stories in Category | HTTP Request               | Fetch stories in selected category | Limit to 1 category      | Split out stories         | ## Get the stories in that category                                                           |
| Split out stories       | Split Out                  | Split stories array into items    | Get Stories in Category  | Pick only title           |                                                                                               |
| Pick only title         | Set                        | Extract only the title field      | Split out stories        | -                        | ## Extract details I am interested in<br>Here I am only extracting the title, but you can also get the summary, maybe the talking points, and so on. |
| Sticky Note             | Sticky Note                | Documentation and instructions    | -                        | -                        | ## Try it out!<br>This is meant to be used in other workflows, but you can manually trigger it by setting a `category` in the `Start` node.<br><br># How it works<br>- We hit the Kagi News API twice<br>-- once to get the category ID<br>-- and once to get the stories in that category<br>- We then do any changes we need to the output, to pick out just the information we are interested in<br><br># Requirements<br>- Nothing! |
| Sticky Note1            | Sticky Note                | Documentation                    | -                        | -                        | ## Get available categories                                                                   |
| Sticky Note2            | Sticky Note                | Documentation                    | -                        | -                        | ## Find the category you want                                                                 |
| Sticky Note3            | Sticky Note                | Documentation                    | -                        | -                        | ## Get the stories in that category                                                           |
| Sticky Note4            | Sticky Note                | Documentation                    | -                        | -                        | ## Extract details I am interested in                                                         |
| Sticky Note5            | Sticky Note                | Documentation                    | -                        | -                        | Here I am only extracting the title, but you can also get the summary, maybe the talking points, and so on. |
| Sticky Note6            | Sticky Note                | Documentation                    | -                        | -                        | This is the only call to a remote service.                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Start Node**  
   - Type: **Execute Workflow Trigger**  
   - Configure input parameter: Name it `category` (string)  
   - This node serves as the entry point for the workflow and accepts the category name to filter news stories.

2. **Add HTTP Request Node: Get Latest Categories**  
   - Type: **HTTP Request**  
   - Method: GET  
   - URL: `https://kite.kagi.com/api/batches/latest/categories`  
   - No authentication needed  
   - Connect Start node output to this node’s input.

3. **Add Split Out Node: Split Categories**  
   - Type: **Split Out**  
   - Field to split out: `categories` (from the HTTP response JSON)  
   - Connect output of Get Latest Categories node to Split Categories node.

4. **Add Filter Node: Filter by wanted category**  
   - Type: **Filter**  
   - Condition:  
     - Left Value: Expression `{{$json.categoryName}}` (category name from split item)  
     - Operator: Equals  
     - Right Value: Expression `{{$('Start').item.json.category}}` (input category)  
     - Case sensitive: Enabled  
   - Connect Split Categories output to this filter.

5. **Add Limit Node: Limit to 1 category**  
   - Type: **Limit**  
   - No special configuration, default limit 1  
   - Connect Filter output to Limit node.

6. **Add HTTP Request Node: Get Stories in Category**  
   - Type: **HTTP Request**  
   - Method: GET  
   - URL with expression:  
     `https://kite.kagi.com/api/batches/latest/categories/{{ $json.id }}/stories?lang=en`  
   - Connect Limit node output to this HTTP Request node.

7. **Add Split Out Node: Split out stories**  
   - Type: **Split Out**  
   - Field to split out: `stories` (from the HTTP response JSON)  
   - Connect output of Get Stories in Category node to this node.

8. **Add Set Node: Pick only title**  
   - Type: **Set**  
   - Clear all fields and add new field:  
     - Name: `title`  
     - Value: Expression `{{$json.title}}`  
   - Connect Split out stories output to this Set node.

9. **Optional: Add Sticky Notes**  
   - Add sticky notes with the content provided for documentation and clarity at appropriate positions.

10. **Save and Activate Workflow**  
    - This workflow can now be called from other workflows by passing the `category` input parameter.  
    - Alternatively, manually trigger it by specifying a category string (e.g., "World").

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                     | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow is designed as a sub-workflow, intended to be invoked by other workflows with a category parameter. It can also be run manually for testing by setting the category input in the Start node.                                         | Workflow usage note                              |
| The only external API calls are to the Kagi News API endpoints: one to get categories and one to get stories for a category.                                                                                                                    | API usage                                        |
| No authentication or special credentials are required to access the public Kagi News API endpoints used here.                                                                                                                                   | API access detail                                |
| The workflow extracts only the `title` of each story, but it can be extended to include other fields such as summary or talking points by modifying the Set node accordingly.                                                                   | Extension note                                   |
| Case sensitivity is enabled when matching categories; ensure input category strings match exactly the category names returned by the API.                                                                                                       | Important usage note                             |
| For more information about the Kagi News API or examples, refer to official Kagi documentation or community forums (no direct links provided in the workflow).                                                                                   | General resource suggestion                       |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.