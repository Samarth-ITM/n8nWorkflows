Benchmark invoice data extraction accuracy with easybits

https://n8nworkflows.xyz/workflows/benchmark-invoice-data-extraction-accuracy-with-easybits-14957


# Benchmark invoice data extraction accuracy with easybits

# Workflow Analysis: Benchmark Invoice Data Extraction Accuracy with easybits

### 1. Workflow Overview
The primary purpose of this workflow is to perform a "stress test" on document extraction accuracy. It allows a user to upload an invoice (in various qualities/formats) and automatically compares the extracted data against a predefined "Ground Truth" (the known correct values). The workflow calculates a per-field success rate and an overall accuracy percentage, presenting the results directly to the user via a web form completion page.

The logic is organized into the following functional blocks:
- **1.1 Input Reception:** Captures the document via a web-based form.
- **1.2 AI Processing:** Extracts structured data from the document using the easybits Extractor.
- **1.3 Reference Data Management:** Provides the expected correct values for comparison.
- **1.4 Data Alignment & Validation:** Merges the extracted data with the ground truth and runs a logical comparison.
- **1.5 Results Reporting:** Displays the final accuracy report to the end-user.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception
**Overview:** This block serves as the entry point, providing a user interface to upload the invoice file.
- **Nodes Involved:** `Document Upload`
- **Node Details:**
    - **Type:** Form Trigger
    - **Configuration:** A form titled "Document Upload Form" with a single file upload field.
    - **Response Mode:** Set to "Last Node" (essential so the browser waits for the final result before redirecting/closing).
    - **Input/Output:** Starts the workflow; passes the binary file to the next node.

#### 2.2 AI Processing
**Overview:** This block leverages the easybits Extractor to convert an unstructured document into structured JSON data.
- **Nodes Involved:** `easybits: Data Extraction`
- **Node Details:**
    - **Type:** easybits Extractor (Community Node)
    - **Configuration:** Utilizes a Pipeline ID and API Key (defined in credentials).
    - **Input/Output:** Receives the binary file from the Form Trigger; outputs structured fields (e.g., `invoice_number`, `total_amount`) under the `data` object.
    - **Potential Failures:** API authentication errors, timeout on large files, or pipeline configuration mismatches on the easybits platform.

#### 2.3 Reference Data Management
**Overview:** Defines the "Gold Standard" values that the extracted data must match to be considered correct.
- **Nodes Involved:** `Ground Truth`
- **Node Details:**
    - **Type:** Set Node
    - **Configuration:** Manually defines 10 specific fields prefixed with `gt_` (e.g., `gt_total_amount: 73.82`, `gt_vendor_name: "easybits GmbH"`).
    - **Input/Output:** Triggered by the extraction node; outputs the static reference dataset.

#### 2.4 Data Alignment & Validation
**Overview:** This block synchronizes the extracted results with the ground truth and calculates the accuracy metrics.
- **Nodes Involved:** `Merge Ground Truth`, `Validation`
- **Node Details:**
    - **Merge Ground Truth:** 
        - **Type:** Merge Node. 
        - **Configuration:** Set to "Combine by Position." It merges the output of the extractor and the ground truth node into a single JSON object.
    - **Validation:**
        - **Type:** Set Node.
        - **Key Expressions:** Uses ternary operators to compare extracted values vs. ground truth. Example: `{{ $json.data.currency === "EUR" ? "correct ✅" : "incorrect ❌" }}`.
        - **Accuracy Calculation:** A mathematical expression that sums the number of correct fields (1 if correct, 0 if incorrect), divides by 10, and multiplies by 100 to get a percentage.
        - **Edge Cases:** Data type mismatches (e.g., comparing a string "73.82" to a number 73.82) could result in "incorrect" statuses.

#### 2.5 Results Reporting
**Overview:** Returns the final calculated report back to the user's browser.
- **Nodes Involved:** `Results in Form`
- **Node Details:**
    - **Type:** Form Node (Completion)
    - **Configuration:** Operation set to "Completion." The message is a dynamic HTML string that lists the `accuracy_percent` and the status of every individual field.
    - **Input/Output:** Receives validated data from the Validation node; outputs the final HTML page to the user.

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Document Upload | Form Trigger | Input UI | - | easybits: Data Extraction | Hosts a simple web form with a single file upload field. Set to "Workflow Finishes" mode. |
| easybits: Data Extraction | easybitsExtractor | Data Extraction | Document Upload | Merge Ground Truth, Ground Truth | Sends the uploaded file to the easybits Extractor API. Returns structured fields under `json.data`. |
| Ground Truth | Set | Reference Data | easybits: Data Extraction | Merge Ground Truth | Holds the known correct values for all 10 fields, prefixed with `gt_`. |
| Merge Ground Truth | Merge | Data Alignment | easybits: Data Extraction, Ground Truth | Validation | Combines the extracted data (Input 1) with the ground truth values (Input 2). |
| Validation | Set | Accuracy Logic | Merge Ground Truth | Results in Form | Compares each extracted field against its ground truth value. Outputs status and overall %. |
| Results in Form | Form | Output UI | Validation | - | Displays the final report as a form completion page. |

---

### 4. Reproducing the Workflow from Scratch

1.  **Create the Entry Point:**
    - Add a **Form Trigger** node named `Document Upload`.
    - Set the Form Title to "Document Upload Form".
    - Add a field: Type = `File`, Label = `Upload`.
    - Set **Response Mode** to "Last Node".

2.  **Configure Extraction:**
    - Add the **easybits Extractor** community node.
    - Connect it to the Form Trigger.
    - Setup credentials using your **API Key** and **Pipeline ID** from `extractor.easybits.tech`.

3.  **Define Reference Data:**
    - Add a **Set** node named `Ground Truth`.
    - Create 10 assignments with the `gt_` prefix (e.g., `gt_amount_due` = 0, `gt_vendor_name` = "easybits GmbH").
    - Connect the easybits node to this node.

4.  **Merge Data Streams:**
    - Add a **Merge** node named `Merge Ground Truth`.
    - Set Mode to **Combine** $\rightarrow$ **Combine by Position**.
    - Connect **Input 1** to the `easybits: Data Extraction` node.
    - Connect **Input 2** to the `Ground Truth` node.

5.  **Implement Validation Logic:**
    - Add a **Set** node named `Validation`.
    - Create 10 string assignments for statuses (e.g., `amount_due_status`) using the expression: `{{ $json.data.amount_due === 0 ? "correct ✅" : "incorrect ❌" }}`.
    - Create one number assignment `accuracy_percent` using the sum of matches divided by 10 multiplied by 100.

6.  **Setup Result Display:**
    - Add a **Form** node named `Results in Form`.
    - Set operation to **Completion**.
    - In the **Completion Message**, use HTML and expressions to display the results (e.g., `Extraction Accuracy: {{ $json.accuracy_percent }}%`).
    - Connect the Validation node to this node.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
| :--- | :--- |
| Pipeline Creation Guide | [extractor.easybits.tech](https://extractor.easybits.tech/) |
| Example Invoices for Testing | [GitHub Repository](https://github.com/felix-sattler-easybits/n8n-workflows/tree/88d7c9818b150e71dd749bf9f665359fa57efcb9/data-extraction-stresstest) |
| Benchmarking Tip | To test other AI solutions, replace the easybits node with an HTTP Request node that returns the same JSON structure under `data`. |