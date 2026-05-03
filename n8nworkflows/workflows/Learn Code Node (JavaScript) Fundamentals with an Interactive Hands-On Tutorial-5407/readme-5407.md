Learn Code Node (JavaScript) Fundamentals with an Interactive Hands-On Tutorial

https://n8nworkflows.xyz/workflows/learn-code-node--javascript--fundamentals-with-an-interactive-hands-on-tutorial-5407


# Learn Code Node (JavaScript) Fundamentals with an Interactive Hands-On Tutorial

### 1. Workflow Overview

This workflow is an interactive, hands-on tutorial designed to teach fundamental JavaScript coding concepts within n8n through processing user data. It demonstrates how to manipulate arrays, process individual items, call external APIs asynchronously, perform aggregations, and generate binary files programmatically in n8n.

The workflow logically divides into the following blocks:

- **1.1 Input Preparation:** Generates and prepares sample user data for processing.
- **1.2 Item-by-Item Processing:** Processes each user individually to enrich data and call an external API.
- **1.3 Aggregation and Summary:** Aggregates processed user data to compute summary statistics.
- **1.4 Expert File Creation:** Generates a downloadable CSV file from the aggregated data.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Preparation

**Overview:**  
This block initializes the workflow by creating a sample dataset of users and splitting it into individual items for further processing.

**Nodes Involved:**  
- `Start Tutorial` (Manual Trigger)  
- `1. Sample Data` (Set)  
- `2. Split Out Users` (SplitOut)

**Node Details:**

- **Start Tutorial**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to manually start the workflow.  
  - *Configuration:* No parameters; triggers the workflow on demand.  
  - *Connections:* Outputs to `1. Sample Data`.  
  - *Failure Modes:* None specific; user-triggered.

- **1. Sample Data**  
  - *Type:* Set  
  - *Role:* Creates the initial dataset containing an array of user objects.  
  - *Configuration:* Defines one field `users` as an array with four user objects, each containing `firstName`, `lastName`, and `birthDate`.  
  - *Key Expressions:* Static JSON array assigned to `users`.  
  - *Connections:* Outputs to `2. Split Out Users`.  
  - *Failure Modes:* None expected; static data.  
  - *Sticky Note:* Explains the purpose of creating sample data and preparing it for processing.

- **2. Split Out Users**  
  - *Type:* SplitOut  
  - *Role:* Splits the `users` array into individual items so that subsequent nodes process each user separately.  
  - *Configuration:* Splits on the field `users`.  
  - *Connections:* Inputs from `1. Sample Data`, outputs to `3. Process Each User`.  
  - *Failure Modes:* Fails if `users` field missing or not an array.  
  - *Sticky Note:* Highlights that this node enables per-item processing by splitting the array.

---

#### Block 1.2: Item-by-Item Processing

**Overview:**  
Processes each user individually to calculate additional properties and enrich data by calling an external API asynchronously.

**Nodes Involved:**  
- `3. Process Each User` (Code)  
- `4. Fetch External Data (Advanced)` (Code)

**Node Details:**

- **3. Process Each User**  
  - *Type:* Code (JavaScript)  
  - *Role:* Runs once per user item to compute full name and age.  
  - *Configuration:*  
    - Mode: `runOnceForEachItem` (iterates per input item).  
    - JavaScript code calculates `fullName` by concatenating first and last names.  
    - Calculates age by subtracting birth date from current date.  
    - Outputs original user fields plus `fullName` and `age`.  
  - *Key Expressions:* Uses `$input.item.json` to access current item.  
  - *Connections:* Inputs from `2. Split Out Users`, outputs to `4. Fetch External Data (Advanced)`.  
  - *Failure Modes:* Date parsing errors; malformed user data.  
  - *Sticky Note:* Explains the per-item execution mode and key coding concepts.

- **4. Fetch External Data (Advanced)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Runs once per user item to asynchronously fetch gender data from an external API based on first name.  
  - *Configuration:*  
    - Mode: `runOnceForEachItem`.  
    - Uses `this.helpers.httpRequest` with `await` to call `https://api.genderize.io?name=FirstName`.  
    - Appends `gender` and `genderProbability` fields to the user data.  
  - *Key Expressions:* Async API call with helper method; merging API response with existing item data.  
  - *Connections:* Inputs from `3. Process Each User`, outputs to `5. Calculate Average Age`.  
  - *Failure Modes:* HTTP request failures (network issues, API downtime, rate limits), JSON parse errors.  
  - *Sticky Note:* Highlights the advanced technique of making asynchronous API calls within code nodes.

---

#### Block 1.3: Aggregation and Summary

**Overview:**  
Aggregates all processed users to compute summary statistics like total users and average age.

**Nodes Involved:**  
- `5. Calculate Average Age` (Code)

**Node Details:**

- **5. Calculate Average Age**  
  - *Type:* Code (JavaScript)  
  - *Role:* Runs once for all items, aggregates ages, and computes average age and total user count.  
  - *Configuration:*  
    - Mode: Runs once for all items (default).  
    - Uses `$items()` to access all input items as an array.  
    - Reduces the age values to sum and calculates average formatted to two decimals.  
    - Outputs a single JSON object with `totalUsers` and `averageAge`.  
  - *Connections:* Inputs from `4. Fetch External Data (Advanced)`, outputs to `6. Create a Binary File (Expert)`.  
  - *Failure Modes:* Empty input array, division by zero if no users.  
  - *Sticky Note:* Explains aggregation concepts and use of `$items()`.

---

#### Block 1.4: Expert File Creation

**Overview:**  
Creates a CSV binary file from all user data enriched with gender info to produce a downloadable report.

**Nodes Involved:**  
- `6. Create a Binary File (Expert)` (Code)

**Node Details:**

- **6. Create a Binary File (Expert)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Runs once for all items, creates CSV content string, converts it to a binary file, and outputs it as binary data for downstream use.  
  - *Configuration:*  
    - Mode: Runs once for all items.  
    - Accesses all items from the node `4. Fetch External Data (Advanced)` using `$("4. Fetch External Data (Advanced)").all()`.  
    - Creates CSV header and rows for `FullName`, `Age`, `GenderGuess`, and a static `ProcessedBy` column.  
    - Uses `this.helpers.prepareBinaryData` with a Node.js Buffer to create binary data for the CSV.  
    - Returns an item with JSON metadata and the binary file under the key `report`.  
  - *Failure Modes:* Buffering or binary conversion errors; missing input data.  
  - *Sticky Note:* Describes key concepts about generating binary files and using helper functions.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                           | Input Node(s)                 | Output Node(s)                   | Sticky Note                                                                                                                      |
|-----------------------------|--------------------|-----------------------------------------|-----------------------------|---------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Start Tutorial              | Manual Trigger     | Workflow entry point                     |                             | 1. Sample Data                  |                                                                                                                                  |
| 1. Sample Data              | Set                | Create sample user data array            | Start Tutorial               | 2. Split Out Users              | Starting point: creates list of users; editable for experimentation.                                                             |
| 2. Split Out Users          | SplitOut           | Split user array into individual items  | 1. Sample Data               | 3. Process Each User            | Splits array so next code nodes process users one by one.                                                                         |
| 3. Process Each User        | Code (JavaScript)  | Per-item processing: add fullName, age  | 2. Split Out Users           | 4. Fetch External Data (Advanced) | Runs once per item; demonstrates $input.item.json and returning enriched data.                                                   |
| 4. Fetch External Data (Advanced) | Code (JavaScript)  | Per-item API call to enrich gender info | 3. Process Each User         | 5. Calculate Average Age        | Uses this.helpers.httpRequest and async/await to call external API; enriches each item with gender data.                         |
| 5. Calculate Average Age    | Code (JavaScript)  | Aggregate all items to compute average age | 4. Fetch External Data (Advanced) | 6. Create a Binary File (Expert) | Runs once for all items; uses $items() to aggregate; outputs summary with total user count and average age.                      |
| 6. Create a Binary File (Expert) | Code (JavaScript)  | Create CSV binary file from aggregated data | 5. Calculate Average Age      |                                 | Creates CSV string and binary file with this.helpers.prepareBinaryData; expert lesson on binary file creation.                  |
| Sticky Note                 | Sticky Note        | Documentation note                      |                             |                                 | Starting point: sample data creation and splitting.                                                                              |
| Sticky Note1                | Sticky Note        | Documentation note                      |                             |                                 | Lesson 1: per-item processing concepts and code behavior.                                                                        |
| Sticky Note2                | Sticky Note        | Documentation note                      |                             |                                 | Advanced lesson: asynchronous API calls in code node using helpers.httpRequest and await.                                        |
| Sticky Note3                | Sticky Note        | Documentation note                      |                             |                                 | Lesson 3: aggregation concepts using $items() and running once for all items.                                                    |
| Sticky Note4                | Sticky Note        | Documentation note                      |                             |                                 | Expert lesson: binary file creation with Buffer and prepareBinaryData helper function.                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**  
   - Add a **Manual Trigger** node named `Start Tutorial`. No additional configuration needed.

2. **Add a Set Node for Sample Data**  
   - Add a **Set** node named `1. Sample Data`.  
   - Add a new field:  
     - Name: `users`  
     - Type: `Array`  
     - Value: JSON array of user objects:  
       ```json
       [
         {"firstName":"Alice","lastName":"Smith","birthDate":"1990-05-15"},
         {"firstName":"Bob","lastName":"Jones","birthDate":"1985-11-22"},
         {"firstName":"Charlie","lastName":"Brown","birthDate":"2001-02-10"},
         {"firstName":"Alex","lastName":"Garcia","birthDate":"1995-07-30"}
       ]
       ```
   - Connect `Start Tutorial` output to this node.

3. **Add a SplitOut Node**  
   - Add a **SplitOut** node named `2. Split Out Users`.  
   - Set the field to split out as `users`.  
   - Connect output of `1. Sample Data` to this node.

4. **Add a Code Node to Process Each User**  
   - Add a **Code** node named `3. Process Each User`.  
   - Set execution mode to **Run Once For Each Item**.  
   - Paste the following JavaScript code:
     ```javascript
     const user = $input.item.json;
     const fullName = `${user.firstName} ${user.lastName}`;
     const birthDate = new Date(user.birthDate);
     const ageDiffMs = Date.now() - birthDate.getTime();
     const ageDate = new Date(ageDiffMs);
     const age = Math.abs(ageDate.getUTCFullYear() - 1970);
     return {
       ...user,
       fullName,
       age
     };
     ```
   - Connect output of `2. Split Out Users` to this node.

5. **Add a Code Node to Fetch External Data per User**  
   - Add a **Code** node named `4. Fetch External Data (Advanced)`.  
   - Set execution mode to **Run Once For Each Item**.  
   - Paste the following JavaScript code:
     ```javascript
     const user = $input.item.json;
     const url = `https://api.genderize.io?name=${user.firstName}`;
     const response = await this.helpers.httpRequest({ url, json: true });
     return {
       ...user,
       gender: response.gender,
       genderProbability: response.probability
     };
     ```
   - Connect output of `3. Process Each User` to this node.

6. **Add a Code Node to Calculate Average Age**  
   - Add a **Code** node named `5. Calculate Average Age`.  
   - Use default mode (runs once for all items).  
   - Paste the following JavaScript code:
     ```javascript
     const allUsers = $items();
     const totalAge = allUsers.reduce((sum, item) => sum + item.json.age, 0);
     const userCount = allUsers.length;
     const averageAge = totalAge / userCount;
     return [{
       json: {
         totalUsers: userCount,
         averageAge: parseFloat(averageAge.toFixed(2))
       }
     }];
     ```
   - Connect output of `4. Fetch External Data (Advanced)` to this node.

7. **Add a Code Node to Create a Binary CSV File**  
   - Add a **Code** node named `6. Create a Binary File (Expert)`.  
   - Use default mode (runs once for all items).  
   - Paste the following JavaScript code:
     ```javascript
     const allUsers = $("4. Fetch External Data (Advanced)").all();
     let csvContent = "FullName,Age,GenderGuess,ProcessedBy\n";
     for (const item of allUsers) {
       const user = item.json;
       const row = `"${user.fullName}",${user.age},${user.gender},n8n`;
       csvContent += row + "\n";
     }
     const binaryData = await this.helpers.prepareBinaryData(Buffer.from(csvContent), 'user_report.csv');
     return [{
       json: {
         reportGenerated: new Date().toISOString(),
         userCount: allUsers.length
       },
       binary: {
         report: binaryData
       }
     }];
     ```
   - Connect output of `5. Calculate Average Age` to this node.

8. **Save and Run**  
   - Save the workflow.  
   - Trigger manually via `Start Tutorial`.  
   - Observe outputs at each step for validation.  
   - Download the binary file from the final node output if desired.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                 |
|----------------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow designed as a stepwise JavaScript coding tutorial within n8n for interactive learning.    | Workflow description                            |
| `this.helpers.httpRequest` enables asynchronous API calls inside Code nodes in n8n.                 | Official n8n documentation                      |
| Binary file creation uses `this.helpers.prepareBinaryData` combined with Node.js Buffer.            | n8n advanced coding techniques                   |
| External API used: https://genderize.io to guess gender from first names; free and public API.     | https://genderize.io                            |
| Sticky Notes in the workflow provide context and lessons for each step; recommended to read.       | Embedded documentation nodes within the workflow|

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. It strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly available.