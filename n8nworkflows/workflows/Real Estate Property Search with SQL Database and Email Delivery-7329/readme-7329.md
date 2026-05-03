Real Estate Property Search with SQL Database and Email Delivery

https://n8nworkflows.xyz/workflows/real-estate-property-search-with-sql-database-and-email-delivery-7329


# Real Estate Property Search with SQL Database and Email Delivery

### 1. Workflow Overview

This workflow automates a **real estate property search** by accepting user input through a web form, dynamically building a SQL query to retrieve matching properties from a Microsoft SQL Server database, converting the results into a CSV file, and emailing the results to the user.

**Target Use Cases:**  
- Real estate agencies or platforms providing users with a personalized property search experience.  
- Users specifying detailed search criteria including price, bedrooms, bathrooms, location, and property status.  
- Delivering search results conveniently via email with an attached CSV file for further analysis.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** User submits search criteria via an interactive form.  
- **1.2 Query Construction:** Build a dynamic SQL query based on the user’s input, adding only relevant filters.  
- **1.3 Database Query Execution:** Run the SQL query on a Microsoft SQL Server containing property listings.  
- **1.4 Data Conversion:** Convert the query results into a CSV file format for easy consumption.  
- **1.5 Email Delivery:** Send a personalized email with summary and CSV attachment to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures user inputs for property search criteria through a web form, triggering the workflow when submitted.

- **Nodes Involved:**  
  - Property Search Form

- **Node Details:**  

  **Property Search Form**  
  - *Type:* Form Trigger  
  - *Role:* Webhook-based form collecting user inputs for property search.  
  - *Configuration:*  
    - Path: `/property-search-form`  
    - Form Title: "🏠 Find Your Perfect Property"  
    - Fields include: Email (required), Property Status (select), Minimum/Maximum Price, Bedrooms, Bathrooms, House Size, Lot Size, State, City, ZIP Code.  
    - Description: "Search through 1000+ real estate properties in our database."  
  - *Inputs:* External user submission via HTTP POST.  
  - *Outputs:* JSON object containing all form fields and values.  
  - *Edge Cases/Potential Failures:*  
    - Missing required email field blocks submission.  
    - Invalid numeric inputs (e.g., negative numbers) are not explicitly validated here; downstream nodes must handle.  
    - Form submission failures (network or server issues).  
  - *Version-Specific:* Uses Form Trigger v2 node, ensure n8n version supports it.  

#### 1.2 Query Construction

- **Overview:**  
  Dynamically builds an optimized SQL query based on the filled form fields, ensuring only specified criteria are included in the WHERE clause.

- **Nodes Involved:**  
  - Build SQL Query

- **Node Details:**  

  **Build SQL Query**  
  - *Type:* Code (JavaScript)  
  - *Role:* Parse form input and generate SQL query string with conditions reflecting user criteria.  
  - *Configuration Highlights:*  
    - Extracts all form data from input JSON.  
    - Helper functions add numeric range conditions (min/max) only if values are valid and non-zero.  
    - Adds string equality conditions for State, City, ZIP Code, and Property Status if provided and non-empty.  
    - Limits results to top 1000 matches ordered by ascending price.  
    - Logs errors and returns error JSON if form data processing fails.  
    - Outputs:  
      - `query`: SQL query string to execute.  
      - `searchCriteria`: structured object summarizing applied filters.  
      - `formData`: original form input.  
      - `totalConditions`: number of WHERE conditions applied.  
      - `userEmail`: email address from form.  
      - `searchTimestamp`: ISO string of search time.  
  - *Inputs:* JSON from Form Trigger node.  
  - *Outputs:* JSON with SQL query and metadata.  
  - *Edge Cases/Potential Failures:*  
    - Invalid or missing numeric values handled by condition checks.  
    - Injection risks mitigated by strict control over query construction (though parameterized queries would be safer).  
    - Parsing errors logged and returned as error JSON.  
  - *Version-Specific:* JavaScript code node v2.  
  - *Notes:* This node is critical for query correctness and performance.

#### 1.3 Database Query Execution

- **Overview:**  
  Executes the constructed SQL query against a Microsoft SQL Server database and retrieves matching property records.

- **Nodes Involved:**  
  - Microsoft SQL

- **Node Details:**  

  **Microsoft SQL**  
  - *Type:* Microsoft SQL Node  
  - *Role:* Runs the SQL command received from the previous node on the realtor database.  
  - *Configuration:*  
    - Operation: Execute Query  
    - Query: Uses expression `{{ $json.query }}` to get the SQL string from the previous node.  
  - *Inputs:* JSON containing SQL query string.  
  - *Outputs:* Array of property records matching the query.  
  - *Credentials:* Requires configured Microsoft SQL Server credentials with read access to `[REALTOR].[dbo].[realtor_usa_price]`.  
  - *Edge Cases/Potential Failures:*  
    - SQL connection errors (network, auth).  
    - Query syntax errors if generated incorrectly.  
    - Large result sets may cause resource constraints; capped at top 1000 rows to mitigate.  
  - *Version-Specific:* Node version 1.1 used.

#### 1.4 Data Conversion

- **Overview:**  
  Converts the array of property objects into a CSV file format, suitable for spreadsheet applications.

- **Nodes Involved:**  
  - Convert to File

- **Node Details:**  

  **Convert to File**  
  - *Type:* Convert to File  
  - *Role:* Transform JSON data from SQL query into a CSV file attachment.  
  - *Configuration:* Default options, automatically infers CSV format from JSON structure.  
  - *Inputs:* Property records from Microsoft SQL node.  
  - *Outputs:* Binary data representing the CSV file.  
  - *Edge Cases/Potential Failures:*  
    - Empty result sets produce empty CSV files.  
    - Data type inconsistencies could affect CSV formatting but generally handled by node.  
  - *Version-Specific:* Node version 1.1.

#### 1.5 Email Delivery

- **Overview:**  
  Sends a personalized email to the user with a summary of their search results and attaches the CSV file for detailed review.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  

  **Send a message**  
  - *Type:* Gmail Node  
  - *Role:* Compose and send an email including a summary message and CSV attachment.  
  - *Configuration Highlights:*  
    - Recipient: User email from form input.  
    - Subject: Includes number of properties found.  
    - Message Body:  
      - Greeting and count of properties found.  
      - Lists top 3 properties with price, bedrooms, bathrooms, and location.  
      - Attach CSV file with all results.  
      - Summarizes user search criteria (price range, bedrooms, location).  
    - Attachments: Binary data from Convert to File node.  
  - *Inputs:*  
    - Text and dynamic expressions referencing previous nodes.  
    - Attachment binary data.  
  - *Credentials:* Gmail OAuth2 credentials required.  
  - *Edge Cases/Potential Failures:*  
    - Email sending failures (auth, quota, network).  
    - Missing or invalid recipient email (validated at form input).  
    - Empty results handled gracefully in message text.  
  - *Version-Specific:* Node version 2.1.

---

### 3. Summary Table

| Node Name           | Node Type         | Functional Role                     | Input Node(s)           | Output Node(s)        | Sticky Note                                                                                       |
|---------------------|-------------------|-----------------------------------|------------------------|-----------------------|-------------------------------------------------------------------------------------------------|
| Property Search Form | Form Trigger      | Receive user input for property search | (start)                | Build SQL Query        | 🎯 1. START HERE! User fills out their dream home criteria - price range, bedrooms, location, etc. This triggers the entire search process! |
| Build SQL Query      | Code (JavaScript) | Build dynamic SQL query based on input | Property Search Form   | Microsoft SQL          | 🧠 2. SMART QUERY BUILDER Takes user input and creates a dynamic SQL query. Only adds conditions for fields the user actually filled out - no empty searches! |
| Microsoft SQL        | Microsoft SQL     | Execute SQL query and retrieve matching properties | Build SQL Query        | Convert to File        | 🔍 3. DATABASE DETECTIVE Searches through 1000+ real estate properties in SQL Server and finds matches based on user criteria. Returns real property data! |
| Convert to File      | Convert to File   | Convert property data to CSV file | Microsoft SQL          | Send a message         | 📊 4. EXCEL MAGIC Transforms search results into a beautiful CSV file that users can open in Excel or Google Sheets for easy viewing and sorting! |
| Send a message       | Gmail             | Send results email with CSV attachment | Convert to File        | (end)                  | 📧 5. DELIVERY SERVICE Sends personalized email with search results attached. User gets their dream properties delivered straight to their inbox! |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Property Search Form Node**  
   - Type: Form Trigger (v2)  
   - Set Path: `property-search-form`  
   - Form Title: "🏠 Find Your Perfect Property"  
   - Add fields:  
     - Email (type: email, required)  
     - Property Status (type: select, options not predefined here)  
     - Minimum Price ($) (number)  
     - Maximum Price ($) (number)  
     - Minimum Bedrooms (number)  
     - Maximum Bedrooms (number)  
     - Minimum Bathrooms (number)  
     - Maximum Bathrooms (number)  
     - Minimum House Size (sqft) (number)  
     - Maximum House Size (sqft) (number)  
     - Minimum Lot Size (acres) (number)  
     - State (text)  
     - City (text)  
     - ZIP Code (text or number)  
   - Form Description: "Search through 1000+ real estate properties in our database"  

2. **Create Build SQL Query Node**  
   - Type: Code (JavaScript, v2)  
   - Connect input from Property Search Form node.  
   - Paste JavaScript code that:  
     - Reads form inputs.  
     - Builds SELECT query for `[REALTOR].[dbo].[realtor_usa_price]` database table.  
     - Adds WHERE conditions only for non-empty inputs (price range, bedrooms, bathrooms, house size, lot size, state, city, ZIP code, property status).  
     - Limits top 1000 results ordered by price ascending.  
     - Returns query string and metadata (search criteria, user email, timestamp).  

3. **Create Microsoft SQL Node**  
   - Type: Microsoft SQL (v1.1)  
   - Operation: Execute Query  
   - Query: Use expression `{{ $json.query }}` to get SQL query from Build SQL Query node.  
   - Configure Microsoft SQL credentials with read access to the realtor database.  
   - Connect input from Build SQL Query node.  

4. **Create Convert to File Node**  
   - Type: Convert to File (v1.1)  
   - Default settings for CSV.  
   - Connect input from Microsoft SQL node.  

5. **Create Send a message Node**  
   - Type: Gmail (v2.1)  
   - Set recipient: Use expression from Property Search Form email field `={{ $('Property Search Form').item.json['Your Email Address'] }}`  
   - Subject: "🏠 Your Dream Home Search Results - {{ $('Microsoft SQL').all().length }} Properties Found!"  
   - Message:  
     ```
     Hi there! 👋

     Great news! We found {{ $('Microsoft SQL').all().length }} properties that match your search criteria.

     🏠 TOP MATCHES:
     {{ $('Microsoft SQL').all().slice(0, 3).map(property => 
     `💰 $${property.json.price.toLocaleString()} - ${property.json.bed}bed/${property.json.bath}bath in ${property.json.city}, ${property.json.state}`
     ).join('\n') }}

     📎 COMPLETE RESULTS ATTACHED
     All {{ $('Microsoft SQL').all().length }} properties are in the attached CSV file - perfect for Excel!

     🎯 YOUR SEARCH:
     - Price Range: ${{ $('Build SQL Query').first().json.searchCriteria.priceRange.min?.toLocaleString() || 'Any' }} - ${{ $('Build SQL Query').first().json.searchCriteria.priceRange.max?.toLocaleString() || 'Any' }}
     - Bedrooms: {{ $('Build SQL Query').first().json.searchCriteria.bedrooms.min || 'Any' }}+
     - Location: {{ $('Build SQL Query').first().json.searchCriteria.location.city || $('Build SQL Query').first().json.searchCriteria.location.state || 'Anywhere' }}

     Happy house hunting! 🏡✨

     Your Property Search Team
     ```
   - Attach the CSV file from Convert to File node binary output.  
   - Configure Gmail OAuth2 credentials.  
   - Connect input from Convert to File node.  

6. **Connect Nodes in Sequence:**  
   - Property Search Form → Build SQL Query → Microsoft SQL → Convert to File → Send a message  

7. **Test the workflow by submitting the form at `/property-search-form` endpoint with sample data.**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| The workflow is designed for use with Microsoft SQL Server, querying the `[REALTOR].[dbo].[realtor_usa_price]` table containing 1000+ properties. | Database schema dependency          |
| Gmail node requires OAuth2 credentials configured in n8n for sending emails on behalf of the user or organization.                                | Gmail OAuth2 setup                  |
| The form trigger node uses webhook path `/property-search-form`; ensure your n8n instance is publicly accessible or behind proper proxy for access. | Webhook public accessibility        |
| CSV file attachment allows users to open results in Excel or Google Sheets for further filtering or analysis.                                      | User data consumption convenience  |
| This workflow is a good example of combining user input, dynamic query construction, database integration, file generation, and email delivery.     | Integration pattern in n8n          |

---

_Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public._