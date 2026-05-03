Joining different datasets

https://n8nworkflows.xyz/workflows/joining-different-datasets-1747


# Joining different datasets

### 1. Workflow Overview

This workflow demonstrates how to join or merge different datasets using n8n’s Merge node, mimicking SQL join operations. It is designed for scenarios where multiple collections of data need to be combined, filtered, or enriched based on matching criteria. It highlights three main use cases:

- Keeping only items from dataset A that have matches in dataset B (inner join-like behavior).
- Enriching items from dataset A with matching data from dataset B (left join-like behavior).
- Appending dataset B’s items below dataset A’s items (union all-like behavior).

The workflow is organized into three main logical blocks based on these use cases, each illustrated with sample data and different merge strategies:

- **1.1 Filtering items from A based on matches in B (Inner join)**
- **1.2 Enriching items from A with data from B (Left join)**
- **1.3 Appending datasets A and B (Union All)**

Additionally, it includes introductory notes explaining the workflow’s purpose and guiding users on how to interact with the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initialization

**Overview:**  
This block triggers the workflow execution and initializes six different datasets used in the merge demonstrations.

**Nodes Involved:**  
- On clicking 'execute'  
- A. Ingredients Needed  
- B. Ingredients in stock  
- A. Ingredients  
- B. Recipe quantities  
- A. Queen  
- B. Led Zeppelin

**Node Details:**

- **On clicking 'execute'**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point to start the workflow manually.  
  - *Config:* No parameters; triggers downstream nodes on execution.  
  - *Connections:* Outputs to all dataset nodes.  
  - *Edge cases:* None, manual trigger; potential user error if not clicked.  
  - *Notes:* Serves as starting point to demonstrate merging scenarios.

- **A. Ingredients Needed**  
  - *Type:* Code  
  - *Role:* Provides dataset A for the first merge scenario (ingredients needed for a recipe).  
  - *Config:* JavaScript returns an array of ingredient objects with a single "Name" property.  
  - *Output:* List of ingredients: Flour, Eggs, Milk, Lemon, Sugar.  
  - *Connections:* Output to "Ingredients in stock from recipe" node.  
  - *Edge cases:* Static data, no failure expected.

- **B. Ingredients in stock**  
  - *Type:* Code  
  - *Role:* Provides dataset B for the first merge scenario (ingredients currently in stock).  
  - *Config:* JavaScript returns an array of ingredient objects with "Name" properties.  
  - *Output:* Eggs, Lemon, Sugar.  
  - *Connections:* Output to "Ingredients in stock from recipe" node.  
  - *Edge cases:* Static data, no failure expected.

- **A. Ingredients**  
  - *Type:* Code  
  - *Role:* Provides dataset A for the second merge scenario (ingredients list).  
  - *Config:* JavaScript returns the same ingredient list as "A. Ingredients Needed".  
  - *Connections:* Output to "Merge recipe" node.  
  - *Edge cases:* Static data, no failure expected.

- **B. Recipe quantities**  
  - *Type:* Code  
  - *Role:* Provides dataset B for the second merge scenario (quantities corresponding to ingredients).  
  - *Config:* JavaScript returns an array with "Name" and "Quantity" properties.  
  - *Output:* Includes Flour (100g), Eggs (2), Salt (50g), Lemon (1), Sugar (6tbsp).  
  - *Connections:* Output to "Merge recipe" node.  
  - *Edge cases:* Static data, no failure expected.

- **A. Queen**  
  - *Type:* Code  
  - *Role:* Provides dataset A for the third merge scenario (band members of Queen).  
  - *Config:* JavaScript returns band member objects with first/last names, instrument, and optional superpower.  
  - *Connections:* Output to "Super Band" node.  
  - *Edge cases:* Static data, no failure expected.

- **B. Led Zeppelin**  
  - *Type:* Code  
  - *Role:* Provides dataset B for the third merge scenario (band members of Led Zeppelin).  
  - *Config:* JavaScript returns band member objects with first/last names, instruments, and optional second instruments.  
  - *Connections:* Output to "Super Band" node.  
  - *Edge cases:* Static data, no failure expected.

---

#### 2.2 Keep items from A if there's a match in B (Inner Join)

**Overview:**  
This block filters ingredients needed (A) to only those that are currently in stock (B), equivalent to an SQL INNER JOIN on "Name".

**Nodes Involved:**  
- Ingredients in stock from recipe (Merge node)  
- Note1 (Sticky Note)

**Node Details:**

- **Ingredients in stock from recipe**  
  - *Type:* Merge (mode: combine)  
  - *Role:* Combines datasets A and B by matching the "Name" field in both, keeping only items present in both.  
  - *Config:*  
    - Mode: Combine  
    - Merge by fields: ["Name" in A] = ["Name" in B]  
  - *Input connections:*  
    - Input 1: A. Ingredients Needed  
    - Input 2: B. Ingredients in stock  
  - *Output:* Only ingredients that exist in both lists (Eggs, Lemon, Sugar).  
  - *Edge cases:*  
    - If either dataset is empty, output will be empty.  
    - Case sensitivity or spelling differences in "Name" fields could cause mismatches.  
  - *Version:* Merge node version 2 is used for mode and mergeByFields support.  
  - *Sticky Note:* "1. Keep items from A if there's a match in B".

---

#### 2.3 Enrich items from A with matching data from B (Left Join)

**Overview:**  
This block enriches the ingredients list (A) with corresponding quantity data from the recipe quantities (B), similar to a SQL LEFT JOIN.

**Nodes Involved:**  
- Merge recipe (Merge node)  
- Note2 (Sticky Note)  
- Note6 (Sticky Note)

**Node Details:**

- **Merge recipe**  
  - *Type:* Merge (mode: combine)  
  - *Role:* Enriches dataset A with matching data from dataset B based on the "Name" field.  
  - *Config:*  
    - Mode: Combine  
    - Join mode: enrichInput1 (left join)  
    - Merge by fields: ["Name" in A] = ["Name" in B]  
  - *Input connections:*  
    - Input 1: A. Ingredients  
    - Input 2: B. Recipe quantities  
  - *Output:* Ingredients from A with quantities added where matches exist; unmatched remain with original data.  
  - *Edge cases:*  
    - Missing quantities for some ingredients (e.g., Milk) will leave those fields empty.  
    - Data type differences in Quantity fields should be handled downstream if needed.  
  - *Sticky Notes:*  
    - "2. Enrich items from A with matching data from B"  
    - "Adds the quantity needed to each ingredient in the recipe. Similar to SQL Left join."

---

#### 2.4 Add items from B below items from A (Union All)

**Overview:**  
This block appends all members of Led Zeppelin (B) below all members of Queen (A), akin to SQL UNION ALL but more flexible regarding fields.

**Nodes Involved:**  
- Super Band (Merge node)  
- Note9 (Sticky Note)

**Node Details:**

- **Super Band**  
  - *Type:* Merge (default mode)  
  - *Role:* Appends dataset B to dataset A without matching or filtering fields, effectively concatenating the two lists.  
  - *Config:* No specific merging fields or modes set, defaults to append behavior.  
  - *Input connections:*  
    - Input 1: A. Queen  
    - Input 2: B. Led Zeppelin  
  - *Output:* Combined list of all band members from both groups.  
  - *Edge cases:*  
    - Different field sets between datasets (e.g., "Superpower" vs "Second Instrument") are preserved individually.  
  - *Sticky Note:*  
    - "This will create a super band by merging Queen and Led Zeppelin. Similar to SQL Union All (more flexible as not requires all fields to be the same)."

---

### 3. Summary Table

| Node Name                     | Node Type       | Functional Role                              | Input Node(s)                               | Output Node(s)                      | Sticky Note                                                                                      |
|-------------------------------|-----------------|----------------------------------------------|---------------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| On clicking 'execute'          | Manual Trigger  | Entry point to start the workflow             | -                                           | A. Ingredients Needed, B. Ingredients in stock, A. Ingredients, B. Recipe quantities, A. Queen, B. Led Zeppelin |                                                                                                |
| A. Ingredients Needed          | Code            | Dataset A for filtering ingredients           | On clicking 'execute'                        | Ingredients in stock from recipe   |                                                                                                |
| B. Ingredients in stock        | Code            | Dataset B for filtering ingredients           | On clicking 'execute'                        | Ingredients in stock from recipe   |                                                                                                |
| Ingredients in stock from recipe| Merge (combine) | Filter ingredients needed to those in stock (inner join) | A. Ingredients Needed, B. Ingredients in stock | -                                 | "1. Keep items from A if there's a match in B"                                                 |
| A. Ingredients                | Code            | Dataset A for enriching ingredients            | On clicking 'execute'                        | Merge recipe                      |                                                                                                |
| B. Recipe quantities           | Code            | Dataset B with quantities for ingredients      | On clicking 'execute'                        | Merge recipe                      |                                                                                                |
| Merge recipe                  | Merge (combine) | Enrich ingredients list with quantities (left join) | A. Ingredients, B. Recipe quantities         | -                                 | "2. Enrich items from A with matching data from B" / "Adds the quantity needed to each ingredient in the recipe / Similar to SQL Left join" |
| A. Queen                     | Code            | Dataset A of Queen band members                 | On clicking 'execute'                        | Super Band                       |                                                                                                |
| B. Led Zeppelin               | Code            | Dataset B of Led Zeppelin band members          | On clicking 'execute'                        | Super Band                       |                                                                                                |
| Super Band                   | Merge           | Append band members from both datasets (union all) | A. Queen, B. Led Zeppelin                     | -                                 | "This will create a super band by merging Queen and Led Zeppelin / Similar to SQL Union All (more flexible as not requires all fields to be the same)" |
| Note                          | Sticky Note     | Annotation for block 3: Union all example      | -                                           | -                                 | "# 3. Add items from B below items from A"                                                     |
| Note1                         | Sticky Note     | Annotation for block 1: Inner join example     | -                                           | -                                 | "# 1. Keep items from A if there's a match in B"                                              |
| Note2                         | Sticky Note     | Annotation for block 2: Left join example      | -                                           | -                                 | "# 2. Enrich items from A with matching data from B"                                          |
| Note4                         | Sticky Note     | General explanation of the Merge node          | -                                           | -                                 | "# Aggregating data with the Merge node\n\n## The merge node is one of the most useful nodes in n8n. In this workflow we show how to merge data from two different sources (similar to SQL joins).\n\n## The most-used operations of the merge node are presented here. For more info, browse the [merge node docs](https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.merge/)\n\n## Click the `Execute Workflow` button and double click on the nodes to see the input and output items." |
| Note6                         | Sticky Note     | Annotation for left join example                | -                                           | -                                 | "Adds the quantity needed to each ingredient in the recipe\n\nSimilar to SQL Left join"       |
| Note8                         | Sticky Note     | Annotation for inner join example                | -                                           | -                                 | "This will keep only the ingredients needed that are also in stock\n\nSimilar to SQL Inner join" |
| Note9                         | Sticky Note     | Annotation for union all example                  | -                                           | -                                 | "This will create a super band by merging Queen and Led Zeppelin\n\nSimilar to SQL Union All \n(more flexible as not requires all fields to be the same)" |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On clicking 'execute'" node**  
   - Type: Manual Trigger  
   - Purpose: Entry point for manual execution.  
   - No parameters to configure.  

2. **Create datasets as Code nodes:**

   - **"A. Ingredients Needed"**  
     - Type: Code  
     - JavaScript:  
       ```js
       return [
         { Name: "Flour" },
         { Name: "Eggs" },
         { Name: "Milk" },
         { Name: "Lemon" },
         { Name: "Sugar" },
       ];
       ```  
     - Connect input from "On clicking 'execute'".

   - **"B. Ingredients in stock"**  
     - Type: Code  
     - JavaScript:  
       ```js
       return [
         { Name: "Eggs" },
         { Name: "Lemon" },
         { Name: "Sugar" },
       ];
       ```  
     - Connect input from "On clicking 'execute'".

   - **"A. Ingredients"**  
     - Type: Code  
     - JavaScript: Same as "A. Ingredients Needed" above.

   - **"B. Recipe quantities"**  
     - Type: Code  
     - JavaScript:  
       ```js
       return [
         { Name: "Flour", Quantity: "100g" },
         { Name: "Eggs", Quantity: 2 },
         { Name: "Salt", Quantity: "50g" },
         { Name: "Lemon", Quantity: 1 },
         { Name: "Sugar", Quantity: "6tbsp" },
       ];
       ```  
     - Connect input from "On clicking 'execute'".

   - **"A. Queen"**  
     - Type: Code  
     - JavaScript:  
       ```js
       return [
         { FirstName: "John", LastName: "Deacon", Instrument: "Drums" },
         { FirstName: "Freddy", LastName: "Mercury", Instrument: "Vocals and Piano", Superpower: "Crowd control" },
         { FirstName: "Brian", LastName: "May", Instrument: "Guitar" },
         { FirstName: "Roger", LastName: "Taylor", Instrument: "Bass" },
       ];
       ```  
     - Connect input from "On clicking 'execute'".

   - **"B. Led Zeppelin"**  
     - Type: Code  
     - JavaScript:  
       ```js
       return [
         { FirstName: "Jimmy", LastName: "Page", Instrument: "Guitar" },
         { FirstName: "Robert", LastName: "Plant", Instrument: "Vocals" },
         { FirstName: "John", LastName: "Bonham", Instrument: "Drums" },
         { FirstName: "John", LastName: "Paul Jones", Instrument: "Bass", "Second Instrument": "Keyboard" },
       ];
       ```  
     - Connect input from "On clicking 'execute'".

3. **Create Merge nodes for each use case:**

   - **"Ingredients in stock from recipe"**  
     - Type: Merge (version 2)  
     - Mode: Combine  
     - Merge By Fields: Match "Name" from Input 1 and Input 2  
     - Inputs:  
       - Input 1: "A. Ingredients Needed"  
       - Input 2: "B. Ingredients in stock"  
     - Purpose: Keep only items from A that have a match in B (inner join).

   - **"Merge recipe"**  
     - Type: Merge (version 2)  
     - Mode: Combine  
     - Join mode: enrichInput1 (left join)  
     - Merge By Fields: Match "Name" fields  
     - Inputs:  
       - Input 1: "A. Ingredients"  
       - Input 2: "B. Recipe quantities"  
     - Purpose: Enrich items from A with matching data from B.

   - **"Super Band"**  
     - Type: Merge (version 2)  
     - Mode: default (append)  
     - Inputs:  
       - Input 1: "A. Queen"  
       - Input 2: "B. Led Zeppelin"  
     - Purpose: Append B items below A items (union all).

4. **Add Sticky Notes for explanations** (optional but recommended for clarity):  
   - Add notes near respective blocks explaining their function and SQL analogy.  
   - Include the provided descriptions and link to [Merge node docs](https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.merge/).

5. **Connect nodes accordingly:**  
   - "On clicking 'execute'" → All six code nodes  
   - Code nodes → Corresponding Merge nodes as inputs described above  

6. **Execute workflow and inspect outputs:**  
   - Run manually via "On clicking 'execute'"  
   - Check each Merge node output to verify expected join behavior.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                         |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------|
| The Merge node is one of the most useful nodes in n8n for combining datasets with SQL-like join operations. | [Merge node docs](https://docs.n8n.io/integrations/core-nodes/n8n-nodes-base.merge/) |
| This workflow demonstrates practical examples of inner joins, left joins, and union all operations.        | Workflow description and sticky notes within the workflow |
| Click the `Execute Workflow` button and double click nodes to view input/output for understanding.         | Instruction for users working with the workflow         |

---

This detailed reference document fully covers the workflow’s structure, node configurations, logical flow, and practical guidance for recreation or modification. It is designed to enable advanced users and automation agents to handle merging datasets with confidence in n8n.