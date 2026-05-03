Customer Personality (BaZi, DISC, Numerology) with Gemini & Google Sheets

https://n8nworkflows.xyz/workflows/customer-personality--bazi--disc--numerology--with-gemini---google-sheets-8051


# Customer Personality (BaZi, DISC, Numerology) with Gemini & Google Sheets

### 1. Workflow Overview

This workflow automates the process of analyzing customer personality traits using three methodologies: BaZi (Chinese astrology), DISC (behavioral assessment), and Numerology. It integrates Google Sheets and Google Gemini (an AI language model) to fetch customer data, compute personality metrics, perform AI-driven analysis, and update results back into multiple Google Sheets tabs. The workflow is designed for businesses seeking to enrich their customer profiles with deep personality insights to enhance sales and marketing strategies.

The workflow’s logic is grouped into these functional blocks:

- **1.1 Input Reception & Filtering:** Triggered by updates in Google Sheets, filters relevant rows for processing.  
- **1.2 Computation of Personality Metrics:** Calculates BaZi and Numerology scores programmatically.  
- **1.3 AI-Based Personality Analysis:** Uses Google Gemini AI nodes to analyze DISC, BaZi, Numerology, and synthesize a customer insight summary.  
- **1.4 Result Aggregation & Sheet Updates:** Merges AI outputs, aggregates data, and updates multiple Google Sheets tabs accordingly.  
- **1.5 Iterative Row Processing:** Splits batch processing for scalability and sequential data handling.  

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Filtering

**Overview:**  
This block triggers the workflow when changes occur in the connected Google Sheet. It filters incoming rows to identify which require processing.

**Nodes Involved:**  
- Google Sheets Trigger  
- Filter  

**Node Details:**  
- **Google Sheets Trigger**  
  - Type: Trigger node for Google Sheets  
  - Role: Listens for new or updated rows in a configured Google Sheet.  
  - Config: Uses default trigger settings for the connected spreadsheet.  
  - Inputs: None (trigger)  
  - Outputs: Passes triggered rows to Filter node  
  - Edge Cases: Trigger may fire on irrelevant changes; hence filtering needed. Authentication errors possible if OAuth expires.

- **Filter**  
  - Type: Conditional filter node  
  - Role: Applies criteria to decide which rows proceed for analysis.  
  - Config: Conditions likely based on presence of required fields or flags in sheet rows.  
  - Inputs: Rows from Google Sheets Trigger  
  - Outputs: Passes filtered rows to the Compute Bazi node  
  - Edge Cases: Expression failures if fields missing; no output if no rows match criteria.

#### 2.2 Computation of Personality Metrics

**Overview:**  
This block programmatically computes BaZi and Numerology values for each filtered row before AI analysis.

**Nodes Involved:**  
- Compute Bazi (Code)  
- Compute Numerology (Code)  
- Loop Over Items (SplitInBatches)  

**Node Details:**  
- **Compute Bazi**  
  - Type: Function (Code) node  
  - Role: Calculates BaZi values from customer birth data or input parameters.  
  - Config: Custom JavaScript code implementing BaZi logic.  
  - Inputs: Rows passing Filter node  
  - Outputs: Passes augmented data to Compute Numerology  
  - Edge Cases: Code errors if input data malformed; missing birthdate fields cause failures.

- **Compute Numerology**  
  - Type: Function (Code) node  
  - Role: Computes numerology numbers based on customer data.  
  - Config: Custom JavaScript code for numerology calculations.  
  - Inputs: Output from Compute Bazi  
  - Outputs: Passes data to Loop Over Items and also triggers an update to sheet4 for intermediate results  
  - Edge Cases: Same as Compute Bazi; possible calculation errors on invalid input.

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes data iteratively in manageable batches for downstream nodes.  
  - Config: Batch size not explicitly specified, defaults apply.  
  - Inputs: Output from Compute Numerology  
  - Outputs: Connects to Get row(s) in sheet and conditional If nodes  
  - Edge Cases: Long batches may cause timeouts; improper batch size may affect performance.

#### 2.3 AI-Based Personality Analysis

**Overview:**  
Uses Google Gemini AI nodes to perform natural language analysis of the computed personality metrics (DISC, BaZi, Numerology) and generate a customer insight summary and sales script.

**Nodes Involved:**  
- If, If1, If2 (Conditional branching)  
- DISC Analysis (Google Gemini)  
- Bazi Analysis (Google Gemini)  
- Numerology analysis (Google Gemini)  
- Summary Customer Insight (Google Gemini)  
- Writing Script for Salers (Google Gemini)  

**Node Details:**  
- **If, If1, If2**  
  - Type: Conditional nodes  
  - Role: Branch processing based on availability or type of analysis required (DISC, BaZi, Numerology).  
  - Config: Conditions likely check flags or field values to route to corresponding AI nodes.  
  - Inputs: Output from Loop Over Items or Get row(s) in sheet  
  - Outputs: Sends data to respective AI analysis nodes or directly to Merge  
  - Edge Cases: Condition misconfiguration could skip analyses or cause dead ends.

- **DISC Analysis**  
  - Type: Google Gemini AI node  
  - Role: Analyzes customer DISC personality traits from input data.  
  - Config: Prompt templates feed computed DISC data; expects textual output.  
  - Inputs: Output from If node  
  - Outputs: Passes results to Update row in sheet3  
  - Edge Cases: API authentication errors, rate limits, or generation errors.

- **Bazi Analysis**  
  - Type: Google Gemini AI node  
  - Role: Analyzes BaZi personality aspects.  
  - Config: Uses BaZi computed data as prompt input.  
  - Inputs: Output from If1 node  
  - Outputs: Passes results to Update row in sheet2  
  - Edge Cases: Same as DISC Analysis.

- **Numerology analysis**  
  - Type: Google Gemini AI node  
  - Role: Analyzes numerology results.  
  - Config: Feeds numerology computation to AI for interpretation.  
  - Inputs: Output from If2 node  
  - Outputs: Passes results to Update row in sheet1  
  - Edge Cases: Same as above.

- **Summary Customer Insight**  
  - Type: Google Gemini AI node  
  - Role: Combines insights from all analyses into a unified customer personality summary.  
  - Config: Uses data from Get row(s) in sheet to generate summary.  
  - Inputs: Output from Get row(s) in sheet  
  - Outputs: Passes result to Writing Script for Salers  
  - Edge Cases: Could fail if input data incomplete or Gemini API issues.

- **Writing Script for Salers**  
  - Type: Google Gemini AI node  
  - Role: Creates a customized sales script based on customer insights.  
  - Config: Input is the customer insight summary.  
  - Inputs: Output from Summary Customer Insight  
  - Outputs: Passes final script to Update row in sheet  
  - Edge Cases: Same API-related failure modes.

#### 2.4 Result Aggregation & Sheet Updates

**Overview:**  
This block merges AI analysis outputs, aggregates data, and updates respective rows in multiple Google Sheets tabs to store the results.

**Nodes Involved:**  
- Update row in sheet1  
- Update row in sheet2  
- Update row in sheet3  
- Update row in sheet4  
- Update row in sheet  
- Merge  
- Aggregate  
- Get row(s) in sheet  

**Node Details:**  
- **Update row in sheet1 / sheet2 / sheet3 / sheet4 / sheet**  
  - Type: Google Sheets node  
  - Role: Writes analysis results back into different sheets or tabs for Numerology, BaZi, DISC, intermediate calculations, and final sales script respectively.  
  - Config: Each targets a specific sheet/tab and updates particular rows based on input data.  
  - Inputs: Corresponding AI nodes or Compute Numerology node  
  - Outputs: Feed into Merge or end the workflow branch  
  - Edge Cases: Auth errors, invalid row references, or write conflicts.

- **Merge**  
  - Type: Merge node  
  - Role: Combines outputs from multiple update operations into a single stream.  
  - Config: Default merge settings with multiple inputs.  
  - Inputs: Update row in sheet1/2/3 outputs  
  - Outputs: Passes combined data to Aggregate  
  - Edge Cases: Data mismatch or ordering issues.

- **Aggregate**  
  - Type: Aggregate node  
  - Role: Performs data aggregation on combined results to prepare for batch processing.  
  - Config: Aggregation type not specified but likely collects or summarizes data.  
  - Inputs: Output from Merge  
  - Outputs: Passes aggregated data to Loop Over Items to restart batch processing cycle.  
  - Edge Cases: Empty input or aggregation expression errors.

- **Get row(s) in sheet**  
  - Type: Google Sheets node  
  - Role: Retrieves updated rows for further AI processing (summary generation).  
  - Config: Queries rows based on incoming batch items.  
  - Inputs: Output from Loop Over Items  
  - Outputs: Passes rows to Summary Customer Insight AI node  
  - Edge Cases: Read permission errors, empty results.

#### 2.5 Iterative Row Processing

**Overview:**  
Manages batch processing of rows for scalable operation and to comply with API limits.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  

**Node Details:**  
- **Loop Over Items** (also referenced earlier)  
  - Type: SplitInBatches  
  - Role: Divides processed data into smaller chunks to avoid API or processing limits.  
  - Config: Batch size unspecified; manages the flow into conditional branches and Get row(s) in sheet.  
  - Inputs: Output from Aggregate or Compute Numerology  
  - Outputs: Routes data through conditional If nodes for each personality analysis path.  
  - Edge Cases: Improper batch size can cause delays or missed data.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                                  | Input Node(s)                   | Output Node(s)                      | Sticky Note                      |
|-------------------------|----------------------------------|-------------------------------------------------|--------------------------------|-----------------------------------|---------------------------------|
| Google Sheets Trigger    | Google Sheets Trigger             | Initiates workflow on sheet changes             | None                           | Filter                            |                                 |
| Filter                  | Filter                           | Filters rows to process                           | Google Sheets Trigger          | Compute Bazi                     |                                 |
| Compute Bazi            | Code                            | Calculates BaZi values                            | Filter                        | Compute Numerology               |                                 |
| Compute Numerology      | Code                            | Computes numerology numbers                       | Compute Bazi                  | Loop Over Items, Update row in sheet4 |                                 |
| Loop Over Items         | SplitInBatches                  | Processes rows in batches                         | Compute Numerology, Aggregate  | Get row(s) in sheet, If2, If1, If |                                 |
| Get row(s) in sheet     | Google Sheets                   | Retrieves rows for summary AI processing         | Loop Over Items               | Summary Customer Insight         |                                 |
| Summary Customer Insight | Google Gemini AI                | Creates combined customer insight summary        | Get row(s) in sheet           | Writing Script for Salers        |                                 |
| Writing Script for Salers| Google Gemini AI                | Generates sales script based on insight          | Summary Customer Insight       | Update row in sheet              |                                 |
| Update row in sheet     | Google Sheets                   | Updates sheet with sales script                   | Writing Script for Salers      | None                           |                                 |
| If2                     | If                             | Routes Numerology analysis                        | Loop Over Items               | Numerology analysis, Merge       |                                 |
| Numerology analysis     | Google Gemini AI                | AI analysis of numerology                         | If2                          | Update row in sheet1             |                                 |
| Update row in sheet1    | Google Sheets                   | Updates sheet1 with numerology analysis          | Numerology analysis           | Merge                          |                                 |
| If1                     | If                             | Routes BaZi analysis                              | Loop Over Items               | Bazi Analysis, Merge             |                                 |
| Bazi Analysis           | Google Gemini AI                | AI analysis of BaZi                               | If1                          | Update row in sheet2             |                                 |
| Update row in sheet2    | Google Sheets                   | Updates sheet2 with BaZi analysis                 | Bazi Analysis                | Merge                          |                                 |
| If                      | If                             | Routes DISC analysis                              | Loop Over Items               | DISC Analysis, Merge             |                                 |
| DISC Analysis           | Google Gemini AI                | AI analysis of DISC                               | If                           | Update row in sheet3             |                                 |
| Update row in sheet3    | Google Sheets                   | Updates sheet3 with DISC analysis                 | DISC Analysis                | Merge                          |                                 |
| Merge                   | Merge                          | Combines outputs from analysis update nodes      | Update row in sheet1, sheet2, sheet3 | Aggregate                     |                                 |
| Aggregate               | Aggregate                      | Aggregates merged data                            | Merge                        | Loop Over Items                 |                                 |
| Update row in sheet4    | Google Sheets                   | Updates sheet4 with intermediate numerology data | Compute Numerology           | None                           |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger node**  
   - Type: Google Sheets Trigger  
   - Configure to listen for new or updated rows on the target spreadsheet and sheet.

2. **Add Filter node**  
   - Connect from Google Sheets Trigger.  
   - Set conditions to filter rows with required data fields for analysis (e.g., birthdate present).

3. **Add Code node "Compute Bazi"**  
   - Connect from Filter node.  
   - Insert JavaScript code to calculate BaZi values from input data.  
   - Ensure input row fields (birthdate, time, etc.) are mapped correctly.

4. **Add Code node "Compute Numerology"**  
   - Connect from Compute Bazi.  
   - JavaScript code to compute numerology numbers from customer data.

5. **Add Google Sheets node "Update row in sheet4"**  
   - Connect also from Compute Numerology.  
   - Configure to update intermediate numerology data in a dedicated sheet/tab.

6. **Add SplitInBatches node "Loop Over Items"**  
   - Connect from Compute Numerology.  
   - Set batch size (default or custom).

7. **Add Google Sheets node "Get row(s) in sheet"**  
   - Connect from Loop Over Items.  
   - Configure to fetch rows to provide full context for AI nodes.

8. **Add Google Gemini node "Summary Customer Insight"**  
   - Connect from Get row(s) in sheet.  
   - Configure prompt to synthesize all personality data into a customer insight summary.

9. **Add Google Gemini node "Writing Script for Salers"**  
   - Connect from Summary Customer Insight.  
   - Configure prompt to generate a sales script based on the insight.

10. **Add Google Sheets node "Update row in sheet"**  
    - Connect from Writing Script for Salers.  
    - Configure to update the sheet with the sales script.

11. **Add three If nodes ("If", "If1", "If2")**  
    - Connect from Loop Over Items.  
    - Configure each condition to check for DISC, BaZi, and Numerology data presence.

12. **Add Google Gemini nodes for each analysis type:**  
    - "DISC Analysis" connected from "If" node.  
    - "Bazi Analysis" connected from "If1" node.  
    - "Numerology analysis" connected from "If2" node.  
    - Configure each with appropriate prompts referencing computed data.

13. **Add Google Sheets "Update row" nodes for analysis results:**  
    - "Update row in sheet3" for DISC Analysis.  
    - "Update row in sheet2" for Bazi Analysis.  
    - "Update row in sheet1" for Numerology analysis.

14. **Add Merge node**  
    - Connect outputs from all three "Update row" nodes.  
    - Set to combine all outputs into a single stream.

15. **Add Aggregate node**  
    - Connect from Merge node.  
    - Configure to aggregate or summarize combined data.

16. **Connect Aggregate output back to Loop Over Items**  
    - Enables batch processing continuation or looping.

17. **Credential Setup:**  
    - Configure Google Sheets OAuth2 credentials to access and modify sheets.  
    - Configure Google Gemini node credentials for AI API access.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| The workflow integrates Google Gemini AI nodes for advanced personality analysis and script generation.   | n8n Google Gemini nodes documentation                                 |
| Batch processing via SplitInBatches improves scalability and respects API rate limits.                    | n8n SplitInBatches node details                                       |
| Ensure that Google Sheets API credentials have proper read/write scopes for all targeted sheets and tabs. | Google Cloud Console API permissions                                  |
| The BaZi and Numerology computations are implemented in custom JavaScript code nodes; verify input formats.| BaZi and Numerology domain knowledge required                         |

---

*Disclaimer: The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.*