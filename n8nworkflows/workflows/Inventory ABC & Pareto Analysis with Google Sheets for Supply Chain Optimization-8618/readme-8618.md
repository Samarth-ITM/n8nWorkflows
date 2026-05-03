Inventory ABC & Pareto Analysis with Google Sheets for Supply Chain Optimization

https://n8nworkflows.xyz/workflows/inventory-abc---pareto-analysis-with-google-sheets-for-supply-chain-optimization-8618


# Inventory ABC & Pareto Analysis with Google Sheets for Supply Chain Optimization

### 1. Workflow Overview

This workflow, titled **"Inventory ABC & Pareto Analysis with Google Sheets for Supply Chain Optimization"**, is designed to analyze inventory sales data from Google Sheets, perform ABC and Pareto analyses, and update analytical results back into Google Sheets for supply chain decision-making. Its primary use case is to help supply chain managers and analysts optimize inventory by categorizing items based on sales contribution and demand variability.

The workflow logically breaks down into these key blocks:

- **1.1 Trigger & Data Retrieval:** Manual start and fetching raw sales data from a Google Sheets document.
- **1.2 Data Filtering & Aggregation:** Cleans data by removing zero sales and aggregates sales by store and day.
- **1.3 Data Transformation & Grouping:** Transposes data and groups sales by items and stores preparing for analysis.
- **1.4 Analytical Computations:** Performs Pareto analysis, demand variability computations, and ABC classification.
- **1.5 Results Export:** Writes back the processed analysis results into specific Google Sheets tabs organized by store and analysis type.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger & Data Retrieval

**Overview:**  
Starts the workflow manually and retrieves the raw transactional sales data from a defined Google Sheet document and sheet tab.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Get row(s) in sheet (Google Sheets)

**Node Details:**  

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Initiates the workflow manually  
  - Config: No parameters needed  
  - Input: None  
  - Output: Triggers next node  
  - Failures: None generally, but workflow won’t start unless manually triggered

- **Get row(s) in sheet**  
  - Type: Google Sheets (Read rows)  
  - Role: Reads all rows from the “Input Data” sheet in the “Data Analytics” Google Sheets document  
  - Config:  
    - Document ID: Linked credential required (Google Sheets OAuth2)  
    - Sheet Name: “Input Data” (gid=0)  
  - Inputs: Trigger node output  
  - Outputs: JSON array of all rows in the sheet  
  - Failures: Authentication errors if credentials invalid, API rate limits, empty or missing sheet errors  
  - Notes: Requires valid credentials and correct sheet access

---

#### 2.2 Data Filtering & Aggregation

**Overview:**  
Filters out rows where quantity sold (QTY) is zero and aggregates sales data daily for each store.

**Nodes Involved:**  
- Filter Out Zero Sales (Filter)  
- Demand Variability x Sales % (Code)  
- TO, QTY GroupBy ITEM (Code)  
- TO GroupBy (STORE, ITEM) (Code)  
- Daily Sales per Store (Code)  
- If (Conditional)  

**Node Details:**  

- **Filter Out Zero Sales**  
  - Type: Filter  
  - Role: Removes rows where QTY equals 0  
  - Config: Condition QTY != 0 (number comparison)  
  - Input: Rows from Google Sheets  
  - Output: Filtered rows with QTY > 0  
  - Failures: Expression errors if QTY missing or non-numeric

- **Demand Variability x Sales %**  
  - Type: Code (JavaScript)  
  - Role: Calculates total quantity, share percentage, mean, standard deviation, and coefficient of variation (CV) for each item’s daily sales (summed across stores)  
  - Config: Custom JS that aggregates by item and day, computes statistics, sorts by share  
  - Input: Filtered sales rows  
  - Output: Items with demand variability metrics  
  - Failures: Possible divide-by-zero if no data, missing fields

- **TO, QTY GroupBy ITEM**  
  - Type: Code (JavaScript)  
  - Role: Aggregates turnover (TO) and quantity (QTY) summed by item  
  - Config: Custom JS that accumulates sums and sorts items by TO descending  
  - Input: Filtered sales rows  
  - Output: Aggregated items with total TO and QTY  
  - Failures: Data type issues if TO or QTY missing or invalid

- **TO GroupBy (STORE, ITEM)**  
  - Type: Code (JavaScript)  
  - Role: Groups and sums turnover (TO) by store and item, outputs a matrix with stores as rows and items as columns (wide format)  
  - Config: Builds sets of unique stores and SKUs, fills zeros for missing combinations  
  - Input: Filtered sales rows  
  - Output: Transformed data suitable for multi-store analysis  
  - Failures: Missing STORE or ITEM fields, potential performance issue on very large datasets

- **Daily Sales per Store**  
  - Type: Code (JavaScript)  
  - Role: Aggregates daily sales quantity and turnover per store and day  
  - Config: Sums QTY and TO grouped by STORE and DAY  
  - Input: Filtered sales rows  
  - Output: Aggregated daily store sales  
  - Failures: Missing DAY or STORE fields, numeric conversion errors

- **If**  
  - Type: Conditional (If)  
  - Role: Routes data based on STORE value, specifically handling “STORE-1” differently  
  - Config: Checks if STORE equals “STORE-1”  
  - Input: Aggregated daily sales per store  
  - Output:  
    - True branch: To “Sales Store 1” node  
    - False branch: To “Sales Store 2” node  
  - Failures: Missing STORE field or unexpected STORE values

---

#### 2.3 Data Transformation & Grouping

**Overview:**  
Transforms the store-item sales matrix from long to wide format and prepares data for multi-store sales analysis and further processing.

**Nodes Involved:**  
- Transpose (Code)  
- Multi-Store Sales (Google Sheets)

**Node Details:**  

- **Transpose**  
  - Type: Code (JavaScript)  
  - Role: Transposes data from store-keyed rows to item-keyed rows with store columns, effectively pivoting the data  
  - Config: Custom JS that iterates stores and items, rearranging the data shape  
  - Input: Output of TO GroupBy (STORE, ITEM)  
  - Output: Transposed sales matrix  
  - Failures: Assumes consistent STORE and ITEM keys, may break if data shape varies

- **Multi-Store Sales**  
  - Type: Google Sheets (Append rows)  
  - Role: Appends multi-store sales data (TO, DAY, QTY) to a designated sheet (“Daily Sales Store 1”) in “Data Analytics” document  
  - Config:  
    - Document ID: Same as input sheet  
    - Sheet Name: Daily Sales Store 1 (by gid)  
    - Columns: TO, DAY, QTY mapped from JSON fields  
    - Operation: Append  
  - Input: Transposed data from “Transpose”  
  - Output: None (side effect: updates sheet)  
  - Failures: Authentication errors, API limits, schema mismatches

---

#### 2.4 Analytical Computations

**Overview:**  
Performs Pareto analysis to rank SKUs by turnover, assigns ABC classes based on cumulative sales contribution, and prepares data for export.

**Nodes Involved:**  
- Pareto Analysis (Code)  
- Update Pareto Sheet (Google Sheets)  
- ABC Class Mapping (Code)  
- ABC XYZ Analysis (Google Sheets)

**Node Details:**  

- **Pareto Analysis**  
  - Type: Code (JavaScript)  
  - Role: Sorts items by turnover, computes cumulative turnover, cumulative share, SKU rank, and cumulative SKU percentages for Pareto charting  
  - Config: JS that normalizes data, calculates cumulative metrics, and ranks SKUs  
  - Input: Aggregated item sales (TO, QTY) from “TO, QTY GroupBy ITEM”  
  - Output: Enriched items with Pareto metrics  
  - Failures: Numeric conversion errors, missing TO or QTY fields

- **Update Pareto Sheet**  
  - Type: Google Sheets (Append rows)  
  - Role: Writes Pareto analysis results into “Pareto” sheet in the “Data Analytics” document  
  - Config:  
    - Columns: Mapped fields for TO, QTY, ITEM, cumulative metrics  
    - Operation: Append  
    - Sheet Name: Pareto (by gid)  
  - Input: Output of Pareto Analysis  
  - Output: None (side effect: sheet updated)  
  - Failures: API limits, auth errors, incorrect mapping

- **ABC Class Mapping**  
  - Type: Code (JavaScript)  
  - Role: Assigns ABC class labels (A, B, C) based on cumulative sales share thresholds (A: top 5%, B: next 15%, C: rest), adds cumulative share field  
  - Config: Sorts items by share, accumulates share, and applies class mapping  
  - Input: Demand variability and sales % data from “Demand Variability x Sales %”  
  - Output: Items enriched with ABC class and cumulative share  
  - Failures: Sorting or numeric errors if share_pct missing or invalid

- **ABC XYZ Analysis**  
  - Type: Google Sheets (Append rows)  
  - Role: Exports ABC classification results and demand variability metrics to the “ABC XYZ” sheet  
  - Config:  
    - Columns: ABC, ITEM, cv_qty, std_qty, mean_qty, cum_share, qty_total, share_qty_pct  
    - Operation: Append  
    - Sheet Name: ABC XYZ (by gid)  
  - Input: Output of ABC Class Mapping  
  - Output: None (side effect: updates sheet)  
  - Failures: Auth errors, API limits, incorrect field mapping

---

#### 2.5 Results Export by Store

**Overview:**  
Saves aggregated daily sales data separately for Store 1 and Store 2 into respective Google Sheets tabs.

**Nodes Involved:**  
- Sales Store 1 (Google Sheets)  
- Sales Store 2 (Google Sheets)

**Node Details:**  

- **Sales Store 1**  
  - Type: Google Sheets (Append rows)  
  - Role: Append daily sales data for STORE-1 (TO, DAY, QTY) into “Daily Sales Store 1” sheet  
  - Config:  
    - Document ID: Same as input  
    - Sheet Name: Daily Sales Store 1 (gid)  
    - Columns: TO, DAY, QTY mapped from JSON  
  - Input: True branch from “If” node (STORE == STORE-1)  
  - Output: None (side effect: sheet updated)  
  - Failures: Auth errors, API limits, missing STORE-1 data

- **Sales Store 2**  
  - Type: Google Sheets (Append rows)  
  - Role: Append daily sales data for stores other than STORE-1 into “Daily Sales Store 2” sheet  
  - Config:  
    - Document ID: Same  
    - Sheet Name: Daily Sales Store 2 (gid)  
    - Columns: TO, DAY, QTY  
  - Input: False branch from “If” node  
  - Output: None (side effect: sheet updated)  
  - Failures: Same as Sales Store 1

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                               | Input Node(s)           | Output Node(s)                   | Sticky Note                                                                                          |
|-------------------------|---------------------|-----------------------------------------------|-------------------------|---------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger      | Initiates workflow manually                    | None                    | Get row(s) in sheet             |                                                                                                    |
| Get row(s) in sheet      | Google Sheets       | Fetches raw sales data from “Input Data” sheet | When clicking ‘Execute workflow’ | Filter Out Zero Sales           | ### Trigger the workflow<br>This starts by collecting the transactional sales from the spreadsheet `Input Data` from the Google Sheet.<br><br>#### How to setup?<br>- **Load records in the Google Sheet Node**:<br>   1. Add your Google Sheet API credentials to access the Google Sheet file<br>   2. Select the file using the list, an URL or an ID<br>   3. Select the sheet `Input Data`<br> [Learn more about the Google Sheet Node](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets) |
| Filter Out Zero Sales    | Filter              | Filters out rows with zero quantity sold      | Get row(s) in sheet      | Demand Variability x Sales %, TO, QTY GroupBy ITEM, TO GroupBy (STORE, ITEM), Daily Sales per Store |                                                                                                    |
| Demand Variability x Sales % | Code (JavaScript) | Computes demand variability and sales share   | Filter Out Zero Sales    | ABC Class Mapping               |                                                                                                    |
| TO, QTY GroupBy ITEM     | Code (JavaScript)   | Aggregates turnover and quantity by item      | Filter Out Zero Sales    | Pareto Analysis                |                                                                                                    |
| TO GroupBy (STORE, ITEM) | Code (JavaScript)   | Aggregates turnover by store and item, outputs wide format | Filter Out Zero Sales    | Transpose                     |                                                                                                    |
| Daily Sales per Store    | Code (JavaScript)   | Aggregates daily sales per store and day       | Filter Out Zero Sales    | If                            |                                                                                                    |
| If                      | Conditional (If)    | Routes data based on STORE value               | Daily Sales per Store    | Sales Store 1, Sales Store 2  |                                                                                                    |
| Sales Store 1            | Google Sheets       | Appends daily sales for STORE-1                | If                      | None                         |                                                                                                    |
| Sales Store 2            | Google Sheets       | Appends daily sales for other stores           | If                      | None                         |                                                                                                    |
| Transpose                | Code (JavaScript)   | Transposes sales matrix from store-row to item-row | TO GroupBy (STORE, ITEM) | Multi-Store Sales            |                                                                                                    |
| Multi-Store Sales        | Google Sheets       | Appends multi-store sales data                  | Transpose                | None                         |                                                                                                    |
| Pareto Analysis          | Code (JavaScript)   | Calculates Pareto cumulative metrics and ranking | TO, QTY GroupBy ITEM     | Update Pareto Sheet            |                                                                                                    |
| Update Pareto Sheet      | Google Sheets       | Writes Pareto analysis results into sheet       | Pareto Analysis          | None                         |                                                                                                    |
| ABC Class Mapping        | Code (JavaScript)   | Assigns ABC class based on cumulative sales share | Demand Variability x Sales % | ABC XYZ Analysis            |                                                                                                    |
| ABC XYZ Analysis         | Google Sheets       | Writes ABC classification and variability results | ABC Class Mapping        | None                         |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Google Sheets Node "Get row(s) in sheet"**  
   - Operation: Read rows  
   - Credentials: Set Google Sheets OAuth2 credentials with access to your target spreadsheet  
   - Document ID: Select or enter your spreadsheet ID (named "Data Analytics")  
   - Sheet Name: Select the sheet tab named "Input Data" (gid=0)  
   - Connect output from Manual Trigger to this node.

3. **Add Filter Node "Filter Out Zero Sales"**  
   - Condition: Numeric field `QTY` not equal to 0  
   - Connect input from "Get row(s) in sheet".

4. **Add Code Node "Demand Variability x Sales %"**  
   - Paste provided JavaScript code that calculates mean, std dev, coefficient of variation, and percentage share by ITEM over days  
   - Connect input from "Filter Out Zero Sales".

5. **Add Code Node "TO, QTY GroupBy ITEM"**  
   - Paste provided JavaScript code that sums turnover (TO) and quantity (QTY) grouped by ITEM, sorted by TO descending  
   - Connect input from "Filter Out Zero Sales".

6. **Add Code Node "Pareto Analysis"**  
   - Paste provided JavaScript code that computes cumulative turnover, cumulative share, SKU rank, and cumulative SKU percentages for Pareto analysis  
   - Connect input from "TO, QTY GroupBy ITEM".

7. **Add Google Sheets Node "Update Pareto Sheet"**  
   - Operation: Append rows  
   - Credentials: Use same Google Sheets OAuth2 credentials  
   - Document ID: Same spreadsheet as before  
   - Sheet Name: Select "Pareto" sheet tab (by gid)  
   - Mapping: Map TO, QTY, ITEM, cum_skus, sku_rank, cum_share, cum_skus_pct, cum_turnover from incoming JSON fields  
   - Connect input from "Pareto Analysis".

8. **Add Code Node "TO GroupBy (STORE, ITEM)"**  
   - Paste provided JavaScript code that aggregates turnover by STORE and ITEM, outputs wide matrix  
   - Connect input from "Filter Out Zero Sales".

9. **Add Code Node "Transpose"**  
   - Paste provided JavaScript code transposing the STORE-ITEM matrix  
   - Connect input from "TO GroupBy (STORE, ITEM)".

10. **Add Google Sheets Node "Multi-Store Sales"**  
    - Operation: Append rows  
    - Credentials: Same Google Sheets OAuth2 credentials  
    - Document ID: Same spreadsheet  
    - Sheet Name: "Daily Sales Store 1" (gid specified)  
    - Columns: Map TO, DAY, QTY  
    - Connect input from "Transpose".

11. **Add Code Node "Daily Sales per Store"**  
    - Paste provided JS code that aggregates daily sales quantity and turnover per STORE and DAY  
    - Connect input from "Filter Out Zero Sales".

12. **Add Conditional Node "If"**  
    - Condition: `$json.STORE` equals "STORE-1"  
    - Connect input from "Daily Sales per Store".

13. **Add Google Sheets Node "Sales Store 1"**  
    - Operation: Append rows  
    - Credentials: Same Google Sheets OAuth2 credentials  
    - Document ID: Same spreadsheet  
    - Sheet Name: "Daily Sales Store 1"  
    - Columns: TO, DAY, QTY  
    - Connect True output from "If".

14. **Add Google Sheets Node "Sales Store 2"**  
    - Operation: Append rows  
    - Credentials: Same OAuth2 credentials  
    - Document ID: Same spreadsheet  
    - Sheet Name: "Daily Sales Store 2"  
    - Columns: TO, DAY, QTY  
    - Connect False output from "If".

15. **Add Code Node "ABC Class Mapping"**  
    - Paste provided JavaScript code that assigns ABC classes based on cumulative sales share thresholds  
    - Connect input from "Demand Variability x Sales %".

16. **Add Google Sheets Node "ABC XYZ Analysis"**  
    - Operation: Append rows  
    - Credentials: Same Google Sheets OAuth2 credentials  
    - Document ID: Same spreadsheet  
    - Sheet Name: "ABC XYZ"  
    - Columns: Map ABC, ITEM, cv_qty, std_qty, mean_qty, cum_share, qty_total, share_qty_pct  
    - Connect input from "ABC Class Mapping".

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                     | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow starts by collecting transactional sales data from the `Input Data` tab of the Google Sheets document “Data Analytics.”                                         | Sticky Note in workflow and Google Sheets Node docs: https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets |
| Use your Google Sheets API credentials with OAuth2 configured properly for all Google Sheets nodes to ensure seamless data access and writing.                                 | n8n documentation for Google Sheets OAuth2 setup                                                             |
| The Pareto and ABC classification thresholds are set as: A = top 5%, B = next 15%, C = remaining 80%. These can be adjusted in the “ABC Class Mapping” code node if needed.    | Business logic embedded in JS code node                                                                       |
| The workflow handles two stores explicitly by routing STORE-1 sales separately from others for individual sheet writes. Expand conditional and sheets to support more stores.  | Logic in “If” node and separate Google Sheets nodes                                                          |
| The workflow assumes consistent data columns: STORE, ITEM, DAY, QTY, and TO (turnover). Missing or malformed data may cause errors in JS code nodes.                          | Data validation recommended before running workflow                                                          |

---

**Disclaimer:** The text provided is extracted exclusively from an automated n8n workflow. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.