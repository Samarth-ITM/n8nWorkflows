Generate Filtered Calendar Views in Baserow with External Calendar Integration

https://n8nworkflows.xyz/workflows/generate-filtered-calendar-views-in-baserow-with-external-calendar-integration-8485


# Generate Filtered Calendar Views in Baserow with External Calendar Integration

---
### 1. Workflow Overview

This workflow automates the creation of personalized calendar views in Baserow, based on specified date fields and filters. It is designed to generate filtered calendar views for multiple entities (e.g., employees, customers, products) by dynamically creating views, applying filters, and configuring visual decorations. The output views are shareable via iCal (.ics) links, enabling integration with external calendar applications like Outlook or Google Calendar.

**Target Use Cases:**

- Task management with deadlines per staff member
- Customer appointment scheduling
- Inventory or delivery date tracking per supplier

**Logical Blocks:**

- **1.1 Input Reception and Credential Setup:** Manual trigger to start, setting user credentials and API host.
- **1.2 Authentication Token Generation:** Creation of JWT token for authenticated API requests.
- **1.3 Configuration of Table and Field IDs:** Setting database, table, and field identifiers, including the JWT token.
- **1.4 Data Retrieval:** Fetching all records from the filter table (e.g., list of employees or customers).
- **1.5 Calendar View Creation:** For each filter record, create a new calendar view in Baserow.
- **1.6 Filter Application:** Apply filter to calendar view to show only relevant records.
- **1.7 Visual Decoration:** Assign background color or decoration based on a single select field.
- **1.8 Sharing and URL Update:** Make the calendar view public (generating an iCal feed) and update records with URLs to the view and iCal feed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Credential Setup

- **Overview:** Starts the workflow manually and sets Baserow user credentials and API host.
- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Set Baserow credentials  
  - Sticky Note1 (authentication instructions)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type:* Manual Trigger  
    - *Role:* Entry point to start the workflow manually.  
    - *Config:* No parameters required.  
    - *Connections:* Outputs to “Set Baserow credentials”.  
    - *Failure Cases:* None (manual trigger).

  - **Set Baserow credentials**  
    - *Type:* Set  
    - *Role:* Stores Baserow username, password, and API host as workflow variables.  
    - *Config:* User must replace placeholders `<your_username>` and `<your_password>` with actual credentials. API host defaults to `https://api.baserow.io`.  
    - *Key Variables:*  
      - Baserow user (email)  
      - Baserow password  
      - API host URL  
    - *Connections:* Outputs to “Create a token”.  
    - *Failure Cases:* Missing or invalid credentials will cause token creation failure.  
    - *Sticky Note:* Authentication instructions reminder.

  - **Sticky Note1**  
    - *Purpose:* Instruction to replace the username and password placeholders.  

#### 2.2 Authentication Token Generation

- **Overview:** Generates a JWT authentication token from the user credentials to authenticate subsequent API calls.
- **Nodes Involved:**  
  - Create a token  

- **Node Details:**

  - **Create a token**  
    - *Type:* HTTP Request  
    - *Role:* POST request to Baserow API’s `/api/user/token-auth/` endpoint to get JWT token.  
    - *Config:*  
      - URL built dynamically from API host.  
      - Request body includes email and password from previous node.  
      - Sends JSON body.  
    - *Expressions:* Uses `{{$json["Baserow user"]}}` and `{{$json["Baserow password"]}}` for request data.  
    - *Connections:* Outputs to “Set table and field ids”.  
    - *Failure Cases:*  
      - Invalid credentials: 401 Unauthorized.  
      - Network or API errors: timeout, host unreachable.  
      - JSON parsing errors if response format changes.  

#### 2.3 Configuration of Table and Field IDs

- **Overview:** Stores JWT token and configures the IDs of database, tables, and fields used in the workflow.
- **Nodes Involved:**  
  - Set table and field ids  
  - Sticky Note2 (field and table description)  

- **Node Details:**

  - **Set table and field ids**  
    - *Type:* Set  
    - *Role:* Assigns JWT token and numeric/string identifiers needed for API requests.  
    - *Config:*  
      - JWT Token is formatted as `JWT {{ $json.access_token }}` from previous node.  
      - IDs for Date table, Date field, Filter field, Status field, Filter table, Database ID, Ics field, and View link field are set here.  
    - *Connections:* Outputs to “Get all records from filter table”.  
    - *Failure Cases:* Missing or invalid IDs cause API calls downstream to fail.  
    - *Sticky Note:* Explains required tables and fields for the workflow.

  - **Sticky Note2**  
    - *Purpose:* Definitions and explanations of each ID and field used in the workflow.

#### 2.4 Data Retrieval

- **Overview:** Retrieves all records from the filter table (e.g., employees or customers) to iterate over and create personalized calendar views.
- **Nodes Involved:**  
  - Get all records from filter table  

- **Node Details:**

  - **Get all records from filter table**  
    - *Type:* Baserow node (API integration)  
    - *Role:* Fetches all records from the configured filter table.  
    - *Config:*  
      - Table ID and Database ID come from previous node.  
      - No additional filters applied; fetches entire dataset.  
    - *Credentials:* Uses Baserow SaaS account credentials stored in n8n.  
    - *Connections:* Outputs to “Create new calendar view”.  
    - *Failure Cases:*  
      - API rate limits or authentication failures.  
      - Empty table (no records) leads to no calendar views created.  

#### 2.5 Calendar View Creation

- **Overview:** Creates a new calendar view for each record from the filter table using the Baserow API.
- **Nodes Involved:**  
  - Create new calendar view  

- **Node Details:**

  - **Create new calendar view**  
    - *Type:* HTTP Request  
    - *Role:* POST request to create a calendar view on the Date table.  
    - *Config:*  
      - URL built dynamically, targeting `/api/database/views/table/{Date table id}/`.  
      - JSON body configures view name dynamically as "Calendar for {Name}" from the record, sets type to calendar, ownership type collaborative, filter type OR, and date_field set.  
      - Header includes Authorization with JWT token.  
    - *Expressions:* Uses `{{$json.Name}}` and table/field IDs from previous nodes.  
    - *Connections:* Outputs to “Create filter”.  
    - *Failure Cases:*  
      - API errors, such as invalid field or table IDs.  
      - Authorization errors if token expired.  
      - Name conflicts or limits on view creation.  

#### 2.6 Filter Application

- **Overview:** Applies a filter on the newly created calendar view to show only records related to the current filter record.
- **Nodes Involved:**  
  - Create filter  

- **Node Details:**

  - **Create filter**  
    - *Type:* HTTP Request  
    - *Role:* POST request to `/api/database/views/{view_id}/filters/` to apply a filter to the view.  
    - *Config:*  
      - Field ID is the filter field (Link to table).  
      - Filter type is `link_row_has`.  
      - Value is the ID of the current filter record.  
      - Authorization header includes JWT token.  
    - *Expressions:* Uses view ID from previous node and filter record ID.  
    - *Connections:* Outputs to “Set background color”.  
    - *Failure Cases:*  
      - Invalid field or view ID.  
      - API errors or rate limits.  

#### 2.7 Visual Decoration

- **Overview:** Assigns a background color decoration to the calendar view items based on a single select field (status).
- **Nodes Involved:**  
  - Set background color  

- **Node Details:**

  - **Set background color**  
    - *Type:* HTTP Request  
    - *Role:* POST request to `/api/database/views/{view_id}/decorations/` to create a decoration rule.  
    - *Config:*  
      - Decoration type: `background_color`.  
      - Value provider type: `single_select_color`.  
      - Configuration references the status field ID.  
      - Order set to 1 (priority).  
      - Authorization header includes JWT token.  
    - *Expressions:* Uses view ID from previous node and status field ID.  
    - *Connections:* Outputs to “Share the view”.  
    - *Failure Cases:*  
      - Missing or invalid status field causes decoration failure.  
      - API errors.  

#### 2.8 Sharing and URL Update

- **Overview:** Shares the calendar view publicly by enabling the iCal feed and updates the filter table records with links to the new calendar view and iCal URL.
- **Nodes Involved:**  
  - Share the view  
  - Update the url's  

- **Node Details:**

  - **Share the view**  
    - *Type:* HTTP Request  
    - *Role:* PATCH request to update the calendar view's `ical_public` property to `true`.  
    - *Config:*  
      - URL targets `/api/database/views/{view_id}/`.  
      - JSON body sets `ical_public: true`.  
      - Authorization header includes JWT token.  
    - *Expressions:* Uses view ID from previous node.  
    - *Connections:* Outputs to “Update the url's”.  
    - *Failure Cases:*  
      - API errors.  
      - Permission issues in Baserow.  

  - **Update the url's**  
    - *Type:* Baserow node (API integration)  
    - *Role:* Updates each record in the filter table with the URLs to the new calendar view and the public iCal feed.  
    - *Config:*  
      - Row ID from current record.  
      - Table ID is the filter table.  
      - Fields updated:  
        - ICS field with `ical_feed_url` from the public view.  
        - View link field with a constructed URL to the Baserow table view.  
      - Uses Baserow SaaS credentials.  
    - *Connections:* End of chain.  
    - *Failure Cases:*  
      - Invalid row or field IDs.  
      - API errors or permission issues.  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                             | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                                              |
|----------------------------|---------------------|--------------------------------------------|------------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Entry trigger to start workflow manually   |                              | Set Baserow credentials          |                                                                                                                                          |
| Set Baserow credentials     | Set                 | Store user credentials and API host         | When clicking ‘Execute workflow’ | Create a token                   | # Authentication: replace <your_username> and <your_password> with your actual Baserow credentials.                                      |
| Create a token              | HTTP Request        | Generate JWT token for API authentication   | Set Baserow credentials       | Set table and field ids           |                                                                                                                                          |
| Set table and field ids     | Set                 | Store JWT token and relevant table/field IDs| Create a token                | Get all records from filter table | # Required tables and fields: explains IDs required.                                                                                      |
| Get all records from filter table | Baserow API Node   | Retrieve all filter table records            | Set table and field ids       | Create new calendar view          |                                                                                                                                          |
| Create new calendar view    | HTTP Request        | Create calendar view per filter record       | Get all records from filter table | Create filter                   |                                                                                                                                          |
| Create filter              | HTTP Request        | Apply filter on calendar view to limit records | Create new calendar view      | Set background color              |                                                                                                                                          |
| Set background color        | HTTP Request        | Set background color decoration on calendar view | Create filter                | Share the view                   |                                                                                                                                          |
| Share the view             | HTTP Request        | Make calendar view public (enable ical feed) | Set background color          | Update the url's                  |                                                                                                                                          |
| Update the url's           | Baserow API Node    | Update filter table records with view and ical URLs | Share the view               |                                  |                                                                                                                                          |
| Sticky Note                | Sticky Note         | Documentation and overview                   |                              |                                  | ## Create personalised calendar views with n8n and Baserow (long detailed explanation and links)                                         |
| Sticky Note1               | Sticky Note         | Authentication reminder                      |                              |                                  | # Authentication: replace <your_username> and <your_password> with your actual Baserow credentials.                                      |
| Sticky Note2               | Sticky Note         | Required tables and fields description       |                              |                                  | # Required tables and fields: Date table, Date field, Filter field, Status field, Filter table, Database ID, Ics field, View link field.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**  
   - Name: `When clicking ‘Execute workflow’`  
   - No special configuration. This is the entry point.

3. **Add a Set node to store Baserow credentials:**  
   - Name: `Set Baserow credentials`  
   - Add three string fields:  
     - `Baserow user` (set to your Baserow email/username)  
     - `Baserow password` (your Baserow password)  
     - `API host` (default: `https://api.baserow.io`)  
   - Connect Manual Trigger → Set Baserow credentials.

4. **Add an HTTP Request node to create JWT token:**  
   - Name: `Create a token`  
   - Method: POST  
   - URL: `={{ $json["API host"] }}/api/user/token-auth/`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "email": "{{ $json["Baserow user"] }}",
       "password": "{{ $json["Baserow password"] }}"
     }
     ```  
   - Send Body: true  
   - Connect Set Baserow credentials → Create a token.

5. **Add a Set node to configure JWT token and all relevant IDs:**  
   - Name: `Set table and field ids`  
   - Add fields with types and values:  
     - `JWT Token` (string), value: `=JWT {{ $json.access_token }}`  
     - `Date table` (number), e.g., 671901  
     - `Date field` (number), e.g., 5533823  
     - `Filter field` (number), e.g., 5533824  
     - `Status field` (number), e.g., 5533825 (optional)  
     - `Filter table` (number), e.g., 671897  
     - `Database ID` (number), e.g., 288404  
     - `Ics field` (string), e.g., `5533800` (optional)  
     - `View link field` (string), e.g., `5533805` (optional)  
   - Connect Create a token → Set table and field ids.

6. **Add a Baserow node to get all records from filter table:**  
   - Name: `Get all records from filter table`  
   - Operation: Get all records  
   - Table ID: `={{ $json["Filter table"] }}`  
   - Database ID: `={{ $json["Database ID"] }}`  
   - Credentials: Use your Baserow SaaS credentials (configured in n8n credentials).  
   - Connect Set table and field ids → Get all records from filter table.

7. **Add an HTTP Request node to create new calendar view:**  
   - Name: `Create new calendar view`  
   - Method: POST  
   - URL: `={{$('Set Baserow credentials').item.json["API host"]}}/api/database/views/table/{{ $('Set table and field ids').item.json['Date table'] }}/`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "name": "Calendar for {{ $json.Name }}",
       "type": "calendar",
       "ownership_type": "collaborative",
       "filter_type": "OR",
       "date_field": {{ $('Set table and field ids').item.json['Date field'] }}
     }
     ```  
   - Headers: Set `Authorization` to `={{ $('Set table and field ids').item.json['JWT Token'] }}`  
   - Connect Get all records from filter table → Create new calendar view.

8. **Add an HTTP Request node to create filter on calendar view:**  
   - Name: `Create filter`  
   - Method: POST  
   - URL: `={{$('Set Baserow credentials').item.json["API host"]}}/api/database/views/{{ $('Create new calendar view').item.json.id }}/filters/`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "field": {{ $('Set table and field ids').item.json['Filter field'] }},
       "type": "link_row_has",
       "value": {{ $json.id }}
     }
     ```  
   - Headers: `Authorization` with JWT token as above  
   - Connect Create new calendar view → Create filter.

9. **Add an HTTP Request node to set background color decoration:**  
   - Name: `Set background color`  
   - Method: POST  
   - URL: `={{$('Set Baserow credentials').item.json["API host"]}}/api/database/views/{{ $json.view }}/decorations/`  
   - Body Content Type: JSON  
   - Body:  
     ```json
     {
       "type": "background_color",
       "value_provider_type": "single_select_color",
       "value_provider_conf": {
         "field_id": {{ $('Set table and field ids').item.json['Status field'] }}
       },
       "order": 1
     }
     ```  
   - Headers: Authorization with JWT token  
   - Connect Create filter → Set background color.

10. **Add an HTTP Request node to share the view (enable iCal public feed):**  
    - Name: `Share the view`  
    - Method: PATCH  
    - URL: `={{$('Set Baserow credentials').item.json["API host"]}}/api/database/views/{{ $json.view }}/`  
    - Body Content Type: JSON  
    - Body:  
      ```json
      {
        "ical_public": true
      }
      ```  
    - Headers: Authorization with JWT token  
    - Connect Set background color → Share the view.

11. **Add a Baserow node to update URLs in filter table:**  
    - Name: `Update the url's`  
    - Operation: Update record  
    - Table ID: `={{ $('Set table and field ids').item.json["Filter table"] }}`  
    - Database ID: `={{ $('Set table and field ids').item.json["Database ID"] }}`  
    - Row ID: `={{ $json.id }}` (current record)  
    - Fields updated:  
      - Ics field: `={{ $json.ical_feed_url }}`  
      - View link field:  
        ``` 
        =https://baserow.cloudron.getbaserow.com/database/1478/table/{{ $('Set table and field ids').item.json['Date table'] }}/{{ $json.id }}
        ```  
    - Credentials: Baserow SaaS credentials  
    - Connect Share the view → Update the url's.

12. **Add Sticky Notes as documentation inside the workflow:**  
    - Copy the detailed project explanation, field definitions, and authentication instructions into Sticky Note nodes for user reference.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow demonstrates how to generate personalized calendar views in Baserow and share them externally via iCal feeds. It supports multiple use cases like task, customer, and inventory management. The setup requires a date field and a link to table field in Baserow. The created views can be filtered and decorated with colors based on status. Authentication uses JWT tokens generated from username/password. | Workflow description and overview from Sticky Note node.                                          |
| Baserow API endpoints used include: `/api/user/token-auth/` for authentication, `/api/database/views/table/{table_id}/` for creating views, `/api/database/views/{view_id}/filters/` for filters, `/api/database/views/{view_id}/decorations/` for decorations, and `/api/database/views/{view_id}` for updating views. Documentation links: https://api.baserow.io/api/redoc/ | Official Baserow API documentation.                                                               |
| The included [Baserow SOP template](https://baserow.io/templates/standard-operating-procedures) is recommended as a base database schema to test this workflow.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | Baserow SOP template link.                                                                         |
| To customize, change the date field, filters, or decorations by modifying the IDs or API request bodies accordingly. Optional steps like coloring or sharing can be added or removed. Extensions could include notifications to users when new views are created.                                                                                                                                                                                                                                                                                                                                                                                                                                               | Customization guidelines.                                                                          |

---

**Disclaimer:**  
The provided text is extracted exclusively from an n8n automated workflow. It complies fully with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.