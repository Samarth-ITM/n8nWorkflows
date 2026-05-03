Learn JavaScript Data Processing with Code Node: Filtering, Analysis & Export Examples

https://n8nworkflows.xyz/workflows/learn-javascript-data-processing-with-code-node--filtering--analysis---export-examples-5729


# Learn JavaScript Data Processing with Code Node: Filtering, Analysis & Export Examples

### 1. Workflow Overview

This workflow, titled **"Learn JavaScript Data Processing with Code Node: Filtering, Analysis & Export Examples"**, is designed as an educational tool to demonstrate practical uses of n8n's Code node for JavaScript data manipulation. It targets users who want to learn or improve their skills in filtering, transforming, analyzing, and formatting JSON data within n8n workflows.

The workflow revolves around a sample dataset representing user records and includes three main logical blocks:

- **1.1 Input Setup:** Manual trigger and sample data definition to simulate incoming user data.
- **1.2 Data Filtering & Transformation:** Filtering users based on age, adding calculated fields, and outputting enriched user objects.
- **1.3 Statistical Analysis:** Aggregating user data into various statistics such as averages, totals, distributions.
- **1.4 Data Formatting for Export:** Formatting the processed data into multiple output formats like CSV, email list strings, and structured JSON for APIs.

Supporting the main logic are sticky notes that explain concepts, provide instructions, and describe use cases.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Setup

- **Overview:**  
  This block initializes the workflow execution manually and sets up sample user data in JSON format. It simulates data input for subsequent processing nodes.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Set Sample Data

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - *Type & Role:* Manual trigger node; initiates workflow execution on user command.  
    - *Configuration:* Default, no parameters.  
    - *Expressions/Variables:* None.  
    - *Inputs:* None (trigger node).  
    - *Outputs:* Connects to "Set Sample Data".  
    - *Edge Cases:* None; manual trigger depends on user interaction.  

  - **Set Sample Data**  
    - *Type & Role:* Set node; defines static JSON data to simulate user records.  
    - *Configuration:*  
      - Sets two fields: `scenario` (string "user_data") and `usersJson` (string containing JSON array of user objects).  
      - Data includes four users with fields: name, age, email, role, salary.  
    - *Expressions/Variables:* The JSON string is stored as a simple string, parsed later in code nodes.  
    - *Inputs:* Receives from Manual Trigger.  
    - *Outputs:* Connects to all three Code nodes.  
    - *Edge Cases:* If JSON string is malformed, subsequent parsing will fail. Should be validated or trusted as static here.  

#### 2.2 Data Filtering & Transformation

- **Overview:**  
  Filters users older than 18 years, calculates a 10% bonus on their salary, creates formatted email strings, and flags eligibility. Outputs each filtered user as an individual item.

- **Nodes Involved:**  
  - Code: Filter & Transform

- **Node Details:**

  - **Code: Filter & Transform**  
    - *Type & Role:* Code node; performs JavaScript filtering and transformation on input data.  
    - *Configuration:*  
      - Parses JSON string from `usersJson` field.  
      - Filters users with `age > 18`.  
      - Adds fields: `bonus` (10% of salary), `fullEmail` (formatted string), `isEligible` (true).  
      - Returns an array where each user is a separate item for n8n downstream consumption.  
    - *Expressions/Variables:* Uses `items[0].json.usersJson` as input. Returns mapped array.  
    - *Inputs:* Receives from "Set Sample Data".  
    - *Outputs:* Passes individual user items downstream (not connected further in this workflow but output visible on execution).  
    - *Edge Cases:*  
      - Parsing error if `usersJson` malformed.  
      - Assumes salary is numeric; non-numeric salaries cause NaN in bonus calculation.  
      - Empty user list returns empty output gracefully.  

#### 2.3 Statistical Analysis

- **Overview:**  
  Aggregates the entire user dataset into a single summary object containing counts, averages, totals, role distribution, min/max age, and eligibility counts.

- **Nodes Involved:**  
  - Code: Calculate Stats

- **Node Details:**

  - **Code: Calculate Stats**  
    - *Type & Role:* Code node; computes aggregated statistics from user data.  
    - *Configuration:*  
      - Parses `usersJson`.  
      - Calculates total employee count, average age, total and average salary.  
      - Creates role distribution map (counts per role).  
      - Determines minimum and maximum age.  
      - Counts users eligible for bonus (age > 18).  
      - Returns a single JSON object with all statistics.  
    - *Expressions/Variables:* Reads from `items[0].json.usersJson`.  
    - *Inputs:* Receives from "Set Sample Data".  
    - *Outputs:* Single summary object downstream (output visible on execution).  
    - *Edge Cases:*  
      - Empty user array leads to division by zero; would produce NaN averages. Should be handled in production code.  
      - Role values assumed non-null strings.  
      - Parsing errors if JSON malformed.  

#### 2.4 Data Formatting for Export

- **Overview:**  
  Prepares the user data for various export scenarios by generating a CSV string, a comma-separated email list, and a structured JSON array for API consumption, including metadata and timestamps.

- **Nodes Involved:**  
  - Code: Format for Export

- **Node Details:**

  - **Code: Format for Export**  
    - *Type & Role:* Code node; produces multi-format output for integration or reporting.  
    - *Configuration:*  
      - Parses `usersJson`.  
      - Creates:  
        - `emailList`: string of emails separated by commas.  
        - `csvFormat`: CSV string with user fields separated by commas, each user on a new line.  
        - `jsonForApi`: array of user objects with normalized fields like `id` (lowercase, underscore), `displayName`, `contact`, `position`, `compensation`, `eligibleForVoting` (age >= 18).  
      - Adds `reportDate` timestamp and `teamOverview` metadata (total members, unique departments, total payroll).  
      - Returns a single JSON object with all formatted data.  
    - *Expressions/Variables:* Uses `items[0].json.usersJson`.  
    - *Inputs:* Receives from "Set Sample Data".  
    - *Outputs:* Single item with formatted report JSON.  
    - *Edge Cases:*  
      - Empty user list results in empty CSV and email list strings.  
      - Assumes user names have single space for ID formatting; multiple spaces could cause inconsistent IDs.  
      - Field values assumed to be strings or numbers; no escaping for CSV commas or special characters.  

#### 2.5 Documentation & Guidance (Sticky Notes)

- **Overview:**  
  Several sticky notes provide workflow purpose, code node basic patterns, example explanations, and quick start tips.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5

- **Node Details:**  
  Each sticky note contains formatted Markdown text explaining specific aspects of the workflow such as:

  - Workflow purpose and author contact  
  - Code node programming patterns and return structures  
  - Descriptions of each example block and their use cases  
  - Quick start instructions and debugging tips  

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                          | Input Node(s)              | Output Node(s)                                 | Sticky Note                                                                                         |
|-------------------------|-----------------------|----------------------------------------|---------------------------|-----------------------------------------------|---------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Starts workflow on manual execution    | None                      | Set Sample Data                               |                                                                                                   |
| Set Sample Data          | Set                   | Defines static sample user JSON data   | When clicking ‘Execute workflow’ | Code: Filter & Transform, Code: Calculate Stats, Code: Format for Export |                                                                                                   |
| Code: Filter & Transform | Code                  | Filters adult users, adds bonus & flags | Set Sample Data            | None (output visible in execution)            | See Sticky Note2 for example explanation                                                          |
| Code: Calculate Stats    | Code                  | Aggregates user data into statistics   | Set Sample Data            | None (output visible in execution)            | See Sticky Note3 for example explanation                                                          |
| Code: Format for Export  | Code                  | Formats data for CSV, email list, API  | Set Sample Data            | None (output visible in execution)            | See Sticky Note4 for example explanation                                                          |
| Sticky Note              | Sticky Note           | Workflow purpose and author info       | None                      | None                                          | ## 🎯 WORKFLOW PURPOSE ... See content above                                                     |
| Sticky Note1             | Sticky Note           | Code node basics and patterns           | None                      | None                                          | ## 🔧 CODE NODE BASICS ... See content above                                                      |
| Sticky Note2             | Sticky Note           | Example 1: Filter & Transform explanation | None                      | None                                          | ## 📊 EXAMPLE 1: FILTER & TRANSFORM ... See content above                                         |
| Sticky Note3             | Sticky Note           | Example 2: Calculate Stats explanation  | None                      | None                                          | ## 📈 EXAMPLE 2: CALCULATE STATS ... See content above                                            |
| Sticky Note4             | Sticky Note           | Example 3: Format for Export explanation | None                      | None                                          | ## 🔄 EXAMPLE 3: FORMAT FOR EXPORT ... See content above                                          |
| Sticky Note5             | Sticky Note           | Quick start guide and tips              | None                      | None                                          | ## 🚀 QUICK START GUIDE ... See content above                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger (n8n-nodes-base.manualTrigger)  
   - No parameters needed.

2. **Create Set Node for Sample Data**  
   - Name: `Set Sample Data`  
   - Type: Set (n8n-nodes-base.set)  
   - Add two fields:  
     - `scenario` (String): Value = `"user_data"`  
     - `usersJson` (String): JSON array string representing users:  
       ```json
       [
         {"name": "Alice Johnson", "age": 28, "email": "alice@example.com", "role": "developer", "salary": 75000},
         {"name": "Bob Smith", "age": 17, "email": "bob@example.com", "role": "intern", "salary": 25000},
         {"name": "Charlie Brown", "age": 35, "email": "charlie@example.com", "role": "manager", "salary": 95000},
         {"name": "Diana Ross", "age": 22, "email": "diana@example.com", "role": "designer", "salary": 60000}
       ]
       ```
   - Connect output of Manual Trigger to this Set node.

3. **Create Code Node for Filtering & Transformation**  
   - Name: `Code: Filter & Transform`  
   - Type: Code (n8n-nodes-base.code)  
   - JavaScript code:
     ```javascript
     const users = JSON.parse(items[0].json.usersJson);

     const adults = users
       .filter(user => user.age > 18)
       .map(user => ({
         ...user,
         bonus: user.salary * 0.1,
         fullEmail: `${user.name} <${user.email}>`,
         isEligible: true
       }));

     return adults.map(user => ({ json: user }));
     ```
   - Connect output of "Set Sample Data" to this node.

4. **Create Code Node for Statistical Calculation**  
   - Name: `Code: Calculate Stats`  
   - Type: Code  
   - JavaScript code:
     ```javascript
     const users = JSON.parse(items[0].json.usersJson);

     const stats = {
       totalEmployees: users.length,
       averageAge: users.reduce((sum, user) => sum + user.age, 0) / users.length,
       totalSalary: users.reduce((sum, user) => sum + user.salary, 0),
       averageSalary: users.reduce((sum, user) => sum + user.salary, 0) / users.length,
       roleDistribution: users.reduce((acc, user) => {
         acc[user.role] = (acc[user.role] || 0) + 1;
         return acc;
       }, {}),
       minAge: Math.min(...users.map(u => u.age)),
       maxAge: Math.max(...users.map(u => u.age)),
       eligibleForBonus: users.filter(u => u.age > 18).length
     };

     return [{ json: stats }];
     ```
   - Connect output of "Set Sample Data" to this node.

5. **Create Code Node for Formatting Export Data**  
   - Name: `Code: Format for Export`  
   - Type: Code  
   - JavaScript code:
     ```javascript
     const users = JSON.parse(items[0].json.usersJson);

     const emailList = users.map(user => user.email).join(', ');
     const csvFormat = users.map(user => 
       `${user.name},${user.age},${user.email},${user.role},${user.salary}`
     ).join('\n');

     const report = {
       reportDate: new Date().toISOString(),
       teamOverview: {
         totalMembers: users.length,
         departments: [...new Set(users.map(u => u.role))],
         payrollTotal: users.reduce((sum, u) => sum + u.salary, 0)
       },
       formattedData: {
         emailList,
         csvFormat,
         jsonForApi: users.map(user => ({
           id: user.name.toLowerCase().replace(' ', '_'),
           displayName: user.name,
           contact: user.email,
           position: user.role,
           compensation: user.salary,
           eligibleForVoting: user.age >= 18
         }))
       }
     };

     return [{ json: report }];
     ```
   - Connect output of "Set Sample Data" to this node.

6. **Add Sticky Notes (Optional but Recommended for Documentation)**  
   - Add Sticky Note nodes with the content provided for workflow purpose, code node basics, and example explanations.  
   - Place them visually near relevant nodes for clarity.

7. **Execution and Testing**  
   - Run the workflow manually.  
   - Inspect outputs of each Code node to verify data processing and transformations.  
   - Modify code as needed to experiment or adapt for real data.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                 | Context or Link                                                    |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| 🎯 WORKFLOW PURPOSE: Demonstrates Code node usage for filtering, stats, formatting; practical learning resource by David Olusola. Coaching and consulting offers available via email.                                                        | Author contact: david@daexai.com                                  |
| 🔧 CODE NODE BASICS: Access input via `items[0].json`, return arrays of objects, use `.map()` for multiple outputs, return format `[{ json: data }]`.                                                                                       | See Sticky Note1 in workflow                                      |
| 🚀 QUICK START GUIDE: Click “Test workflow” to run, check outputs, edit JavaScript, use `console.log` for debugging, handle edge cases, return correct n8n format.                                                                          | See Sticky Note5 in workflow                                      |
| 📊 Example 1: Filtering & Transformation suitable for lead qualification, segmentation, enrichment.                                                                                                                                          | See Sticky Note2                                                  |
| 📈 Example 2: Statistical aggregation for dashboards, KPI, BI.                                                                                                                                                                              | See Sticky Note3                                                  |
| 🔄 Example 3: Multi-format export for APIs, email marketing, data exports.                                                                                                                                                                  | See Sticky Note4                                                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.